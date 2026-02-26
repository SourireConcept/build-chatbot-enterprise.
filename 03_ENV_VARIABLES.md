# 03 — VARIABLES D'ENVIRONNEMENT : Template complet

## Instructions
1. Copier ce fichier en `.env` pour le développement local
2. Sur Vercel : configurer chaque variable dans Settings → Environment Variables
3. JAMAIS committer de vraies valeurs dans git (`.env` dans `.gitignore`)
4. Les variables marquées `[REQUIRED]` sont obligatoires au démarrage

---

## Fichier `.env.example` à créer

```bash
# ============================================================
# [APP_NAME] API — Variables d'environnement
# ============================================================
# IMPORTANT: Copier en .env et remplir les valeurs réelles.
# Ne jamais committer .env dans git.
# ============================================================

# ─────────────────────────────────────────────────────────────
# ENVIRONNEMENT
# ─────────────────────────────────────────────────────────────
NODE_ENV=development                    # development | production | test [REQUIRED]

# ─────────────────────────────────────────────────────────────
# SÉCURITÉ / AUTH ADMIN
# ─────────────────────────────────────────────────────────────
# Générer avec : node -e "const bcrypt = require('bcryptjs'); bcrypt.hash('votre-mot-de-passe', 12).then(console.log)"
ADMIN_PASSWORD_HASH=                    # Hash bcrypt du mot de passe admin [REQUIRED]
# Générer avec : node -e "console.log(require('crypto').randomBytes(64).toString('hex'))"
JWT_SECRET=                             # Secret JWT (min 64 chars) [REQUIRED]
# Générer avec : node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
ENCRYPTION_KEY=                         # Clé chiffrement 32 chars hex [REQUIRED]

ADMIN_PHONE=                            # Numéro WhatsApp admin (format international +33...)
ADMIN_EMAIL=                            # Email de l'administrateur
ADMIN_ALLOWED_ORIGINS=http://localhost:3000,https://votre-domaine.com
CRON_SECRET=                            # Secret pour les routes cron Vercel [REQUIRED]
SECURITY_ALERT_EMAIL=                   # Email pour alertes sécurité
ADMIN_SUPERCHAT_STRICT=false            # Mode strict super chat (true en prod recommandé)
ADMIN_EMAIL_ALLOWED_DOMAINS=votre-domaine.com  # Domaines SMTP autorisés (séparés par virgules)
DEBUG_WEBHOOK_SIGNATURE=false           # Logs signatures webhook (dev seulement)
PUBLIC_DEBUG_LOGS=false                 # Exposer logs debug en réponse API (JAMAIS en prod)
DISABLE_LOGS=false                      # Désactiver tous les logs console

# ─────────────────────────────────────────────────────────────
# SUPABASE (Base de données)
# ─────────────────────────────────────────────────────────────
SUPABASE_URL=                           # https://xxx.supabase.co [REQUIRED]
SUPABASE_ANON_KEY=                      # Clé publique Supabase [REQUIRED]
SUPABASE_SERVICE_ROLE_KEY=              # Clé service role (backend seulement) [REQUIRED]

# ─────────────────────────────────────────────────────────────
# HUGGING FACE (LLM Principal)
# ─────────────────────────────────────────────────────────────
HF_API_TOKEN=                           # Token HuggingFace [REQUIRED]
HF_TOKEN=                               # Alias HF_API_TOKEN (certains modules)

# Modèles LLM configurables
HF_CHAT_MODEL_V3=moonshotai/Kimi-K2.5:fastest  # Modèle principal chatbot V3
HF_MODEL_FAST=Qwen/Qwen2.5-7B-Instruct        # Modèle rapide (classification)
HF_MODEL_HEAVY=Qwen/Qwen2.5-72B-Instruct      # Modèle lourd (admin, agent runtime)
HF_MODEL_ENTITY=Qwen/Qwen2.5-7B-Instruct      # Extraction entités
HF_MODEL_SUMMARY=Qwen/Qwen2.5-7B-Instruct     # Résumés session
HF_MEMORY_MODEL=Qwen/Qwen2.5-7B-Instruct      # Extraction mémoire
HF_MEMORY_MAX_TOKENS=512
HF_MEMORY_TEMPERATURE=0.3

# Embeddings
HF_EMBED_MODEL_V2=BAAI/bge-m3          # Modèle embeddings V2 (1024D)
HF_EMBED_DIM_V2=1024                   # Dimension embeddings V2

# Provider d'inférence
HF_INFERENCE_PROVIDER=nebius           # nebius | together | fireworks | auto
HF_SPACE_URL=                          # URL HF Space (si utilisation d'un Space custom)
HF_ENABLE_SPACE_FALLBACK=false         # Fallback vers HF Space si inference échoue
HF_FALLBACK_MODEL=Qwen/Qwen2.5-7B-Instruct  # Modèle fallback si principal inaccessible

# Seuils
HF_HEAVY_TRIGGER_SCORE=0.7             # Score min pour déclencher le modèle lourd
HF_FORCE_MODEL=                        # Forcer un modèle spécifique (debug)

# Politique modèle
HF_MODEL_POLICY=auto                   # auto | fast | heavy | force

# Cache embeddings
EMBEDDING_CACHE_TTL_MS=900000          # 15 minutes
EMBEDDING_CACHE_MAX=500
EMBEDDING_RETRY_ATTEMPTS=3
EMBEDDING_RETRY_BASE_DELAY_MS=500

# ─────────────────────────────────────────────────────────────
# LLM FALLBACK CHAIN
# ─────────────────────────────────────────────────────────────
# Ordre de fallback : HF → Perplexity → OpenAI → OpenRouter → xAI

# Providers primaires et fallback (noms séparés par virgules)
CHAT_PRIMARY_PROVIDERS=hf              # Providers primaires chat
CHAT_FALLBACK_PROVIDERS=perplexity,openai,openrouter,xai
ADMIN_PRIMARY_PROVIDERS=hf             # Providers primaires admin
ADMIN_FALLBACK_PROVIDERS=perplexity,openai
AGENT_PRIMARY_PROVIDERS=hf
AGENT_FALLBACK_PROVIDERS=openai,openrouter

# Perplexity
PERPLEXITY_API_KEY=                    # Clé API Perplexity
PERPLEXITY_MODEL=llama-3.1-sonar-large-128k-online  # Modèle Perplexity

# OpenAI (ou compatible OpenAI API)
OPENAI_API_KEY=                        # Clé API OpenAI
OPENAI_MODEL=gpt-4o-mini
OPENAI_BASE_URL=https://api.openai.com/v1  # Changer pour API compatible

# OpenRouter
OPENROUTER_API_KEY=                    # Clé API OpenRouter
OPENROUTER_MODEL=anthropic/claude-3-haiku
OPENROUTER_BASE_URL=https://openrouter.ai/api/v1

# xAI (Grok)
XAI_API_KEY=                           # Clé API xAI
XAI_MODEL=grok-beta
XAI_BASE_URL=https://api.x.ai/v1

# ─────────────────────────────────────────────────────────────
# AUTOROUTE AI PRO (Publication sociale automatisée)
# ─────────────────────────────────────────────────────────────
GEMINI_API_KEY=                        # Clé API Google Gemini
GOOGLE_AI_API_KEY=                     # Alias Gemini
GEMINI_TEXT_MODEL=gemini-2.5-flash     # Modèle Gemini pour génération texte

AUTOROUTE_DISABLE_HF=false             # Désactiver HF dans Autoroute
AUTOROUTE_DISABLE_GEMINI=false         # Désactiver Gemini dans Autoroute
AUTOROUTE_INSTRUCTIONS=                # Instructions système Autoroute (override DB)
AUTOROUTE_DEFAULT_TOPIC=               # Thème par défaut des publications
AUTOROUTE_TEXT_HF_MODEL=THUDM/glm-4-plus  # Modèle HF pour texte Autoroute
AUTOROUTE_TEXT_HF_FALLBACK_MODELS=Qwen/Qwen2.5-72B-Instruct  # Fallbacks virgule-séparés

# Génération image
AUTOROUTE_IMAGE_INSTRUCTIONS=          # Instructions génération images
AUTOROUTE_MEDIA_PACK=glm-flux-wan      # Pack modèle : glm-flux-wan | glm-flux-hunyuan | glm-sdxl-wan | glm-sdxl-hunyuan | auto
AUTOROUTE_IMAGE_QUALITY_HINT=high      # low | medium | high
AUTOROUTE_IMAGE_QA_PROVIDER=hf         # Provider QA image (BLIP captioning)
AUTOROUTE_IMAGE_QA_MODEL=Salesforce/blip-image-captioning-large
AUTOROUTE_IMAGE_QA_MAX_ATTEMPTS=3
AUTOROUTE_IMAGE_UPSCALE=false          # Activer upscaling image
AUTOROUTE_IMAGE_UPSCALE_PROVIDER=hf
AUTOROUTE_IMAGE_UPSCALE_MODEL=

# Génération vidéo
AUTOROUTE_VIDEO_INSTRUCTIONS=          # Instructions génération vidéos

# Publication
AUTOROUTE_PUBLICATION_MODE_DEFAULT=auto  # auto | manual | disabled
AUTOROUTE_TIMEZONE=Europe/Paris
AUTOROUTE_CRON_WINDOW_MINUTES=30       # Fenêtre de déclenchement cron

# ─────────────────────────────────────────────────────────────
# WIX (si WIX_ENABLED)
# ─────────────────────────────────────────────────────────────
WIX_API_KEY=                           # Clé API Wix [REQUIRED si Wix activé]
WIX_SITE_ID=                           # ID du site Wix [REQUIRED si Wix activé]
WIX_ACCOUNT_ID=                        # ID compte Wix

# Performance
WIX_TIMEOUT_MS=8000                    # Timeout appels Wix (ms)
WIX_RETRY_ATTEMPTS=3                   # Tentatives retry
WIX_RETRY_BASE_DELAY_MS=300            # Délai base retry (ms)

# Circuit breaker
WIX_CB_FAILURE_THRESHOLD=5             # Seuil ouverture circuit breaker
WIX_CB_OPEN_MS=30000                   # Durée circuit ouvert (ms)

# Sauvegarde
WIX_SAVE_RETRY_ATTEMPTS=3
WIX_SAVE_RETRY_DELAY_MS=1000

# URLs
WIX_PUBLIC_BASE_URL=https://votre-site.com  # URL publique du site Wix

# Commandes
WIX_ORDER_SEARCH_PAGES=3               # Pages max de recherche commandes
WIX_STORES_APP_ID=                     # App ID Wix Stores

# Collections
WIX_PROMOS_COLLECTION_ID=              # ID collection promotions

# Checkout
WIX_CHECKOUT_ENABLED=true             # Activer checkout Wix
WIX_CHECKOUT_PRIMARY=true             # Wix Checkout prioritaire sur Stripe
WIX_CHECKOUT_FALLBACK_STRIPE=true     # Fallback Stripe si Wix échoue
WIX_CHECKOUT_MODE=live                 # live | sandbox
WIX_CHECKOUT_AUTO_CREATE_ORDER=false  # Auto-créer commande après checkout
WIX_CHECKOUT_TIMEOUT_MS=10000

# Webhooks
WIX_WEBHOOK_ENABLED=true
WIX_WEBHOOK_SECRET=                    # Secret validation webhooks Wix

# Bookings
WIX_BOOKINGS_CACHE_ENABLED=true

# Invoices
WIX_INVOICES_ENABLED=true
WIX_INVOICE_DUE_DAYS=30               # Délai paiement factures (jours)
WIX_INVOICE_LOCALE=fr                 # Locale factures Wix
WIX_INVOICE_TIMEOUT_MS=10000

# Fallbacks
WIX_FALLBACK_ON_HF_ERROR=true         # Fallback Wix si HF indisponible

# ─────────────────────────────────────────────────────────────
# COUPONS
# ─────────────────────────────────────────────────────────────
COUPON_MAX_FIXED_AMOUNT=50             # Montant max coupon fixe (€)
COUPON_CODE_BYTES=8                    # Entropie code coupon (octets)

# ─────────────────────────────────────────────────────────────
# STRIPE (Paiement fallback)
# ─────────────────────────────────────────────────────────────
STRIPE_SECRET_KEY=                     # Clé secrète Stripe (sk_live_... ou sk_test_...)

# ─────────────────────────────────────────────────────────────
# EMAIL SMTP
# ─────────────────────────────────────────────────────────────
SMTP_HOST=                             # Serveur SMTP (ex: ssl0.ovh.net)
SMTP_PORT=465                          # Port SMTP (465 pour SSL)
SMTP_USER=                             # Email SMTP
SMTP_PASS=                             # Mot de passe SMTP
SMTP_FROM=                             # Adresse expéditeur (ex: "Mon App <contact@monsite.com>")
NOTIFICATION_EMAIL=                    # Email de notification interne

# ─────────────────────────────────────────────────────────────
# EMAIL IMAP (lecture emails admin)
# ─────────────────────────────────────────────────────────────
IMAP_HOST=                             # Serveur IMAP
IMAP_PORT=993                          # Port IMAP SSL
IMAP_USER=                             # Identifiant IMAP
IMAP_PASS=                             # Mot de passe IMAP
IMAP_FOLDER=INBOX                      # Dossier à surveiller

# ─────────────────────────────────────────────────────────────
# WHATSAPP BUSINESS API
# ─────────────────────────────────────────────────────────────
WHATSAPP_ACCESS_TOKEN=                 # Token WhatsApp Business API
WHATSAPP_PHONE_NUMBER_ID=              # ID numéro téléphone WhatsApp Business
WHATSAPP_APP_SECRET=                   # Secret app Meta (validation webhooks)
WHATSAPP_VERIFY_TOKEN=                 # Token vérification webhook WhatsApp
META_APP_SECRET=                       # Secret app Meta global

# ─────────────────────────────────────────────────────────────
# FACEBOOK MESSENGER
# ─────────────────────────────────────────────────────────────
FB_PAGE_ID=                            # ID page Facebook principale
FB_PAGE_ACCESS_TOKEN=                  # Token accès page Facebook
FACEBOOK_APP_SECRET=                   # Secret app Facebook
MESSENGER_VERIFY_TOKEN=                # Token vérification webhook Messenger
WEBHOOK_VERIFY_TOKEN=                  # Token vérification webhook global

# ─────────────────────────────────────────────────────────────
# INSTAGRAM
# ─────────────────────────────────────────────────────────────
IG_USER_ID=                            # ID utilisateur Instagram Business
IG_ACCESS_TOKEN=                       # Token accès Instagram Graph API

# ─────────────────────────────────────────────────────────────
# TIKTOK
# ─────────────────────────────────────────────────────────────
TIKTOK_ACCESS_TOKEN=                   # Token TikTok messaging
TIKTOK_VERIFY_TOKEN=                   # Token vérification webhook TikTok
TIKTOK_PUBLISH_ACCESS_TOKEN=           # Token TikTok Content Publishing API
TIKTOK_PUBLISH_OPEN_ID=               # Open ID TikTok pour publication

# ─────────────────────────────────────────────────────────────
# MULTI-COMPTES SOCIAUX (pour Autoroute multi-pages)
# Format: SOCIAL_[PLATFORM]_[FIELD]_[ACCOUNT_ID]
# ─────────────────────────────────────────────────────────────
SOCIAL_APPROVAL_REQUIRED=true          # Approbation admin avant publication
# Comptes Facebook
SOCIAL_FB_PAGE_ID_FB1=                 # ID page Facebook compte 1
SOCIAL_FB_TOKEN_FB1=                   # Token page Facebook compte 1
# Comptes Instagram
SOCIAL_IG_USER_ID_IG1=
SOCIAL_IG_TOKEN_IG1=
# Comptes TikTok
SOCIAL_TIKTOK_OPEN_ID_TK1=
SOCIAL_TIKTOK_TOKEN_TK1=
# Comptes YouTube (via webhook custom)
SOCIAL_YT_WEBHOOK_URL_YT1=
SOCIAL_YT_WEBHOOK_TOKEN_YT1=
# Multi-pages (JSON stringifié)
PAGE_TOKENS=                           # {"page_id_1": "token_1", "page_id_2": "token_2"}

# ─────────────────────────────────────────────────────────────
# GOOGLE DRIVE
# ─────────────────────────────────────────────────────────────
GOOGLE_DRIVE_API_KEY=                  # Clé API Google Drive simple
# OU service account (prioritaire sur API key)
GOOGLE_DRIVE_SERVICE_ACCOUNT_JSON=     # JSON service account (inline, stringifié)
GOOGLE_DRIVE_SERVICE_ACCOUNT_PATH=     # OU chemin vers fichier JSON service account
GOOGLE_DRIVE_SCOPES=https://www.googleapis.com/auth/drive.file
# OU OAuth (si Service Account non disponible)
GOOGLE_DRIVE_CLIENT_ID=
GOOGLE_DRIVE_CLIENT_SECRET=
GOOGLE_DRIVE_REFRESH_TOKEN=
# Dossiers
WORKFLOWS_DEFAULT_DRIVE_FOLDER_ID=     # ID dossier Drive par défaut pour workflows
AUTOROUTE_DRIVE_FOLDER_ID=             # ID dossier Drive pour assets Autoroute

# ─────────────────────────────────────────────────────────────
# AGENT RUNTIME AUTONOME
# ─────────────────────────────────────────────────────────────
AGENT_ENABLED=true                     # Activer l'agent runtime autonome
AGENT_MAX_STEPS=20                     # Budget steps par run
AGENT_MAX_TOOL_CALLS=50               # Budget tool calls par run
AGENT_MAX_DURATION_MS=300000          # Durée max run (5 minutes)
AGENT_ALLOWED_RISK_LEVELS=low,medium  # Niveaux risque autorisés sans approbation
AGENT_APPROVAL_FLOW=true              # Activer les approbations admin pour high risk
AGENT_APPROVAL_TTL_HOURS=24           # Durée validité approbation
AGENT_EVENTS_RETENTION_DAYS=30        # Rétention events en base
AGENT_APPROVAL_NOTIFY_CHANNELS=email,whatsapp

# Modèle IA pour l'agent
AGENT_AI_MODEL=Qwen/Qwen2.5-72B-Instruct
AGENT_AI_TEMPERATURE=0.2
AGENT_AI_MAX_TOKENS=2048

# ─────────────────────────────────────────────────────────────
# CONTACT LOOKUP
# ─────────────────────────────────────────────────────────────
CONTACT_LOOKUP_TIMEOUT_MS=700          # Timeout lookup contact (ms)
CONTACT_LOOKUP_CACHE_ENABLED=true
CONTACT_LOOKUP_CACHE_TTL_MS=300000    # Cache 5 minutes
CONTACT_LOOKUP_CACHE_MAX=1000         # Entrées max en cache

# ─────────────────────────────────────────────────────────────
# SESSION CHATBOT
# ─────────────────────────────────────────────────────────────
CHAT_HISTORY_MAX_TURNS=50             # Historique max (tours de conversation)
CHAT_SESSION_TTL_DAYS=90              # TTL sessions
CHAT_SESSION_CLEANUP_INTERVAL_MS=3600000  # Intervalle nettoyage (1h)
CHAT_MEMORY_V2_AUDIT=false            # Audit logs mémoire V2

# ─────────────────────────────────────────────────────────────
# FEATURE FLAGS CHATBOT
# ─────────────────────────────────────────────────────────────
V3_ENABLED=true                        # Activer moteur V3
V3_KILL_SWITCH=false                   # Kill switch moteur V3 (bascule sur legacy)
CHAT_ENGINE_V2_ENABLED=false           # Activer moteur V2 (compat)
CHAT_ENGINE_V2_CANARY_PERCENT=0        # % trafic routé vers V2 (canary)

# Guards
CONVERSATION_GUARD_ENABLED=true       # Guard scope conversation
ROLE_LOCK_ENABLED=true                # Verrouillage rôle chatbot
SCOPE_POLICY_MODE=strict              # strict | permissive | off
COMPETITOR_POLICY=block               # block | redirect | mention
PRODUCT_QUERY_STRICT_MODE=false       # Mode strict requêtes produits
ENABLE_SUPERVISOR=true                # Supervisor agent actif

# Topic lock
TOPIC_LOCK_ENABLED=true
TOPIC_LOCK_TURNS=3                    # Nombre de tours de verrouillage sujet

# Web
WEB_ALLOWLIST_ONLY=true               # Liens uniquement depuis allowlist
WEB_EXPERT_MODE=false                 # Mode expert (réponses détaillées)
WEB_SHOW_EXTERNAL_LINKS=false         # Autoriser liens externes

# Debug
AI_TIMING_LOGS=false                  # Logs timing IA (dev seulement)

# V2 compat
V2_RUNTIME_GUARD_CHECK_INTERVAL_MS=30000

# ─────────────────────────────────────────────────────────────
# MCP SERVER (Model Context Protocol)
# ─────────────────────────────────────────────────────────────
MCP_SERVER_URL=                        # URL serveur MCP (optionnel)
MCP_TIMEOUT_MS=5000
MCP_FALLBACK_ON_HF_ERROR=true

# ─────────────────────────────────────────────────────────────
# VERCEL
# ─────────────────────────────────────────────────────────────
VERCEL_OIDC_TOKEN=                     # Token OIDC Vercel (auto-injecté en prod)
EDGE_CONFIG=                           # Edge Config Vercel (optionnel)

# ─────────────────────────────────────────────────────────────
# SENTRY (Monitoring)
# ─────────────────────────────────────────────────────────────
SENTRY_DSN=                            # DSN Sentry (optionnel)

# ─────────────────────────────────────────────────────────────
# WORKFLOWS / MEDIA
# ─────────────────────────────────────────────────────────────
WORKFLOW_MEDIA_BUCKET=media             # Nom bucket Supabase Storage pour médias

# ─────────────────────────────────────────────────────────────
# TRUSTPILOT (optionnel)
# ─────────────────────────────────────────────────────────────
TRUSTPILOT_API_KEY=
TRUSTPILOT_BUSINESS_UNIT_ID=

# ─────────────────────────────────────────────────────────────
# PRODUITS SPECIFIQUES (adapter à votre secteur)
# ─────────────────────────────────────────────────────────────
# Ex: PREMIUM_GEL_CONCENTRATION=40 pour un secteur dentaire
# Ajouter ici les constantes métier spécifiques à votre domaine
# PREMIUM_PRODUCT_THRESHOLD=40        # Seuil produit premium
STRICT_CATALOG_ONLY=false             # Répondre UNIQUEMENT sur les produits du catalogue
```

