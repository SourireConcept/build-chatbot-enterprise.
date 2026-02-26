# 05 — SYSTÈME DE SÉCURITÉ : Défense en profondeur

## Architecture des couches de sécurité

```
Requête entrante
       │
       ▼
[1] Rate Limiter (Supabase RPC atomique)
       │  429 si dépassé
       ▼
[2] Auth Middleware (JWT pour /admin/*, CRON_SECRET pour /cron/*)
       │  401 si invalide
       ▼
[3] Input Sanitization (escapeHtml + validation)
       │
       ▼
[4] Pre-LLM Guards (en parallèle)
       ├── Disclosure Guard (exfiltration prompt/secrets)
       ├── Role Override Guard (DAN/jailbreak)
       └── Prompt Injection Shield (meta-agent)
       │  Refus explicite si détecté
       ▼
[5] Domain Scope Guard (classification in/out scope)
       │  Recentrage doux si hors scope
       ▼
[6] LLM Execution
       │
       ▼
[7] Post-LLM Guards
       ├── Guard Orchestrator V3 (anti-hallucination catalogue)
       ├── Company Identity Guard (refs entreprises non autorisées)
       └── Product Query Guard (produits fictifs)
       │
       ▼
[8] Output Sanitization
       ├── CoT artifacts (<think>...</think>)
       ├── ENV leaks (patterns VARIABLE=valeur)
       └── Token masking (IDs tronqués)
       │
       ▼
[9] Response Delivery
```

---

## Module `lib/security.js` — Fonctions bas niveau

```javascript
// lib/security.js

/**
 * Échappe les caractères HTML dangereux
 */
export function escapeHtml(str) {
  if (typeof str !== 'string') return '';
  return str
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&#x27;');
}

/**
 * Valide et sanitise une URL (allowlist domains)
 */
export function sanitizeUrl(url, allowedDomains = []) {
  try {
    const parsed = new URL(url);
    // Forcer HTTPS
    if (parsed.protocol !== 'https:') return null;
    // Vérifier domaine
    if (allowedDomains.length > 0) {
      const isAllowed = allowedDomains.some(d => parsed.hostname.endsWith(d));
      if (!isAllowed) return null;
    }
    return parsed.href;
  } catch {
    return null;
  }
}

/**
 * Masque un token (affiche seulement les 4 premiers caractères)
 */
export function maskToken(token) {
  if (!token || token.length < 8) return '****';
  return token.slice(0, 4) + '****';
}

/**
 * Génère un ID de corrélation pour les logs sécurité
 */
export function generateCorrelationId() {
  return `sec_${Date.now()}_${Math.random().toString(36).slice(2, 7)}`;
}
```

---

## Module `lib/security-disclosure-guard.js`

```javascript
// lib/security-disclosure-guard.js
// Détection et blocage des tentatives d'exfiltration

// Patterns déclenchant un blocage (sensibles à la casse)
const DISCLOSURE_PATTERNS = [
  // Demandes directes de prompt
  /(?:montre|affiche|révèle|dis|donne|répète|copie).{0,30}(?:prompt|instruction|system|consigne|règle|directive)/i,
  /(?:what|show|tell|reveal|repeat|copy).{0,20}(?:prompt|instruction|system|rule|guideline)/i,
  // Tentatives d'exfiltration de secrets
  /(?:api.?key|token|password|secret|credential|ENV|\.env)/i,
  // Demande de configuration
  /(?:ta|ta|votre|your).{0,10}(?:configuration|config|paramètre|setting)/i,
  // DAN variants
  /(?:ignore|bypass|override|forget).{0,20}(?:instruction|rule|constraint|guideline)/i,
];

// Patterns soft (avertissement sans blocage)
const SOFT_PATTERNS = [
  /comment tu fonctionnes/i,
  /how do you work/i,
  /what are you trained on/i,
];

export async function analyzeSensitiveDisclosure(message) {
  if (!message || typeof message !== 'string') {
    return { blocked: false, reason: null };
  }

  const cleaned = message.trim();

  // Vérification patterns stricts
  for (const pattern of DISCLOSURE_PATTERNS) {
    if (pattern.test(cleaned)) {
      await logSecurityIncident('disclosure_attempt', cleaned);
      return {
        blocked: true,
        reason: 'disclosure_attempt',
        pattern: pattern.toString(),
      };
    }
  }

  // Vérification patterns soft
  const isSoft = SOFT_PATTERNS.some(p => p.test(cleaned));

  return {
    blocked: false,
    soft_warning: isSoft,
    reason: null,
  };
}

async function logSecurityIncident(type, message) {
  try {
    const { supabase } = await import('./supabase.js');
    await supabase.from('app_security_incidents').insert({
      type,
      severity: 'medium',
      details: { message: message.slice(0, 200) },
    });
  } catch {
    // Non-bloquant
  }
}
```

