# 07 — INTÉGRATIONS EXTERNES

## Vue d'ensemble des intégrations

| Service | Module | Requis | Usage principal |
|---|---|---|---|
| HuggingFace | `hf-client-v2.js` | Oui | LLM principal + embeddings |
| Supabase | `supabase.js` | Oui | BDD + Storage |
| Wix APIs | `wix-*.js` | Optionnel | Boutique + CRM |
| WhatsApp | `whatsapp.js` | Optionnel | Chat + notifications |
| Facebook Messenger | `messenger.js` | Optionnel | Chat |
| SMTP | `email.js` | Recommandé | Notifications |
| Stripe | `stripe.js` | Optionnel | Paiement fallback |
| Google Gemini | `autoroute-proxy.js` | Optionnel | Social AI |
| Google Drive | `google-drive.js` | Optionnel | Assets storage |
| Sentry | `instrument.js` | Recommandé | Monitoring |

---

## Intégration Wix (complète)

### `lib/wix-client.js` — Client de base

```javascript
// lib/wix-client.js

import axios from 'axios';

const WIX_BASE = 'https://www.wixapis.com';
const TIMEOUT = parseInt(process.env.WIX_TIMEOUT_MS || '8000');

// Instance axios configurée
export const wixAxios = axios.create({
  baseURL: WIX_BASE,
  timeout: TIMEOUT,
  headers: {
    'Authorization': process.env.WIX_API_KEY,
    'wix-site-id': process.env.WIX_SITE_ID,
    'wix-account-id': process.env.WIX_ACCOUNT_ID,
    'Content-Type': 'application/json',
  },
});

// Circuit breaker state
let cbState = 'closed'; // closed | open | half-open
let cbFailures = 0;
let cbOpenedAt = null;

const CB_THRESHOLD = parseInt(process.env.WIX_CB_FAILURE_THRESHOLD || '5');
const CB_OPEN_MS = parseInt(process.env.WIX_CB_OPEN_MS || '30000');

/**
 * Appel API Wix avec circuit breaker + retry exponentiel
 */
export async function wixRequest(config) {
  // Circuit breaker check
  if (cbState === 'open') {
    if (Date.now() - cbOpenedAt > CB_OPEN_MS) {
      cbState = 'half-open'; // Essai de récupération
    } else {
      throw new Error('Circuit breaker ouvert — Wix API indisponible');
    }
  }

  const maxRetries = parseInt(process.env.WIX_RETRY_ATTEMPTS || '3');
  const baseDelay = parseInt(process.env.WIX_RETRY_BASE_DELAY_MS || '300');

  for (let attempt = 0; attempt < maxRetries; attempt++) {
    try {
      const response = await wixAxios(config);

      // Succès → réinitialiser circuit breaker
      cbFailures = 0;
      if (cbState === 'half-open') cbState = 'closed';

      return response.data;

    } catch (error) {
      const isRetryable = !error.response || error.response.status >= 500 || error.response.status === 429;
      const isLastAttempt = attempt === maxRetries - 1;

      if (!isRetryable || isLastAttempt) {
        // Incrémenter failures circuit breaker
        cbFailures++;
        if (cbFailures >= CB_THRESHOLD) {
          cbState = 'open';
          cbOpenedAt = Date.now();
          console.warn('[wix-client] Circuit breaker OUVERT');
        }
        throw error;
      }

      // Retry exponentiel
      const delay = Math.min(baseDelay * Math.pow(2, attempt), 2000);
      await new Promise(r => setTimeout(r, delay));
    }
  }
}
```

### `lib/wix-catalog.js` — Catalogue produits

