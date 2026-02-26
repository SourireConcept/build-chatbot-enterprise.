# Skill : Créer un chatbot IA complet (Blueprint Enterprise)

## DESCRIPTION DU SKILL
Ce skill te donne les instructions COMPLÈTES pour créer de zéro un système de chatbot IA enterprise complet. Tu dois LIRE tous les fichiers dans `docs/chatbot-blueprint/` avant de commencer. Chaque fichier contient le blueprint exhaustif d'un sous-système.

---

## CONTEXTE DU PROJET À CRÉER

Crée un **système d'agent IA conversationnel et opérationnel enterprise** déployé sur **Vercel** avec les caractéristiques suivantes :

- **Chatbot multi-canal** (widget web, WhatsApp, Messenger, TikTok)
- **Agent commercial proactif** avec cadence de capture leads
- **Admin panel God Mode** (chat IA avec accès outils complet)
- **Agent runtime autonome** multi-étapes (plan → act → observe → decide)
- **Publication sociale automatisée** (texte + image + vidéo → réseaux sociaux)
- **Intégration Wix totale** (boutique, checkout, CRM, blog, webhooks)
- **Sécurité défense-en-profondeur** (guardrails, rate limiting, JWT, scan secrets)
- **Mémoire cognitive persistante** (session, entités, RAG hybride)

**Stack technique** :
- Runtime : Node.js 22.x, ESM pur (`type: "module"`)
- Serveur : Express.js (déployé comme API Vercel serverless)
- Base de données : Supabase (PostgreSQL + pgvector + Storage)
- LLM principal : HuggingFace Inference API (modèles configurables)
- LLM fallback : Perplexity → OpenAI → OpenRouter → xAI
- Déploiement : Vercel (edge, crons, analytics)

---

## INSTRUCTIONS POUR MOI (CLAUDE)

### ÉTAPE 1 — Lire la documentation blueprint
Lis ces fichiers dans l'ordre :
1. `docs/chatbot-blueprint/00_OVERVIEW.md` — Vue d'ensemble
2. `docs/chatbot-blueprint/01_ARCHITECTURE.md` — Architecture complète
3. `docs/chatbot-blueprint/02_DATABASE_SCHEMA.md` — Schéma BDD + migrations
4. `docs/chatbot-blueprint/03_ENV_VARIABLES.md` — Toutes les variables d'env
5. `docs/chatbot-blueprint/04_AI_ENGINE.md` — Moteur IA V3 (cœur du système)
6. `docs/chatbot-blueprint/05_SECURITY_LAYERS.md` — Système de sécurité
7. `docs/chatbot-blueprint/06_WIZARDS_FSM.md` — Wizards FSM
8. `docs/chatbot-blueprint/07_INTEGRATIONS.md` — Intégrations externes
9. `docs/chatbot-blueprint/08_ADMIN_PANEL.md` — Panel admin
10. `docs/chatbot-blueprint/09_AGENT_RUNTIME.md` — Agent autonome
11. `docs/chatbot-blueprint/10_SOCIAL_AUTOROUTE.md` — Publication sociale
12. `docs/chatbot-blueprint/11_DEPLOYMENT.md` — Déploiement
13. `docs/chatbot-blueprint/12_TESTING.md` — Tests

### ÉTAPE 2 — Demander les informations manquantes
Avant de créer quoi que ce soit, demande à l'utilisateur :
- `[APP_NAME]` : Nom de l'application (ex: "MonBot Expert")
- `[BRAND_NAME]` : Nom de la marque/entreprise (ex: "MonEntreprise")
- `[BRAND_COLOR]` : Couleur principale hex (ex: "#3a86ff")
- `[DOMAIN]` : Domaine du site (ex: "monsite.com")
- `[PRODUCT_DOMAIN]` : Secteur/type de produits (ex: "produits cosmétiques professionnels")
- `[ADMIN_EMAIL]` : Email administrateur
- `[WIX_ENABLED]` : Intégration Wix ? (oui/non — si non, omettre tous les modules wix-*)
- `[SOCIAL_ENABLED]` : Publication sociale automatisée ? (oui/non)
- `[AGENT_ENABLED]` : Agent runtime autonome ? (oui/non)

