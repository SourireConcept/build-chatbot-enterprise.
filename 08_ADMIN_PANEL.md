# 08 — ADMIN PANEL : Interface God Mode

## Concept

L'admin panel est une **SPA HTML/CSS/JS natif** (~600 Ko) sans aucun framework frontend. Il offre un accès complet à tous les aspects du système via une interface unifiée.

**Points clés :**
- Zéro dépendance frontend (pas de React, Vue, Angular)
- Auth JWT sécurisé (httpOnly cookie + postMessage one-shot)
- Thème personnalisable (palette marque via variables CSS)
- Composants réutilisables (NEO-style design system)

---

## Structure de `public/admin.html`

```html
<!DOCTYPE html>
<html lang="fr">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>[APP_NAME] — Admin</title>
  <style>
    /* ──────────────────────────────────
       DESIGN SYSTEM (variables CSS)
       ────────────────────────────────── */
    :root {
      --brand-color: [BRAND_COLOR];       /* ex: #3a86ff */
      --brand-light: [BRAND_COLOR_LIGHT]; /* ex: #dce8ff */
      --bg-dark: #0a0a0f;
      --bg-card: #16161e;
      --bg-input: #1e1e2e;
      --text-primary: #e2e8f0;
      --text-muted: #64748b;
      --border-color: #2d2d3f;
      --success: #10b981;
      --warning: #f59e0b;
      --danger: #ef4444;
      --font-mono: 'JetBrains Mono', 'Fira Code', monospace;
      --radius: 8px;
      --shadow: 0 4px 20px rgba(0,0,0,0.4);
    }

    /* ──────────────────────────────────
       RESET + BASE
       ────────────────────────────────── */
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body {
      background: var(--bg-dark);
      color: var(--text-primary);
      font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
      min-height: 100vh;
    }

    /* ──────────────────────────────────
       LAYOUT
       ────────────────────────────────── */
    .app-layout {
      display: grid;
      grid-template-columns: 240px 1fr;
      grid-template-rows: 56px 1fr;
      height: 100vh;
      overflow: hidden;
    }

    /* Header */
    .app-header {
      grid-column: 1 / -1;
      background: var(--bg-card);
      border-bottom: 1px solid var(--border-color);
      display: flex;
      align-items: center;
      padding: 0 20px;
      gap: 16px;
    }

    .app-header .logo {
      font-size: 18px;
      font-weight: 700;
      color: var(--brand-color);
    }

    .app-header .status-badge {
      margin-left: auto;
      display: flex;
      align-items: center;
      gap: 8px;
      font-size: 12px;
      color: var(--text-muted);
    }

    /* Sidebar navigation */
    .app-nav {
      background: var(--bg-card);
      border-right: 1px solid var(--border-color);
      overflow-y: auto;
      padding: 16px 0;
    }

    .nav-section {
      margin-bottom: 24px;
    }

    .nav-section-title {
      font-size: 10px;
      font-weight: 700;
      text-transform: uppercase;
      letter-spacing: 1px;
      color: var(--text-muted);
      padding: 0 16px 8px;
    }

    .nav-item {
      display: flex;
      align-items: center;
      gap: 10px;
      padding: 10px 16px;
      cursor: pointer;
      border-radius: 0;
      transition: background 0.15s;
      font-size: 14px;
      color: var(--text-muted);
      border: none;
      background: none;
      width: 100%;
      text-align: left;
    }

    .nav-item:hover { background: rgba(255,255,255,0.05); color: var(--text-primary); }
    .nav-item.active { background: rgba(var(--brand-color-rgb), 0.15); color: var(--brand-color); border-left: 3px solid var(--brand-color); }

    /* Main content */
    .app-main {
      overflow-y: auto;
      padding: 24px;
    }

    /* ──────────────────────────────────
       COMPOSANTS NEO
       ────────────────────────────────── */
    .neo-card {
      background: var(--bg-card);
      border: 1px solid var(--border-color);
      border-radius: var(--radius);
      padding: 20px;
      box-shadow: var(--shadow);
    }

    .neo-btn {
      display: inline-flex;
      align-items: center;
      gap: 8px;
      padding: 10px 20px;
      border-radius: var(--radius);
      font-size: 14px;
      font-weight: 600;
      cursor: pointer;
      border: none;
      transition: all 0.15s;
    }

    .neo-btn-primary {
      background: var(--brand-color);
      color: white;
    }
    .neo-btn-primary:hover { filter: brightness(1.1); }

    .neo-btn-ghost {
      background: transparent;
      border: 1px solid var(--border-color);
      color: var(--text-primary);
    }
    .neo-btn-ghost:hover { border-color: var(--brand-color); color: var(--brand-color); }

    .neo-input {
      width: 100%;
      background: var(--bg-input);
      border: 1px solid var(--border-color);
      border-radius: var(--radius);
      padding: 10px 14px;
      color: var(--text-primary);
      font-size: 14px;
      outline: none;
      transition: border-color 0.15s;
    }
    .neo-input:focus { border-color: var(--brand-color); }

    .neo-table { width: 100%; border-collapse: collapse; }
    .neo-table th, .neo-table td {
      padding: 12px 16px;
      text-align: left;
      border-bottom: 1px solid var(--border-color);
      font-size: 14px;
    }
    .neo-table th { color: var(--text-muted); font-weight: 600; font-size: 12px; text-transform: uppercase; }
    .neo-table tr:hover td { background: rgba(255,255,255,0.02); }

    .badge {
      display: inline-block;
      padding: 2px 8px;
      border-radius: 20px;
      font-size: 11px;
      font-weight: 600;
    }
    .badge-success { background: rgba(16,185,129,0.15); color: #10b981; }
    .badge-warning { background: rgba(245,158,11,0.15); color: #f59e0b; }
    .badge-danger { background: rgba(239,68,68,0.15); color: #ef4444; }
    .badge-info { background: rgba(59,130,246,0.15); color: #3b82f6; }

    /* ──────────────────────────────────
       SECTIONS SPÉCIFIQUES
       ────────────────────────────────── */

    /* Super Chat */
    .chat-container {
      display: flex;
      flex-direction: column;
      height: calc(100vh - 200px);
    }

    .chat-messages {
      flex: 1;
      overflow-y: auto;
      padding: 16px;
      display: flex;
      flex-direction: column;
      gap: 12px;
    }

    .chat-message {
      max-width: 80%;
      padding: 12px 16px;
      border-radius: var(--radius);
      font-size: 14px;
      line-height: 1.6;
    }

    .chat-message.user {
      background: var(--brand-color);
      color: white;
      align-self: flex-end;
    }

    .chat-message.assistant {
      background: var(--bg-input);
      border: 1px solid var(--border-color);
      align-self: flex-start;
    }

    .chat-input-area {
      padding: 16px;
      border-top: 1px solid var(--border-color);
      display: flex;
      gap: 12px;
    }

    /* Stats cards */
    .stats-grid {
      display: grid;
      grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
      gap: 16px;
      margin-bottom: 24px;
    }

    .stat-card {
      background: var(--bg-card);
      border: 1px solid var(--border-color);
      border-radius: var(--radius);
      padding: 20px;
    }

    .stat-value {
      font-size: 32px;
      font-weight: 700;
      color: var(--brand-color);
    }

    .stat-label {
      font-size: 13px;
      color: var(--text-muted);
      margin-top: 4px;
    }
  </style>
</head>

<body>
  <!-- Login Modal -->
  <div id="loginModal" class="modal-overlay" style="display:flex; position:fixed; inset:0; background:rgba(0,0,0,0.8); z-index:1000; align-items:center; justify-content:center;">
    <div class="neo-card" style="width:360px;">
      <h2 style="margin-bottom:20px; color:var(--brand-color);">[APP_NAME] — Admin</h2>
      <input type="password" id="passwordInput" class="neo-input" placeholder="Mot de passe admin" style="margin-bottom:12px;">
      <button class="neo-btn neo-btn-primary" style="width:100%;" onclick="login()">Se connecter</button>
      <div id="loginError" style="color:var(--danger); font-size:13px; margin-top:8px; display:none;"></div>
    </div>
  </div>

  <!-- App Layout -->
  <div class="app-layout" style="display:none;" id="appLayout">
    <!-- Header -->
    <header class="app-header">
      <span class="logo">[APP_NAME]</span>
      <div class="status-badge">
        <span id="statusDot" style="width:8px;height:8px;background:#10b981;border-radius:50%;display:inline-block;"></span>
        <span id="statusText">Connecté</span>
        <button class="neo-btn neo-btn-ghost" style="padding:4px 12px; font-size:12px;" onclick="logout()">Déconnexion</button>
      </div>
    </header>

    <!-- Sidebar -->
    <nav class="app-nav">
      <div class="nav-section">
        <div class="nav-section-title">Chatbot</div>
        <button class="nav-item active" onclick="showSection('superchat')">💬 Super Chat</button>
        <button class="nav-item" onclick="showSection('leads')">👥 Leads</button>
        <button class="nav-item" onclick="showSection('orders')">📦 Commandes</button>
        <button class="nav-item" onclick="showSection('debug')">🔍 Debug Chat</button>
      </div>
      <div class="nav-section">
        <div class="nav-section-title">Contenu</div>
        <button class="nav-item" onclick="showSection('prompt')">📝 Prompt</button>
        <button class="nav-item" onclick="showSection('facts')">🏢 Facts Entreprise</button>
        <button class="nav-item" onclick="showSection('knowledge')">📚 Base de connaissances</button>
      </div>
      <div class="nav-section">
        <div class="nav-section-title">Automation</div>
        <button class="nav-item" onclick="showSection('workflows')">⚙️ Workflows</button>
        <button class="nav-item" onclick="showSection('autoroute')">🤖 Autoroute IA</button>
        <button class="nav-item" onclick="showSection('social-queue')">📅 Queue Sociale</button>
      </div>
      <div class="nav-section">
        <div class="nav-section-title">Agent Runtime</div>
        <button class="nav-item" onclick="showSection('agent-control')">🎛️ Contrôle Agents</button>
        <button class="nav-item" onclick="showSection('objectives')">🎯 Objectifs</button>
        <button class="nav-item" onclick="showSection('runs')">📊 Runs & Timeline</button>
        <button class="nav-item" onclick="showSection('approvals')">✅ Approbations</button>
      </div>
      <div class="nav-section">
        <div class="nav-section-title">Système</div>
        <button class="nav-item" onclick="showSection('runtime-flags')">🏁 Flags Runtime</button>
        <button class="nav-item" onclick="showSection('security')">🔒 Sécurité</button>
        <button class="nav-item" onclick="showSection('logs')">📋 Logs Admin</button>
        <button class="nav-item" onclick="showSection('settings')">⚙️ Paramètres</button>
      </div>
    </nav>

    <!-- Main Content -->
    <main class="app-main" id="mainContent">
      <!-- Les sections sont injectées dynamiquement ici -->
    </main>
  </div>

  <script>
    // ──────────────────────────────────
    // AUTH
    // ──────────────────────────────────
    let adminToken = null;

    async function login() {
      const password = document.getElementById('passwordInput').value;
      if (!password) return;

      try {
        const res = await fetch('/api/admin/login', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ password }),
          credentials: 'include',
        });

        const data = await res.json();

        if (res.ok) {
          adminToken = data.token;
          document.getElementById('loginModal').style.display = 'none';
          document.getElementById('appLayout').style.display = 'grid';
          showSection('superchat');
          loadDashboardStats();
        } else {
          const error = document.getElementById('loginError');
          error.textContent = data.error || 'Erreur d\'authentification';
          error.style.display = 'block';
        }
      } catch (err) {
        console.error('Login error:', err);
      }
    }

    function logout() {
      adminToken = null;
      document.getElementById('loginModal').style.display = 'flex';
      document.getElementById('appLayout').style.display = 'none';
      document.getElementById('passwordInput').value = '';
    }

    async function apiFetch(url, options = {}) {
      return fetch(url, {
        ...options,
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${adminToken}`,
          ...options.headers,
        },
        credentials: 'include',
      });
    }

    // ──────────────────────────────────
    // NAVIGATION
    // ──────────────────────────────────
    function showSection(section) {
      document.querySelectorAll('.nav-item').forEach(el => el.classList.remove('active'));
      document.querySelector(`[onclick="showSection('${section}')"]`)?.classList.add('active');

      const main = document.getElementById('mainContent');

      switch (section) {
        case 'superchat': renderSuperChat(main); break;
        case 'leads': renderLeads(main); break;
        case 'orders': renderOrders(main); break;
        case 'prompt': renderPrompt(main); break;
        case 'facts': renderFacts(main); break;
        case 'workflows': renderWorkflows(main); break;
        case 'autoroute': renderAutoroute(main); break;
        case 'runtime-flags': renderRuntimeFlags(main); break;
        case 'security': renderSecurity(main); break;
        case 'agent-control': renderAgentControl(main); break;
        case 'objectives': renderObjectives(main); break;
        case 'runs': renderRuns(main); break;
        default: main.innerHTML = `<div class="neo-card"><h2>${section}</h2><p style="color:var(--text-muted)">Section en cours de développement</p></div>`;
      }
    }

    // ──────────────────────────────────
    // SUPER CHAT
    // ──────────────────────────────────
    const chatHistory = [];

    function renderSuperChat(container) {
      container.innerHTML = `
        <h2 style="margin-bottom:16px;">💬 Super Chat — God Mode</h2>
        <div class="neo-card chat-container">
          <div class="chat-messages" id="chatMessages"></div>
          <div class="chat-input-area">
            <input type="text" id="chatInput" class="neo-input" placeholder="Commande admin, question, outil..." onkeydown="if(event.key==='Enter')sendAdminMessage()">
            <button class="neo-btn neo-btn-primary" onclick="sendAdminMessage()">Envoyer</button>
          </div>
        </div>`;
      renderChatHistory();
    }

    function renderChatHistory() {
      const container = document.getElementById('chatMessages');
      if (!container) return;
      container.innerHTML = chatHistory.map(m => `
        <div class="chat-message ${m.role}">
          ${m.content.replace(/\n/g, '<br>').replace(/\*\*(.+?)\*\*/g, '<strong>$1</strong>').replace(/`(.+?)`/g, '<code>$1</code>')}
        </div>`).join('');
      container.scrollTop = container.scrollHeight;
    }

    async function sendAdminMessage() {
      const input = document.getElementById('chatInput');
      const message = input.value.trim();
      if (!message) return;

      input.value = '';
      chatHistory.push({ role: 'user', content: message });
      renderChatHistory();

      try {
        const res = await apiFetch('/api/admin/chat', {
          method: 'POST',
          body: JSON.stringify({ message }),
        });
        const data = await res.json();
        chatHistory.push({ role: 'assistant', content: data.reply || data.error || 'Erreur' });
        renderChatHistory();
      } catch (err) {
        chatHistory.push({ role: 'assistant', content: `❌ Erreur: ${err.message}` });
        renderChatHistory();
      }
    }

    // ──────────────────────────────────
    // LEADS
    // ──────────────────────────────────
    async function renderLeads(container) {
      container.innerHTML = `
        <h2 style="margin-bottom:16px;">👥 Leads</h2>
        <div class="neo-card">
          <div id="leadsTable" style="color:var(--text-muted)">Chargement...</div>
        </div>`;

      const res = await apiFetch('/api/admin/leads');
      const data = await res.json();
      const leads = data.leads || [];

      document.getElementById('leadsTable').innerHTML = `
        <table class="neo-table">
          <thead><tr><th>Nom</th><th>Email</th><th>Téléphone</th><th>Source</th><th>Score</th><th>Date</th></tr></thead>
          <tbody>${leads.map(l => `
            <tr>
              <td>${l.name || '—'}</td>
              <td>${l.email}</td>
              <td>${l.phone || '—'}</td>
              <td><span class="badge badge-info">${l.source || 'chatbot'}</span></td>
              <td>${l.score || 0}</td>
              <td>${new Date(l.created_at).toLocaleDateString('fr-FR')}</td>
            </tr>`).join('')}
          </tbody>
        </table>`;
    }

    // ──────────────────────────────────
    // PROMPT STORE
    // ──────────────────────────────────
    async function renderPrompt(container) {
      container.innerHTML = `
        <h2 style="margin-bottom:16px;">📝 Prompt Store</h2>
        <div class="neo-card">
          <p style="color:var(--text-muted);margin-bottom:12px;">Prompt principal du chatbot (DB-first, prioritaire sur le code)</p>
          <textarea id="promptContent" class="neo-input" rows="20" style="font-family:var(--font-mono);font-size:13px;resize:vertical;">Chargement...</textarea>
          <div style="margin-top:12px;display:flex;gap:8px;">
            <button class="neo-btn neo-btn-primary" onclick="savePrompt()">💾 Sauvegarder</button>
            <button class="neo-btn neo-btn-ghost" onclick="renderPrompt(document.getElementById('mainContent'))">↩ Recharger</button>
          </div>
        </div>`;

      const res = await apiFetch('/api/admin/prompt?name=chatbot_main');
      const data = await res.json();
      document.getElementById('promptContent').value = data.content || '';
    }

    async function savePrompt() {
      const content = document.getElementById('promptContent').value;
      await apiFetch('/api/admin/prompt', {
        method: 'POST',
        body: JSON.stringify({ name: 'chatbot_main', content }),
      });
      alert('✅ Prompt sauvegardé !');
    }

    // ──────────────────────────────────
    // RUNTIME FLAGS
    // ──────────────────────────────────
    async function renderRuntimeFlags(container) {
      const res = await apiFetch('/api/admin/runtime-flags');
      const flags = await res.json();

      container.innerHTML = `
        <h2 style="margin-bottom:16px;">🏁 Flags Runtime (sans redéploiement)</h2>
        <div class="neo-card">
          <div style="display:grid;gap:16px;">
            ${Object.entries(flags).map(([key, value]) => `
              <div style="display:flex;align-items:center;justify-content:space-between;padding:12px;background:var(--bg-input);border-radius:var(--radius);">
                <div>
                  <div style="font-weight:600;">${key}</div>
                  <div style="font-size:12px;color:var(--text-muted);">${getFlagDescription(key)}</div>
                </div>
                <label style="position:relative;display:inline-block;width:48px;height:24px;">
                  <input type="checkbox" ${value ? 'checked' : ''} onchange="toggleFlag('${key}', this.checked)" style="opacity:0;width:0;height:0;">
                  <span style="position:absolute;cursor:pointer;inset:0;background:${value ? 'var(--brand-color)' : 'var(--border-color)'};border-radius:24px;transition:0.3s;"></span>
                </label>
              </div>`).join('')}
          </div>
        </div>`;
    }

    async function toggleFlag(key, value) {
      await apiFetch('/api/admin/runtime-flags', {
        method: 'POST',
        body: JSON.stringify({ key, value }),
      });
    }

    function getFlagDescription(key) {
      const descriptions = {
        v3KillSwitch: 'Basculer sur le moteur legacy si V3 défaillant',
        conversationGuardEnabled: 'Activer le guard de scope conversationnel',
        roleLockEnabled: 'Empêcher les tentatives de jailbreak',
        topicLockEnabled: 'Verrouiller le sujet sur N tours',
        socialApprovalRequired: 'Exiger approbation avant publication sociale',
      };
      return descriptions[key] || key;
    }

    // ──────────────────────────────────
    // AGENT CONTROL
    // ──────────────────────────────────
    async function renderAgentControl(container) {
      const res = await apiFetch('/api/admin/agent/registry');
      const { agents } = await res.json();

      container.innerHTML = `
        <h2 style="margin-bottom:16px;">🎛️ Contrôle Agents</h2>
        <div style="display:grid;gap:16px;">
          ${(agents || []).map(agent => `
            <div class="neo-card" style="display:flex;justify-content:space-between;align-items:center;">
              <div>
                <div style="font-weight:700;">${agent.name}</div>
                <div style="font-size:13px;color:var(--text-muted);">${agent.description || ''}</div>
              </div>
              <div style="display:flex;align-items:center;gap:12px;">
                <span class="badge ${agent.enabled ? 'badge-success' : 'badge-danger'}">${agent.enabled ? 'Actif' : 'Inactif'}</span>
                <button class="neo-btn neo-btn-ghost" onclick="toggleAgent('${agent.name}', ${!agent.enabled})">
                  ${agent.enabled ? 'Désactiver' : 'Activer'}
                </button>
              </div>
            </div>`).join('')}
        </div>`;
    }

    async function toggleAgent(name, enabled) {
      await apiFetch('/api/admin/agent/registry', {
        method: 'POST',
        body: JSON.stringify({ agent_name: name, enabled }),
      });
      renderAgentControl(document.getElementById('mainContent'));
    }

    // ──────────────────────────────────
    // INIT
    // ──────────────────────────────────
    async function loadDashboardStats() {
      // Chargement optionnel de statistiques initiales
    }

    // Entrée clavier pour login
    document.getElementById('passwordInput').addEventListener('keydown', e => {
      if (e.key === 'Enter') login();
    });
  </script>
