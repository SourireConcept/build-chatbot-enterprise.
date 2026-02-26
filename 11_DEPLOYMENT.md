# 11 — DÉPLOIEMENT : Vercel + Supabase + Configuration

## Stack de déploiement

| Composant | Service | Configuration |
|---|---|---|
| API Backend | Vercel Serverless | `api/index.js` → `/api/*` |
| Frontend statique | Vercel CDN | `public/` → `/` |
| Base de données | Supabase (PostgreSQL 15) | pgvector activé |
| Storage médias | Supabase Storage | bucket `media` |
| Crons | Vercel Cron Jobs | `vercel.json` |
| Monitoring | Sentry | DSN configuré |

---

## Fichier `vercel.json`

```json
{
  "version": 2,
  "buildCommand": "echo 'No build needed'",
  "outputDirectory": "public",
  "framework": null,

  "functions": {
    "api/index.js": {
      "maxDuration": 60,
      "memory": 1024
    }
  },

  "rewrites": [
    {
      "source": "/api/(.*)",
      "destination": "/api/index.js"
    },
    {
      "source": "/admin",
      "destination": "/admin.html"
    },
    {
      "source": "/(.*)",
      "destination": "/$1"
    }
  ],

  "headers": [
    {
      "source": "/api/(.*)",
      "headers": [
        { "key": "Access-Control-Allow-Origin", "value": "*" },
        { "key": "Access-Control-Allow-Methods", "value": "GET,POST,PUT,DELETE,OPTIONS" },
        { "key": "Access-Control-Allow-Headers", "value": "Content-Type,Authorization,X-Session-Id" }
      ]
    },
    {
      "source": "/(.*)",
      "headers": [
        { "key": "X-Frame-Options", "value": "DENY" },
        { "key": "X-Content-Type-Options", "value": "nosniff" },
        { "key": "Referrer-Policy", "value": "strict-origin-when-cross-origin" }
      ]
    }
  ],

  "crons": [
    {
      "path": "/api/cron/autoroute-daily",
      "schedule": "0 9 * * *"
    },
    {
      "path": "/api/cron/security-scan",
      "schedule": "0 0 * * *"
    },
    {
      "path": "/api/cron/session-cleanup",
      "schedule": "0 */6 * * *"
    },
    {
      "path": "/api/cron/agent-maintenance",
      "schedule": "0 */2 * * *"
    }
  ]
}
```

---

## Fichier `package.json`

```json
{
  "name": "[app-name]-api",
  "version": "1.0.0",
  "description": "API [APP_NAME] — Chatbot IA Enterprise",
  "type": "module",
  "main": "api/index.js",
  "engines": {
    "node": ">=22.0.0"
  },
  "scripts": {
    "start": "node server.js",
    "dev": "node --watch server.js",
    "test": "node --experimental-vm-modules node_modules/.bin/jest",
    "test:coverage": "node --experimental-vm-modules node_modules/.bin/jest --coverage",
    "test:e2e": "playwright test",
    "lint": "eslint . --ext .js",
    "ingest-kb": "node scripts/ingest-knowledge-v2.mjs",
    "reindex-kb": "node scripts/reindex-knowledge-v2.mjs",
    "generate-secrets": "node scripts/generate-secrets.js",
    "validate-env": "node validate-env.js"
  },
  "dependencies": {
    "@google/genai": "^0.7.0",
    "@huggingface/inference": "^3.3.0",
    "@sentry/node": "^8.0.0",
    "@sentry/profiling-node": "^8.0.0",
    "@supabase/supabase-js": "^2.46.0",
    "axios": "^1.7.0",
    "bcryptjs": "^2.4.3",
    "cookie-parser": "^1.4.7",
    "cors": "^2.8.5",
    "cron-parser": "^4.9.0",
    "dotenv": "^16.4.5",
    "express": "^4.21.0",
    "ffmpeg-static": "^5.2.0",
    "fluent-ffmpeg": "^2.1.3",
    "googleapis": "^144.0.0",
    "imapflow": "^1.0.162",
    "jsonwebtoken": "^9.0.2",
    "mailparser": "^3.7.1",
    "nodemailer": "^6.9.15",
    "pdf-parse": "^1.1.1",
    "pdfkit": "^0.15.0",
    "stripe": "^16.12.0"
  },
  "devDependencies": {
    "@playwright/test": "^1.48.0",
    "eslint": "^9.0.0",
    "jest": "^29.7.0",
    "supertest": "^7.0.0"
  }
}
```

