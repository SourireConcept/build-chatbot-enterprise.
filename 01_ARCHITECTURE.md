# 01 — ARCHITECTURE : Flux de données et modules

## Vue d'ensemble topologique

```
Internet
   │
[Vercel Edge / CDN]
   │
   ├── /public/*              → Fichiers statiques (admin.html, widget, embed.js)
   └── /api/* → api/index.js  → Express App (tous les endpoints)
                    │
                    ├── Middleware global
                    │   ├── CORS (allowlist domaines)
                    │   ├── JSON body parser (limit 10mb)
                    │   ├── Rate limiter (Supabase RPC atomique)
                    │   └── Sentry error handler
                    │
                    ├── Routes publiques
                    │   ├── POST /api/chat → handleChatRequestV3
                    │   ├── POST /api/webhook/* → handlers WhatsApp/Messenger/TikTok
                    │   └── GET /api/health
                    │
                    ├── Routes admin (JWT requis)
                    │   ├── POST /api/admin/login
                    │   ├── POST /api/admin/chat → handleAdminRequest
                    │   └── GET/POST /api/admin/* → controllers admin
                    │
                    └── Routes cron (CRON_SECRET requis)
                        └── POST /api/cron/* → handlers planifiés
```

---

## Flux 1 : Requête Chatbot (chemin principal)

```
1. Utilisateur envoie message
   POST /api/chat { message, session_id, language, context }

2. Rate Limiter
   → Supabase RPC atomique (10 req/min/session)
   → fail-closed si BDD indisponible

3. Résolution identité (chat-anon-id.js)
   → cookie anon v4 UUID | contactId Wix Chat
   → header x-forwarded-for → pays (Vercel geo)

4. Chat Router (chat-router.js)
   detectEngineReason() → classify:
   - "token"           : message matche un des 15 tokens d'action
   - "wizard"          : session a un wizard actif (trackingState, contactFormState…)
   - "business_intent" : signal d'achat détecté → catalog_tools
   - "conversation"    : discussion générale → general_chat
   - "fallback"        : moteur legacy d'urgence

5. Guardrails PRÉ-traitement (en parallèle)
   - analyzeSensitiveDisclosureRequest() → détection exfiltration prompt/secrets
   - analyzeRoleOverride()               → détection DAN/jailbreak
   - analyzePromptInjection()            → meta-agent shield

6. Moteur V3 (ai-v3.js → handleChatRequestV3)

   a. SESSION STORE V2
      loadSession(session_id) → Supabase sourire_chat_sessions
      → historique 50 tours max (TTL 90 jours)
      → résumé LLM si historique > seuil (compression heuristique/LLM)

   b. ENTITY MEMORY V2
      getEntities(session_id) → nom, email, tel, préférences, intentions
      → cache mémoire LRU (TTL 15min, max 200 entrées)

   c. RAG HYBRIDE
      retrieveChunksV2(message) → BAAI/bge-m3 embeddings → pgvector similarity
      webKnowledgeHybrid(message) → RAG + web knowledge statique
      → cache embeddings (TTL 15min, max 500 entrées)

   d. CONTEXTE PROVIDER
      getChatContext() → company facts DB + prompt runtime DB-first
      → topic lock (verrouillage sujet N tours)

   e. CONTACT LOOKUP
      findContact(email | tel) → Supabase → Wix CRM (timeout 700ms, cache 5min)

   f. BEHAVIOR SCORING (meta-agent)
      computeBehaviorScore() → score 0-100
      detectAbandonRisk() → signaux fuite (concurrent, hésitation, prix)
      selectStrategy() → mode stratégique (éducatif/relance/urgence)

   g. SALES AGENT CADENCE
      Si message #3 → demander prénom (si pas déjà connu)
      Si message #5 → demander email (si pas déjà connu)
      → capture continue sur chaque message

   h. ROUTING PATH (selon reason détecté)

      ┌── "token" ──────────────────────────────────────────────────────┐
      │   responses.js → token lookup direct                            │
      │   ex: __SHOW_CATALOG__ → carousel produits depuis Wix API       │
      └─────────────────────────────────────────────────────────────────┘

      ┌── "wizard_fsm" ─────────────────────────────────────────────────┐
      │   wizard-state-machine.js → getActiveWizard()                   │
      │   → dispatchToWizard(wizard_type, message, session)             │
      │   ex: tracking-wizard → collecte orderId → wix-catalog lookup   │
      └─────────────────────────────────────────────────────────────────┘

      ┌── "catalog_tools" ──────────────────────────────────────────────┐
      │   LLM avec function calling (2 rounds max)                      │
      │   Tools disponibles:                                             │
      │   - search_wix_products(query, category, limit)                 │
      │   - get_product_details(product_id)                             │
      │   - search_wix_catalog(keywords[])                              │
      │   Promise.all() pour appels outils en parallèle                 │
      │   Guard orchestrator V3 POST-LLM                                │
      │   → carousel construit UNIQUEMENT depuis catalogEvidence         │
      └─────────────────────────────────────────────────────────────────┘

      ┌── "general_chat" ───────────────────────────────────────────────┐
      │   hf-client-v2.js → InferenceClient                             │
      │   → prompt enrichi (historique + entités + RAG + facts)         │
      │   → llm-fallback.js si HF échoue                                │
      │   (Perplexity → OpenAI → OpenRouter → xAI)                      │
      └─────────────────────────────────────────────────────────────────┘

      ┌── "fallback" ───────────────────────────────────────────────────┐
      │   ai.js (moteur legacy) → réponse simple sans guardrails avancés │
      └─────────────────────────────────────────────────────────────────┘

   i. GUARD ORCHESTRATOR V3 (post-LLM)
      anti-hallucination executoire :
      → bloque carousel si pas de catalogEvidence
      → bloque prix si pas de preuves API
      → bloque liens si pas dans allowedDomains

   j. OUTPUT SANITIZATION
      → nettoyage <think>...</think> (CoT artifacts)
      → nettoyage ENV leaks (regex sur output)
      → normalisation prix/stock (arrondi, devise)
      → escapeHtml sur fragments HTML

   k. RESPONSE FORMAT
      → injection boutons interactifs (interactive.js)
      → format JSON standard : {reply, action, meta}

7. PERSISTANCE (en parallèle, non-bloquant)
   - saveSession() → historique + résumé
   - upsertEntities() → entités mises à jour
   - extractMemoryFacts() → memory-engine → faits sémantiques
   - syncLead() si email présent → sourire_leads + Wix Data ChatbotContact

8. Retour réponse JSON
   {reply, action, meta} → 200 OK
```