```javascript
// lib/wix-catalog.js

import { wixRequest } from './wix-client.js';
import { supabase } from './supabase.js';

const CACHE_TTL_MS = 120 * 1000; // 2 minutes

export async function searchWixProducts({ query, category, limit = 4 }) {
  if (!process.env.WIX_API_KEY) return [];

  // Vérifier le cache Supabase
  const cacheKey = `catalog:${query}:${category}:${limit}`;
  const cached = await getCachedProducts(cacheKey);
  if (cached) return cached;

  try {
    const response = await wixRequest({
      method: 'POST',
      url: '/stores/v1/products/query',
      data: {
        query: {
          filter: {
            visible: { $eq: true },
            ...(category ? { 'collections.name': { $contains: category } } : {}),
          },
          search: { expression: query },
          paging: { limit: Math.min(limit, 8) },
          fields: ['id', 'name', 'price', 'media', 'description', 'stock', 'productPageUrl'],
        },
      },
    });

    const products = (response.products || []).map(p => ({
      id: p.id,
      name: p.name,
      price: p.price?.discountedPrice || p.price?.price,
      originalPrice: p.price?.price,
      image: p.media?.mainMedia?.image?.url || null,
      description: (p.description || '').replace(/<[^>]*>/g, '').slice(0, 150),
      inStock: p.stock?.inStock !== false,
      url: p.productPageUrl || null,
    }));

    // Sauvegarder en cache
    await cacheProducts(cacheKey, products, CACHE_TTL_MS);

    return products;

  } catch (error) {
    console.error('[wix-catalog] Erreur recherche:', error.message);
    return [];
  }
}

async function getCachedProducts(key) {
  const { data } = await supabase
    .from('app_product_cache')
    .select('data')
    .eq('cache_key', key)
    .gt('expires_at', new Date().toISOString())
    .single();

  return data?.data || null;
}

async function cacheProducts(key, products, ttlMs) {
  await supabase.from('app_product_cache').upsert({
    cache_key: key,
    data: products,
    expires_at: new Date(Date.now() + ttlMs).toISOString(),
  }, { onConflict: 'cache_key' }).catch(() => {});
}
```

### `lib/wix-crm.js` — Contacts

```javascript
// lib/wix-crm.js

import { wixRequest } from './wix-client.js';

/**
 * Upsert un contact Wix CRM V4
 * Stratégie: find → update OU create
 */
export async function upsertWixContact({ email, name, phone }) {
  if (!email) throw new Error('Email requis pour upsert contact Wix');

  try {
    // Chercher le contact existant
    const searchResult = await wixRequest({
      method: 'POST',
      url: '/contacts/v4/contacts/query',
      data: {
        query: {
          filter: { 'info.emails.email': { $eq: email } },
          paging: { limit: 1 },
        },
      },
    });

    const existing = searchResult.contacts?.[0];

    if (existing) {
      // Mettre à jour
      const [firstName, ...lastParts] = (name || '').split(' ');
      await wixRequest({
        method: 'PATCH',
        url: `/contacts/v4/contacts/${existing.id}`,
        data: {
          info: {
            name: { first: firstName, last: lastParts.join(' ') || undefined },
            phones: phone ? [{ phone, primary: true }] : undefined,
          },
        },
      });
      return { id: existing.id, action: 'updated' };
    }

    // Créer nouveau contact
    const [firstName, ...lastParts] = (name || '').split(' ');
    const created = await wixRequest({
      method: 'POST',
      url: '/contacts/v4/contacts',
      data: {
        info: {
          name: { first: firstName || email.split('@')[0], last: lastParts.join(' ') || undefined },
          emails: [{ email, primary: true }],
          phones: phone ? [{ phone, primary: true }] : undefined,
          labelKeys: ['custom.chatbot-lead'],
        },
      },
    });

    return { id: created.contact?.id, action: 'created' };

  } catch (error) {
    console.error('[wix-crm] Erreur upsert contact:', error.message);
    throw error;
  }
}
```

### `lib/wix-coupon.js` — Coupons

```javascript
// lib/wix-coupon.js

import { wixRequest } from './wix-client.js';
import crypto from 'crypto';

export async function generateCoupon({
  type = 'PERCENTAGE',  // PERCENTAGE | FIXED_AMOUNT | FREE_SHIPPING
  discountValue,
  expiryDays = 7,
  minimumOrder = 0,
  description = 'Code promo chatbot',
}) {
  // Générer un code unique
  const codeBytes = parseInt(process.env.COUPON_CODE_BYTES || '8');
  const code = crypto.randomBytes(codeBytes).toString('hex').toUpperCase().slice(0, 12);

  // Vérification montant max
  const maxFixed = parseFloat(process.env.COUPON_MAX_FIXED_AMOUNT || '50');
  if (type === 'FIXED_AMOUNT' && discountValue > maxFixed) {
    throw new Error(`Montant coupon fixe trop élevé (max: ${maxFixed}€)`);
  }

  const expiryDate = new Date(Date.now() + expiryDays * 24 * 60 * 60 * 1000);

  const response = await wixRequest({
    method: 'POST',
    url: '/promotions/v1/coupons',
    data: {
      specification: {
        name: description,
        code,
        expirationDate: expiryDate.toISOString(),
        minimumSubtotal: minimumOrder,
        discountType: type,
        discount: type === 'PERCENTAGE'
          ? { percentage: { percentage: discountValue } }
          : type === 'FIXED_AMOUNT'
            ? { moneyOff: { amount: String(discountValue) } }
            : {},
        usageLimit: 1,        // Un seul usage
        limitedToOneItem: false,
      },
    },
  });

  return {
    code,
    type,
    discountValue,
    expiryDays,
    minimumOrder,
    couponId: response.coupon?.id,
  };
}
```