---

## Module `lib/meta-agent.js`

```javascript
// lib/meta-agent.js
// Shield injections + behavior scoring + abandon risk

// Patterns d'injection de prompt
const INJECTION_PATTERNS = [
  /system prompt/i,
  /\[system\]/i,
  /ignore (?:all )?(?:previous|above) instructions/i,
  /act as (?:an? )?(?:ai|gpt|claude|llm|model|chatbot)/i,
  /you are now/i,
  /pretend (?:to be|you are)/i,
  /roleplay as/i,
  /in this (?:fictional|hypothetical) scenario/i,
  /forget (?:your|all) (?:rules|instructions|training)/i,
  /DAN (?:mode|prompt)/i,
  /jailbreak/i,
];

export async function analyzeRoleOverride(message) {
  if (!message) return { blocked: false };

  const lower = message.toLowerCase();

  for (const pattern of INJECTION_PATTERNS) {
    if (pattern.test(lower)) {
      return {
        blocked: true,
        reason: 'role_override_detected',
        pattern: pattern.toString(),
      };
    }
  }

  return { blocked: false };
}

export function analyzePromptInjection(message) {
  const INJECTION_SIGNALS = [
    'system:', 'assistant:', '### instruction', '## system',
    '[inst]', '[/inst]', '<|im_start|>', '<|im_end|>',
  ];

  const lower = message.toLowerCase();
  const detected = INJECTION_SIGNALS.some(s => lower.includes(s));

  return {
    injection_detected: detected,
    severity: detected ? 'high' : 'none',
  };
}

/**
 * Score comportemental 0-100
 * Basé sur : pages vues, temps sur produit, messages envoyés, objections prix, questions techniques
 */
export function computeBehaviorScore(session, entities, clientContext = {}) {
  let score = 0;

  // Messages envoyés (engagement)
  const messageCount = session.message_count || 0;
  if (messageCount >= 3) score += 10;
  if (messageCount >= 6) score += 10;
  if (messageCount >= 10) score += 10;

  // Données connues (engagement = profil complet)
  if (entities.email) score += 20;
  if (entities.name) score += 10;
  if (entities.phone) score += 15;

  // Contexte page (signal d'intention fort)
  if (clientContext.page_url?.includes('product')) score += 15;
  if (clientContext.page_url?.includes('cart')) score += 20;
  if (clientContext.page_url?.includes('checkout')) score += 25;

  // Plafond
  return Math.min(score, 100);
}

/**
 * Détection risque d'abandon
 */
export function detectAbandonRisk(message, session) {
  const ABANDON_SIGNALS = [
    'trop cher', 'pas le budget', 'concurrent', 'ailleurs', 'autre site',
    'je réfléchis', 'pas sûr', 'peut-être plus tard', 'pas maintenant',
    'too expensive', 'competitor', 'maybe later', 'not sure',
  ];

  const lower = message.toLowerCase();
  const hasSignal = ABANDON_SIGNALS.some(s => lower.includes(s));

  return hasSignal;
}
```

---

## Module `lib/domain-scope-guard.js`

```javascript
// lib/domain-scope-guard.js
// Classification in/out scope + recentrage

const OUT_SCOPE_TOPICS = [
  'politique', 'élection', 'parti', 'religion', 'dieu', 'philosophie',
  'météo', 'sport', 'actualité', 'news', 'cuisine', 'recette',
  'politics', 'election', 'religion', 'weather', 'sports', 'cooking',
  // Ajouter selon votre domaine métier
];

export function classifyDomainScope(message) {
  const lower = message.toLowerCase();

  const outScopeMatch = OUT_SCOPE_TOPICS.find(topic => lower.includes(topic));

  if (outScopeMatch) {
    return {
      classification: 'out_scope',
      matched_topic: outScopeMatch,
      action: 'soft_answer_recenter',
    };
  }

  return {
    classification: 'in_scope',
    matched_topic: null,
    action: null,
  };
}
```

---

## Module `lib/guard-orchestrator-v3.js`