---

## Flux 2 : Admin Chat (God Mode)

```
Admin → admin.html
→ POST /api/admin/chat (JWT requis)
→ handleAdminRequest() (admin-ai.js)

1. Parse message admin
2. LLM Qwen 72B génère JSON strict :
   {
     "thought": "Raisonnement interne",
     "tool": "nom_outil",
     "args": { ... },
     "reply": "Réponse à afficher"
   }
3. Exécution outil (allowlist dynamique depuis agent registry) :
   - read_database(table, filters) → Supabase query
   - execute_sql(sql) → Supabase RPC
   - send_email(to, subject, body) → SMTP (domaines allowlistés)
   - read_emails() → IMAP
   - send_whatsapp(to, message) → Graph API v18
   - wix_search_orders(query) → Wix Orders API
   - wix_update_product(id, data) → Wix Stores API
   - wix_create_coupon(params) → Wix Coupons
   - manage_workflow(action, config) → workflows.js
   - read_file(path) → lecture fichier local
   - list_directory(path) → ls répertoire
   - etc.
4. Log dans app_admin_logs (action + args + result)
5. Retour réponse structurée avec logs admin
```

---

## Flux 3 : Agent Runtime Autonome

```
Admin → Créer objectif (POST /api/admin/agent/objectives)
  { description, tools_allowed, budget: {maxSteps, maxToolCalls, maxDurationMs} }

Admin → Démarrer run (POST /api/admin/agent/objectives/:id/run)

Orchestrator loop (orchestrator.js) :
  BOUCLE jusqu'à done=true ou budget épuisé:

  1. Chargement objectif + config budget
     → résumé run précédent injecté dans contexte

  2. LLM (Qwen 72B) génère étape JSON strict :
     {
       "thought": "Analyse de la situation",
       "tool": "tool_name",
       "args": { ... },
       "done": false,
       "summary": null
     }

  3. Tool Gateway (tool-gateway.js) :
     → validation args (schemas.js → zod)
     → risk check :
       - low  : exécution directe
       - medium : exécution avec log
       - high : pause + notifyAdmin (email + WhatsApp)

  4. executeAgentTool() :
     → timeout selon risk level (low:10s, medium:30s, high:60s)
     → retry exponentiel (max 3 tentatives)
     → résultat → observation injectée dans prochain tour

  5. Ledger (run-ledger.js) :
     → persist step + tool_call + event dans Supabase
     → app_agent_run_steps, app_agent_tool_calls, app_agent_events

  6. Budget check :
     → steps >= maxSteps → finaliser (budget épuisé)
     → duration >= maxDurationMs → finaliser (timeout)
     → done=true → finaliser (objectif atteint)

Admin Approbation (si risk=high) :
  → Notification admin : email + WhatsApp avec détails action
  → Admin: POST /api/admin/agent/approvals/:id/approve | /reject
  → Si approuvé : run reprend avec résumé
  → Si rejeté : run finalisé avec statut REJECTED
```

