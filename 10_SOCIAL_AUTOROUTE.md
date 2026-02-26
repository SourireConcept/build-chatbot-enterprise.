# 10 — SOCIAL AUTOROUTE AI PRO : Publication automatisée

## Concept

Autoroute est un système de **génération et publication de contenu social automatisé**. Il génère du texte, des images et des vidéos via des LLMs, puis publie sur plusieurs plateformes sociales.

**Pipeline :** Déclencheur (cron/manuel) → Génération texte → Génération média → Sauvegarde → Approbation → Publication

---

## Module `lib/autoroute-proxy.js`

```javascript
// lib/autoroute-proxy.js
// Pipeline Autoroute AI Pro

import { generateWithHF } from './hf-client-v2.js';
import { supabase } from './supabase.js';

// ──────────────────────────────────────────────────
// GÉNÉRATION TEXTE
// Chain: Perplexity → Gemini → HF GLM → HF Qwen fallback
// ──────────────────────────────────────────────────

export async function generateAutoroutePost({ topic, instructions, language = 'fr' }) {
  const customInstructions = instructions || process.env.AUTOROUTE_INSTRUCTIONS || '';
  const defaultTopic = topic || process.env.AUTOROUTE_DEFAULT_TOPIC || 'actualité du secteur';

  const systemPrompt = `${customInstructions}
Génère un post engageant pour les réseaux sociaux sur le thème: "${defaultTopic}".
Langue: ${language}. Ton: professionnel mais accessible. Max 280 caractères (version courte) + version longue (500 mots).
Format JSON: {"short": "post court", "long": "article long", "hashtags": ["#tag1", "#tag2"], "emoji": "🎯"}`;

  // Essayer les providers dans l'ordre
  const providers = [
    () => callPerplexity(systemPrompt, topic),
    () => callGemini(systemPrompt, topic),
    () => callHFModel(process.env.AUTOROUTE_TEXT_HF_MODEL || 'THUDM/glm-4-plus', systemPrompt, topic),
    ...getHFFallbackModels().map(model => () => callHFModel(model, systemPrompt, topic)),
  ];

  for (const provider of providers) {
    try {
      if (process.env.AUTOROUTE_DISABLE_PERPLEXITY === 'true' && provider === providers[0]) continue;
      if (process.env.AUTOROUTE_DISABLE_GEMINI === 'true' && provider === providers[1]) continue;
      if (process.env.AUTOROUTE_DISABLE_HF === 'true' && providers.indexOf(provider) >= 2) continue;

      const result = await provider();
      if (result) return result;
    } catch (error) {
      console.warn(`[autoroute] Provider échoué:`, error.message);
    }
  }

  throw new Error('Tous les providers Autoroute ont échoué');
}

async function callPerplexity(systemPrompt, topic) {
  if (!process.env.PERPLEXITY_API_KEY) return null;

  const model = process.env.PERPLEXITY_MODEL || 'llama-3.1-sonar-large-128k-online';

  const response = await fetch('https://api.perplexity.ai/chat/completions', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${process.env.PERPLEXITY_API_KEY}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      model,
      messages: [
        { role: 'system', content: systemPrompt },
        { role: 'user', content: `Génère le contenu sur: ${topic}` },
      ],
      max_tokens: 1024,
      temperature: 0.7,
    }),
  });

  if (!response.ok) throw new Error(`Perplexity ${response.status}`);
  const data = await response.json();
  return parseAutorouteContent(data.choices[0].message.content);
}

async function callGemini(systemPrompt, topic) {
  const apiKey = process.env.GEMINI_API_KEY || process.env.GOOGLE_AI_API_KEY;
  if (!apiKey) return null;

  const model = process.env.GEMINI_TEXT_MODEL || 'gemini-2.5-flash';

  const response = await fetch(
    `https://generativelanguage.googleapis.com/v1beta/models/${model}:generateContent?key=${apiKey}`,
    {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        contents: [{ parts: [{ text: `${systemPrompt}\n\nSujet: ${topic}` }] }],
        generationConfig: { maxOutputTokens: 1024, temperature: 0.7 },
      }),
    }
  );

  if (!response.ok) throw new Error(`Gemini ${response.status}`);
  const data = await response.json();
  const text = data.candidates?.[0]?.content?.parts?.[0]?.text || '';
  return parseAutorouteContent(text);
}