---

## Fichier `server.js` (lancement local)

```javascript
// server.js
// Wrapper de lancement local (dev uniquement)

import './instrument.js'; // Sentry doit être importé en premier
import dotenv from 'dotenv';
dotenv.config();

import { validateEnvironment } from './validate-env.js';
validateEnvironment();

import app from './api/index.js';

const PORT = process.env.PORT || 3000;

app.listen(PORT, () => {
  console.log(`\n🚀 [APP_NAME] API démarrée sur http://localhost:${PORT}`);
  console.log(`📊 Admin panel: http://localhost:${PORT}/admin.html`);
  console.log(`🤖 Chatbot widget: http://localhost:${PORT}/chatbot-widget.html`);
  console.log(`🔍 Health check: http://localhost:${PORT}/api/health\n`);
});

export default app;
```

---

## Fichier `instrument.js` (Sentry)

```javascript
// instrument.js
// Init Sentry (profiling-node) — doit être le premier import

import * as Sentry from '@sentry/node';
import { nodeProfilingIntegration } from '@sentry/profiling-node';

if (process.env.SENTRY_DSN) {
  Sentry.init({
    dsn: process.env.SENTRY_DSN,
    integrations: [nodeProfilingIntegration()],
    tracesSampleRate: process.env.NODE_ENV === 'production' ? 0.1 : 1.0,
    profilesSampleRate: 1.0,
    environment: process.env.NODE_ENV || 'development',
    release: process.env.npm_package_version,
  });

  console.log('[sentry] Monitoring initialisé');
} else {
  console.warn('[sentry] SENTRY_DSN non configuré — monitoring désactivé');
}
```

---

## Fichier `api/index.js` (squelette complet)

```javascript
// api/index.js
// Entrypoint Express — TOUTES les routes

import './instrument.js';
import * as Sentry from '@sentry/node';
import express from 'express';
import cors from 'cors';
import cookieParser from 'cookie-parser';

import { validateEnvironment } from '../validate-env.js';
validateEnvironment();

// ── MODULES ──
import { handleChatRequestV3 } from '../lib/ai-v3.js';
import { handleAdminRequest } from '../lib/admin-ai.js';
import { requireAuth, requireCronSecret, generateToken, verifyAdminPassword } from '../lib/jwt.js';
import { rateLimiterMiddleware } from '../lib/rate-limiter.js';
import { supabase } from '../lib/supabase.js';
import { runSecurityScan } from '../lib/security-agent.js';
import { runAgentObjective } from '../lib/agent/orchestrator.js';
import { generateAutoroutePost, generateAutorouteMedia, saveAndQueuePost } from '../lib/autoroute-proxy.js';
import { runWorkflowsForEvent } from '../lib/workflows.js';
import { verifyWhatsAppWebhook } from '../lib/whatsapp.js';
import { sendEmail } from '../lib/email.js';

const app = express();

// ── MIDDLEWARE GLOBAL ──
const allowedOrigins = (process.env.ADMIN_ALLOWED_ORIGINS || '').split(',').filter(Boolean);

app.use(cors({
  origin: (origin, callback) => {
    if (!origin || allowedOrigins.includes(origin) || allowedOrigins.includes('*')) {
      callback(null, true);
    } else {
      callback(null, false); // Silencieux, pas d'erreur
    }
  },
  credentials: true,
}));