---

## Flux 4 : Publication Sociale (Autoroute AI Pro)

```
Déclencheur : cron quotidien (09:00) OU admin manuel

1. generateAutoroutePost() (autoroute-proxy.js)
   Chain de fallback texte :
   Perplexity sonar-pro
   → Gemini 2.5 Flash
   → HF GLM-5-Plus ou Qwen
   → HF fallback (liste modèles)

2. generateAutorouteMedia() (autoroute-proxy.js)
   Mode image : HF model packs
   → pack "glm-flux-wan" (défaut)
   → pack "glm-flux-hunyuan"
   → pack "glm-sdxl-wan"
   → pack "glm-sdxl-hunyuan"
   Mode vidéo : HF HunyuanVideo → Wan (fallback)
   Post-processing : QA BLIP captioning + upscale optionnel

3. saveAutorouteCollectionWithRetry()
   → Wix Data : PostsWithMedia collection
   → Supabase : app_social_publication_queue (statut PENDING)

4. publishWixBlogPost() (si mode auto)
   → wix-blog.js → Wix Blog V3

5. Si SOCIAL_APPROVAL_REQUIRED=true :
   → Admin notifié (email + WhatsApp)
   → Admin approuve/rejette dans panel

6. executeSocialQueueDispatch() :
   → runWorkflowsForEvent("autoroute_post_approved", data)
   → workflows.js dispatch :
     ├── Facebook Pages API
     ├── Instagram Business API
     ├── TikTok Content Publishing API
     └── YouTube (via webhook custom)

7. Google Drive upload (si configuré)
   → assets images/vidéos sauvegardés dans Drive
```

---

## Flux 5 : Génération Devis/Facture

```
Utilisateur demande devis → __GENERATE_QUOTE__ token détecté

handleQuoteRequest() (ai/quote-handler.js)
→ generateQuote() (quotes.js)

Cascade devis :
  1. Vérification prix via Wix API (anti-tampering, prix côté serveur)
  2. Si items ont productId → Wix Checkout
     → wix-checkout.js → createWixCheckout()
     → taxes + livraison calculées par Wix
     → lien checkout Wix (TTL 1h)
  3. Si Wix Checkout échoue ou items custom → Wix Invoice
     → wix-invoices.js → createWixInvoice()
     → lien facture Wix (PDF + paiement en ligne)
  4. Dernier recours → Stripe Checkout
     → stripe.js → createStripeSession()
     → vérification prix côté Wix (anti-tampering)

PDF local (si demandé) :
  → PDFKit → génération PDF (logo, couleurs marque)
  → Supabase Storage → upload
  → URL signée retournée

Persistance :
  → app_quotes (référence devis)
  → app_wix_invoices (si invoice Wix)

Notifications :
  → notifyAdmin() : email + WhatsApp (nouveau devis)
  → notifyCustomer() : email brandé avec lien paiement
```

---

## Modules clés et leurs interactions

```
api/index.js
  └── chat route
        ├── rate-limiter.js     (protection)
        ├── chat-anon-id.js     (identité)
        ├── chat-router.js      (classification)
        ├── meta-agent.js       (guardrails injection)
        └── ai-v3.js            (moteur principal)
              ├── session-store-v2.js   (mémoire court terme)
              ├── entity-memory-v2.js   (entités nommées)
              ├── context-manager.js    (contexte long terme)
              ├── retrieval-v2.js       (RAG)
              ├── web-knowledge-hybrid.js (RAG + web)
              ├── chat-context-provider.js (prompt DB)
              ├── topic-lock.js         (verrouillage sujet)
              ├── contact-lookup.js     (CRM lookup)
              ├── hf-client-v2.js       (LLM HF)
              ├── llm-fallback.js       (chain fallback)
              ├── wix-tools-v3.js       (function calling)
              ├── wix-catalog.js        (catalogue)
              ├── guard-orchestrator-v3.js (anti-hallucination)
              ├── domain-scope-guard.js (scope)
              ├── security-disclosure-guard.js (exfiltration)
              ├── company-identity-guard.js (marques)
              ├── interactive.js        (boutons HTML)
              ├── leads.js              (sync leads)
              └── memory-engine.js      (extraction faits)
```