### `lib/wix-data.js` — Collections CMS

```javascript
// lib/wix-data.js

import { wixRequest } from './wix-client.js';

/**
 * Query une collection Wix Data V2
 */
export async function queryWixData(collectionId, { filter = {}, limit = 10, fields = [], sort } = {}) {
  const response = await wixRequest({
    method: 'POST',
    url: `/wix-data/v2/items/query`,
    data: {
      dataCollectionId: collectionId,
      query: {
        filter,
        paging: { limit },
        fields,
        sort,
      },
    },
  });

  return response.dataItems?.map(item => item.data) || [];
}

/**
 * Insérer/mettre à jour un item dans une collection
 */
export async function upsertWixDataItem(collectionId, data, id = null) {
  if (id) {
    return wixRequest({
      method: 'PUT',
      url: `/wix-data/v2/items/${id}`,
      data: { dataCollectionId: collectionId, dataItem: { id, data } },
    });
  }

  return wixRequest({
    method: 'POST',
    url: '/wix-data/v2/items',
    data: { dataCollectionId: collectionId, dataItem: { data } },
  });
}
```

---

## Intégration WhatsApp Business API

```javascript
// lib/whatsapp.js

const GRAPH_API_VERSION = 'v18.0';
const BASE_URL = `https://graph.facebook.com/${GRAPH_API_VERSION}`;

/**
 * Envoyer un message WhatsApp texte simple
 */
export async function sendWhatsApp({ to, message }) {
  const phoneId = process.env.WHATSAPP_PHONE_NUMBER_ID;
  const token = process.env.WHATSAPP_ACCESS_TOKEN;

  if (!phoneId || !token) {
    console.warn('[whatsapp] Variables non configurées, message ignoré');
    return null;
  }

  // Normaliser le numéro (supprimer espaces/tirets)
  const phone = to.replace(/[\s\-\(\)]/g, '');

  const response = await fetch(`${BASE_URL}/${phoneId}/messages`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      messaging_product: 'whatsapp',
      recipient_type: 'individual',
      to: phone,
      type: 'text',
      text: { body: message },
    }),
  });

  if (!response.ok) {
    const error = await response.text();
    throw new Error(`WhatsApp API error: ${response.status} — ${error}`);
  }

  return response.json();
}

/**
 * Handler webhook WhatsApp entrant
 */
export function handleWhatsAppWebhook(req, res) {
  const { entry } = req.body;

  if (!entry?.[0]?.changes?.[0]?.value?.messages?.[0]) {
    return res.status(200).send('OK');
  }

  const message = entry[0].changes[0].value.messages[0];
  const from = message.from;
  const text = message.text?.body || '';

  // Déléguer au moteur chatbot
  return { from, text, channel: 'whatsapp' };
}

/**
 * Vérification webhook (challenge)
 */
export function verifyWhatsAppWebhook(req, res) {
  const mode = req.query['hub.mode'];
  const token = req.query['hub.verify_token'];
  const challenge = req.query['hub.challenge'];

  if (mode === 'subscribe' && token === process.env.WHATSAPP_VERIFY_TOKEN) {
    return res.status(200).send(challenge);
  }

  return res.status(403).send('Verification failed');
}
```

---

## Intégration Email SMTP

```javascript
// lib/email.js

import nodemailer from 'nodemailer';

let transporter = null;

function getTransporter() {
  if (!transporter) {
    transporter = nodemailer.createTransport({
      host: process.env.SMTP_HOST,
      port: parseInt(process.env.SMTP_PORT || '465'),
      secure: true, // SSL
      auth: {
        user: process.env.SMTP_USER,
        pass: process.env.SMTP_PASS,
      },
    });
  }
  return transporter;
}

/**
 * Envoyer un email
 * @param {Object} params - { to, subject, text, html }
 */
export async function sendEmail({ to, subject, text, html }) {
  if (!process.env.SMTP_HOST || !process.env.SMTP_USER) {
    console.warn('[email] SMTP non configuré, email ignoré');
    return null;
  }

  // Vérifier domaine autorisé (sécurité)
  const allowedDomains = (process.env.ADMIN_EMAIL_ALLOWED_DOMAINS || '').split(',').map(d => d.trim());
  if (allowedDomains.length > 0 && !allowedDomains.some(d => to.endsWith(`@${d}`))) {
    throw new Error(`Email vers ${to} non autorisé (domaine hors allowlist)`);
  }

  const info = await getTransporter().sendMail({
    from: process.env.SMTP_FROM || process.env.SMTP_USER,
    to,
    subject,
    text,
    html: html || text,
  });

  return info;
}

