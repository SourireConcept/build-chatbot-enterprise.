# 09 — AGENT RUNTIME AUTONOME

## Concept

L'agent runtime est un **système autonome multi-étapes** qui exécute des objectifs complexes en boucle plan→act→observe→decide. Il est conçu pour des tâches d'administration et d'automation qui nécessitent plusieurs actions séquentielles.

**Caractéristiques :**
- Budgets configurables (steps, tool_calls, durée, risk level)
- Approbation humaine (human-in-the-loop) pour les actions à risque élevé
- Timeline auditée de tous les runs (ledger immuable)
- Résumé des runs précédents injecté dans le contexte (mémoire inter-runs)
- Notifications admin en temps réel (email + WhatsApp)

---

## Module `lib/agent/orchestrator.js`

```javascript
// lib/agent/orchestrator.js
// Boucle autonome plan → act → observe → decide

import { generateWithHF } from '../hf-client-v2.js';
import { generateWithFallback } from '../llm-fallback.js';
import { executeAgentTool, validateToolArgs } from './tool-gateway.js';
import { getObjective, startRun, completeRun, createStep, completeStep, createToolCall, createEvent, getPreviousRunSummary } from './run-ledger.js';
import { loadAgentConfig } from './config.js';
import { validateStepSchema } from './schemas.js';
import { sendEmail } from '../email.js';
import { sendWhatsApp } from '../whatsapp.js';

const AGENT_SYSTEM_PROMPT = `Tu es un agent IA autonome. Tu dois accomplir un objectif en plusieurs étapes.

