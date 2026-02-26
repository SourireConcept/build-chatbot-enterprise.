# 04 — MOTEUR IA V3 : Implémentation complète

## Vue d'ensemble du moteur

Le moteur V3 (`lib/ai-v3.js`) est le cœur du chatbot. Il orchestre :
- La mémoire (session + entités + RAG)
- Le routing intelligent (tokens / wizards / catalog / général)
- Le function calling pour le catalogue
- Les guardrails pre/post LLM
- La persistance asynchrone

**Taille typique** : ~150 Ko de code ESM, ~3000-4000 lignes

---

## Structure du fichier `lib/ai-v3.js`

```javascript
// lib/ai-v3.js
// Moteur principal chatbot V3
// Version: 4.3 — ESM pur

import { createClient } from '@supabase/supabase-js';
import { loadSession, saveSession } from './session-store-v2.js';
import { getEntities, upsertEntity } from './entity-memory-v2.js';
import { retrieveChunksV2 } from './retrieval-v2.js';
import { webKnowledgeHybrid } from './web-knowledge-hybrid.js';
import { getChatContext } from './chat-context-provider.js';
import { getTopicLock, updateTopicLock } from './topic-lock.js';
import { findContact } from './contact-lookup.js';
import { computeBehaviorScore, detectAbandonRisk } from './meta-agent.js';
import { generateWithHF } from './hf-client-v2.js';
import { generateWithFallback } from './llm-fallback.js';
import { callWixTools } from './wix-tools-v3.js';
import { guardOrchestratorV3 } from './guard-orchestrator-v3.js';
import { analyzeSensitiveDisclosure } from './security-disclosure-guard.js';
import { analyzeRoleOverride } from './meta-agent.js';
import { classifyDomainScope } from './domain-scope-guard.js';
import { syncLead } from './leads.js';
import { extractMemoryFacts } from './memory-engine.js';
import { formatInteractiveButtons } from './interactive.js';
import { detectLanguage } from './language.js';
import { getResponse } from './responses.js';
import { dispatchToWizard, getActiveWizard } from './ai/wizard-state-machine.js';
import { getRuntimeState } from './ai-runtime-state.js';
import { getInstructions } from './instructions.js';

// ────────────────────────────────────────────────
// CONSTANTE : Tokens d'action déterministes
// ────────────────────────────────────────────────
const ACTION_TOKENS = new Set([
  '__SHOW_CATALOG__',
  '__GENERATE_QUOTE__',
  '__TRACK_ORDER__',
  '__CONTACT_FORM__',
  '__GET_COUPON__',
  '__CUSTOMS_HELP__',
  '__START_EORI__',
  '__START_CHECKOUT__',
  '__BOOK_SERVICE__',
  '__SHOW_BOOKINGS__',
  '__SHOW_PROMOTIONS__',
  '__SHOW_BLOG__',
  '__TRUSTPILOT_REVIEW__',
  '__SALES_AGENT_PROACTIVE__',
  '__RESET_CHAT__',
]);

// ────────────────────────────────────────────────
// EXPORT PRINCIPAL
// ────────────────────────────────────────────────
export async function handleChatRequestV3(req, res) {
  const startTime = Date.now();

  // 1. Extraction paramètres
  const {
    message = '',
    session_id: rawSessionId,
    language: langHint,
    context: clientContext = {},
  } = req.body;

  // Normalisation
  const sessionId = rawSessionId || generateSessionId();
  const language = langHint || detectLanguage(message) || 'fr';

  try {
    // 2. Guard pré-traitement (en parallèle)
    const [disclosureCheck, roleCheck, scopeCheck] = await Promise.allSettled([
      analyzeSensitiveDisclosure(message),
      analyzeRoleOverride(message),
      classifyDomainScope(message),
    ]);

    // Extraire résultats
    const disclosure = disclosureCheck.value || { blocked: false };
    const roleOverride = roleCheck.value || { blocked: false };
    const scope = scopeCheck.value || { classification: 'in_scope' };

    if (disclosure.blocked) {
      return res.json({
        reply: getResponse('disclosure_blocked', language),
        action: null,
        meta: { engine: 'v3', guardrails: { disclosure_blocked: true } },
      });
    }

    if (roleOverride.blocked) {
      return res.json({
        reply: getResponse('role_override_blocked', language),
        action: null,
        meta: { engine: 'v3', guardrails: { role_override_blocked: true } },
      });
    }

    // 3. Charger session + entités (en parallèle)
    const [session, entities] = await Promise.all([
      loadSession(sessionId),
      getEntities(sessionId),
    ]);

    const messageCount = (session.message_count || 0) + 1;

    // 4. RAG + Contexte (en parallèle)
    const [ragChunks, chatContext] = await Promise.all([
      retrieveChunksV2(message).catch(() => []),
      getChatContext(sessionId, language),
    ]);

    // 5. Contact lookup (non-bloquant, timeout 700ms)
    const contact = await Promise.race([
      findContact({ email: entities.email, phone: entities.phone }),
      new Promise(resolve => setTimeout(() => resolve(null), 700)),
    ]);

    // 6. Behavior scoring
    const [behaviorScore, abandonRisk] = await Promise.all([
      computeBehaviorScore(session, entities, clientContext),
      detectAbandonRisk(message, session),
    ]);

    // 7. Runtime state (feature flags)
    const runtimeState = await getRuntimeState();

    // 8. Sales agent cadence
    const salesSignal = computeSalesAgentCadence(messageCount, entities);

    // 9. Déterminer routing path
    const routingPath = determineRoutingPath(message, session, runtimeState);

    // 10. Exécuter selon le path
    let reply = '';
    let action = null;
    let catalogEvidence = null;

    switch (routingPath) {
      case 'token': {
        const tokenResponse = handleActionToken(message, language, entities, session);
        reply = tokenResponse.reply;
        action = tokenResponse.action;
        break;
      }

      case 'wizard_fsm': {
        const wizardResult = await dispatchToWizard(message, session, language);
        reply = wizardResult.reply;
        action = wizardResult.action;
        break;
      }

      case 'catalog_tools': {
        const toolResult = await runCatalogToolCalling(
          message, session, entities, ragChunks, chatContext, language
        );
        reply = toolResult.reply;
        action = toolResult.action;
        catalogEvidence = toolResult.evidence;
        break;
      }

      default: // general_chat
      {
        const chatResult = await runGeneralChat(
          message, session, entities, ragChunks, chatContext, language,
          salesSignal, scope, behaviorScore
        );
        reply = chatResult.reply;
        action = chatResult.action;
        break;
      }
    }

    // 11. Guard orchestrator post-LLM
    const guardedReply = await guardOrchestratorV3(reply, action, catalogEvidence, language);
    reply = guardedReply.reply;
    action = guardedReply.action;

    // 12. Sanitisation output
    reply = sanitizeOutput(reply);

    // 13. Injection boutons interactifs
    reply = formatInteractiveButtons(reply, language);

    // 14. Persistance asynchrone (non-bloquant)
    persistAsync(sessionId, message, reply, entities, session, language);

    // 15. Sync lead si email présent
    if (entities.email) {
      syncLead({
        session_id: sessionId,
        email: entities.email,
        name: entities.name,
        phone: entities.phone,
        source: 'chatbot_v3',
      }).catch(console.error);
    }

    // 16. Réponse
    return res.json({
      reply,
      action,
      meta: buildMeta({
        sessionId, language, messageCount, routingPath,
        disclosure, roleOverride, scope,
        behaviorScore, abandonRisk,
        ragChunks, catalogEvidence,
        startTime,
      }),
    });

  } catch (error) {
    console.error('[ai-v3] Erreur critique:', error);
    return res.status(500).json({
      reply: getResponse('error_generic', language),
      action: null,
      meta: { engine: 'v3', error: process.env.NODE_ENV !== 'production' ? error.message : undefined },
    });
  }
}

// ────────────────────────────────────────────────
// ROUTING PATH
// ────────────────────────────────────────────────
function determineRoutingPath(message, session, runtimeState) {
  // Kill switch → legacy
  if (runtimeState.v3KillSwitch) return 'fallback';

  // Token déterministe
  const trimmed = message.trim().toUpperCase();
  if (ACTION_TOKENS.has(trimmed)) return 'token';

  // Wizard actif en session
  if (getActiveWizard(session)) return 'wizard_fsm';

  // Signal business (catalog)
  if (hasCatalogIntent(message)) return 'catalog_tools';

  // Chat général
  return 'general_chat';
}

// ────────────────────────────────────────────────
// FUNCTION CALLING CATALOGUE
// ────────────────────────────────────────────────
async function runCatalogToolCalling(message, session, entities, ragChunks, chatContext, language) {
  // Définition des tools pour function calling
  const tools = [
    {
      type: 'function',
      function: {
        name: 'search_wix_products',
        description: 'Rechercher des produits dans le catalogue par texte libre',
        parameters: {
          type: 'object',
          properties: {
            query: { type: 'string', description: 'Texte de recherche' },
            category: { type: 'string', description: 'Catégorie produit (optionnel)' },
            limit: { type: 'integer', default: 4, maximum: 8 },
          },
          required: ['query'],
        },
      },
    },
    {
      type: 'function',
      function: {
        name: 'get_product_details',
        description: 'Obtenir les détails complets d\'un produit par son ID',
        parameters: {
          type: 'object',
          properties: {
            product_id: { type: 'string' },
          },
          required: ['product_id'],
        },
      },
    },
  ];

  // Prompt enrichi pour le function calling
  const systemPrompt = buildSystemPrompt(chatContext, entities, ragChunks, language);
  const history = formatHistory(session.messages?.slice(-10) || []);

  // Round 1 : LLM décide quels outils appeler
  let llmResponse = await generateWithHF({
    model: process.env.HF_CHAT_MODEL_V3,
    messages: [
      { role: 'system', content: systemPrompt },
      ...history,
      { role: 'user', content: message },
    ],
    tools,
    tool_choice: 'auto',
    max_tokens: 1024,
    temperature: 0.7,
  });

  // Exécuter les tool calls en parallèle
  const toolCalls = llmResponse.tool_calls || [];
  if (toolCalls.length > 0) {
    const toolResults = await Promise.all(
      toolCalls.map(tc => callWixTools(tc.function.name, JSON.parse(tc.function.arguments)))
    );

    // Collecter les preuves catalogue (pour guard orchestrator)
    const evidence = toolResults.flatMap(r => r.products || []);

    // Round 2 : LLM génère la réponse finale avec les résultats
    const finalMessages = [
      { role: 'system', content: systemPrompt },
      ...history,
      { role: 'user', content: message },
      { role: 'assistant', content: '', tool_calls: toolCalls },
      ...toolResults.map((result, i) => ({
        role: 'tool',
        tool_call_id: toolCalls[i].id,
        content: JSON.stringify(result),
      })),
    ];

    llmResponse = await generateWithHF({
      model: process.env.HF_CHAT_MODEL_V3,
      messages: finalMessages,
      max_tokens: 1024,
      temperature: 0.7,
    });

    return {
      reply: llmResponse.content || llmResponse,
      action: evidence.length > 0 ? { type: 'SHOW_CAROUSEL', data: { products: evidence.slice(0, 4) } } : null,
      evidence,
    };
  }

  return {
    reply: llmResponse.content || llmResponse,
    action: null,
    evidence: [],
  };
}

// ────────────────────────────────────────────────
// CHAT GÉNÉRAL
// ────────────────────────────────────────────────
async function runGeneralChat(message, session, entities, ragChunks, chatContext, language, salesSignal, scope, behaviorScore) {
  const systemPrompt = buildSystemPrompt(chatContext, entities, ragChunks, language, salesSignal, scope, behaviorScore);
  const history = formatHistory(session.messages?.slice(-20) || []);

  try {
    const response = await generateWithHF({
      model: process.env.HF_CHAT_MODEL_V3,
      messages: [
        { role: 'system', content: systemPrompt },
        ...history,
        { role: 'user', content: message },
      ],
      max_tokens: 512,
      temperature: 0.8,
    });

    return {
      reply: response.content || response,
      action: null,
    };
  } catch (hfError) {
    // Fallback chain
    const fallbackResponse = await generateWithFallback({
      messages: [
        { role: 'system', content: systemPrompt },
        ...history,
        { role: 'user', content: message },
      ],
      max_tokens: 512,
    });

    return {
      reply: fallbackResponse.content || fallbackResponse,
      action: null,
    };
  }
}

// ────────────────────────────────────────────────
// CONSTRUCTION PROMPT SYSTÈME
// ────────────────────────────────────────────────
function buildSystemPrompt(chatContext, entities, ragChunks, language, salesSignal, scope, behaviorScore) {
  const sections = [];

  // Instructions principales (DB-first)
  sections.push(chatContext.instructions || process.env.FALLBACK_INSTRUCTIONS || 'Tu es un assistant IA helpful.');

  // Contexte entreprise
  if (chatContext.companyFacts?.length > 0) {
    sections.push('\n## Informations entreprise\n' + chatContext.companyFacts.map(f => `- ${f.key}: ${f.value}`).join('\n'));
  }

  // Profil utilisateur (entités)
  if (entities && Object.keys(entities).some(k => entities[k])) {
    const entityLines = [];
    if (entities.name) entityLines.push(`- Nom: ${entities.name}`);
    if (entities.email) entityLines.push(`- Email: ${entities.email}`);
    if (entities.phone) entityLines.push(`- Téléphone: ${entities.phone}`);
    if (entities.preferences?.length) entityLines.push(`- Préférences: ${entities.preferences.join(', ')}`);
    if (entityLines.length > 0) {
      sections.push('\n## Profil client connu\n' + entityLines.join('\n'));
    }
  }

  // Knowledge RAG
  if (ragChunks?.length > 0) {
    sections.push('\n## Base de connaissances pertinente\n' + ragChunks.map(c => c.content).join('\n\n'));
  }

  // Signal commercial
  if (salesSignal) {
    sections.push('\n## Instruction commerciale\n' + salesSignal);
  }

  // Scope guard
  if (scope?.classification === 'out_scope') {
    sections.push('\n## IMPORTANT: Le sujet semble hors de ton domaine. Ramène doucement la conversation sur les produits et services.');
  }

  // Langue
  sections.push(`\n## Langue: Réponds TOUJOURS en ${language === 'fr' ? 'français' : language === 'en' ? 'anglais' : language === 'de' ? 'allemand' : 'arabe'}.`);

  return sections.join('\n');
}