</body>
</html>
```

---

## Routes Admin (api/index.js — extraits)

```javascript
// Routes admin dans api/index.js
import express from 'express';
import { requireAuth, requireCronSecret, generateToken, verifyAdminPassword } from '../lib/jwt.js';
import { handleAdminRequest } from '../lib/admin-ai.js';
import { supabase } from '../lib/supabase.js';

const router = express.Router();

// ── LOGIN ──
router.post('/admin/login', async (req, res) => {
  const { password } = req.body;

  // Rate limit login (anti-bruteforce)
  const { allowed } = await checkRateLimit(`login:${req.ip}`, 'login');
  if (!allowed) return res.status(429).json({ error: 'Trop de tentatives' });

  try {
    const valid = await verifyAdminPassword(password);
    if (!valid) return res.status(401).json({ error: 'Mot de passe incorrect' });

    const token = generateToken({ role: 'admin' });

    // Cookie httpOnly
    res.cookie('admin_token', token, {
      httpOnly: true,
      secure: process.env.NODE_ENV === 'production',
      sameSite: 'strict',
      maxAge: 24 * 60 * 60 * 1000, // 24h
    });

    return res.json({ token, expires_in: '24h' });

  } catch (error) {
    return res.status(500).json({ error: 'Erreur serveur' });
  }
});