### ÉTAPE 3 — Créer la structure du projet

Crée exactement cette structure (remplace `[APP_NAME]` par le slug de l'app) :

```
[app-name]-api/
├── server.js                    # Wrapper lancement local
├── package.json                 # ESM, Node 22, dépendances complètes
├── vercel.json                  # Config Vercel (rewrites, CORS, crons)
├── validate-env.js              # Validation vars au démarrage
├── instrument.js                # Init Sentry
├── jest.config.cjs              # Config Jest
├── .env.example                 # Template complet variables (JAMAIS de vraies valeurs)
├── api/
│   └── index.js                 # Entrypoint Express (TOUTES les routes)
├── lib/
│   ├── — MOTEUR IA —
│   ├── ai-v3.js                 # Moteur principal V3
│   ├── ai-routing.js            # Heuristiques routing
│   ├── ai-runtime-state.js      # Flags runtime live
│   ├── — ADMIN —
│   ├── admin-ai.js              # Agent Admin God Mode
│   ├── admin-tools.js           # Outils admin
│   ├── — AGENT RUNTIME —
│   └── agent/
│       ├── config.js
│       ├── schemas.js
│       ├── agent-registry.js
│       ├── run-ledger.js
│       ├── tool-gateway.js
│       └── orchestrator.js
│   ├── — SOUS-MOTEURS IA —
│   └── ai/
│       ├── quote-handler.js
│       ├── invoice-handler.js
│       ├── training-intent-policy.js
│       ├── wizard-state-machine.js
│       └── wizards/
│           ├── contact-wizard.js
│           ├── tracking-wizard.js
│           ├── coupon-wizard.js
│           ├── eori-wizard.js
│           ├── checkout-address-wizard.js
│           └── wizard-utils.js
│   ├── — SÉCURITÉ —
│   ├── security.js
│   ├── security-agent.js
│   ├── security-audit.js
│   ├── security-disclosure-guard.js
│   ├── domain-scope-guard.js
│   ├── guard-orchestrator-v3.js
│   ├── company-identity-guard.js
│   ├── product-query-guard.js
│   ├── meta-agent.js
│   ├── wix-security-guard.js    # (si WIX_ENABLED)
│   ├── — MÉMOIRE & RAG —
│   ├── memory-engine.js
│   ├── memory-audit-v2.js
│   ├── session-store-v2.js
│   ├── entity-memory-v2.js
│   ├── context-manager.js
│   ├── retrieval-v2.js
│   ├── rag.js
│   ├── web-knowledge-hybrid.js
│   ├── knowledge.js
│   ├── knowledge-sync.js
│   ├── — ROUTING CHATBOT —
│   ├── chat-router.js
│   ├── chat-context-provider.js
│   ├── chat-anon-id.js
│   ├── topic-lock.js
│   ├── — LLM / PROVIDERS —
│   ├── hf-client-v2.js
│   ├── llm-fallback.js
│   ├── autoroute-proxy.js       # (si SOCIAL_ENABLED)
│   ├── — WIX API — (si WIX_ENABLED)
│   ├── wix-catalog.js
│   ├── wix-tools-v3.js
│   ├── wix-checkout.js
│   ├── wix-invoices.js
│   ├── wix-coupon.js
│   ├── wix-crm.js
│   ├── wix-data.js
│   ├── wix-blog.js
│   ├── wix-bookings.js
│   ├── wix-client.js
│   ├── wix-webhook.js
│   ├── wix-chat-contact-id.js
│   ├── collections.js
│   ├── — PAIEMENTS —
│   ├── quotes.js
│   ├── stripe.js
│   ├── orders.js
│   ├── — LEADS & CONTACTS —
│   ├── leads.js
│   ├── contact-lookup.js
│   ├── — NOTIFICATIONS —
│   ├── notifications.js
│   ├── email.js
│   ├── email-templates.js
│   ├── email-normalization.js
│   ├── — MESSAGERIE SOCIALE —
│   ├── whatsapp.js
│   ├── messenger.js
│   ├── tiktok.js
│   ├── social-queue.js          # (si SOCIAL_ENABLED)
│   ├── social-account-resolver.js # (si SOCIAL_ENABLED)
│   └── social/
│       ├── facebook.js
│       ├── instagram.js
│       └── tiktok-publish.js
│   ├── — WORKFLOWS —
│   ├── workflows.js
│   ├── workflow-templates.js
│   ├── workflow-media.js
│   ├── google-drive.js
│   ├── — UTILITAIRES —
│   ├── instructions.js          # Prompt runtime du chatbot (DB-first)
│   ├── prompt-store.js          # Stockage prompt admin DB-first
│   ├── language.js
│   ├── translations.js
│   ├── responses.js             # Réponses pré-définies (tokens d'action)
│   ├── interactive.js           # Boutons HTML interactifs
│   ├── rate-limiter.js
│   ├── jwt.js
│   ├── supabase.js
│   ├── crypto.js
│   ├── allowed-domains.js
│   ├── site-indexer.js
│   ├── news.js
│   ├── order-tracking-parser.js
│   └── company-facts.js
├── public/
│   ├── admin.html               # SPA Admin panel (~600 Ko)
│   ├── chatbot-widget.html      # Widget chatbot embed
│   ├── fullscreen.html          # Chat plein écran
│   └── embed.js                 # Script intégration widget
├── supabase/
│   └── migrations/              # 30 fichiers SQL (voir 02_DATABASE_SCHEMA.md)
├── scripts/
│   ├── ingest-knowledge-v2.mjs  # Ingestion KB → Supabase
│   ├── reindex-knowledge-v2.mjs # Réindexation
│   └── scan-secrets.cjs         # Scan secrets pre-commit
└── tests/                       # 90+ fichiers Jest
```

### ÉTAPE 4 — Remplacer les placeholders

Remplace systématiquement dans TOUT le code généré :
- `[APP_NAME]` → nom de l'application
- `[BRAND_NAME]` → nom de la marque
- `[BRAND_COLOR]` → couleur hex principale
- `[DOMAIN]` → domaine du site
- `[PRODUCT_DOMAIN]` → secteur produits
- `[ADMIN_EMAIL]` → email admin
- `app_` (préfixe tables BDD) → préfixe adapté (ex: `bot_`, `myapp_`)
- `[PREFIX]` → même préfixe dans les migrations SQL

### ÉTAPE 5 — Ordre de création des fichiers

Respecte cet ordre (dépendances) :
1. `package.json` + `.env.example`
2. `supabase/migrations/` (30 fichiers — voir `02_DATABASE_SCHEMA.md`)
3. `lib/supabase.js` + `lib/jwt.js` + `lib/crypto.js`
4. `lib/rate-limiter.js` + `lib/session-store-v2.js`
5. `lib/language.js` + `lib/translations.js` + `lib/allowed-domains.js`
6. `lib/hf-client-v2.js` + `lib/llm-fallback.js`
7. `lib/rag.js` + `lib/retrieval-v2.js` + `lib/memory-engine.js`
8. `lib/entity-memory-v2.js` + `lib/context-manager.js` + `lib/topic-lock.js`
9. `lib/knowledge.js` + `lib/knowledge-sync.js` + `lib/web-knowledge-hybrid.js`
10. Modules sécurité (tous les security-*.js + guard-*.js + meta-agent.js)
11. Modules Wix (si WIX_ENABLED) dans l'ordre : wix-client.js → wix-catalog.js → wix-crm.js → wix-data.js → le reste
12. `lib/leads.js` + `lib/contact-lookup.js` + `lib/orders.js`
13. `lib/email.js` + `lib/email-templates.js` + `lib/notifications.js`
14. `lib/whatsapp.js` + `lib/messenger.js` + `lib/tiktok.js`
15. Wizards : `wizard-utils.js` → wizards individuels → `wizard-state-machine.js`
16. `lib/quotes.js` + `lib/stripe.js`
17. `lib/chat-router.js` + `lib/chat-context-provider.js` + `lib/chat-anon-id.js`
18. `lib/ai-routing.js` + `lib/ai-runtime-state.js`
19. `lib/ai-v3.js` (moteur principal — CŒUR du système)
20. `lib/admin-tools.js` + `lib/admin-ai.js`
21. Agent runtime (si AGENT_ENABLED) : config.js → schemas.js → registry → ledger → gateway → orchestrator
22. Workflows + social (si SOCIAL_ENABLED)
23. `lib/company-facts.js` + `lib/prompt-store.js` + `lib/instructions.js`
24. `api/index.js` (toutes les routes Express)
25. `validate-env.js` + `server.js` + `vercel.json` + `instrument.js`
26. `public/chatbot-widget.html` + `public/embed.js`
27. `public/admin.html` (SPA complète)
28. Scripts d'ingestion KB
29. `tests/` (suite Jest complète)

### ÉTAPE 6 — Points critiques à respecter

**SÉCURITÉ OBLIGATOIRE :**
- Jamais de valeurs hardcodées pour les secrets (toujours `process.env.XXX`)
- Rate limiter fail-closed (bloquer si DB indisponible)
- JWT fail-fast en production (erreur si pas de secret)
- Sanitiser TOUT l'input utilisateur
- Nettoyer tout l'output LLM (CoT `<think>...</think>`, artifacts ENV)
- Blocage opérations destructives Wix (wix-security-guard.js sur axios)

**PATTERNS OBLIGATOIRES :**
- ESM pur (`import/export` natif), jamais de `require()`
- Retry exponentiel sur toutes les APIs externes (baseDelay × 2^attempt)
- Circuit breaker Wix (threshold + open window configurables)
- Cache multi-niveaux (mémoire LRU + Supabase) pour les appels coûteux
- Feature flags via ENV (presque tout activable/désactivable sans redéploiement)
- DB-first pour les prompts, company facts, sessions (tout en Supabase)
- Evidence-based carousel : carousel produits UNIQUEMENT depuis preuves API, jamais depuis texte LLM
- Metadata rich sur les réponses API (`meta.*` pour debugging — non cassant)

**RÉPONSE API STANDARD :**
```json
{
  "reply": "Texte de la réponse",
  "action": {
    "type": "SHOW_CAROUSEL | ADD_TO_CART | START_CHECKOUT | OPEN_URL | SHOW_PDF | null",
    "data": {}
  },
  "meta": {
    "engine": "v3",
    "routing_path": "catalog_tools | wizard_fsm | general_chat | fallback",
    "session_id": "...",
    "language": "fr",
    "guardrails": {},
    "memory": {},
    "behavior": {}
  }
}
```

---

## RÉSUMÉ DES LIVRABLES

Une fois ce skill exécuté, le projet livré doit :
1. ✅ Se déployer sur Vercel avec `vercel --prod` sans erreur
2. ✅ Avoir tous les tests Jest passants
3. ✅ Avoir un chatbot fonctionnel sur `/public/chatbot-widget.html`
4. ✅ Avoir un admin panel complet sur `/public/admin.html`
5. ✅ Avoir toutes les migrations Supabase prêtes à appliquer
6. ✅ Avoir un `.env.example` documenté et complet
7. ✅ Avoir les scripts d'ingestion de la base de connaissances
8. ✅ N'avoir AUCUNE valeur sensible hardcodée

**Temps estimé de génération** : procède module par module, ne saute aucune étape.

---

*Skill généré depuis le blueprint docs/chatbot-blueprint/ — Lire ces fichiers pour les détails complets.*