app.use(express.json({ limit: '10mb' }));
app.use(express.urlencoded({ extended: true, limit: '10mb' }));
app.use(cookieParser());

// ──────────────────────────────────────────────────
// ROUTES PUBLIQUES
// ──────────────────────────────────────────────────

// Health check
app.get('/api/health', (req, res) => {
  res.json({
    status: 'ok',
    version: process.env.npm_package_version || '1.0.0',
    env: process.env.NODE_ENV,
    timestamp: new Date().toISOString(),
  });
});

// Chatbot principal
app.post('/api/chat',
  rateLimiterMiddleware('chat'),
  handleChatRequestV3
);

// Webhooks entrants
app.get('/api/webhook/whatsapp', verifyWhatsAppWebhook);
app.post('/api/webhook/whatsapp', async (req, res) => {
  // Extraire le message WhatsApp et le traiter comme un chat
  const { entry } = req.body;
  const msg = entry?.[0]?.changes?.[0]?.value?.messages?.[0];
  if (!msg) return res.status(200).send('OK');

  const from = msg.from;
  const text = msg.text?.body || '';
  const sessionId = `wa_${from}`;

  // Traiter comme requête chat
  req.body = { message: text, session_id: sessionId, language: 'fr', context: { channel: 'whatsapp' } };
  await handleChatRequestV3(req, {
    json: (data) => {
      if (data.reply) {
        const { sendWhatsApp } = require('./lib/whatsapp.js');
        sendWhatsApp({ to: from, message: data.reply }).catch(console.error);
      }
      return { json: () => {}, status: () => ({ json: () => {} }) };
    },
    status: () => ({ json: () => {} }),
  });

  return res.status(200).send('OK');
});

// Webhooks Messenger (vérification + réception)
app.get('/api/webhook/messenger', (req, res) => {
  const mode = req.query['hub.mode'];
  const token = req.query['hub.verify_token'];
  const challenge = req.query['hub.challenge'];
  if (mode === 'subscribe' && token === process.env.MESSENGER_VERIFY_TOKEN) {
    return res.status(200).send(challenge);
  }
  return res.status(403).send('Forbidden');
});

app.post('/api/webhook/messenger', async (req, res) => {
  // Parser et traiter les messages Messenger
  res.status(200).send('EVENT_RECEIVED');
});

// Catalogue public (avec cache)
app.get('/api/catalog', async (req, res) => {
  const { query = '', limit = 12 } = req.query;
  const { searchWixProducts } = await import('./lib/wix-catalog.js');
  const products = await searchWixProducts({ query, limit: parseInt(limit) });
  return res.json({ products });
});

// ──────────────────────────────────────────────────
// ROUTES ADMIN (JWT requis)
// ──────────────────────────────────────────────────

// Login
app.post('/api/admin/login', rateLimiterMiddleware('login'), async (req, res) => {
  const { password } = req.body;
  try {
    const valid = await verifyAdminPassword(password);
    if (!valid) return res.status(401).json({ error: 'Mot de passe incorrect' });
    const token = generateToken({ role: 'admin' });
    res.cookie('admin_token', token, { httpOnly: true, secure: process.env.NODE_ENV === 'production', sameSite: 'strict', maxAge: 86400000 });
    return res.json({ token, expires_in: '24h' });
  } catch (error) {
    return res.status(500).json({ error: 'Erreur serveur' });
  }
});

// Super Chat
app.post('/api/admin/chat', requireAuth, async (req, res) => {
  const result = await handleAdminRequest(req.body.message, req.admin);
  return res.json(result);
});

// Leads
app.get('/api/admin/leads', requireAuth, async (req, res) => {
  const { limit = 100, offset = 0 } = req.query;
  const { data: leads } = await supabase.from('app_leads').select('*').order('created_at', { ascending: false }).range(parseInt(offset), parseInt(offset) + parseInt(limit) - 1);
  return res.json({ leads: leads || [] });
});

