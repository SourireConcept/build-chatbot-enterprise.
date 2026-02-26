# 06 — WIZARDS FSM : Flux guidés multi-étapes

## Concept

Les wizards sont des **machines d'état finie (FSM)** qui guident l'utilisateur à travers un flux structuré (formulaire multi-étapes). Chaque wizard :
1. Est activé par un token d'action (`__TRACK_ORDER__`, `__CONTACT_FORM__`, etc.)
2. Stocke son état dans la session (`session.trackingState`, `session.contactFormState`, etc.)
3. Intercepte les messages suivants pour collecter les infos nécessaires
4. Peut être interrompu par des commandes urgentes (`annuler`, `cancel`, `urgent`)
5. Se termine et nettoie son état une fois complété

---

## Module `lib/ai/wizard-state-machine.js`

```javascript
// lib/ai/wizard-state-machine.js
// Dispatcher central des wizards

import { handleTrackingWizard } from './wizards/tracking-wizard.js';
import { handleContactWizard } from './wizards/contact-wizard.js';
import { handleCouponWizard } from './wizards/coupon-wizard.js';
import { handleEoriWizard } from './wizards/eori-wizard.js';
import { handleCheckoutAddressWizard } from './wizards/checkout-address-wizard.js';

// Mapping wizard_type → handler
const WIZARD_HANDLERS = {
  tracking: handleTrackingWizard,
  contact: handleContactWizard,
  coupon: handleCouponWizard,
  eori: handleEoriWizard,
  checkout_address: handleCheckoutAddressWizard,
};

/**
 * Retourne le wizard actif en session (ou null)
 */
export function getActiveWizard(session) {
  if (session?.trackingState?.active) return 'tracking';
  if (session?.contactFormState?.active) return 'contact';
  if (session?.couponState?.active) return 'coupon';
  if (session?.eoriState?.active) return 'eori';
  if (session?.checkoutAddressState?.active) return 'checkout_address';
  return null;
}

/**
 * Démarre un wizard
 */
export function startWizard(session, wizardType) {
  switch (wizardType) {
    case 'tracking':
      session.trackingState = { active: true, step: 'ask_order_id', attempts: 0 };
      break;
    case 'contact':
      session.contactFormState = { active: true, step: 'ask_name', data: {} };
      break;
    case 'coupon':
      session.couponState = { active: true, step: 'check_eligibility' };
      break;
    case 'eori':
      session.eoriState = { active: true, step: 'ask_country', data: {} };
      break;
    case 'checkout_address':
      session.checkoutAddressState = { active: true, step: 'ask_name', data: {} };
      break;
  }
}

/**
 * Dispatch vers le bon wizard
 */
export async function dispatchToWizard(message, session, language) {
  // Commande d'interruption universelle
  const INTERRUPT_COMMANDS = ['annuler', 'cancel', 'stop', 'quitter', 'exit', 'abort'];
  if (INTERRUPT_COMMANDS.some(cmd => message.toLowerCase().trim() === cmd)) {
    return interruptAllWizards(session, language);
  }

  const activeWizard = getActiveWizard(session);
  if (!activeWizard) return null;

  const handler = WIZARD_HANDLERS[activeWizard];
  if (!handler) return null;

  return handler(message, session, language);
}

function interruptAllWizards(session, language) {
  // Nettoyer tous les états wizard
  delete session.trackingState;
  delete session.contactFormState;
  delete session.couponState;
  delete session.eoriState;
  delete session.checkoutAddressState;

  const replies = {
    fr: 'Très bien, j\'ai annulé le formulaire en cours. Comment puis-je vous aider ?',
    en: 'Sure, I\'ve cancelled the current form. How can I help you?',
    de: 'Verstanden, ich habe das Formular abgebrochen. Wie kann ich Ihnen helfen?',
  };

  return {
    reply: replies[language] || replies.fr,
    action: null,
  };
}
```

---

## Wizard 1 : Suivi de commande