// ── SUPER CHAT ──
router.post('/admin/chat', requireAuth, async (req, res) => {
  const { message } = req.body;
  const result = await handleAdminRequest(message, req.admin);
  return res.json(result);
});

// ── LEADS ──
router.get('/admin/leads', requireAuth, async (req, res) => {
  const { data: leads } = await supabase
    .from('app_leads')
    .select('*')
    .order('created_at', { ascending: false })
    .limit(100);
  return res.json({ leads: leads || [] });
});

// ── PROMPT STORE ──
router.get('/admin/prompt', requireAuth, async (req, res) => {
  const name = req.query.name || 'chatbot_main';
  const { data } = await supabase
    .from('app_prompt_store')
    .select('content, version, updated_at')
    .eq('name', name)
    .single();
  return res.json(data || { content: '' });
});

router.post('/admin/prompt', requireAuth, async (req, res) => {
  const { name = 'chatbot_main', content } = req.body;
  await supabase.from('app_prompt_store').upsert(
    { name, content, updated_at: new Date().toISOString() },
    { onConflict: 'name' }
  );
  return res.json({ success: true });
});

// ── RUNTIME FLAGS ──
router.get('/admin/runtime-flags', requireAuth, async (req, res) => {
  const { data } = await supabase.from('app_runtime_state').select('key, value');
  const flags = Object.fromEntries((data || []).map(r => [r.key, r.value]));
  return res.json(flags);
});