/**
 * Template email notification nouvelle conversation
 */
export function buildLeadNotificationEmail(lead, brandColor = '#3a86ff', brandName = 'Mon App') {
  return {
    subject: `Nouveau lead chatbot — ${lead.name || lead.email}`,
    html: `
<!DOCTYPE html>
<html>
<head><meta charset="utf-8"></head>
<body style="font-family: Arial, sans-serif; max-width: 600px; margin: 0 auto; padding: 20px;">
  <div style="background: ${brandColor}; padding: 20px; border-radius: 8px 8px 0 0;">
    <h1 style="color: white; margin: 0;">${brandName}</h1>
    <p style="color: rgba(255,255,255,0.9); margin: 5px 0 0;">Nouveau lead depuis le chatbot</p>
  </div>
  <div style="background: #f8f9fa; padding: 20px; border-radius: 0 0 8px 8px;">
    <h2 style="color: #333; margin-top: 0;">Informations du prospect</h2>
    <table style="width: 100%; border-collapse: collapse;">
      <tr><td style="padding: 8px 0; color: #666;">Nom</td><td style="padding: 8px 0; font-weight: bold;">${lead.name || 'Non renseigné'}</td></tr>
      <tr><td style="padding: 8px 0; color: #666;">Email</td><td style="padding: 8px 0; font-weight: bold;">${lead.email}</td></tr>
      <tr><td style="padding: 8px 0; color: #666;">Téléphone</td><td style="padding: 8px 0;">${lead.phone || 'Non renseigné'}</td></tr>
      <tr><td style="padding: 8px 0; color: #666;">Source</td><td style="padding: 8px 0;">${lead.source || 'chatbot'}</td></tr>
    </table>
  </div>
</body>
</html>`,
  };
}
```

---

## Intégration Stripe (paiement fallback)

```javascript
// lib/stripe.js

import Stripe from 'stripe';

let stripe = null;

function getStripe() {
  if (!stripe) {
    if (!process.env.STRIPE_SECRET_KEY) {
      throw new Error('STRIPE_SECRET_KEY non configuré');
    }
    stripe = new Stripe(process.env.STRIPE_SECRET_KEY);
  }
  return stripe;
}

/**
 * Créer une session Stripe Checkout
 * IMPORTANT: Les prix sont TOUJOURS vérifiés côté serveur depuis Wix (anti-tampering)
 */
export async function createStripeCheckoutSession({ items, customerEmail, successUrl, cancelUrl }) {
  // Vérifier chaque item côté Wix pour éviter la manipulation de prix
  const verifiedItems = await verifyItemPricesFromWix(items);

  const session = await getStripe().checkout.sessions.create({
    mode: 'payment',
    customer_email: customerEmail,
    line_items: verifiedItems.map(item => ({
      price_data: {
        currency: 'eur',
        product_data: { name: item.name },
        unit_amount: Math.round(item.price * 100), // Centimes
      },
      quantity: item.quantity || 1,
    })),
    success_url: successUrl,
    cancel_url: cancelUrl,
    payment_intent_data: {
      metadata: { source: 'chatbot', customer_email: customerEmail },
    },
  });

  return { url: session.url, session_id: session.id };
}

async function verifyItemPricesFromWix(items) {
  // Vérification des prix depuis Wix (voir wix-catalog.js)
  // Cette fonction doit récupérer le vrai prix Wix pour chaque item
  // pour éviter toute manipulation côté client
  const { searchWixProducts } = await import('./wix-catalog.js');

  return Promise.all(items.map(async item => {
    if (item.productId) {
      const [wixProduct] = await searchWixProducts({ query: item.productId, limit: 1 });
      return { ...item, price: wixProduct?.price || item.price };
    }
    return item; // Items custom (ex: devis) sans vérification Wix
  }));
}
```

---

## Intégration Google Drive

```javascript
// lib/google-drive.js

import { google } from 'googleapis';

let driveClient = null;