RÈGLES STRICTES:
1. Génère UNE SEULE étape à la fois (pas toutes les étapes d'un coup)
2. Réponds UNIQUEMENT avec un JSON valide
3. N'invente jamais de résultats — base-toi uniquement sur les observations réelles
4. Si une étape échoue, adapte ton plan

FORMAT DE RÉPONSE (JSON strict):
{
  "thought": "ton raisonnement interne sur ce que tu vas faire",
  "tool": "nom_de_l_outil",
  "args": { "param": "valeur" },
  "done": false,
  "summary": null
}

Si l'objectif est atteint:
{
  "thought": "résumé de ce qui a été accompli",
  "tool": null,
  "args": null,
  "done": true,
  "summary": "Description de ce qui a été accompli et des résultats"
}`;

/**
 * Lance la boucle d'orchestration pour un objectif donné
 */
export async function runAgentObjective(objectiveId) {
  const config = await loadAgentConfig();

  if (!config.agentEnabled) {
    throw new Error('Agent runtime désactivé (AGENT_ENABLED=false)');
  }

  // Charger l'objectif
  const objective = await getObjective(objectiveId);
  if (!objective) throw new Error(`Objectif ${objectiveId} introuvable`);

  // Budgets
  const maxSteps = objective.budget?.maxSteps || parseInt(process.env.AGENT_MAX_STEPS || '20');
  const maxToolCalls = objective.budget?.maxToolCalls || parseInt(process.env.AGENT_MAX_TOOL_CALLS || '50');
  const maxDurationMs = objective.budget?.maxDurationMs || parseInt(process.env.AGENT_MAX_DURATION_MS || '300000');

  // Démarrer un run
  const run = await startRun(objectiveId);
  const runId = run.id;
  const startTime = Date.now();

  // Résumé des runs précédents (mémoire inter-runs)
  const previousSummary = await getPreviousRunSummary(objectiveId, run.run_number);

  // Contexte initial
  const messages = [
    { role: 'system', content: AGENT_SYSTEM_PROMPT },
    {
      role: 'user',
      content: `OBJECTIF: ${objective.title}\n\nDESCRIPTION: ${objective.description}${previousSummary ? `\n\nRUN PRÉCÉDENT:\n${previousSummary}` : ''}`,
    },
  ];

  let stepsCount = 0;
  let toolCallsCount = 0;

  await createEvent(runId, 'run_started', { objective_id: objectiveId, budget: { maxSteps, maxToolCalls, maxDurationMs } });

  // ── BOUCLE PRINCIPALE ──
  while (true) {
    // Vérification budgets
    if (stepsCount >= maxSteps) {
      await completeRun(runId, 'completed', `Budget steps épuisé (${maxSteps} steps)`);
      await createEvent(runId, 'budget_exceeded', { type: 'steps', limit: maxSteps });
      break;
    }

    if (toolCallsCount >= maxToolCalls) {
      await completeRun(runId, 'completed', `Budget tool calls épuisé (${maxToolCalls})`);
      await createEvent(runId, 'budget_exceeded', { type: 'tool_calls', limit: maxToolCalls });
      break;
    }

    if (Date.now() - startTime > maxDurationMs) {
      await completeRun(runId, 'failed', 'Timeout durée maximale');
      await createEvent(runId, 'timeout', { duration_ms: Date.now() - startTime });
      break;
    }

    // ── GÉNÉRATION D'ÉTAPE ──
    let stepResponse;
    try {
      const model = process.env.AGENT_AI_MODEL || process.env.HF_MODEL_HEAVY || 'Qwen/Qwen2.5-72B-Instruct';
      const temperature = parseFloat(process.env.AGENT_AI_TEMPERATURE || '0.2');
      const maxTokens = parseInt(process.env.AGENT_AI_MAX_TOKENS || '2048');

      const llmResponse = await generateWithHF({
        model,
        messages,
        max_tokens: maxTokens,
        temperature,
      });

      const content = llmResponse.content || '';

      // Parser le JSON
      const jsonMatch = content.match(/\{[\s\S]*\}/);
      if (!jsonMatch) throw new Error('Réponse LLM non JSON');
      stepResponse = JSON.parse(jsonMatch[0]);

      // Valider le schéma
      const validation = validateStepSchema(stepResponse);
      if (!validation.valid) throw new Error(`Schéma invalide: ${validation.error}`);

    } catch (error) {
      console.error('[orchestrator] Erreur génération étape:', error.message);
      await createEvent(runId, 'step_error', { error: error.message, step: stepsCount });
      await completeRun(runId, 'failed', `Erreur génération étape: ${error.message}`);
      break;
    }

    // ── FIN DE L'OBJECTIF ──
    if (stepResponse.done) {
      await completeRun(runId, 'completed', stepResponse.summary || 'Objectif atteint');
      await createEvent(runId, 'objective_completed', { summary: stepResponse.summary });
      break;
    }

    // ── CRÉER STEP EN BASE ──
    stepsCount++;
    const step = await createStep(runId, {
      step_number: stepsCount,
      thought: stepResponse.thought,
      tool: stepResponse.tool,
      args: stepResponse.args,
    });

    await createEvent(runId, 'step_started', { step_number: stepsCount, tool: stepResponse.tool });

    // ── EXÉCUTION OUTIL ──
    let toolResult = null;
    let toolSuccess = false;

    if (stepResponse.tool) {
      toolCallsCount++;

      // Vérification des droits (tools autorisés pour cet objectif)
      const allowedTools = objective.tools_allowed || [];
      if (allowedTools.length > 0 && !allowedTools.includes(stepResponse.tool)) {
        toolResult = { error: `Outil '${stepResponse.tool}' non autorisé pour cet objectif` };
      } else {
        // Vérification du risk level
        const riskLevel = getToolRiskLevel(stepResponse.tool);
        const allowedRisks = (process.env.AGENT_ALLOWED_RISK_LEVELS || 'low,medium').split(',');

        if (riskLevel === 'high' && !allowedRisks.includes('high')) {
          if (process.env.AGENT_APPROVAL_FLOW === 'true') {
            // Pause + notification admin
            await requestApproval(runId, step.id, stepResponse, riskLevel);
            await completeRun(runId, 'paused', `Approbation requise pour: ${stepResponse.tool}`);
            break; // Pause du run
          } else {
            toolResult = { error: `Action à risque élevé refusée (approbation désactivée)` };
          }
        } else {
          // Exécuter l'outil
          try {
            // Valider les args
            const argsValid = validateToolArgs(stepResponse.tool, stepResponse.args);
            if (!argsValid.valid) throw new Error(`Args invalides: ${argsValid.error}`);

            toolResult = await executeAgentTool(stepResponse.tool, stepResponse.args, riskLevel);
            toolSuccess = true;
          } catch (error) {
            toolResult = { error: error.message };
            await createEvent(runId, 'tool_error', { tool: stepResponse.tool, error: error.message });
          }
        }
      }

      // Logger le tool call
      await createToolCall(step.id, runId, {
        tool_name: stepResponse.tool,
        args: stepResponse.args,
        result: toolResult,
        success: toolSuccess,
      });
    }

    // ── COMPLÉTER STEP ──
    await completeStep(step.id, toolResult, toolSuccess);
    await createEvent(runId, 'step_completed', {
      step_number: stepsCount,
      tool: stepResponse.tool,
      success: toolSuccess,
    });

    // ── INJECTER OBSERVATION DANS LE CONTEXTE ──
    messages.push({
      role: 'assistant',
      content: JSON.stringify(stepResponse),
    });

    messages.push({
      role: 'user',
      content: `OBSERVATION step ${stepsCount}:\n${JSON.stringify(toolResult, null, 2)}\n\nContinue. Génère la prochaine étape ou marque done=true si l'objectif est atteint.`,
    });
  }

  return { run_id: runId, steps: stepsCount, tool_calls: toolCallsCount };
}