router.post('/admin/runtime-flags', requireAuth, async (req, res) => {
  const { key, value } = req.body;
  await supabase.from('app_runtime_state').upsert(
    { key, value, updated_at: new Date().toISOString() },
    { onConflict: 'key' }
  );
  return res.json({ success: true });
});

// ── AGENT REGISTRY ──
router.get('/admin/agent/registry', requireAuth, async (req, res) => {
  const { data: agents } = await supabase.from('app_agent_registry').select('*').order('name');
  return res.json({ agents: agents || [] });
});

router.post('/admin/agent/registry', requireAuth, async (req, res) => {
  const { agent_name, enabled } = req.body;
  await supabase.from('app_agent_registry').update({ enabled }).eq('name', agent_name);
  return res.json({ success: true });
});
```

---

## Module `lib/admin-ai.js`

```javascript
// lib/admin-ai.js
// Agent Admin God Mode

import { generateWithHF } from './hf-client-v2.js';
import { generateWithFallback } from './llm-fallback.js';
import { executeAdminTool } from './admin-tools.js';
import { supabase } from './supabase.js';

const ADMIN_SYSTEM_PROMPT = `Tu es l'assistant IA de l'admin de [APP_NAME]. Tu as accès à des outils pour gérer la base de données, envoyer des emails, interagir avec les APIs.

IMPORTANT:
- Réponds TOUJOURS avec un JSON valide: {"thought": "...", "tool": "nom_outil", "args": {...}, "reply": "..."}
- Si aucun outil n'est nécessaire: {"thought": "...", "tool": null, "args": null, "reply": "..."}
- Sois précis et concis dans ton reply
- "thought" est ton raisonnement interne (non affiché à l'utilisateur)

Outils disponibles: read_database, execute_sql, send_email, send_whatsapp, read_emails, wix_search_products, wix_search_orders, wix_create_coupon, manage_workflow, generate_content, security_scan`;