```javascript
// lib/ai/wizards/tracking-wizard.js

import { searchOrders } from '../../orders.js';

export async function handleTrackingWizard(message, session, language) {
  const state = session.trackingState;

  switch (state.step) {
    case 'ask_order_id': {
      // Vérifier si le message contient un numéro de commande
      const orderIdMatch = message.match(/\b([A-Z0-9]{6,20})\b/);
      const emailMatch = message.match(/\b[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}\b/);

      if (orderIdMatch) {
        state.orderId = orderIdMatch[1];
        state.step = 'searching';
        return searchAndReturnOrder(state, session, language);
      }

      if (emailMatch) {
        state.email = emailMatch[0];
        state.step = 'searching';
        return searchAndReturnOrder(state, session, language);
      }

      // Demander le numéro de commande
      const prompts = {
        fr: 'Pour suivre votre commande, merci de me donner votre **numéro de commande** ou votre **adresse email** de commande.',
        en: 'To track your order, please give me your **order number** or **email address**.',
      };

      return { reply: prompts[language] || prompts.fr, action: null };
    }

    case 'searching': {
      return searchAndReturnOrder(state, session, language);
    }

    default: {
      delete session.trackingState;
      return { reply: 'Une erreur s\'est produite. Réessayons.', action: null };
    }
  }
}

async function searchAndReturnOrder(state, session, language) {
  try {
    const orders = await searchOrders({
      orderId: state.orderId,
      email: state.email,
    });

    // Nettoyer l'état wizard
    delete session.trackingState;

    if (!orders || orders.length === 0) {
      const notFound = {
        fr: 'Je n\'ai pas trouvé de commande avec ces informations. Vérifiez le numéro ou contactez-nous.',
        en: 'No order found with this information. Please check the number or contact us.',
      };
      return { reply: notFound[language] || notFound.fr, action: null };
    }

    const order = orders[0];
    const statusMap = {
      fr: {
        PENDING: '⏳ En attente',
        PROCESSING: '🔄 En traitement',
        SHIPPED: '📦 Expédiée',
        DELIVERED: '✅ Livrée',
        CANCELLED: '❌ Annulée',
      },
    };

    const statusLabel = statusMap[language]?.[order.status] || order.status;
    const reply = `**Commande ${order.id}**\nStatut : ${statusLabel}\n${order.tracking_url ? `[Suivre le colis](${order.tracking_url})` : ''}`;

    return { reply, action: null };

  } catch (error) {
    delete session.trackingState;
    return { reply: 'Erreur lors de la recherche de commande. Veuillez réessayer.', action: null };
  }
}
```

---

## Wizard 2 : Formulaire de contact

```javascript
// lib/ai/wizards/contact-wizard.js

import { sendEmail } from '../../email.js';
import { syncLead } from '../../leads.js';

const STEPS = ['ask_name', 'ask_email', 'ask_message', 'confirm'];

export async function handleContactWizard(message, session, language) {
  const state = session.contactFormState;

  switch (state.step) {
    case 'ask_name': {
      if (message.trim().length >= 2) {
        state.data.name = message.trim();
        state.step = 'ask_email';
        return {
          reply: `Merci ${state.data.name} ! Quelle est votre **adresse email** pour que nous puissions vous répondre ?`,
          action: null,
        };
      }
      return { reply: 'Quel est votre prénom ou nom ?', action: null };
    }

    case 'ask_email': {
      const emailMatch = message.match(/\b[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}\b/);
      if (emailMatch) {
        state.data.email = emailMatch[0].toLowerCase();
        state.step = 'ask_message';
        return { reply: 'Parfait ! Quel est votre **message** ou votre demande ?', action: null };
      }
      return { reply: 'Merci d\'entrer une adresse email valide.', action: null };
    }

    case 'ask_message': {
      if (message.trim().length >= 5) {
        state.data.message = message.trim();
        state.step = 'confirm';
        return {
          reply: `Voici votre message :\n\n**Nom** : ${state.data.name}\n**Email** : ${state.data.email}\n**Message** : ${state.data.message}\n\nConfirmez-vous l'envoi ? (oui/non)`,
          action: null,
        };
      }
      return { reply: 'Merci de décrire votre demande en quelques mots.', action: null };
    }

    case 'confirm': {
      const isYes = /^(oui|yes|ok|confirme|envoyer|send|y)$/i.test(message.trim());
      const isNo = /^(non|no|annuler|cancel|n)$/i.test(message.trim());

      if (isYes) {
        // Envoyer l'email
        await submitContactForm(state.data);
        delete session.contactFormState;
        return {
          reply: `✅ Votre message a été envoyé ! Nous vous répondrons à **${state.data.email}** dans les plus brefs délais.`,
          action: null,
        };
      }

      if (isNo) {
        delete session.contactFormState;
        return { reply: 'Message annulé. Comment puis-je vous aider autrement ?', action: null };
      }

      return { reply: 'Répondez **oui** pour envoyer ou **non** pour annuler.', action: null };
    }
  }
}