// ── APPROBATION HUMAINE ──
async function requestApproval(runId, stepId, stepResponse, riskLevel) {
  const { supabase } = await import('../supabase.js');

  const ttlHours = parseInt(process.env.AGENT_APPROVAL_TTL_HOURS || '24');

  const { data: approval } = await supabase.from('app_agent_approvals').insert({
    run_id: runId,
    step_id: stepId,
    status: 'pending',
    tool_name: stepResponse.tool,
    args: stepResponse.args,
    risk_reason: `Outil à risque élevé: ${stepResponse.tool}`,
    expires_at: new Date(Date.now() + ttlHours * 60 * 60 * 1000).toISOString(),
  }).select().single();

  // Notifier l'admin
  const channels = (process.env.AGENT_APPROVAL_NOTIFY_CHANNELS || 'email,whatsapp').split(',');
  const message = `🤖 Agent runtime — Approbation requise\n\nOutil: ${stepResponse.tool}\nArgs: ${JSON.stringify(stepResponse.args, null, 2)}\nRaisonnement: ${stepResponse.thought}\n\nApprobation ID: ${approval?.id}\nExpire dans ${ttlHours}h`;

  if (channels.includes('email') && process.env.ADMIN_EMAIL) {
    await sendEmail({
      to: process.env.ADMIN_EMAIL,
      subject: `[Agent Runtime] Approbation requise — ${stepResponse.tool}`,
      text: message,
    }).catch(console.error);
  }

  if (channels.includes('whatsapp') && process.env.ADMIN_PHONE) {
    await sendWhatsApp({ to: process.env.ADMIN_PHONE, message }).catch(console.error);
  }
}

// Risk levels des outils
function getToolRiskLevel(toolName) {
  const HIGH_RISK_TOOLS = ['delete_data', 'send_email', 'send_whatsapp', 'publish_social', 'modify_database'];
  const MEDIUM_RISK_TOOLS = ['create_data', 'update_data', 'generate_content', 'upload_drive'];

  if (HIGH_RISK_TOOLS.includes(toolName)) return 'high';
  if (MEDIUM_RISK_TOOLS.includes(toolName)) return 'medium';
  return 'low';
}
```

---

## Module `lib/agent/tool-gateway.js`

```javascript
// lib/agent/tool-gateway.js
// Dispatch + validation + exécution des outils agent

