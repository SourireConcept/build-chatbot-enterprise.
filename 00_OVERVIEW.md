# 00 — OVERVIEW : Chatbot IA Enterprise Blueprint

## Qu'est-ce que ce blueprint ?

Ce blueprint décrit un **système d'agent IA conversationnel enterprise complet**, prêt pour la production sur Vercel. Il est issu d'une implémentation réelle en production (V4.3), anonymisé et généralisé pour être réutilisable.

---

## Capacités du système

### 1. Chatbot conversationnel avancé
- **Multi-canal** : widget web embed, WhatsApp Business API, Facebook Messenger, Instagram, TikTok
- **Multilingue** : détection automatique de la langue (FR, EN, DE, AR + extension facile)
- **Mémoire cognitive persistante** : session 90 jours, entités nommées, RAG hybride
- **Wizards FSM** : 5 flux guidés (suivi commande, contact, coupon, douanes EORI, checkout)
- **Tokens d'action** : 15+ actions déterministes interceptées avant le NLP
- **Proactive trigger** : message commercial silencieux après 60s d'inactivité

### 2. Agent commercial proactif
- Cadence automatique : prénom au message #3, email au message #5
- Capture leads continue sur chaque message utilisateur
- Sync immédiate si email présent (BDD locale + Wix Data)
- Scoring comportemental (pages vues, temps produit, objections prix)
- Détection risque d'abandon (concurrent, hésitation, objection prix)

### 3. Admin Panel God Mode
- Interface SPA sans framework frontend (~600 Ko HTML/CSS/JS natif)
- Chat IA admin avec accès total aux outils (DB, email, Wix, WhatsApp, fichiers)
- Sections : Leads, Commandes, Workflows, Autoroute, Prompts, Facts, Queue Sociale, Agents
- Auth JWT sécurisé (httpOnly cookie + postMessage one-shot)
- Runtime flags live (activation/désactivation sans redéploiement)

### 4. Agent Runtime Autonome
- Boucle plan → act → observe → decide
- Budgets configurables (steps, tool_calls, durée, risk level)
- Approbation humaine pour les actions à risque élevé
- Notifications admin (email + WhatsApp) sur pause
- Timeline auditée de tous les runs

### 5. Publication Sociale Automatisée (Autoroute AI Pro)
- Génération texte : Perplexity → Gemini → HuggingFace (chain de fallback)
- Génération image : HF model packs (4 presets)
- Génération vidéo : HunyuanVideo / Wan (fallback configurable)
- Publication : Facebook Pages, Instagram Business, TikTok, YouTube, Wix Blog
- Queue d'approbation admin avec validation manuelle optionnelle
- Cron Vercel quotidien (configurable)

### 6. Intégration Wix complète (optionnelle)
- Catalogue Stores : recherche live + circuit breaker + retry + cache Supabase
- CRM V4 : contacts (upsert find/update/create)
- Data V2 : 70+ collections CMS
- Bookings V2 : services de réservation
- Blog V3 : publication + media extraction robuste
- Checkout eCommerce : création checkout + taxes/livraison + ordre optionnel
- Invoices REST : création/envoi/marquage payé
- Webhooks : validation signature + idempotence

### 7. Sécurité défense-en-profondeur
- JWT admin (bcrypt + 24h expiry)
- Rate limiting atomique (RPC Supabase) : 10 req/min chat, 5 req/min login
- Guardrails LLM : role override / DAN / jailbreak → refus sécurisé
- Disclosure guard : blocage exfiltration prompt/secrets/CoT/ENV
- Guard orchestrator : anti-hallucination executoire (carousel uniquement depuis preuves API)
- Meta-agent shield : détection injections
- Scan secrets pre-commit
- Security agent autonome (scan incidents + alertes + auto-fix)

---

## Tech Stack Complet

| Couche | Technologie |
|---|---|
| Runtime | Node.js 22.x, ESM natif |
| Serveur | Express.js → Vercel Serverless |
| Base de données | Supabase (PostgreSQL 15 + pgvector + Storage) |
| LLM principal | HuggingFace Inference API (modèle configurable) |
| LLM admin | HuggingFace Qwen 72B (ou équivalent) |
| LLM fallback | Perplexity → OpenAI → OpenRouter → xAI |
| LLM social | Perplexity sonar-pro → Gemini 2.5 Flash → HF |
| Embeddings | HF BAAI/bge-m3 (1024D) via pgvector |
| Embeddings legacy | HF all-MiniLM-L6-v2 (384D) |
| Images AI | HF model packs (Flux + SDXL + Wan) |
| Vidéos AI | HF HunyuanVideo / Wan |
| Email | SMTP (nodemailer, port 465 SSL) |
| WhatsApp | Meta Graph API v18 |
| Paiement | Wix Checkout → Wix Invoices → Stripe (cascade) |
| PDF | PDFKit |
| Vidéo processing | ffmpeg-static + fluent-ffmpeg |
| Drive | Google Drive (Service Account / OAuth) |
| Monitoring | Sentry (profiling-node) |
| Tests | Jest (unitaires) + Playwright (E2E) |
| Déploiement | Vercel (edge, crons, analytics) |

---

## Points d'Entrée (Routes Express)