// Prompt Store
app.get('/api/admin/prompt', requireAuth, async (req, res) => {
  const name = req.query.name || 'chatbot_main';
  const { data } = await supabase.from('app_prompt_store').select('*').eq('name', name).single();
  return res.json(data || { content: '' });
});
app.post('/api/admin/prompt', requireAuth, async (req, res) => {
  const { name = 'chatbot_main', content } = req.body;
  await supabase.from('app_prompt_store').upsert({ name, content, updated_at: new Date().toISOString() }, { onConflict: 'name' });
  return res.json({ success: true });
});

// Company Facts
app.get('/api/admin/facts', requireAuth, async (req, res) => {
  const { data } = await supabase.from('app_company_facts').select('*').eq('active', true).order('category');
  return res.json({ facts: data || [] });
});
app.post('/api/admin/facts', requireAuth, async (req, res) => {
  const { key, value, category } = req.body;
  await supabase.from('app_company_facts').upsert({ key, value, category, updated_at: new Date().toISOString() }, { onConflict: 'key' });
  return res.json({ success: true });
});

// Runtime Flags
app.get('/api/admin/runtime-flags', requireAuth, async (req, res) => {
  const { data } = await supabase.from('app_runtime_state').select('key, value');
  return res.json(Object.fromEntries((data || []).map(r => [r.key, r.value])));
});
app.post('/api/admin/runtime-flags', requireAuth, async (req, res) => {
  const { key, value } = req.body;
  await supabase.from('app_runtime_state').upsert({ key, value, updated_at: new Date().toISOString() }, { onConflict: 'key' });
  return res.json({ success: true });
});

// Agent Registry
app.get('/api/admin/agent/registry', requireAuth, async (req, res) => {
  const { data: agents } = await supabase.from('app_agent_registry').select('*').order('name');
  return res.json({ agents: agents || [] });
});
app.post('/api/admin/agent/registry', requireAuth, async (req, res) => {
  const { agent_name, enabled } = req.body;
  await supabase.from('app_agent_registry').update({ enabled, updated_at: new Date().toISOString() }).eq('name', agent_name);
  return res.json({ success: true });
});

// Agent Objectives
app.get('/api/admin/agent/objectives', requireAuth, async (req, res) => {
  const { data } = await supabase.from('app_agent_objectives').select('*').order('created_at', { ascending: false });
  return res.json({ objectives: data || [] });
});
app.post('/api/admin/agent/objectives', requireAuth, async (req, res) => {
  const { data: objective } = await supabase.from('app_agent_objectives').insert(req.body).select().single();
  return res.json({ objective });
});
app.post('/api/admin/agent/objectives/:id/run', requireAuth, async (req, res) => {
  if (process.env.AGENT_ENABLED !== 'true') return res.status(503).json({ error: 'Agent désactivé' });
  runAgentObjective(req.params.id).catch(console.error);
  return res.json({ message: 'Run démarré', objective_id: req.params.id });
});

// Agent Approvals
app.get('/api/admin/agent/approvals', requireAuth, async (req, res) => {
  const { data } = await supabase.from('app_agent_approvals').select('*').eq('status', 'pending').order('created_at', { ascending: false });
  return res.json({ approvals: data || [] });
});
app.post('/api/admin/agent/approvals/:id/approve', requireAuth, async (req, res) => {
  await supabase.from('app_agent_approvals').update({ status: 'approved', approved_by: 'admin', approved_at: new Date().toISOString() }).eq('id', req.params.id);
  return res.json({ approved: true });
});
app.post('/api/admin/agent/approvals/:id/reject', requireAuth, async (req, res) => {
  await supabase.from('app_agent_approvals').update({ status: 'rejected', approved_by: 'admin', approved_at: new Date().toISOString() }).eq('id', req.params.id);
  return res.json({ rejected: true });
});