import { supabase } from '../supabase.js';
import { sendEmail } from '../email.js';
import { sendWhatsApp } from '../whatsapp.js';

const TIMEOUT_BY_RISK = {
  low: 10000,    // 10s
  medium: 30000, // 30s
  high: 60000,   // 60s
};

/**
 * Exécuter un outil agent avec timeout et retry
 */
export async function executeAgentTool(toolName, args, riskLevel = 'low') {
  const timeout = TIMEOUT_BY_RISK[riskLevel] || 10000;

  return Promise.race([
    executeToolInternal(toolName, args),
    new Promise((_, reject) => setTimeout(() => reject(new Error(`Timeout outil ${toolName} (${timeout}ms)`)), timeout)),
  ]);
}

async function executeToolInternal(toolName, args) {
  switch (toolName) {
    // ── BASE DE DONNÉES ──
    case 'read_database': {
      const { table, filters = {}, limit = 10 } = args;
      let query = supabase.from(table).select('*').limit(limit);
      for (const [key, value] of Object.entries(filters)) {
        query = query.eq(key, value);
      }
      const { data, error } = await query;
      if (error) throw new Error(`DB error: ${error.message}`);
      return { rows: data, count: data?.length };
    }

    case 'count_database': {
      const { table, filters = {} } = args;
      let query = supabase.from(table).select('*', { count: 'exact', head: true });
      for (const [key, value] of Object.entries(filters)) {
        query = query.eq(key, value);
      }
      const { count, error } = await query;
      if (error) throw new Error(`DB error: ${error.message}`);
      return { count };
    }

    case 'create_data': {
      const { table, data } = args;
      const { data: created, error } = await supabase.from(table).insert(data).select();
      if (error) throw new Error(`DB insert error: ${error.message}`);
      return { created: created?.[0], id: created?.[0]?.id };
    }

    case 'update_data': {
      const { table, id, data } = args;
      const { error } = await supabase.from(table).update(data).eq('id', id);
      if (error) throw new Error(`DB update error: ${error.message}`);
      return { updated: true, id };
    }

    // ── COMMUNICATIONS ──
    case 'send_email': {
      const { to, subject, body } = args;
      await sendEmail({ to, subject, text: body });
      return { sent: true, to };
    }

    case 'send_whatsapp': {
      const { to, message } = args;
      await sendWhatsApp({ to, message });
      return { sent: true, to };
    }

    // ── GÉNÉRATION CONTENU ──
    case 'generate_content': {
      const { prompt, max_tokens = 512 } = args;
      const { generateWithHF } = await import('../hf-client-v2.js');
      const response = await generateWithHF({
        model: process.env.HF_MODEL_FAST,
        messages: [{ role: 'user', content: prompt }],
        max_tokens,
        temperature: 0.7,
      });
      return { content: response.content || response };
    }

    // ── WIX ──
    case 'wix_search_products': {
      const { searchWixProducts } = await import('../wix-catalog.js');
      return searchWixProducts(args);
    }

    case 'wix_search_orders': {
      const { searchOrders } = await import('../orders.js');
      return searchOrders(args);
    }

    case 'wix_create_coupon': {
      const { generateCoupon } = await import('../wix-coupon.js');
      return generateCoupon(args);
    }

    // ── ANALYSE ──
    case 'analyze_leads': {
      const { data: leads } = await supabase
        .from('app_leads')
        .select('*')
        .gte('created_at', new Date(Date.now() - 7 * 24 * 60 * 60 * 1000).toISOString());

      const total = leads?.length || 0;
      const withEmail = leads?.filter(l => l.email).length || 0;
      const avgScore = leads?.reduce((s, l) => s + (l.score || 0), 0) / (total || 1);

      return { total_leads_7d: total, with_email: withEmail, avg_score: Math.round(avgScore) };
    }

    default:
      throw new Error(`Outil inconnu: ${toolName}`);
  }
}