### Chat public
- `POST /api/chat` — Chatbot principal (canal web)
- `POST /api/webhook/whatsapp` — WhatsApp entrant
- `POST /api/webhook/messenger` — Facebook Messenger entrant
- `POST /api/webhook/tiktok` — TikTok entrant
- `GET /api/webhook/*` — Vérification webhooks (challenge)

### Admin (auth JWT requis)
- `POST /api/admin/login` — Authentification
- `POST /api/admin/chat` — Super Chat God Mode
- `GET /api/admin/leads` — Gestion leads
- `GET /api/admin/orders` — Commandes
- `GET/POST /api/admin/workflows` — Workflows
- `GET/POST /api/admin/autoroute/*` — Génération sociale
- `GET/POST /api/admin/agent/*` — Agent runtime autonome
- `GET/POST /api/admin/facts` — Company facts
- `GET/POST /api/admin/prompt` — Prompt store
- `GET/POST /api/admin/runtime-flags` — Feature flags live
- `POST /api/admin/security/scan` — Security agent scan

### Crons Vercel (auth CRON_SECRET)
- `POST /api/cron/daily` — Autoroute quotidien (09:00)
- `POST /api/cron/security-scan` — Scan sécurité (00:00)
- `POST /api/cron/session-cleanup` — Nettoyage sessions
- `POST /api/cron/agent-maintenance` — Maintenance agent runtime

### Utilitaires
- `GET /api/health` — Health check
- `GET /api/catalog` — Catalogue produits (cache)
- `POST /api/quote` — Génération devis
- `POST /api/trustpilot-review` — Avis Trustpilot

---

## Schéma de données Supabase (30 tables)

**Préfixe conseillé** : `app_` (remplacer par votre préfixe)

| Catégorie | Tables |
|---|---|
| Sessions/Mémoire | `app_chat_sessions`, `app_entity_memory`, `app_knowledge_chunks`, `app_knowledge_chunks_v2`, `app_conversation_contexts` |
| Business | `app_leads`, `app_quotes`, `app_wix_invoices`, `app_product_cache` |
| Admin | `app_admin_logs`, `app_admin_config`, `app_admin_conversation` |
| Agents | `app_agent_registry`, `app_agent_tool_assignments`, `app_agent_objectives`, `app_agent_runs`, `app_agent_run_steps`, `app_agent_tool_calls`, `app_agent_approvals`, `app_agent_events`, `app_agent_config` |
| Contenu | `app_company_facts`, `app_company_facts_versions`, `app_prompt_store` |
| Sécurité | `app_security_incidents`, `app_rate_limits` |
| Workflows | `app_workflows`, `app_social_publication_queue`, `app_wix_checkout_payments` |
| Runtime | `app_runtime_state` |

---

## Conventions globales du projet

### Code
- **ESM pur** : `import/export` natif, jamais `require()`
- **Fonctions pures isolées** : chaque module exporte des fonctions testables indépendamment
- **Fail-safe par défaut** : rate limiter fail-closed, JWT fail-fast en prod
- **Feature flags ENV** : presque toute fonctionnalité activable/désactivable sans redéploiement

### Nommage
- **Fichiers** : kebab-case (`ai-v3.js`, `chat-router.js`)
- **Fonctions** : camelCase (`handleChatRequestV3`, `analyzeRoleOverride`)
- **Tables BDD** : snake_case avec préfixe (`app_chat_sessions`)
- **Variables ENV** : SCREAMING_SNAKE_CASE (`HF_API_TOKEN`, `WIX_API_KEY`)
- **Tokens d'action** : `__CAPS_SNAKE__` (`__SHOW_CATALOG__`, `__GENERATE_QUOTE__`)

### Résilience
- Circuit breaker sur toutes les APIs externes
- Retry exponentiel (`baseDelay × 2^attempt`, cap 2s)
- Timeout race sur contact lookup (700ms), Wix (8s)
- Cache multi-niveaux (mémoire LRU + Supabase)

### Sécurité
- Sanitiser TOUT l'input utilisateur (`escapeHtml`, regex filtering)
- Nettoyer TOUT l'output LLM (`<think>...</think>`, CoT artifacts, leaks ENV)
- Allowlist stricte SMTP (domaines configurés uniquement)
- Idempotence webhooks (déduplication events)
- Pas de raw SQL (Supabase JS uniquement sauf RPCs définies)

---

## Flux de déploiement initial

1. **Supabase** : créer projet → appliquer 30 migrations → activer pgvector → configurer RLS
2. **Variables ENV** : copier `.env.example` → remplir toutes les valeurs → configurer sur Vercel
3. **Ingestion KB** : `node scripts/ingest-knowledge-v2.mjs` → chunks + embeddings → Supabase
4. **Premier déploiement** : `vercel --prod`
5. **Configurer webhooks** : WhatsApp → `https://[domain]/api/webhook/whatsapp`, Messenger, TikTok
6. **Vérifier health** : `GET /api/health` → `{status: "ok", version: "4.3"}`
7. **Configurer prompt** : Admin Panel → Prompt Store → sauvegarder prompt chatbot initial
8. **Configurer company facts** : Admin Panel → Facts Entreprise → sauvegarder

---

*Pour les détails de chaque sous-système, lire les fichiers 01 à 12.*