export async function handleAdminRequest(message, adminPayload = {}) {
  const model = process.env.HF_MODEL_HEAVY || 'Qwen/Qwen2.5-72B-Instruct';

  try {
    const response = await generateWithHF({
      model,
      messages: [
        { role: 'system', content: ADMIN_SYSTEM_PROMPT },
        { role: 'user', content: message },
      ],
      max_tokens: 2048,
      temperature: 0.3,
    });

    const content = response.content || response;

    // Parser le JSON
    const jsonMatch = content.match(/\{[\s\S]*\}/);
    if (!jsonMatch) {
      return { reply: content, tool: null, result: null };
    }

    const parsed = JSON.parse(jsonMatch[0]);

    // Exécuter l'outil si présent
    let toolResult = null;
    if (parsed.tool) {
      toolResult = await executeAdminTool(parsed.tool, parsed.args || {});

      // Logger dans admin_logs
      await supabase.from('app_admin_logs').insert({
        level: 'info',
        category: 'admin_chat',
        message: `Tool: ${parsed.tool}`,
        metadata: { args: parsed.args, result: toolResult, message: message.slice(0, 100) },
      });
    }

    return {
      thought: parsed.thought,
      tool: parsed.tool,
      args: parsed.args,
      result: toolResult,
      reply: parsed.reply || 'Opération effectuée.',
    };

  } catch (error) {
    console.error('[admin-ai] Erreur:', error);
    return { reply: `Erreur: ${error.message}`, tool: null, result: null };
  }
}
```