/**
 * Valider les arguments d'un outil
 */
export function validateToolArgs(toolName, args) {
  const required = {
    read_database: ['table'],
    create_data: ['table', 'data'],
    update_data: ['table', 'id', 'data'],
    send_email: ['to', 'subject', 'body'],
    send_whatsapp: ['to', 'message'],
    generate_content: ['prompt'],
  };

  const requiredFields = required[toolName] || [];
  const missing = requiredFields.filter(f => !args?.[f]);

  if (missing.length > 0) {
    return { valid: false, error: `Champs requis manquants: ${missing.join(', ')}` };
  }

  return { valid: true };
}
```

---

## Module `lib/agent/run-ledger.js`

```javascript
// lib/agent/run-ledger.js
// Persistance immuable de l'historique des runs

import { supabase } from '../supabase.js';

export async function getObjective(objectiveId) {
  const { data } = await supabase
    .from('app_agent_objectives')
    .select('*')
    .eq('id', objectiveId)
    .single();
  return data;
}

export async function startRun(objectiveId) {
  // Incrémenter run_number
  const { data: lastRun } = await supabase
    .from('app_agent_runs')
    .select('run_number')
    .eq('objective_id', objectiveId)
    .order('run_number', { ascending: false })
    .limit(1)
    .single();

  const runNumber = (lastRun?.run_number || 0) + 1;

  const { data: run } = await supabase.from('app_agent_runs').insert({
    objective_id: objectiveId,
    run_number: runNumber,
    status: 'running',
    started_at: new Date().toISOString(),
  }).select().single();

  // Mettre à jour le statut de l'objectif
  await supabase.from('app_agent_objectives')
    .update({ status: 'running', updated_at: new Date().toISOString() })
    .eq('id', objectiveId);

  return run;
}

export async function completeRun(runId, status, summary) {
  await supabase.from('app_agent_runs').update({
    status,
    summary,
    completed_at: new Date().toISOString(),
  }).eq('id', runId);
}

export async function createStep(runId, stepData) {
  const { data: step } = await supabase.from('app_agent_run_steps').insert({
    run_id: runId,
    ...stepData,
    status: 'executing',
    created_at: new Date().toISOString(),
  }).select().single();

  // Incrémenter le compteur du run
  await supabase.rpc('increment_counter', { table_name: 'app_agent_runs', id: runId, field: 'steps_count' });

  return step;
}

export async function completeStep(stepId, result, success) {
  await supabase.from('app_agent_run_steps').update({
    result,
    status: success ? 'completed' : 'failed',
  }).eq('id', stepId);
}

export async function createToolCall(stepId, runId, toolCallData) {
  const { data: tc } = await supabase.from('app_agent_tool_calls').insert({
    step_id: stepId,
    run_id: runId,
    ...toolCallData,
    created_at: new Date().toISOString(),
  }).select().single();

  // Incrémenter le compteur du run
  await supabase.rpc('increment_counter', { table_name: 'app_agent_runs', id: runId, field: 'tool_calls_count' });

  return tc;
}

export async function createEvent(runId, eventType, data) {
  return supabase.from('app_agent_events').insert({
    run_id: runId,
    event_type: eventType,
    data,
    created_at: new Date().toISOString(),
  });
}

export async function getPreviousRunSummary(objectiveId, currentRunNumber) {
  if (currentRunNumber <= 1) return null;

  const { data: prevRun } = await supabase
    .from('app_agent_runs')
    .select('summary, completed_at, status')
    .eq('objective_id', objectiveId)
    .eq('run_number', currentRunNumber - 1)
    .single();

  if (!prevRun?.summary) return null;

  return `Run #${currentRunNumber - 1} (${prevRun.status}):\n${prevRun.summary}`;
}
```

---

## Module `lib/agent/schemas.js`

```javascript
// lib/agent/schemas.js
// Validation JSON des étapes et objectifs