async function submitContactForm(data) {
  const notifEmail = process.env.NOTIFICATION_EMAIL;
  if (notifEmail) {
    await sendEmail({
      to: notifEmail,
      subject: `Nouveau message de ${data.name} — Chatbot`,
      text: `Nom: ${data.name}\nEmail: ${data.email}\nMessage:\n${data.message}`,
    }).catch(console.error);
  }

  // Sync lead
  await syncLead({
    email: data.email,
    name: data.name,
    source: 'contact_wizard',
  }).catch(console.error);
}
```

---

## Wizard 3 : Code promo

```javascript
// lib/ai/wizards/coupon-wizard.js

import { generateCoupon, validateCouponEligibility } from '../../wix-coupon.js';

export async function handleCouponWizard(message, session, language) {
  const state = session.couponState;

  switch (state.step) {
    case 'check_eligibility': {
      const entities = session.entities || {};

      // Vérifier éligibilité (email connu = client identifié)
      const isEligible = Boolean(entities.email);

      if (isEligible) {
        state.step = 'generate';
        return generateCouponForUser(state, session, entities, language);
      }

      // Demander l'email pour vérifier l'éligibilité
      state.step = 'ask_email';
      return {
        reply: 'Pour vous offrir un code promo, j\'ai besoin de votre **email** pour vérifier votre éligibilité.',
        action: null,
      };
    }

    case 'ask_email': {
      const emailMatch = message.match(/\b[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}\b/);
      if (emailMatch) {
        state.email = emailMatch[0].toLowerCase();
        state.step = 'generate';
        return generateCouponForUser(state, session, { email: state.email }, language);
      }
      return { reply: 'Merci d\'entrer votre adresse email.', action: null };
    }

    case 'generated': {
      delete session.couponState;
      return { reply: 'Votre code promo a déjà été généré.', action: null };
    }
  }
}

async function generateCouponForUser(state, session, entities, language) {
  try {
    const coupon = await generateCoupon({
      type: 'PERCENTAGE',  // ou 'FIXED_AMOUNT'
      discountValue: 10,   // 10% de réduction
      expiryDays: 7,       // Valide 7 jours
      description: `Code promo chatbot — ${entities.email}`,
    });

    state.step = 'generated';
    delete session.couponState;

    return {
      reply: `🎉 Voici votre code promo exclusif : **\`${coupon.code}\`**\n\n- Réduction : ${coupon.discountValue}%\n- Valide : ${coupon.expiryDays} jours\n- Condition : commande min. ${coupon.minimumOrder || 0}€`,
      action: null,
    };
  } catch (error) {
    delete session.couponState;
    return { reply: 'Impossible de générer un code promo pour l\'instant. Réessayez plus tard.', action: null };
  }
}
```

---

## Wizard 4 : EORI (douanes)

```javascript
// lib/ai/wizards/eori-wizard.js
// Assistance EORI — procédures douanières par pays