async function callHFModel(model, systemPrompt, topic) {
  const response = await generateWithHF({
    model,
    messages: [
      { role: 'system', content: systemPrompt },
      { role: 'user', content: `Génère le contenu sur: ${topic}` },
    ],
    max_tokens: 1024,
    temperature: 0.7,
  });

  return parseAutorouteContent(response.content || '');
}

function getHFFallbackModels() {
  return (process.env.AUTOROUTE_TEXT_HF_FALLBACK_MODELS || 'Qwen/Qwen2.5-72B-Instruct').split(',').map(m => m.trim());
}

function parseAutorouteContent(text) {
  // Essayer de parser du JSON
  const jsonMatch = text.match(/\{[\s\S]*\}/);
  if (jsonMatch) {
    try {
      return JSON.parse(jsonMatch[0]);
    } catch {}
  }
  // Fallback: retourner le texte brut comme post court
  return { short: text.slice(0, 280), long: text, hashtags: [], emoji: '' };
}

// ──────────────────────────────────────────────────
// GÉNÉRATION MÉDIA (Image/Vidéo)
// ──────────────────────────────────────────────────

const IMAGE_PACKS = {
  'glm-flux-wan': {
    textModel: 'THUDM/glm-4-plus',
    imageModel: 'black-forest-labs/FLUX.1-dev',
  },
  'glm-flux-hunyuan': {
    textModel: 'THUDM/glm-4-plus',
    imageModel: 'hunyuandit-community/HunyuanDiT-v1.1-Diffusers-Distilled',
  },
  'glm-sdxl-wan': {
    textModel: 'THUDM/glm-4-plus',
    imageModel: 'stabilityai/stable-diffusion-xl-base-1.0',
  },
  'glm-sdxl-hunyuan': {
    textModel: 'THUDM/glm-4-plus',
    imageModel: 'stabilityai/stable-diffusion-xl-base-1.0',
  },
};

export async function generateAutorouteMedia({ prompt, type = 'image', pack }) {
  const selectedPack = pack || process.env.AUTOROUTE_MEDIA_PACK || 'glm-flux-wan';

  if (type === 'image') {
    return generateImage(prompt, selectedPack);
  } else if (type === 'video') {
    return generateVideo(prompt);
  }

  throw new Error(`Type média non supporté: ${type}`);
}