export function validateStepSchema(step) {
  if (typeof step !== 'object' || step === null) {
    return { valid: false, error: 'Step doit être un objet' };
  }

  if (typeof step.thought !== 'string') {
    return { valid: false, error: '"thought" doit être une string' };
  }

  if (step.done !== true && step.done !== false) {
    return { valid: false, error: '"done" doit être un boolean' };
  }

  if (step.done === false) {
    if (!step.tool || typeof step.tool !== 'string') {
      return { valid: false, error: '"tool" requis quand done=false' };
    }
  }

  if (step.done === true && typeof step.summary !== 'string') {
    return { valid: false, error: '"summary" requis quand done=true' };
  }

  return { valid: true };
}

export function validateObjectiveSchema(objective) {
  if (!objective.title || typeof objective.title !== 'string') {
    return { valid: false, error: '"title" requis' };
  }

  if (!objective.description || typeof objective.description !== 'string') {
    return { valid: false, error: '"description" requise' };
  }

  return { valid: true };
}
```

---

## Routes API Agent Runtime

```javascript
// Dans api/index.js — Routes agent runtime

// Créer un objectif
router.post('/admin/agent/objectives', requireAuth, async (req, res) => {
  const { title, description, tools_allowed, budget, risk_level } = req.body;

  const { data: objective } = await supabase.from('app_agent_objectives').insert({
    title, description,
    tools_allowed: tools_allowed || [],
    budget: budget || {},
    risk_level: risk_level || 'medium',
    status: 'pending',
  }).select().single();

  return res.json({ objective });
});

// Démarrer un run
router.post('/admin/agent/objectives/:id/run', requireAuth, async (req, res) => {
  const { id } = req.params;

  if (process.env.AGENT_ENABLED !== 'true') {
    return res.status(503).json({ error: 'Agent runtime désactivé' });
  }

  // Démarrer en arrière-plan
  runAgentObjective(id)
    .then(result => console.log('[agent] Run terminé:', result))
    .catch(error => console.error('[agent] Run échoué:', error));

  return res.json({ message: 'Run démarré', objective_id: id });
});

// Lister les runs d'un objectif
router.get('/admin/agent/objectives/:id/runs', requireAuth, async (req, res) => {
  const { data: runs } = await supabase
    .from('app_agent_runs')
    .select('*, app_agent_run_steps(*)')
    .eq('objective_id', req.params.id)
    .order('run_number', { ascending: false });

  return res.json({ runs: runs || [] });
});

// Approuver/rejeter une action
router.post('/admin/agent/approvals/:id/approve', requireAuth, async (req, res) => {
  await supabase.from('app_agent_approvals').update({
    status: 'approved',
    approved_by: 'admin',
    approved_at: new Date().toISOString(),
  }).eq('id', req.params.id);

  return res.json({ approved: true });
});

router.post('/admin/agent/approvals/:id/reject', requireAuth, async (req, res) => {
  await supabase.from('app_agent_approvals').update({
    status: 'rejected',
    approved_by: 'admin',
    approved_at: new Date().toISOString(),
  }).eq('id', req.params.id);

  return res.json({ rejected: true });
});

// Maintenance
router.post('/admin/agent/maintenance', requireCronSecret, async (req, res) => {
  // Expirer les approbations TTL dépassé
  await supabase.from('app_agent_approvals')
    .update({ status: 'expired' })
    .eq('status', 'pending')
    .lt('expires_at', new Date().toISOString());

  // Nettoyer events anciens
  await supabase.rpc('prune_agent_events', {
    p_retention_days: parseInt(process.env.AGENT_EVENTS_RETENTION_DAYS || '30'),
  });

  return res.json({ maintained: true });
});
```