---

## Variables générées automatiquement

Ces variables n'ont PAS besoin d'être configurées manuellement :

| Variable | Source | Description |
|---|---|---|
| `VERCEL_URL` | Vercel | URL du déploiement actuel |
| `VERCEL_ENV` | Vercel | `production` \| `preview` \| `development` |
| `VERCEL_REGION` | Vercel | Région edge |

---

## Script de génération des secrets

```bash
#!/bin/bash
# generate-secrets.sh
# Génère les secrets requis

echo "=== Génération secrets [APP_NAME] ==="
echo ""

echo "JWT_SECRET:"
node -e "console.log(require('crypto').randomBytes(64).toString('hex'))"
echo ""

echo "ENCRYPTION_KEY:"
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
echo ""

echo "CRON_SECRET:"
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
echo ""

echo "ADMIN_PASSWORD_HASH (pour mot de passe 'MonMotDePasse123'):"
node -e "
const bcrypt = require('bcryptjs');
bcrypt.hash('MonMotDePasse123', 12).then(h => console.log(h));
"
echo ""

echo "=== Fin génération ==="
```

---

## Validation au démarrage (validate-env.js)

```javascript
// validate-env.js
// Appelé au démarrage pour vérifier les variables requises

const REQUIRED_VARS = [
  'NODE_ENV',
  'ADMIN_PASSWORD_HASH',
  'JWT_SECRET',
  'ENCRYPTION_KEY',
  'CRON_SECRET',
  'SUPABASE_URL',
  'SUPABASE_ANON_KEY',
  'SUPABASE_SERVICE_ROLE_KEY',
  'HF_API_TOKEN',
];

const REQUIRED_IN_PRODUCTION = [
  'SMTP_HOST',
  'SMTP_USER',
  'SMTP_PASS',
  'NOTIFICATION_EMAIL',
];

export function validateEnvironment() {
  const missing = [];

  for (const v of REQUIRED_VARS) {
    if (!process.env[v]) missing.push(v);
  }

  if (process.env.NODE_ENV === 'production') {
    for (const v of REQUIRED_IN_PRODUCTION) {
      if (!process.env[v]) missing.push(`${v} (requis en production)`);
    }
  }

  if (missing.length > 0) {
    console.error('❌ Variables d\'environnement manquantes :');
    missing.forEach(v => console.error(`  - ${v}`));
    if (process.env.NODE_ENV === 'production') {
      process.exit(1); // Fail-fast en production
    } else {
      console.warn('⚠️  Mode développement: continuation malgré les variables manquantes');
    }
  } else {
    console.log('✅ Toutes les variables d\'environnement sont présentes');
  }
}
```