// Autoroute
app.post('/api/admin/autoroute/generate', requireAuth, async (req, res) => {
  const { topic } = req.body;
  const content = await generateAutoroutePost({ topic });
  const media = await generateAutorouteMedia({ prompt: content.short, type: 'image' }).catch(() => null);
  const queueItem = await saveAndQueuePost({ content, media });
  return res.json({ content, media, queue_item: queueItem });
});

// Social Queue
app.get('/api/admin/social-queue', requireAuth, async (req, res) => {
  const { data } = await supabase.from('app_social_publication_queue').select('*').order('created_at', { ascending: false }).limit(50);
  return res.json({ queue: data || [] });
});
app.post('/api/admin/social-queue/:id/approve', requireAuth, async (req, res) => {
  const { data: item } = await supabase.from('app_social_publication_queue').select('*').eq('id', req.params.id).single();
  if (!item) return res.status(404).json({ error: 'Non trouvé' });
  await supabase.from('app_social_publication_queue').update({ status: 'approved', approved_by: 'admin', approved_at: new Date().toISOString() }).eq('id', req.params.id);
  const result = await runWorkflowsForEvent('autoroute_post_approved', item.metadata);
  await supabase.from('app_social_publication_queue').update({ status: 'published', published_at: new Date().toISOString() }).eq('id', req.params.id);
  return res.json({ approved: true, result });
});

// Logs admin
app.get('/api/admin/logs', requireAuth, async (req, res) => {
  const { level, limit = 100 } = req.query;
  let query = supabase.from('app_admin_logs').select('*').order('created_at', { ascending: false }).limit(parseInt(limit));
  if (level) query = query.eq('level', level);
  const { data } = await query;
  return res.json({ logs: data || [] });
});

// Sécurité
app.post('/api/admin/security/scan', requireAuth, async (req, res) => {
  const result = await runSecurityScan();
  return res.json(result);
});

// ──────────────────────────────────────────────────
// ROUTES CRON (CRON_SECRET requis)
// ──────────────────────────────────────────────────

app.post('/api/cron/autoroute-daily', requireCronSecret, async (req, res) => {
  try {
    if (process.env.AUTOROUTE_PUBLICATION_MODE_DEFAULT === 'disabled') return res.json({ skipped: true });
    const content = await generateAutoroutePost({});
    const media = await generateAutorouteMedia({ prompt: content.short }).catch(() => null);
    const item = await saveAndQueuePost({ content, media });
    if (process.env.SOCIAL_APPROVAL_REQUIRED !== 'true') {
      await runWorkflowsForEvent('autoroute_post_approved', { content, media });
      await supabase.from('app_social_publication_queue').update({ status: 'published', published_at: new Date().toISOString() }).eq('id', item.id);
    }
    return res.json({ success: true, item_id: item.id });
  } catch (error) {
    return res.status(500).json({ error: error.message });
  }
});

app.post('/api/cron/security-scan', requireCronSecret, async (req, res) => {
  const result = await runSecurityScan();
  return res.json(result);
});

app.post('/api/cron/session-cleanup', requireCronSecret, async (req, res) => {
  const ttlDays = parseInt(process.env.CHAT_SESSION_TTL_DAYS || '90');
  const { data } = await supabase.rpc('prune_expired_sessions', { p_ttl_days: ttlDays });
  return res.json({ cleaned: data || 0 });
});

app.post('/api/cron/agent-maintenance', requireCronSecret, async (req, res) => {
  await supabase.from('app_agent_approvals').update({ status: 'expired' }).eq('status', 'pending').lt('expires_at', new Date().toISOString());
  await supabase.rpc('prune_agent_events', { p_retention_days: parseInt(process.env.AGENT_EVENTS_RETENTION_DAYS || '30') });
  return res.json({ maintained: true });
});