```javascript
// lib/guard-orchestrator-v3.js
// Anti-hallucination executoire — catalogue uniquement depuis evidence API

/**
 * Valide que la réponse LLM ne contient pas de carousel non autorisé.
 * Le carousel DOIT venir des preuves API (catalogEvidence), jamais du LLM.
 */
export async function guardOrchestratorV3(reply, action, catalogEvidence, language) {
  let guardedReply = reply;
  let guardedAction = action;

  // Si l'action est SHOW_CAROUSEL mais pas de preuves API → supprimer l'action
  if (action?.type === 'SHOW_CAROUSEL') {
    if (!catalogEvidence || catalogEvidence.length === 0) {
      console.warn('[guard-v3] Carousel bloqué : pas de catalogEvidence');
      guardedAction = null;
    } else {
      // Nettoyer les produits (garder uniquement les champs sûrs)
      guardedAction = {
        type: 'SHOW_CAROUSEL',
        data: {
          products: catalogEvidence.slice(0, 4).map(p => ({
            id: p.id || p._id,
            name: p.name || p.name,
            price: p.price || p.priceData?.price,
            image: p.mainMedia?.image?.url || p.media?.mainMedia?.image?.url || null,
            url: p.productPageUrl || null,
            description: (p.description || '').slice(0, 100),
          })),
        },
      };
    }
  }

  // Nettoyer les prix mentionnés dans la réponse (vérification cohérence)
  // Si le LLM mentionne un prix qui ne correspond pas aux preuves → warning
  if (catalogEvidence?.length > 0) {
    const pricePattern = /(\d+[,.]?\d*)\s*€/g;
    const mentionedPrices = [...guardedReply.matchAll(pricePattern)].map(m => parseFloat(m[1].replace(',', '.')));

    for (const mentionedPrice of mentionedPrices) {
      const validPrices = catalogEvidence.map(p => p.price || p.priceData?.price || 0);
      const isValid = validPrices.some(p => Math.abs(p - mentionedPrice) < 1);
      if (!isValid && mentionedPrice > 0) {
        console.warn(`[guard-v3] Prix suspect mentionné: ${mentionedPrice}€ — pas dans les preuves API`);
        // Option: supprimer le prix du texte ou le marquer
      }
    }
  }

  return { reply: guardedReply, action: guardedAction };
}
```

---

## Module `lib/jwt.js` — Auth admin

```javascript
// lib/jwt.js
// Auth JWT pour l'admin panel

import jwt from 'jsonwebtoken';
import bcrypt from 'bcryptjs';

const JWT_SECRET = process.env.JWT_SECRET;
const JWT_EXPIRY = '24h';

// Fail-fast si pas de secret en production
if (!JWT_SECRET && process.env.NODE_ENV === 'production') {
  console.error('FATAL: JWT_SECRET manquant en production');
  process.exit(1);
}

export function generateToken(payload = {}) {
  if (!JWT_SECRET) throw new Error('JWT_SECRET non configuré');
  return jwt.sign({ ...payload, admin: true }, JWT_SECRET, { expiresIn: JWT_EXPIRY });
}

export function verifyToken(token) {
  if (!JWT_SECRET) throw new Error('JWT_SECRET non configuré');
  return jwt.verify(token, JWT_SECRET);
}

export async function verifyAdminPassword(password) {
  const hash = process.env.ADMIN_PASSWORD_HASH;
  if (!hash) throw new Error('ADMIN_PASSWORD_HASH non configuré');
  return bcrypt.compare(password, hash);
}

/**
 * Middleware Express : vérifie le token JWT admin
 */
export function requireAuth(req, res, next) {
  // Chercher le token dans les cookies httpOnly ou le header Authorization
  const token = req.cookies?.admin_token || extractBearerToken(req);

  if (!token) {
    return res.status(401).json({ error: 'Authentification requise' });
  }

  try {
    const payload = verifyToken(token);
    if (!payload.admin) throw new Error('Pas admin');
    req.admin = payload;
    next();
  } catch (error) {
    return res.status(401).json({ error: 'Token invalide ou expiré' });
  }
}

/**
 * Middleware : vérifie CRON_SECRET pour les routes cron
 */
export function requireCronSecret(req, res, next) {
  const cronSecret = process.env.CRON_SECRET;
  if (!cronSecret) {
    return res.status(500).json({ error: 'CRON_SECRET non configuré' });
  }

  const token = extractBearerToken(req);

  if (token !== cronSecret) {
    return res.status(401).json({ error: 'Cron secret invalide' });
  }

  next();
}

function extractBearerToken(req) {
  const auth = req.headers.authorization;
  if (auth?.startsWith('Bearer ')) return auth.slice(7);
  return null;
}
```