const EORI_PROCEDURES = {
  FR: {
    name: 'France',
    url: 'https://pro.douane.gouv.fr/',
    steps: [
      '1. Rendez-vous sur le portail douane.gouv.fr',
      '2. Section "EORI" → Demande de numéro EORI',
      '3. Fournissez votre numéro SIRET/SIREN',
      '4. Délai d\'obtention : 3-5 jours ouvrables',
    ],
    requirements: ['Numéro SIRET/SIREN', 'Extrait Kbis', 'Justificatif d\'identité dirigeant'],
  },
  BE: {
    name: 'Belgique',
    url: 'https://finances.belgium.be/',
    steps: [
      '1. Via MyMinfin (finances.belgium.be)',
      '2. Section "Douane et accises"',
      '3. Demande automatique liée au numéro de TVA BE',
    ],
    requirements: ['Numéro TVA belge (BE0...)'],
  },
  DE: {
    name: 'Allemagne',
    url: 'https://www.zoll.de/',
    steps: [
      '1. Portail Zoll.de → EORI-Nummer',
      '2. Formulaire "Antrag auf Erteilung einer EORI-Nummer"',
      '3. Délai : 2-3 jours ouvrables',
    ],
    requirements: ['Umsatzsteuer-Identifikationsnummer (USt-IdNr.)', 'Handelsregisternummer'],
  },
  IT: {
    name: 'Italie',
    url: 'https://www.agenziadogane.gov.it/',
    steps: [
      '1. Portail Agenzia delle Dogane',
      '2. EORI → Richiesta numero EORI',
      '3. Délai : 5-7 jours ouvrables',
    ],
    requirements: ['Partita IVA', 'Codice fiscale'],
  },
  // Ajouter d'autres pays selon les besoins
};

export async function handleEoriWizard(message, session, language) {
  const state = session.eoriState;

  switch (state.step) {
    case 'ask_country': {
      const countryMap = {
        'france': 'FR', 'fr': 'FR', 'français': 'FR',
        'belgique': 'BE', 'belgium': 'BE', 'be': 'BE',
        'allemagne': 'DE', 'germany': 'DE', 'deutschland': 'DE', 'de': 'DE',
        'italie': 'IT', 'italy': 'IT', 'it': 'IT',
      };

      const lower = message.toLowerCase().trim();
      const countryCode = countryMap[lower];

      if (countryCode) {
        state.data.country = countryCode;
        state.step = 'show_procedure';
        return showEoriProcedure(state, language);
      }

      return {
        reply: 'Dans quel **pays** êtes-vous établi(e) ? (France, Belgique, Allemagne, Italie, autre)',
        action: null,
      };
    }

    case 'show_procedure': {
      return showEoriProcedure(state, language);
    }
  }
}

function showEoriProcedure(state, language) {
  const procedure = EORI_PROCEDURES[state.data.country];

  if (!procedure) {
    delete state.active;
    return {
      reply: `Pour obtenir votre numéro EORI dans votre pays, consultez le site des douanes locales. Votre pays est-il dans l'UE ? Si oui, contactez l'agence des douanes nationale.`,
      action: null,
    };
  }

  const steps = procedure.steps.join('\n');
  const requirements = procedure.requirements.join('\n- ');

  const reply = `## Obtenir votre numéro EORI — ${procedure.name}\n\n**Étapes :**\n${steps}\n\n**Documents requis :**\n- ${requirements}\n\n**Site officiel :** [Accéder](${procedure.url})`;

  delete state.active; // Fin du wizard EORI
  return { reply, action: { type: 'OPEN_URL', data: { url: procedure.url } } };
}
```

---

## Wizard 5 : Adresse checkout

```javascript
// lib/ai/wizards/checkout-address-wizard.js

import { createWixCheckout } from '../../wix-checkout.js';