async function generateImage(prompt, pack) {
  const packConfig = IMAGE_PACKS[pack] || IMAGE_PACKS['glm-flux-wan'];

  // 1. Améliorer le prompt image via LLM
  const imageInstructions = process.env.AUTOROUTE_IMAGE_INSTRUCTIONS || '';
  const enhancedPrompt = await generateWithHF({
    model: packConfig.textModel,
    messages: [
      {
        role: 'system',
        content: `${imageInstructions}\nTu génères des prompts détaillés pour la génération d'images. Réponds uniquement avec le prompt optimisé.`,
      },
      { role: 'user', content: `Prompt image pour: ${prompt}` },
    ],
    max_tokens: 256,
    temperature: 0.8,
  }).then(r => r.content || prompt).catch(() => prompt);

  // 2. Générer l'image
  const { InferenceClient } = await import('@huggingface/inference');
  const client = new InferenceClient(process.env.HF_API_TOKEN);

  const imageBlob = await client.textToImage({
    model: packConfig.imageModel,
    inputs: enhancedPrompt,
    parameters: {
      num_inference_steps: process.env.AUTOROUTE_IMAGE_QUALITY_HINT === 'high' ? 50 : 30,
    },
  });

  // 3. Uploader sur Supabase Storage
  const filename = `autoroute_${Date.now()}.png`;
  const buffer = Buffer.from(await imageBlob.arrayBuffer());

  const { data: uploadData, error } = await supabase.storage
    .from(process.env.WORKFLOW_MEDIA_BUCKET || 'media')
    .upload(`autoroute/${filename}`, buffer, { contentType: 'image/png' });

  if (error) throw new Error(`Upload échoué: ${error.message}`);

  const { data: urlData } = supabase.storage
    .from(process.env.WORKFLOW_MEDIA_BUCKET || 'media')
    .getPublicUrl(`autoroute/${filename}`);

  // 4. QA optionnel (BLIP captioning)
  let qaCaption = null;
  if (process.env.AUTOROUTE_IMAGE_QA_PROVIDER === 'hf') {
    try {
      qaCaption = await client.imageToText({
        model: process.env.AUTOROUTE_IMAGE_QA_MODEL || 'Salesforce/blip-image-captioning-large',
        data: buffer,
      });
    } catch {}
  }

  return {
    type: 'image',
    url: urlData.publicUrl,
    filename,
    prompt: enhancedPrompt,
    qa_caption: qaCaption,
  };
}

async function generateVideo(prompt) {
  const { InferenceClient } = await import('@huggingface/inference');
  const client = new InferenceClient(process.env.HF_API_TOKEN);

  // Essayer HunyuanVideo en premier, puis Wan
  const videoModels = ['tencent/HunyuanVideo', 'Wan-AI/Wan2.1-T2V-14B'];

  for (const model of videoModels) {
    try {
      const videoBlob = await client.textToVideo({
        model,
        inputs: prompt,
        parameters: { num_frames: 49 },
      });

      const filename = `autoroute_${Date.now()}.mp4`;
      const buffer = Buffer.from(await videoBlob.arrayBuffer());

      await supabase.storage
        .from(process.env.WORKFLOW_MEDIA_BUCKET || 'media')
        .upload(`autoroute/${filename}`, buffer, { contentType: 'video/mp4' });

      const { data: urlData } = supabase.storage
        .from(process.env.WORKFLOW_MEDIA_BUCKET || 'media')
        .getPublicUrl(`autoroute/${filename}`);

      return { type: 'video', url: urlData.publicUrl, filename };

    } catch (error) {
      console.warn(`[autoroute] Modèle vidéo ${model} échoué:`, error.message);
    }
  }

  throw new Error('Génération vidéo impossible — tous les modèles ont échoué');
}

// ──────────────────────────────────────────────────
// SAUVEGARDE ET QUEUE
// ──────────────────────────────────────────────────

export async function saveAndQueuePost({ content, media, platforms = ['facebook', 'instagram'] }) {
  // Sauvegarder dans la queue
  const { data: queueItem } = await supabase.from('app_social_publication_queue').insert({
    content: content.short || content.long || content,
    media_url: media?.url || null,
    media_type: media?.type || null,
    platforms,
    status: process.env.SOCIAL_APPROVAL_REQUIRED === 'true' ? 'pending' : 'approved',
    metadata: { content_full: content, media_metadata: media },
  }).select().single();

  return queueItem;
}
```

---

## Module `lib/workflows.js`

```javascript
// lib/workflows.js
// Moteur de workflows configurable

import { supabase } from './supabase.js';

/**
 * Déclencher les workflows abonnés à un événement
 */
export async function runWorkflowsForEvent(eventName, data = {}) {
  const { data: workflows } = await supabase
    .from('app_workflows')
    .select('*')
    .eq('trigger_event', eventName)
    .eq('enabled', true);

  if (!workflows || workflows.length === 0) return [];

  const results = [];

  for (const workflow of workflows) {
    try {
      const result = await executeWorkflow(workflow, data);
      results.push({ workflow_id: workflow.id, name: workflow.name, result });
    } catch (error) {
      console.error(`[workflows] Workflow ${workflow.name} échoué:`, error.message);
      results.push({ workflow_id: workflow.id, name: workflow.name, error: error.message });
    }
  }

  return results;
}