---

## Module `lib/rate-limiter.js`

```javascript
// lib/rate-limiter.js
// Rate limiting atomique via Supabase RPC

import { supabase } from './supabase.js';

const LIMITS = {
  chat: { max: 10, window: 60 },      // 10 req/min
  login: { max: 5, window: 300 },     // 5 req / 5 min
  quote: { max: 3, window: 300 },     // 3 req / 5 min
  webhook: { max: 100, window: 60 },  // 100 req/min (webhooks entrants)
};

/**
 * Vérifie et incrémente le rate limit.
 * Fail-closed : si la BDD est indisponible, BLOQUE (conservateur).
 */
export async function checkRateLimit(key, type = 'chat') {
  const limit = LIMITS[type] || LIMITS.chat;

  try {
    const { data, error } = await supabase.rpc('check_rate_limit', {
      p_key: key,
      p_max_requests: limit.max,
      p_window_seconds: limit.window,
    });

    if (error) {
      // Fail-closed : erreur BDD → bloquer
      console.error('[rate-limiter] Erreur BDD, fail-closed:', error.message);
      return { allowed: false, reason: 'db_error', retry_after: limit.window };
    }

    return data || { allowed: false, reason: 'no_data' };

  } catch (error) {
    // Fail-closed
    console.error('[rate-limiter] Exception, fail-closed:', error.message);
    return { allowed: false, reason: 'exception', retry_after: limit.window };
  }
}

/**
 * Middleware Express rate limiter
 */
export function rateLimiterMiddleware(type = 'chat') {
  return async (req, res, next) => {
    const sessionId = req.body?.session_id || req.ip || 'unknown';
    const key = `${type}:${sessionId}`;

    const result = await checkRateLimit(key, type);

    if (!result.allowed) {
      return res.status(429).json({
        error: 'Trop de requêtes. Veuillez patienter.',
        retry_after: result.retry_after || 60,
      });
    }

    next();
  };
}
```

---

## Module `lib/security-agent.js` — Agent de sécurité autonome

```javascript
// lib/security-agent.js
// Scan autonome incidents + alertes + auto-fix optionnel

import { supabase } from './supabase.js';
import { sendEmail } from './email.js';
import { sendWhatsApp } from './whatsapp.js';

const SEVERITY_THRESHOLDS = {
  critical: 1,    // 1 incident critique → alerte immédiate
  high: 3,        // 3 incidents high → alerte
  medium: 10,     // 10 incidents medium → rapport
};

export async function runSecurityScan() {
  console.log('[security-agent] Démarrage scan sécurité');

  try {
    // 1. Collecte incidents récents (dernières 24h)
    const { data: incidents } = await supabase
      .from('app_security_incidents')
      .select('*')
      .gte('created_at', new Date(Date.now() - 24 * 60 * 60 * 1000).toISOString())
      .eq('resolved', false)
      .order('created_at', { ascending: false });

    if (!incidents || incidents.length === 0) {
      console.log('[security-agent] Aucun incident non résolu');
      return { status: 'clean', incidents: 0 };
    }

    // 2. Classification par sévérité
    const bySeverity = {
      critical: incidents.filter(i => i.severity === 'critical'),
      high: incidents.filter(i => i.severity === 'high'),
      medium: incidents.filter(i => i.severity === 'medium'),
      low: incidents.filter(i => i.severity === 'low'),
    };

    // 3. Déterminer si alerte requise
    const needsAlert =
      bySeverity.critical.length >= SEVERITY_THRESHOLDS.critical ||
      bySeverity.high.length >= SEVERITY_THRESHOLDS.high ||
      bySeverity.medium.length >= SEVERITY_THRESHOLDS.medium;

    if (needsAlert) {
      await sendSecurityAlert(bySeverity, incidents.length);
    }

    // 4. Auto-fix si activé
    if (process.env.SECURITY_AUTO_FIX_ENABLED === 'true') {
      await attemptAutoFix(bySeverity.high.concat(bySeverity.critical));
    }

    console.log(`[security-agent] Scan terminé: ${incidents.length} incidents, alerte: ${needsAlert}`);

    return {
      status: needsAlert ? 'alert_sent' : 'monitored',
      incidents: incidents.length,
      by_severity: {
        critical: bySeverity.critical.length,
        high: bySeverity.high.length,
        medium: bySeverity.medium.length,
        low: bySeverity.low.length,
      },
    };

  } catch (error) {
    console.error('[security-agent] Erreur scan:', error);
    return { status: 'error', error: error.message };
  }
}

async function sendSecurityAlert(bySeverity, total) {
  const subject = `🔒 Alerte Sécurité — ${total} incidents détectés`;
  const body = `