// ──────────────────────────────────────────────────
// SENTRY ERROR HANDLER (doit être après toutes les routes)
// ──────────────────────────────────────────────────
Sentry.setupExpressErrorHandler(app);

// Handler erreurs global
app.use((err, req, res, next) => {
  console.error('[express] Erreur non gérée:', err);
  return res.status(500).json({ error: 'Erreur serveur interne', reply: 'Une erreur est survenue. Veuillez réessayer.' });
});

export default app;
```

---

## Procédure de déploiement initial

### 1. Supabase Setup

```bash
# 1. Créer un projet Supabase
# 2. Activer pgvector dans l'éditeur SQL:
CREATE EXTENSION IF NOT EXISTS vector;

# 3. Appliquer toutes les migrations (dans l'ordre):
# Copier-coller chaque migration SQL depuis docs/chatbot-blueprint/02_DATABASE_SCHEMA.md
# OU utiliser Supabase CLI:
supabase db push

# 4. Configurer le bucket Storage:
# Dashboard Supabase → Storage → New bucket → "media" (public)
```

### 2. Variables d'environnement Vercel

```bash
# Installer Vercel CLI
npm i -g vercel

# Se connecter
vercel login

# Configurer les variables (pour chaque variable de .env.example)
vercel env add SUPABASE_URL production
vercel env add SUPABASE_ANON_KEY production
# ... etc pour toutes les variables

# OU importer depuis .env:
vercel env import .env production
```

### 3. Premier déploiement

```bash
# Déploiement production
vercel --prod

# Vérifier le health check
curl https://votre-app.vercel.app/api/health
# Attendu: {"status":"ok","version":"1.0.0","env":"production"}
```

### 4. Ingestion base de connaissances

```bash
# Créer les fichiers de connaissances dans knowledge/
mkdir -p knowledge
# Créer des fichiers .md avec vos contenus (FAQ, produits, politiques, etc.)

# Ingérer dans Supabase (embeddings + pgvector)
node scripts/ingest-knowledge-v2.mjs

# Vérifier l'ingestion
curl -X POST https://votre-app.vercel.app/api/admin/chat \
  -H "Authorization: Bearer YOUR_ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"message": "Combien de chunks de connaissances sont indexés ?"}'
```

### 5. Configuration webhooks

```bash
# WhatsApp Business API
# → Meta Business Suite → WhatsApp → Configuration
# Webhook URL: https://votre-app.vercel.app/api/webhook/whatsapp
# Verify Token: valeur de WHATSAPP_VERIFY_TOKEN

# Facebook Messenger
# → Meta Business Suite → Messenger → Paramètres
# Webhook URL: https://votre-app.vercel.app/api/webhook/messenger
# Verify Token: valeur de MESSENGER_VERIFY_TOKEN

# Wix Webhooks
# → Wix Dashboard → API Keys → Webhooks
# URL: https://votre-app.vercel.app/api/webhook/wix
```

### 6. Configuration initiale du chatbot

```bash
# 1. Accéder à l'admin panel
open https://votre-app.vercel.app/admin.html

# 2. Se connecter avec le mot de passe admin

# 3. Configurer le prompt principal:
# Admin Panel → Prompt Store → chatbot_main → Saisir votre prompt

# 4. Configurer les facts entreprise:
# Admin Panel → Facts Entreprise → Ajouter:
#   - brand_name: Votre Marque
#   - product_domain: Vos produits
#   - contact_email: contact@votre-domaine.com
#   - currency: EUR
#   - language_default: fr

# 5. Tester le chatbot:
# Admin Panel → Debug Chat → Tester plusieurs messages
```

---

## Widget embed (`public/embed.js`)

```javascript
// public/embed.js
// Script d'intégration du chatbot sur n'importe quel site