async function executeWorkflow(workflow, triggerData) {
  const steps = workflow.steps || [];
  const settings = workflow.settings || {};
  const rateLimitMs = settings.rate_limit_ms || 0;

  const stepResults = [];

  for (const step of steps) {
    try {
      if (rateLimitMs > 0 && stepResults.length > 0) {
        await new Promise(r => setTimeout(r, rateLimitMs));
      }

      const result = await executeWorkflowStep(step, triggerData, stepResults);
      stepResults.push({ step_type: step.type, result, success: true });

    } catch (error) {
      console.error(`[workflows] Step ${step.type} échoué:`, error.message);
      stepResults.push({ step_type: step.type, error: error.message, success: false });

      // Arrêter si step critique
      if (step.abort_on_error) break;
    }
  }

  // Mettre à jour last_run
  await supabase.from('app_workflows').update({
    last_run_at: new Date().toISOString(),
    last_run_status: stepResults.every(s => s.success) ? 'success' : 'partial',
  }).eq('id', workflow.id);

  return stepResults;
}

async function executeWorkflowStep(step, triggerData, previousResults) {
  const config = step.config || {};

  switch (step.type) {
    // ── PUBLICATION SOCIALE ──
    case 'publish_facebook': {
      const { publishFacebook } = await import('./social/facebook.js');
      return publishFacebook({
        pageId: config.page_id || process.env.FB_PAGE_ID,
        content: config.content || triggerData.content?.short,
        mediaUrl: config.media_url || triggerData.media?.url,
      });
    }

    case 'publish_instagram': {
      const { publishInstagram } = await import('./social/instagram.js');
      return publishInstagram({
        userId: config.user_id || process.env.IG_USER_ID,
        caption: config.content || triggerData.content?.short,
        mediaUrl: config.media_url || triggerData.media?.url,
        mediaType: triggerData.media?.type || 'IMAGE',
      });
    }

    case 'publish_tiktok': {
      const { publishTikTok } = await import('./social/tiktok-publish.js');
      return publishTikTok({
        openId: config.open_id || process.env.TIKTOK_PUBLISH_OPEN_ID,
        videoUrl: config.media_url || triggerData.media?.url,
        caption: config.content || triggerData.content?.short,
      });
    }

    case 'publish_wix_blog': {
      const { publishWixBlogPost } = await import('./wix-blog.js');
      return publishWixBlogPost({
        title: config.title || triggerData.content?.title || 'Nouvel article',
        content: config.content || triggerData.content?.long,
        coverImageUrl: triggerData.media?.url,
      });
    }

    // ── COMMUNICATIONS ──
    case 'send_email': {
      const { sendEmail } = await import('./email.js');
      return sendEmail({
        to: config.to,
        subject: config.subject,
        text: config.body || triggerData.content?.short,
      });
    }

    case 'send_whatsapp': {
      const { sendWhatsApp } = await import('./whatsapp.js');
      return sendWhatsApp({
        to: config.to || process.env.ADMIN_PHONE,
        message: config.message || triggerData.content?.short,
      });
    }

    // ── STOCKAGE ──
    case 'upload_drive': {
      const { uploadToDrive } = await import('./google-drive.js');
      return uploadToDrive({
        content: config.content || triggerData.media?.buffer,
        mimeType: config.mime_type || 'image/png',
        filename: config.filename || `autoroute_${Date.now()}.png`,
        folderId: config.folder_id || process.env.AUTOROUTE_DRIVE_FOLDER_ID,
      });
    }

    // ── DONNÉES ──
    case 'save_to_database': {
      const { supabase } = await import('./supabase.js');
      const { data } = await supabase.from(config.table).insert(config.data || triggerData);
      return { saved: true, data };
    }

    // ── HTTP ──
    case 'webhook': {
      const response = await fetch(config.url, {
        method: config.method || 'POST',
        headers: { 'Content-Type': 'application/json', ...config.headers },
        body: JSON.stringify(config.body || triggerData),
      });
      return { status: response.status, ok: response.ok };
    }

    // ── ATTENTE ──
    case 'delay': {
      await new Promise(r => setTimeout(r, config.ms || 1000));
      return { waited_ms: config.ms };
    }

    default:
      throw new Error(`Type de step inconnu: ${step.type}`);
  }
}
```

---

## Modules publication sociale

### `lib/social/facebook.js`

```javascript
// lib/social/facebook.js