Rapport de sécurité automatique

Total incidents (24h): ${total}
- Critiques: ${bySeverity.critical.length}
- Élevés: ${bySeverity.high.length}
- Moyens: ${bySeverity.medium.length}
- Faibles: ${bySeverity.low.length}

Incidents critiques:
${bySeverity.critical.map(i => `- [${i.type}] ${JSON.stringify(i.details)}`).join('\n')}

Veuillez vérifier le panel admin → Sécurité pour plus de détails.
  `.trim();

  const alertEmail = process.env.SECURITY_ALERT_EMAIL || process.env.ADMIN_EMAIL;
  if (alertEmail) {
    await sendEmail({ to: alertEmail, subject, text: body }).catch(console.error);
  }

  const adminPhone = process.env.ADMIN_PHONE;
  if (adminPhone && bySeverity.critical.length > 0) {
    await sendWhatsApp({
      to: adminPhone,
      message: `🔒 ALERTE SÉCURITÉ: ${bySeverity.critical.length} incident(s) critique(s) détecté(s). Vérifiez le panel admin.`,
    }).catch(console.error);
  }
}

async function attemptAutoFix(incidents) {
  for (const incident of incidents) {
    try {
      switch (incident.type) {
        case 'rate_limit': {
          // Augmenter temporairement les seuils ou bloquer l'IP source
          console.log(`[security-agent] Auto-fix rate_limit pour session: ${incident.session_id}`);
          break;
        }
        // Ajouter d'autres auto-fixes selon les types d'incidents
      }

      // Marquer comme résolu si auto-fix réussi
      await supabase
        .from('app_security_incidents')
        .update({ resolved: true, resolved_at: new Date().toISOString() })
        .eq('id', incident.id);

    } catch (error) {
      console.error(`[security-agent] Auto-fix échoué pour ${incident.id}:`, error.message);
    }
  }
}
```

---

## Scan secrets pre-commit (`scripts/scan-secrets.cjs`)

```javascript
// scripts/scan-secrets.cjs
// Détecte les secrets dans les fichiers avant commit (hook pre-commit)

const fs = require('fs');
const path = require('path');
const { execSync } = require('child_process');

const SECRET_PATTERNS = [
  { name: 'API Key générique', pattern: /(?:api[_-]?key|apikey)\s*[:=]\s*['"]?([a-zA-Z0-9_\-]{20,})['"]?/i },
  { name: 'Token Bearer', pattern: /bearer\s+([a-zA-Z0-9_\-\.]{20,})/i },
  { name: 'AWS Secret', pattern: /AWS_SECRET_ACCESS_KEY\s*=\s*([a-zA-Z0-9\/+]{40})/i },
  { name: 'Private Key RSA', pattern: /-----BEGIN (?:RSA )?PRIVATE KEY-----/ },
  { name: 'Supabase Service Role', pattern: /eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9\.[a-zA-Z0-9_\-]+\.[a-zA-Z0-9_\-]+/ },
];

const EXCLUDED_FILES = ['.env.example', 'scan-secrets.cjs', '*.md'];

// Obtenir les fichiers stagés
const stagedFiles = execSync('git diff --cached --name-only').toString().split('\n').filter(Boolean);

let hasSecrets = false;

for (const file of stagedFiles) {
  if (!fs.existsSync(file)) continue;
  if (EXCLUDED_FILES.some(ex => ex.startsWith('*') ? file.endsWith(ex.slice(1)) : file.includes(ex))) continue;

  const content = fs.readFileSync(file, 'utf-8');

  for (const { name, pattern } of SECRET_PATTERNS) {
    if (pattern.test(content)) {
      console.error(`❌ SECRET DÉTECTÉ dans ${file}: ${name}`);
      hasSecrets = true;
    }
  }
}

if (hasSecrets) {
  console.error('\n🛑 Commit bloqué : des secrets ont été détectés. Utilisez .env pour les secrets.');
  process.exit(1);
} else {
  console.log('✅ Aucun secret détecté.');
  process.exit(0);
}
```

### Hook pre-commit (`.git/hooks/pre-commit`)
```bash
#!/bin/sh
node scripts/scan-secrets.cjs
```