// ────────────────────────────────────────────────
// SANITISATION OUTPUT
// ────────────────────────────────────────────────
function sanitizeOutput(text) {
  if (!text || typeof text !== 'string') return '';

  // Supprimer les CoT artifacts (<think>...</think>)
  text = text.replace(/<think>[\s\S]*?<\/think>/gi, '').trim();

  // Supprimer les leaks ENV (patterns comme VARIABLE_NAME=valeur)
  text = text.replace(/[A-Z_]{5,}=[^\s]+/g, '[REDACTED]');

  // Supprimer les tokens d'action dans le texte visible
  text = text.replace(/__[A-Z_]+__/g, '');

  // Nettoyer les espaces multiples
  text = text.replace(/\n{3,}/g, '\n\n').trim();

  return text;
}

// ────────────────────────────────────────────────
// SALES AGENT CADENCE
// ────────────────────────────────────────────────
function computeSalesAgentCadence(messageCount, entities) {
  // Message #3 : demander le prénom si pas connu
  if (messageCount === 3 && !entities.name) {
    return 'Au fil de la conversation, si l\'occasion se présente naturellement, tu peux demander le prénom du client.';
  }
  // Message #5 : demander l'email si pas connu
  if (messageCount === 5 && !entities.email) {
    return 'Si approprié, tu peux proposer d\'envoyer des informations par email et demander l\'adresse email du client.';
  }
  return null;
}