const GRAPH_API = 'https://graph.facebook.com/v18.0';

export async function publishFacebook({ pageId, content, mediaUrl, accessToken }) {
  const token = accessToken || process.env.FB_PAGE_ACCESS_TOKEN;
  if (!token || !pageId) throw new Error('Facebook: pageId et token requis');

  // Avec média
  if (mediaUrl) {
    // 1. Uploader la photo
    const photoRes = await fetch(`${GRAPH_API}/${pageId}/photos`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ url: mediaUrl, caption: content, access_token: token, published: true }),
    });
    return photoRes.json();
  }

  // Sans média (post texte)
  const postRes = await fetch(`${GRAPH_API}/${pageId}/feed`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ message: content, access_token: token }),
  });

  if (!postRes.ok) throw new Error(`Facebook API: ${postRes.status}`);
  return postRes.json();
}
```

### `lib/social/instagram.js`

```javascript
// lib/social/instagram.js
// Instagram Content Publishing API (via Graph API)

const GRAPH_API = 'https://graph.facebook.com/v18.0';

export async function publishInstagram({ userId, caption, mediaUrl, mediaType = 'IMAGE', accessToken }) {
  const token = accessToken || process.env.IG_ACCESS_TOKEN;
  const igUserId = userId || process.env.IG_USER_ID;
  if (!token || !igUserId) throw new Error('Instagram: userId et token requis');

  // Étape 1: Créer le media container
  const containerRes = await fetch(`${GRAPH_API}/${igUserId}/media`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      image_url: mediaUrl,
      caption,
      access_token: token,
    }),
  });

  if (!containerRes.ok) throw new Error(`Instagram container: ${containerRes.status}`);
  const { id: containerId } = await containerRes.json();

  // Étape 2: Publier le container
  const publishRes = await fetch(`${GRAPH_API}/${igUserId}/media_publish`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ creation_id: containerId, access_token: token }),
  });

  if (!publishRes.ok) throw new Error(`Instagram publish: ${publishRes.status}`);
  return publishRes.json();
}
```

### `lib/social/tiktok-publish.js`

```javascript
// lib/social/tiktok-publish.js
// TikTok Content Posting API

export async function publishTikTok({ openId, videoUrl, caption, accessToken }) {
  const token = accessToken || process.env.TIKTOK_PUBLISH_ACCESS_TOKEN;
  const tikTokOpenId = openId || process.env.TIKTOK_PUBLISH_OPEN_ID;
  if (!token || !tikTokOpenId) throw new Error('TikTok: openId et token requis');

  // Initialiser l'upload vidéo
  const initRes = await fetch('https://open.tiktokapis.com/v2/post/publish/video/init/', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      post_info: {
        title: caption?.slice(0, 150) || '',
        privacy_level: 'PUBLIC_TO_EVERYONE',
        disable_duet: false,
        disable_comment: false,
        disable_stitch: false,
        video_cover_timestamp_ms: 1000,
      },
      source_info: {
        source: 'PULL_FROM_URL',
        video_url: videoUrl,
      },
    }),
  });

  if (!initRes.ok) throw new Error(`TikTok init: ${initRes.status}`);
  return initRes.json();
}
```

---

## Cron quotidien (vercel.json)

```json
{
  "crons": [
    {
      "path": "/api/cron/autoroute-daily",
      "schedule": "0 9 * * *"
    }
  ]
}
```

## Handler cron Autoroute

```javascript
// Dans api/index.js