const STEPS_ORDER = ['ask_name', 'ask_email', 'ask_phone', 'ask_country', 'ask_city', 'ask_zip', 'ask_address', 'confirm'];

export async function handleCheckoutAddressWizard(message, session, language) {
  const state = session.checkoutAddressState;
  const data = state.data;

  switch (state.step) {
    case 'ask_name':
      if (message.trim().length >= 2) {
        data.name = message.trim();
        state.step = 'ask_email';
        return ask('email', language);
      }
      return ask('name', language);

    case 'ask_email': {
      const m = message.match(/\b[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}\b/);
      if (m) { data.email = m[0]; state.step = 'ask_phone'; return ask('phone', language); }
      return { reply: 'Email invalide. Essayez de nouveau.', action: null };
    }

    case 'ask_phone':
      if (message.length > 6) {
        data.phone = message.trim();
        state.step = 'ask_country';
        return ask('country', language);
      }
      return ask('phone', language);

    case 'ask_country':
      data.country = message.trim();
      state.step = 'ask_city';
      return ask('city', language);

    case 'ask_city':
      data.city = message.trim();
      state.step = 'ask_zip';
      return ask('zip', language);

    case 'ask_zip':
      data.zip = message.trim();
      state.step = 'ask_address';
      return ask('address', language);

    case 'ask_address':
      data.address = message.trim();
      state.step = 'confirm';
      return {
        reply: `Récapitulatif :\n**${data.name}** — ${data.email}\n📍 ${data.address}, ${data.zip} ${data.city}, ${data.country}\n\nConfirmez ? (oui/non)`,
        action: null,
      };

    case 'confirm': {
      if (/^(oui|yes|ok)$/i.test(message.trim())) {
        return createCheckout(state, session, language);
      }
      delete session.checkoutAddressState;
      return { reply: 'Annulé. Comment puis-je vous aider ?', action: null };
    }
  }
}

function ask(field, language) {
  const questions = {
    name: 'Quel est votre **nom complet** ?',
    email: 'Votre **adresse email** ?',
    phone: 'Votre **numéro de téléphone** ?',
    country: 'Votre **pays** de livraison ?',
    city: 'Votre **ville** ?',
    zip: 'Votre **code postal** ?',
    address: 'Votre **adresse complète** (numéro + rue) ?',
  };
  return { reply: questions[field] || 'Continuons...', action: null };
}

async function createCheckout(state, session, language) {
  try {
    const cartItems = session.cartItems || [];

    const checkout = await createWixCheckout({
      items: cartItems,
      shippingAddress: state.data,
    });

    delete session.checkoutAddressState;

    return {
      reply: `✅ Votre panier est prêt ! Cliquez pour finaliser votre commande.`,
      action: {
        type: 'START_WIX_CHECKOUT',
        data: { checkout_url: checkout.url },
      },
    };
  } catch (error) {
    delete session.checkoutAddressState;
    return { reply: 'Impossible de créer le checkout. Réessayez ou contactez-nous.', action: null };
  }
}
```

---

## Utilitaires wizards

```javascript
// lib/ai/wizards/wizard-utils.js

/**
 * Vérifie si le message est une commande d'interruption
 */
export function isInterruptCommand(message) {
  const commands = ['annuler', 'cancel', 'stop', 'quitter', 'exit', 'abort', 'retour'];
  return commands.includes(message.toLowerCase().trim());
}

/**
 * Formatte un message wizard avec indicateur de progression
 */
export function formatWizardStep(step, totalSteps, message) {
  return `[${step}/${totalSteps}] ${message}`;
}

/**
 * Extrait un montant monétaire depuis un message
 */
export function extractAmount(message) {
  const match = message.match(/(\d+(?:[,\.]\d{1,2})?)\s*(?:€|EUR|euros?)?/i);
  if (match) return parseFloat(match[1].replace(',', '.'));
  return null;
}

/**
 * Valide un code postal français
 */
export function isValidFrenchPostalCode(code) {
  return /^\d{5}$/.test(code);
}
```