// ────────────────────────────────────────────────
// HELPERS
// ────────────────────────────────────────────────
function generateSessionId() {
  return `anon_${Date.now()}_${Math.random().toString(36).slice(2, 9)}`;
}

function hasCatalogIntent(message) {
  const catalogKeywords = [
    'produit', 'article', 'cherche', 'trouve', 'prix', 'acheter', 'commander',
    'stock', 'disponible', 'catalogue', 'gamme', 'modèle', 'référence',
    'product', 'price', 'buy', 'order', 'stock', 'available',
  ];
  const lower = message.toLowerCase();
  return catalogKeywords.some(k => lower.includes(k));
}

function formatHistory(messages) {
  return messages.map(m => ({
    role: m.role === 'user' ? 'user' : 'assistant',
    content: m.content || '',
  }));
}

function buildMeta({ sessionId, language, messageCount, routingPath, disclosure, roleOverride, scope, behaviorScore, abandonRisk, ragChunks, catalogEvidence, startTime }) {
  return {
    engine: 'v3',
    routing_path: routingPath,
    session_id: sessionId,
    language,
    message_count: messageCount,
    duration_ms: Date.now() - startTime,
    guardrails: {
      disclosure_blocked: disclosure?.blocked || false,
      role_override_blocked: roleOverride?.blocked || false,
      scope_classification: scope?.classification || 'in_scope',
    },
    memory: {
      rag_chunks_used: ragChunks?.length || 0,
    },
    behavior: {
      score: behaviorScore || 0,
      abandon_risk: abandonRisk || false,
    },
    wix: {
      catalog_evidence: catalogEvidence ? catalogEvidence.length > 0 : false,
      products_found: catalogEvidence?.length || 0,
    },
  };
}