---

## Conventions API

### Requête entrante
```json
{
  "message": "string (required)",
  "session_id": "string (optional, généré si absent)",
  "language": "fr|en|de|ar (optional, auto-détecté)",
  "context": {
    "page_url": "string",
    "page_title": "string",
    "cart_items": [],
    "contact_id": "string (Wix Chat)"
  }
}
```

### Réponse standard
```json
{
  "reply": "Texte de la réponse en Markdown",
  "action": {
    "type": "SHOW_CAROUSEL|ADD_TO_WIX_CART|START_WIX_CHECKOUT|OPEN_URL|SHOW_QUOTE_PDF|null",
    "data": {
      "products": [],
      "url": "string",
      "pdf_url": "string"
    }
  },
  "meta": {
    "engine": "v3|legacy|v2",
    "routing_path": "catalog_tools|wizard_fsm|general_chat|fallback|token",
    "session_id": "uuid",
    "language": "fr",
    "message_count": 5,
    "guardrails": {
      "disclosure_blocked": false,
      "role_override_blocked": false,
      "injection_detected": false,
      "scope_classification": "in_scope"
    },
    "memory": {
      "entities_found": ["nom", "email"],
      "rag_chunks_used": 3,
      "session_turns": 12
    },
    "behavior": {
      "score": 72,
      "abandon_risk": false,
      "strategy": "relance_douce",
      "profile": "PriceSensitive"
    },
    "wix": {
      "catalog_evidence": true,
      "products_found": 4
    }
  }
}
```

### Tokens d'action (15 tokens déterministes)
```javascript
const ACTION_TOKENS = {
  '__SHOW_CATALOG__': 'Afficher le catalogue produits',
  '__GENERATE_QUOTE__': 'Générer un devis',
  '__TRACK_ORDER__': 'Suivi de commande (start wizard)',
  '__CONTACT_FORM__': 'Formulaire de contact (start wizard)',
  '__GET_COUPON__': 'Obtenir un code promo (start wizard)',
  '__CUSTOMS_HELP__': 'Aide douanes/EORI',
  '__START_EORI__': 'Démarrer wizard EORI',
  '__START_CHECKOUT__': 'Démarrer checkout address wizard',
  '__BOOK_SERVICE__': 'Réservation service',
  '__SHOW_BOOKINGS__': 'Voir les disponibilités',
  '__SHOW_PROMOTIONS__': 'Afficher les promotions',
  '__SHOW_BLOG__': 'Articles/actualités',
  '__TRUSTPILOT_REVIEW__': 'Inviter à laisser un avis',
  '__SALES_AGENT_PROACTIVE__': 'Trigger message commercial proactif',
  '__RESET_CHAT__': 'Réinitialiser la conversation'
};
```

---

## Gestion des erreurs

### Stratégie globale
1. **Erreur API externe** : retry exponentiel → fallback → log + réponse gracieuse
2. **Erreur BDD** : retry 1x → fail gracieux (réponse sans mémoire si session)
3. **Erreur LLM** : provider fallback chain → legacy engine → message d'erreur poli
4. **Erreur auth** : 401 avec message clair (pas de détails internes)
5. **Rate limit** : 429 avec `Retry-After` header
6. **Erreur interne** : Sentry capture → 500 avec message générique (jamais de stack trace en prod)

### Circuit breaker Wix
```javascript
// État interne du circuit breaker
let cbState = 'closed'; // closed | open | half-open
let failures = 0;
const CB_THRESHOLD = parseInt(process.env.WIX_CB_FAILURE_THRESHOLD || '5');
const CB_OPEN_MS = parseInt(process.env.WIX_CB_OPEN_MS || '30000');

// Sur chaque appel Wix :
// - Succès → reset failures
// - Échec → failures++; si failures >= CB_THRESHOLD → ouvrir le circuit
// - Circuit ouvert → retourner erreur immédiatement (pas d'appel API)
// - Après CB_OPEN_MS → half-open (1 essai test)
```