(function() {
  'use strict';

  const CHATBOT_URL = 'https://votre-app.vercel.app';
  const WIDGET_WIDTH = 400;
  const WIDGET_HEIGHT = 600;
  const TRIGGER_DELAY_MS = 60000; // 60 secondes

  // Créer le container
  const container = document.createElement('div');
  container.id = 'ai-chatbot-container';
  container.style.cssText = `
    position: fixed;
    bottom: 20px;
    right: 20px;
    z-index: 9999;
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
  `;

  // Bouton déclencheur
  const trigger = document.createElement('button');
  trigger.id = 'chatbot-trigger';
  trigger.innerHTML = '💬';
  trigger.style.cssText = `
    width: 56px;
    height: 56px;
    border-radius: 50%;
    background: [BRAND_COLOR];
    border: none;
    cursor: pointer;
    font-size: 24px;
    box-shadow: 0 4px 20px rgba(0,0,0,0.3);
    transition: transform 0.2s;
    display: flex;
    align-items: center;
    justify-content: center;
    margin-left: auto;
  `;

  // Badge notifications
  const badge = document.createElement('span');
  badge.id = 'chatbot-badge';
  badge.style.cssText = `
    position: absolute;
    top: -4px;
    right: -4px;
    width: 12px;
    height: 12px;
    background: #ef4444;
    border-radius: 50%;
    display: none;
    animation: pulse 2s infinite;
  `;
  trigger.appendChild(badge);

  // iframe
  const iframe = document.createElement('iframe');
  iframe.src = `${CHATBOT_URL}/chatbot-widget.html`;
  iframe.style.cssText = `
    width: ${WIDGET_WIDTH}px;
    height: ${WIDGET_HEIGHT}px;
    border: none;
    border-radius: 12px;
    box-shadow: 0 8px 40px rgba(0,0,0,0.3);
    display: none;
    margin-bottom: 12px;
    background: white;
  `;
  iframe.allow = 'clipboard-write';

  // Toggle
  let isOpen = false;
  trigger.addEventListener('click', () => {
    isOpen = !isOpen;
    iframe.style.display = isOpen ? 'block' : 'none';
    trigger.innerHTML = isOpen ? '✕' : '💬';
    badge.style.display = 'none';
    if (!iframe.appendChild) trigger.appendChild(badge);
  });

  // Trigger proactif après 60s
  setTimeout(() => {
    if (!isOpen) {
      badge.style.display = 'block';
      // Envoyer token proactif au widget via postMessage
      iframe.contentWindow?.postMessage({ type: 'PROACTIVE_TRIGGER' }, CHATBOT_URL);
    }
  }, TRIGGER_DELAY_MS);

  // Écouter postMessage du widget
  window.addEventListener('message', (event) => {
    if (event.origin !== CHATBOT_URL) return;
    if (event.data.type === 'CLOSE_CHATBOT') {
      isOpen = false;
      iframe.style.display = 'none';
      trigger.innerHTML = '💬';
    }
    if (event.data.type === 'ADD_TO_CART' && event.data.productId) {
      // Intégration panier personnalisée
      window.dispatchEvent(new CustomEvent('chatbot-add-to-cart', { detail: event.data }));
    }
  });

  container.appendChild(iframe);
  container.appendChild(trigger);
  document.body.appendChild(container);

  // Style animation pulse
  const style = document.createElement('style');
  style.textContent = `
    @keyframes pulse {
      0%, 100% { transform: scale(1); opacity: 1; }
      50% { transform: scale(1.2); opacity: 0.8; }
    }
  `;
  document.head.appendChild(style);
})();
```

---

## Script d'ingestion de connaissances

```javascript
// scripts/ingest-knowledge-v2.mjs

import { createClient } from '@supabase/supabase-js';
import { InferenceClient } from '@huggingface/inference';
import { readdir, readFile } from 'fs/promises';
import { join, extname } from 'path';
import dotenv from 'dotenv';
dotenv.config();