async function persistAsync(sessionId, userMessage, assistantReply, entities, session, language) {
  try {
    const updatedMessages = [
      ...(session.messages || []),
      { role: 'user', content: userMessage, timestamp: Date.now() },
      { role: 'assistant', content: assistantReply, timestamp: Date.now() },
    ];

    // Limiter à CHAT_HISTORY_MAX_TURNS tours
    const maxTurns = parseInt(process.env.CHAT_HISTORY_MAX_TURNS || '50') * 2;
    const trimmedMessages = updatedMessages.slice(-maxTurns);

    await saveSession(sessionId, trimmedMessages, language);

    // Extraction entités depuis le message utilisateur
    const newEntities = await extractEntitiesFromMessage(userMessage);
    if (newEntities) {
      for (const [type, value] of Object.entries(newEntities)) {
        await upsertEntity(sessionId, type, value);
      }
    }

    // Extraction faits mémoire
    await extractMemoryFacts(sessionId, userMessage, assistantReply).catch(() => {});

  } catch (error) {
    console.error('[ai-v3] Erreur persistAsync:', error.message);
    // Non-bloquant, on ignore l'erreur
  }
}

// Extraction basique d'entités sans LLM (regex)
function extractEntitiesFromMessage(message) {
  const entities = {};

  // Email
  const emailMatch = message.match(/\b[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}\b/);
  if (emailMatch) entities.email = emailMatch[0].toLowerCase();

  // Téléphone (format international + formats courants)
  const phoneMatch = message.match(/(?:\+\d{1,3}[-.\s]?)?\(?\d{2,4}\)?[-.\s]?\d{2,4}[-.\s]?\d{2,4}[-.\s]?\d{0,4}/);
  if (phoneMatch?.[0]?.replace(/\D/g, '').length >= 9) {
    entities.phone = phoneMatch[0].trim();
  }

  // Prénom (heuristique basique)
  const nameMatch = message.match(/(?:je m'appelle|je suis|c'est|my name is|I'm)\s+([A-ZÀ-Ü][a-zà-ü]+)/i);
  if (nameMatch) entities.name = nameMatch[1];

  return Object.keys(entities).length > 0 ? entities : null;
}
```

---

## Module `lib/chat-router.js`

```javascript
// lib/chat-router.js
// Détection du moteur et du chemin de routing

import { getActiveWizard } from './ai/wizard-state-machine.js';

export function detectEngineReason(message, session) {
  const ACTION_TOKENS = new Set([
    '__SHOW_CATALOG__', '__GENERATE_QUOTE__', '__TRACK_ORDER__',
    '__CONTACT_FORM__', '__GET_COUPON__', '__CUSTOMS_HELP__',
    '__START_EORI__', '__START_CHECKOUT__', '__BOOK_SERVICE__',
    '__SHOW_BOOKINGS__', '__SHOW_PROMOTIONS__', '__SHOW_BLOG__',
    '__TRUSTPILOT_REVIEW__', '__SALES_AGENT_PROACTIVE__', '__RESET_CHAT__',
  ]);

  const trimmed = message.trim().toUpperCase();

  // 1. Token déterministe
  if (ACTION_TOKENS.has(trimmed)) return 'token';

  // 2. Wizard actif
  if (session && getActiveWizard(session)) return 'wizard';

  // 3. Intent commercial (catalog)
  if (hasCatalogIntent(message)) return 'business_intent';

  // 4. Chat général
  return 'conversation';
}

function hasCatalogIntent(message) {
  const FR = ['produit', 'article', 'prix', 'acheter', 'commander', 'stock', 'disponible', 'catalogue', 'gamme'];
  const EN = ['product', 'price', 'buy', 'order', 'stock', 'available', 'catalog'];
  const keywords = [...FR, ...EN];
  const lower = message.toLowerCase();
  return keywords.some(k => lower.includes(k));
}
```

---

## Module `lib/hf-client-v2.js`

```javascript
// lib/hf-client-v2.js
// Client HuggingFace Inference avec retry et function calling

import { InferenceClient } from '@huggingface/inference';

let client = null;

function getClient() {
  if (!client) {
    client = new InferenceClient(process.env.HF_API_TOKEN);
  }
  return client;
}

export async function generateWithHF({
  model,
  messages,
  tools,
  tool_choice,
  max_tokens = 512,
  temperature = 0.7,
  provider,
  retries = 3,
  baseDelay = 500,
}) {
  const hfClient = getClient();
  const inferenceProvider = provider || process.env.HF_INFERENCE_PROVIDER || 'nebius';

  for (let attempt = 0; attempt < retries; attempt++) {
    try {
      const response = await hfClient.chatCompletion({
        model,
        messages,
        tools,
        tool_choice,
        max_tokens,
        temperature,
        provider: inferenceProvider,
      });

      return response.choices[0].message;

    } catch (error) {
      const isRetryable = error.status === 429 || error.status >= 500 || error.code === 'ECONNRESET';

      if (!isRetryable || attempt === retries - 1) {
        throw error;
      }

      const delay = Math.min(baseDelay * Math.pow(2, attempt), 2000);
      await new Promise(r => setTimeout(r, delay));
    }
  }
}

export async function generateEmbedding(text, model, dimension) {
  const hfClient = getClient();
  const embModel = model || process.env.HF_EMBED_MODEL_V2 || 'BAAI/bge-m3';
  const dim = dimension || parseInt(process.env.HF_EMBED_DIM_V2 || '1024');

  const response = await hfClient.featureExtraction({
    model: embModel,
    inputs: text,
    parameters: { normalize: true },
  });

  // Normaliser la sortie (parfois tableau de tableaux)
  const embedding = Array.isArray(response[0]) ? response[0] : response;

  // Tronquer/padder à la dimension attendue
  const result = new Array(dim).fill(0);
  for (let i = 0; i < Math.min(embedding.length, dim); i++) {
    result[i] = embedding[i];
  }

  return result;
}
```

---

## Module `lib/llm-fallback.js`

```javascript
// lib/llm-fallback.js
// Chain de fallback LLM : Perplexity → OpenAI → OpenRouter → xAI

export async function generateWithFallback({ messages, max_tokens = 512, temperature = 0.7 }) {
  const providers = (process.env.CHAT_FALLBACK_PROVIDERS || 'perplexity,openai,openrouter,xai').split(',');

  for (const provider of providers) {
    try {
      switch (provider.trim()) {
        case 'perplexity': {
          if (!process.env.PERPLEXITY_API_KEY) continue;
          return await callOpenAICompatible({
            baseUrl: 'https://api.perplexity.ai',
            apiKey: process.env.PERPLEXITY_API_KEY,
            model: process.env.PERPLEXITY_MODEL || 'llama-3.1-sonar-large-128k-online',
            messages, max_tokens, temperature,
          });
        }
        case 'openai': {
          if (!process.env.OPENAI_API_KEY) continue;
          return await callOpenAICompatible({
            baseUrl: process.env.OPENAI_BASE_URL || 'https://api.openai.com/v1',
            apiKey: process.env.OPENAI_API_KEY,
            model: process.env.OPENAI_MODEL || 'gpt-4o-mini',
            messages, max_tokens, temperature,
          });
        }
        case 'openrouter': {
          if (!process.env.OPENROUTER_API_KEY) continue;
          return await callOpenAICompatible({
            baseUrl: process.env.OPENROUTER_BASE_URL || 'https://openrouter.ai/api/v1',
            apiKey: process.env.OPENROUTER_API_KEY,
            model: process.env.OPENROUTER_MODEL || 'anthropic/claude-3-haiku',
            messages, max_tokens, temperature,
          });
        }
        case 'xai': {
          if (!process.env.XAI_API_KEY) continue;
          return await callOpenAICompatible({
            baseUrl: process.env.XAI_BASE_URL || 'https://api.x.ai/v1',
            apiKey: process.env.XAI_API_KEY,
            model: process.env.XAI_MODEL || 'grok-beta',
            messages, max_tokens, temperature,
          });
        }
      }
    } catch (error) {
      console.warn(`[llm-fallback] Provider ${provider} échoué:`, error.message);
      // Continuer vers le prochain provider
    }
  }

  // Tous les providers ont échoué
  throw new Error('Tous les providers LLM ont échoué');
}

async function callOpenAICompatible({ baseUrl, apiKey, model, messages, max_tokens, temperature }) {
  const response = await fetch(`${baseUrl}/chat/completions`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${apiKey}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({ model, messages, max_tokens, temperature }),
  });

  if (!response.ok) {
    const error = await response.text();
    throw new Error(`HTTP ${response.status}: ${error}`);
  }

  const data = await response.json();
  return { content: data.choices[0].message.content };
}
```

---

## Module `lib/session-store-v2.js`

```javascript
// lib/session-store-v2.js
// Session store persistant (Supabase) avec TTL et compression

import { supabase } from './supabase.js';

const TABLE = 'app_chat_sessions';
const TTL_DAYS = parseInt(process.env.CHAT_SESSION_TTL_DAYS || '90');

export async function loadSession(sessionId) {
  const { data, error } = await supabase
    .from(TABLE)
    .select('*')
    .eq('session_id', sessionId)
    .single();

  if (error || !data) {
    return { session_id: sessionId, messages: [], message_count: 0 };
  }

  return data;
}

export async function saveSession(sessionId, messages, language = 'fr') {
  const messageCount = messages.filter(m => m.role === 'user').length;

  const { error } = await supabase
    .from(TABLE)
    .upsert({
      session_id: sessionId,
      messages,
      message_count: messageCount,
      metadata: { language, last_updated: Date.now() },
      updated_at: new Date().toISOString(),
    }, { onConflict: 'session_id' });

  if (error) {
    console.error('[session-store] Erreur sauvegarde:', error.message);
  }
}

export async function resetSession(sessionId) {
  const { error } = await supabase.rpc('reset_chat_session', {
    p_session_id: sessionId,
  });

  if (error) {
    console.error('[session-store] Erreur reset:', error.message);
  }
}
```

---

## Module `lib/memory-engine.js`

```javascript
// lib/memory-engine.js
// Extraction de faits sémantiques depuis la conversation

import { generateWithHF } from './hf-client-v2.js';
import { supabase } from './supabase.js';

const MEMORY_CATEGORIES = [
  'preference',   // Ce que le client aime/n'aime pas
  'intent',       // Ce que le client veut faire
  'sentiment',    // Ton émotionnel
  'complaint',    // Plaintes/insatisfactions
  'purchase',     // Informations d'achat passées
  'context',      // Contexte situation du client
];

export async function extractMemoryFacts(sessionId, userMessage, assistantReply) {
  if (!sessionId || !userMessage) return;

  try {
    const model = process.env.HF_MEMORY_MODEL || process.env.HF_MODEL_FAST;
    const maxTokens = parseInt(process.env.HF_MEMORY_MAX_TOKENS || '512');
    const temperature = parseFloat(process.env.HF_MEMORY_TEMPERATURE || '0.3');

    const prompt = `Extrait les faits importants de cet échange client.
Réponds UNIQUEMENT avec un JSON valide ou null si rien d'important.

Message client: "${userMessage}"
Réponse assistant: "${assistantReply.slice(0, 200)}"

JSON attendu (ou null):
{"category": "preference|intent|complaint|context", "fact": "fait extrait court", "confidence": 0.0-1.0}`;

    const response = await generateWithHF({
      model,
      messages: [{ role: 'user', content: prompt }],
      max_tokens: maxTokens,
      temperature,
    });

    const content = response.content || '';
    const jsonMatch = content.match(/\{[\s\S]*\}/);

    if (jsonMatch) {
      const fact = JSON.parse(jsonMatch[0]);
      if (fact.category && fact.fact && fact.confidence > 0.6) {
        await supabase.from('app_entity_memory').upsert({
          session_id: sessionId,
          entity_type: fact.category,
          entity_value: fact.fact,
          confidence: fact.confidence,
          source_message: userMessage.slice(0, 100),
        }, { onConflict: 'session_id,entity_type,entity_value' });
      }
    }
  } catch (error) {
    // Non-bloquant
    console.warn('[memory-engine] Extraction échouée:', error.message);
  }
}
```