router.post('/cron/autoroute-daily', requireCronSecret, async (req, res) => {
  if (process.env.AUTOROUTE_PUBLICATION_MODE_DEFAULT === 'disabled') {
    return res.json({ skipped: true, reason: 'disabled' });
  }

  try {
    // 1. Générer le texte
    const content = await generateAutoroutePost({
      topic: process.env.AUTOROUTE_DEFAULT_TOPIC,
      language: 'fr',
    });

    // 2. Générer le média
    const media = await generateAutorouteMedia({
      prompt: content.short,
      type: 'image',
      pack: process.env.AUTOROUTE_MEDIA_PACK,
    }).catch(err => {
      console.warn('[cron/autoroute] Média échoué (continuer sans):', err.message);
      return null;
    });

    // 3. Sauvegarder et mettre en queue
    const queueItem = await saveAndQueuePost({
      content,
      media,
      platforms: ['facebook', 'instagram'],
    });

    // 4. Si pas d'approbation requise → publier immédiatement
    if (process.env.SOCIAL_APPROVAL_REQUIRED !== 'true') {
      await runWorkflowsForEvent('autoroute_post_approved', { content, media, queue_item_id: queueItem.id });
    } else {
      // Notifier l'admin pour approbation
      const adminEmail = process.env.NOTIFICATION_EMAIL || process.env.ADMIN_EMAIL;
      if (adminEmail) {
        await sendEmail({
          to: adminEmail,
          subject: 'Contenu social à approuver — Autoroute',
          text: `Nouveau contenu généré:\n\n${content.short}\n\nApprouvez dans le panel admin → Queue Sociale.`,
        });
      }
    }

    return res.json({ success: true, queue_item_id: queueItem.id });

  } catch (error) {
    console.error('[cron/autoroute] Erreur:', error);
    return res.status(500).json({ error: error.message });
  }
});
```

---

## Route approbation sociale (admin)

```javascript
// Approuver un post en attente
router.post('/admin/social-queue/:id/approve', requireAuth, async (req, res) => {
  const { id } = req.params;

  // Récupérer l'item
  const { data: item } = await supabase
    .from('app_social_publication_queue')
    .select('*')
    .eq('id', id)
    .single();

  if (!item) return res.status(404).json({ error: 'Item non trouvé' });

  // Marquer comme approuvé
  await supabase.from('app_social_publication_queue').update({
    status: 'approved',
    approved_by: 'admin',
    approved_at: new Date().toISOString(),
  }).eq('id', id);

  // Déclencher la publication
  const result = await runWorkflowsForEvent('autoroute_post_approved', {
    content: item.metadata?.content_full,
    media: { url: item.media_url, type: item.media_type },
    platforms: item.platforms,
  });

  // Marquer comme publié
  await supabase.from('app_social_publication_queue').update({
    status: 'published',
    published_at: new Date().toISOString(),
  }).eq('id', id);

  return res.json({ approved: true, publish_results: result });
});

// Rejeter un post
router.post('/admin/social-queue/:id/reject', requireAuth, async (req, res) => {
  await supabase.from('app_social_publication_queue').update({ status: 'rejected' }).eq('id', req.params.id);
  return res.json({ rejected: true });
});

// Lister la queue
router.get('/admin/social-queue', requireAuth, async (req, res) => {
  const { data: queue } = await supabase
    .from('app_social_publication_queue')
    .select('*')
    .order('created_at', { ascending: false })
    .limit(50);
  return res.json({ queue: queue || [] });
});
```