const supabase = createClient(process.env.SUPABASE_URL, process.env.SUPABASE_SERVICE_ROLE_KEY);
const hf = new InferenceClient(process.env.HF_API_TOKEN);

const EMBED_MODEL = process.env.HF_EMBED_MODEL_V2 || 'BAAI/bge-m3';
const EMBED_DIM = parseInt(process.env.HF_EMBED_DIM_V2 || '1024');
const CHUNK_SIZE = 500; // Caractères par chunk
const KNOWLEDGE_DIR = './knowledge';

async function chunkText(text, size = CHUNK_SIZE) {
  const chunks = [];
  const sentences = text.split(/(?<=[.!?])\s+/);
  let current = '';

  for (const sentence of sentences) {
    if ((current + sentence).length > size && current.length > 0) {
      chunks.push(current.trim());
      current = sentence;
    } else {
      current += ' ' + sentence;
    }
  }
  if (current.trim()) chunks.push(current.trim());
  return chunks;
}

async function embedText(text) {
  const response = await hf.featureExtraction({
    model: EMBED_MODEL,
    inputs: text,
    parameters: { normalize: true },
    provider: process.env.HF_INFERENCE_PROVIDER || 'nebius',
  });
  const embedding = Array.isArray(response[0]) ? response[0] : response;
  return embedding.slice(0, EMBED_DIM);
}

async function ingestFile(filePath, category = 'general') {
  console.log(`📄 Ingestion: ${filePath}`);
  const content = await readFile(filePath, 'utf-8');
  const chunks = await chunkText(content);

  for (let i = 0; i < chunks.length; i++) {
    const chunk = chunks[i];
    console.log(`  Chunk ${i + 1}/${chunks.length} (${chunk.length} chars)`);

    const embedding = await embedText(chunk);

    await supabase.from('app_knowledge_chunks_v2').insert({
      content: chunk,
      embedding,
      source: filePath,
      category,
      chunk_index: i,
      total_chunks: chunks.length,
      language: 'fr',
    });

    // Petit délai pour éviter le rate limit
    await new Promise(r => setTimeout(r, 200));
  }

  console.log(`  ✅ ${chunks.length} chunks ingérés`);
}

async function main() {
  console.log('🚀 Ingestion base de connaissances V2\n');

  try {
    const files = await readdir(KNOWLEDGE_DIR);
    const markdownFiles = files.filter(f => extname(f) === '.md' || extname(f) === '.txt');

    if (markdownFiles.length === 0) {
      console.log(`⚠️  Aucun fichier .md/.txt trouvé dans ${KNOWLEDGE_DIR}/`);
      console.log('Créez des fichiers de connaissances (FAQ, produits, politiques...) dans ce répertoire.');
      return;
    }

    for (const file of markdownFiles) {
      const category = file.replace(/\.(md|txt)$/, '').split('-')[0];
      await ingestFile(join(KNOWLEDGE_DIR, file), category);
    }

    // Vérifier le compte total
    const { count } = await supabase
      .from('app_knowledge_chunks_v2')
      .select('*', { count: 'exact', head: true });

    console.log(`\n✅ Ingestion terminée! Total chunks en BDD: ${count}`);

  } catch (error) {
    console.error('❌ Erreur:', error.message);
    process.exit(1);
  }
}

main();
```

---

## Jest configuration (`jest.config.cjs`)

```javascript
// jest.config.cjs
module.exports = {
  testEnvironment: 'node',
  transform: {},
  extensionsToTreatAsEsm: ['.js'],
  moduleNameMapper: {
    '^(\\.{1,2}/.*)\\.js$': '$1',
  },
  testMatch: ['**/tests/**/*.test.js'],
  collectCoverageFrom: ['lib/**/*.js', 'api/**/*.js', '!lib/knowledge.js'],
  coverageThreshold: {
    global: { branches: 60, functions: 60, lines: 70, statements: 70 },
  },
  setupFiles: ['./tests/setup.js'],
};
```