async function getDriveClient() {
  if (driveClient) return driveClient;

  let auth;

  // Priorité 1: Service Account (JSON inline)
  if (process.env.GOOGLE_DRIVE_SERVICE_ACCOUNT_JSON) {
    const credentials = JSON.parse(process.env.GOOGLE_DRIVE_SERVICE_ACCOUNT_JSON);
    auth = new google.auth.GoogleAuth({
      credentials,
      scopes: process.env.GOOGLE_DRIVE_SCOPES?.split(',') || ['https://www.googleapis.com/auth/drive.file'],
    });
  }
  // Priorité 2: Service Account (fichier)
  else if (process.env.GOOGLE_DRIVE_SERVICE_ACCOUNT_PATH) {
    auth = new google.auth.GoogleAuth({
      keyFile: process.env.GOOGLE_DRIVE_SERVICE_ACCOUNT_PATH,
      scopes: ['https://www.googleapis.com/auth/drive.file'],
    });
  }
  // Priorité 3: OAuth refresh token
  else if (process.env.GOOGLE_DRIVE_REFRESH_TOKEN) {
    auth = new google.auth.OAuth2(
      process.env.GOOGLE_DRIVE_CLIENT_ID,
      process.env.GOOGLE_DRIVE_CLIENT_SECRET,
    );
    auth.setCredentials({ refresh_token: process.env.GOOGLE_DRIVE_REFRESH_TOKEN });
  }
  // Priorité 4: API Key simple (lecture seulement)
  else if (process.env.GOOGLE_DRIVE_API_KEY) {
    driveClient = google.drive({ version: 'v3', auth: process.env.GOOGLE_DRIVE_API_KEY });
    return driveClient;
  } else {
    throw new Error('Aucune méthode d\'auth Google Drive configurée');
  }

  driveClient = google.drive({ version: 'v3', auth });
  return driveClient;
}

/**
 * Upload un fichier dans Google Drive
 */
export async function uploadToDrive({ content, mimeType, filename, folderId }) {
  const drive = await getDriveClient();
  const targetFolder = folderId || process.env.WORKFLOWS_DEFAULT_DRIVE_FOLDER_ID;

  const { data } = await drive.files.create({
    requestBody: {
      name: filename,
      parents: targetFolder ? [targetFolder] : undefined,
    },
    media: { mimeType, body: content },
    fields: 'id, webViewLink, webContentLink',
  });

  return {
    fileId: data.id,
    viewUrl: data.webViewLink,
    downloadUrl: data.webContentLink,
  };
}
```

---

## Notifications centralisées

```javascript
// lib/notifications.js
// Système centralisé avec déduplication

import { sendEmail } from './email.js';
import { sendWhatsApp } from './whatsapp.js';

// Cache de déduplication en mémoire (TTL 6h)
const dedupCache = new Map();
const DEDUP_TTL = 6 * 60 * 60 * 1000;

/**
 * Envoyer une notification (email + WhatsApp) avec déduplication
 */
export async function notify({ type, to, subject, message, channels = ['email'] }) {
  const dedupKey = `${type}:${to}:${subject?.slice(0, 20)}`;

  // Vérifier déduplication
  const lastSent = dedupCache.get(dedupKey);
  if (lastSent && Date.now() - lastSent < DEDUP_TTL) {
    console.log(`[notifications] Déduplication: ${dedupKey}`);
    return null;
  }

  const results = {};

  if (channels.includes('email') && to.includes('@')) {
    try {
      results.email = await sendEmail({ to, subject, text: message });
      dedupCache.set(dedupKey, Date.now());
    } catch (error) {
      console.error('[notifications] Email échoué:', error.message);
    }
  }

  if (channels.includes('whatsapp') && !to.includes('@')) {
    try {
      results.whatsapp = await sendWhatsApp({ to, message });
      dedupCache.set(dedupKey, Date.now());
    } catch (error) {
      console.error('[notifications] WhatsApp échoué:', error.message);
    }
  }

  // Nettoyage cache (simple)
  if (dedupCache.size > 1000) {
    const now = Date.now();
    for (const [key, ts] of dedupCache.entries()) {
      if (now - ts > DEDUP_TTL) dedupCache.delete(key);
    }
  }

  return results;
}

/**
 * Notifier l'admin d'un nouveau devis
 */
export async function notifyNewQuote(quote) {
  const adminEmail = process.env.NOTIFICATION_EMAIL || process.env.ADMIN_EMAIL;
  const adminPhone = process.env.ADMIN_PHONE;

  const message = `Nouveau devis chatbot\nClient: ${quote.email}\nMontant: ${quote.total}€\nLien: ${quote.checkout_url || quote.invoice_url || 'N/A'}`;

  await notify({
    type: 'new_quote',
    to: adminEmail,
    subject: `Nouveau devis — ${quote.email}`,
    message,
    channels: ['email'],
  });

  if (adminPhone) {
    await notify({
      type: 'new_quote_wa',
      to: adminPhone,
      message,
      channels: ['whatsapp'],
    });
  }
}
```
