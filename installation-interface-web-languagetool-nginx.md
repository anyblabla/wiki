---
title: Interface web LanguageTool (frontend statique)
description: Apprenez à installer une interface web statique pour LanguageTool sur Debian ou Ubuntu. Ce guide détaille la mise en place de Nginx et la configuration via Nginx Proxy Manager.
published: false
date: 2026-03-29T00:32:57.386Z
tags: nginx, npm, debian, ubuntu, auto-hébergement, debian13, debian 12, languagetool, open-source, ubuntu 22.04, ubuntu 24.04
editor: markdown
dateCreated: 2026-03-29T00:29:32.235Z
---

### Prérequis
- Une VM ou LXC sous Debian 12/13 ou Ubuntu 22.04/24.04
- Nginx Proxy Manager déjà installé et fonctionnel
- Votre instance LanguageTool (API) opérationnelle (ex. `languagetool.votredomaine.com`)
- Un nom de domaine pointant vers votre Nginx Proxy Manager (ex. `languagetool-web.votredomaine.com`)

---

### Étape 1 : Installation de NGINX sur la VM/LXC

Connectez-vous en **root** à votre machine et exécutez :

```bash
apt update && apt install nginx -y
```

---

### Étape 2 : Préparation du dossier de l’interface

```bash
rm /var/www/html/index.html
chown -R www-data:www-data /var/www/html
```

---

### Étape 3 : Placement des fichiers

**1. Logo LanguageTool (`languagetool.svg`)**

Téléchargez le fichier SVG officiel LanguageTool et placez-le dans le dossier :

![languagetool.svg](/installation-interface-web-languagetool-nginx/languagetool.svg)

```bash
mv languagetool.svg /var/www/html/
```

**2. Création du fichier `index.html`**

```bash
nano /var/www/html/index.html
```

Collez **exactement** le code suivant :

```html
<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>LanguageTool — Correcteur orthographique</title>
    <link rel="icon" href="languagetool.svg" type="image/svg+xml">
    <style>
        /* Variables de couleurs adaptatives */
        :root {
            --bg-body: #f8f9fa;
            --bg-editor: #ffffff;
            --text-main: #333333;
            --text-title: #222222;
            --border-color: #dddddd;
            --header-bg: #1565c0;
            --status-gray: #777777;
            --hint-color: #666666;
            --popup-bg: #ffffff;
            --popup-shadow: rgba(0,0,0,0.18);
            --api-header-bg: #f1f3f4;
        }
        @media (prefers-color-scheme: dark) {
            :root {
                --bg-body: #121212;
                --bg-editor: #1e1e1e;
                --text-main: #e0e0e0;
                --text-title: #ffffff;
                --border-color: #444444;
                --header-bg: #0d47a1;
                --status-gray: #aaaaaa;
                --hint-color: #999999;
                --popup-bg: #252525;
                --popup-shadow: rgba(0,0,0,0.5);
                --api-header-bg: #2d2d2d;
            }
        }
        * { box-sizing: border-box; margin: 0; padding: 0; }
        body {
            font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
            background: var(--bg-body);
            color: var(--text-main);
            min-height: 100vh;
            display: flex;
            flex-direction: column;
            transition: background 0.3s, color 0.3s;
        }
        header {
            background: var(--header-bg);
            color: white;
            padding: 0 2rem;
            height: 56px;
            display: flex;
            align-items: center;
            justify-content: space-between;
            box-shadow: 0 2px 4px rgba(0,0,0,0.2);
        }
        header .logo {
            font-size: 1.2rem;
            font-weight: 600;
            display: flex;
            align-items: center;
            gap: 0.5rem;
        }
        header .logo img { width: 28px; height: 28px; display: block; flex-shrink: 0; }
        header nav a {
            color: rgba(255,255,255,0.85);
            text-decoration: none;
            margin-left: 1.5rem;
            font-size: 0.9rem;
        }
        header nav a:hover { color: white; }
        main {
            flex: 1;
            max-width: 900px;
            width: 100%;
            margin: 0 auto;
            padding: 3rem 1.5rem 2rem;
        }
        h1 {
            text-align: center;
            font-size: 2rem;
            font-weight: 400;
            margin-bottom: 2rem;
            color: var(--text-title);
        }
        .lang-bar {
            display: flex;
            align-items: center;
            gap: 0.5rem;
            margin-bottom: 0.5rem;
            font-size: 0.9rem;
            color: var(--status-gray);
        }
        .lang-bar select {
            border: none;
            background: transparent;
            font-size: 0.9rem;
            font-weight: 600;
            color: #42a5f5;
            cursor: pointer;
            outline: none;
        }
        .editor-wrapper {
            background: var(--bg-editor);
            border: 1px solid var(--border-color);
            border-radius: 6px;
            min-height: 220px;
            position: relative;
            padding: 1rem;
            font-size: 1rem;
            line-height: 1.7;
            cursor: text;
        }
        #editor {
            outline: none;
            min-height: 180px;
            white-space: pre-wrap;
            word-wrap: break-word;
            color: var(--text-main);
        }
        #editor:empty::before {
            content: "Saisissez ou collez votre texte ici…";
            color: #888;
        }
        .editor-footer {
            display: flex;
            justify-content: space-between;
            align-items: center;
            margin-top: 0.5rem;
            font-size: 0.8rem;
            color: #888;
        }
        .btn-clear {
            background: none;
            border: none;
            color: #888;
            cursor: pointer;
            font-size: 0.8rem;
            display: flex;
            align-items: center;
            gap: 0.3rem;
        }
        .btn-clear:hover { color: #e53935; }
        mark.lt-error {
            background: rgba(229, 57, 53, 0.2);
            border-bottom: 2px solid #ff5252;
            border-radius: 2px;
            cursor: pointer;
            padding: 0 1px;
        }
        mark.lt-warning {
            background: rgba(255, 152, 0, 0.2);
            border-bottom: 2px solid #ffb74d;
            border-radius: 2px;
            cursor: pointer;
            padding: 0 1px;
        }
        #popup {
            display: none;
            position: fixed;
            z-index: 1000;
            background: var(--popup-bg);
            border: 1px solid var(--border-color);
            border-radius: 8px;
            box-shadow: 0 4px 20px var(--popup-shadow);
            min-width: 260px;
            max-width: 360px;
            overflow: hidden;
            color: var(--text-main);
        }
        #popup.visible { display: block; }
        .popup-header {
            padding: 0.75rem 1rem;
            border-bottom: 1px solid var(--border-color);
            font-size: 0.85rem;
            color: var(--status-gray);
            background: var(--api-header-bg);
        }
        .popup-message { padding: 0.75rem 1rem; font-size: 0.9rem; font-weight: 500; }
        .popup-suggestions { padding: 0 1rem 0.75rem; display: flex; flex-wrap: wrap; gap: 0.4rem; }
        .popup-suggestions button {
            background: #1565c0;
            border: none;
            color: white;
            padding: 0.3rem 0.7rem;
            border-radius: 4px;
            font-size: 0.85rem;
            cursor: pointer;
        }
        .popup-ignore {
            display: block; width: 100%; text-align: center; padding: 0.5rem;
            font-size: 0.8rem; color: var(--status-gray); background: none;
            border: none; border-top: 1px solid var(--border-color); cursor: pointer;
        }
        .btn-check {
            display: block; margin: 1.5rem auto 0.5rem auto;
            background: #1565c0; color: white; border: none;
            padding: 0.85rem 2.5rem; font-size: 0.9rem; font-weight: 600;
            border-radius: 4px; cursor: pointer; transition: background 0.2s;
        }
        .btn-check:hover { background: #0d47a1; }
        #status { text-align: center; font-size: 0.85rem; min-height: 1.5rem; margin-bottom: 0.5rem; }
        #status.error-count { color: #ff5252; font-weight: 600; }
        #status.success { color: #66bb6a; font-weight: 600; }
        .api-section { margin-top: 2.5rem; display: none; }
        .api-section.visible { display: block; }
        .api-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 1.5rem; }
        @media (max-width: 600px) { .api-grid { grid-template-columns: 1fr; } }
        .api-box { background: #1e1e2e; border: 1px solid var(--border-color); border-radius: 6px; overflow: hidden; }
        .api-box-header {
            padding: 0.6rem 1rem; background: var(--api-header-bg);
            border-bottom: 1px solid var(--border-color);
            font-size: 0.8rem; font-weight: 600; color: var(--status-gray);
            text-transform: uppercase;
        }
        .api-box pre {
            padding: 1rem; font-size: 0.78rem; line-height: 1.6;
            overflow-x: auto; max-height: 320px; color: #cdd6f4; margin: 0;
            white-space: pre-wrap;
        }
        .json-key { color: #89dceb; } .json-str { color: #a6e3a1; }
        .js-keyword { color: #cba6f7; } .js-string { color: #a6e3a1; }
        footer {
            background: var(--header-bg); color: rgba(255,255,255,0.85);
            text-align: center; padding: 2rem; font-size: 0.85rem; margin-top: 3rem;
        }
        footer a { color: white; text-decoration: underline; }

        /* --- PERSO 1 : Logo cliquable en bas à gauche --- */
        #custom-logo {
            position: fixed;
            bottom: 15px;
            left: 15px;
            z-index: 10000;
            transition: transform 0.3s ease;
        }
        @media (max-width: 767px) {
            #custom-logo { display: none; }
        }
        #custom-logo:hover { transform: scale(1.1); }
        #custom-logo img {
            height: auto;
            border-radius: 5px;
            box-shadow: 0 2px 8px rgba(0,0,0,0.3);
            cursor: pointer;
        }
    </style>
</head>
<body>

<!-- PERSO 1 : Logo cliquable en bas à gauche (supprimer toute la balise <a> si non désiré) -->
<a href="https://votresite.com" id="custom-logo" target="_blank">
    <img src="https://votresite.com/logo.png" alt="Logo personnalisé" width="50" height="50">
</a>

<header>
    <div class="logo">
        <img src="languagetool.svg" alt="" width="28" height="28">
        LanguageTool
    </div>
    <nav>
        <a href="https://languagetool.org/http-api/swagger-ui/#/default" target="_blank">Doc Officielle</a>
        <!-- PERSO 2 : Lien vers ton Swagger personnel (supprimer la ligne si non désiré) -->
        <a href="https://swagger.votredomaine.com" target="_blank" title="Accès LAN/VPN uniquement" style="opacity: 0.7;">Mon Swagger</a>
    </nav>
</header>

<main>
    <h1>Correcteur orthographique & grammatical</h1>
    <div class="lang-bar">
        Langue :
        <select id="langSelect">
            <option value="fr">Français</option>
            <option value="en-US">Anglais (US)</option>
            <option value="en-GB">Anglais (UK)</option>
            <option value="de">Allemand</option>
            <option value="es">Espagnol</option>
            <option value="it">Italien</option>
            <option value="pt">Portugais</option>
            <option value="nl">Néerlandais</option>
        </select>
    </div>
    <div class="editor-wrapper">
        <div id="editor" contenteditable="true" spellcheck="false"></div>
        <div class="editor-footer">
            <span id="charCount">0 caractère</span>
            <button class="btn-clear" onclick="clearEditor()">✕ Effacer</button>
        </div>
    </div>
    <button class="btn-check" id="checkBtn" onclick="checkText()">CORRIGER LE TEXTE</button>
    <div id="status"></div>
    <!-- PERSO 3 : Message d'astuce (supprimer tout le bloc <p> si non désiré) -->
    <p style="text-align: center; font-size: 0.8rem; color: var(--hint-color); margin-bottom: 1.5rem;">
        <i>Astuce : Cliquez d'abord sur "CORRIGER LE TEXTE", puis sur les mots soulignés pour voir les suggestions.</i>
    </p>
    <div class="api-section" id="apiSection">
        <div class="api-grid">
            <div class="api-box">
                <div class="api-box-header">Requête</div>
                <pre id="apiRequest"></pre>
            </div>
            <div class="api-box">
                <div class="api-box-header">Réponse</div>
                <pre id="apiResponse"></pre>
            </div>
        </div>
    </div>
</main>

<div id="popup">
    <div class="popup-header" id="popupCategory"></div>
    <div class="popup-message" id="popupMessage"></div>
    <div class="popup-suggestions" id="popupSuggestions"></div>
    <button class="popup-ignore" onclick="closePopup()">Ignorer</button>
</div>

<footer>
    <strong>LanguageTool Interface</strong><br>
    Correcteur orthographique et grammatical libre<br>
    Propulsé par <a href="https://languagetool.org" target="_blank">LanguageTool</a>
</footer>

<script>
    const editor = document.getElementById('editor');
    const charCount = document.getElementById('charCount');
    const status = document.getElementById('status');
    const popup = document.getElementById('popup');
    const apiSection = document.getElementById('apiSection');
    const apiRequest = document.getElementById('apiRequest');
    const apiResponse = document.getElementById('apiResponse');
    let matches = [];

    editor.addEventListener('input', () => {
        const len = editor.innerText.length;
        charCount.textContent = len + (len <= 1 ? ' caractère' : ' caractères');
        if (editor.querySelector('mark')) {
            stripMarks();
            status.textContent = '';
            status.className = '';
        }
    });

    function clearEditor() {
        editor.innerText = '';
        charCount.textContent = '0 caractère';
        status.textContent = '';
        status.className = '';
        matches = [];
        closePopup();
        apiSection.classList.remove('visible');
    }

    function stripMarks() { editor.innerHTML = editor.innerText; }

    document.addEventListener('click', (e) => {
        if (!popup.contains(e.target) && !e.target.closest('mark')) closePopup();
    });

    function closePopup() { popup.classList.remove('visible'); }

    async function checkText() {
        const text = editor.innerText.trim();
        if (!text) return;

        const lang = document.getElementById('langSelect').value;
        const btn = document.getElementById('checkBtn');
        btn.disabled = true;
        btn.textContent = 'Analyse en cours…';
        status.textContent = '';
        status.className = '';
        closePopup();
        editor.innerText = text;

        const params = new URLSearchParams({ language: lang, text });
        updateRequestBox(lang, text);

        try {
            // PERSO 4 : URL de votre API LanguageTool – À MODIFIER
            const res = await fetch('https://languagetool.votredomaine.com/v2/check', {
                method: 'POST',
                headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
                body: params
            });
            const data = await res.json();
            matches = data.matches || [];

            updateResponseBox(data);
            apiSection.classList.add('visible');

            if (matches.length === 0) {
                status.textContent = '✅ Aucune erreur détectée !';
                status.className = 'success';
            } else {
                status.textContent = matches.length + (matches.length === 1 ? ' erreur détectée' : ' erreurs détectées');
                status.className = 'error-count';
                applyHighlights(text, matches);
            }
        } catch (e) {
            status.textContent = '❌ Erreur de connexion à l\'API.';
            status.className = 'error-count';
        }
        btn.disabled = false;
        btn.textContent = 'CORRIGER LE TEXTE';
    }

    function applyHighlights(text, matches) {
        const sorted = [...matches].sort((a, b) => b.offset - a.offset);
        let result = text;
        sorted.forEach(m => {
            const idx = matches.indexOf(m);
            const cls = (m.rule.issueType === 'style' || m.rule.issueType === 'locale-violation') ? 'lt-warning' : 'lt-error';
            const before = result.substring(0, m.offset);
            const wrong = result.substring(m.offset, m.offset + m.length);
            const after = result.substring(m.offset + m.length);
            result = before + `<mark class="${cls}" data-index="${idx}" onclick="showPopup(event,${idx})">${escapeHtml(wrong)}</mark>` + after;
        });
        result = result.replace(/\n/g, '<br>');
        editor.innerHTML = result;
    }

    function escapeHtml(str) { return str.replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;'); }

    function showPopup(e, index) {
        e.stopPropagation();
        const m = matches[index];
        if (!m) return;
        document.getElementById('popupCategory').textContent = m.rule.category.name || 'Erreur';
        document.getElementById('popupMessage').textContent = m.message;
        const suggestionsEl = document.getElementById('popupSuggestions');
        suggestionsEl.innerHTML = '';
        if (m.replacements && m.replacements.length > 0) {
            m.replacements.slice(0, 5).forEach(r => {
                const btn = document.createElement('button');
                btn.textContent = r.value;
                btn.onclick = () => applySuggestion(index, r.value);
                suggestionsEl.appendChild(btn);
            });
        }
        const rect = e.target.getBoundingClientRect();
        popup.classList.add('visible');
        let top = rect.bottom + window.scrollY + 8;
        let left = rect.left + window.scrollX;
        if (left + 370 > window.innerWidth) left = window.innerWidth - 375;
        popup.style.top = top + 'px'; popup.style.left = left + 'px';
    }

    function applySuggestion(index, replacement) {
        const m = matches[index];
        const currentText = editor.innerText;
        const newText = currentText.substring(0, m.offset) + replacement + currentText.substring(m.offset + m.length);
        const diff = replacement.length - m.length;
        matches.forEach((match, i) => { if (i !== index && match.offset > m.offset) match.offset += diff; });
        matches.splice(index, 1);
        editor.innerText = newText;
        closePopup();
        if (matches.length === 0) {
            status.textContent = '✅ Toutes les erreurs ont été corrigées !';
            status.className = 'success';
        } else {
            status.textContent = matches.length + (matches.length === 1 ? ' erreur restante' : ' erreurs restantes');
            applyHighlights(newText, matches);
        }
    }

    function updateRequestBox(lang, text) {
        const displayText = text.length > 80 ? text.substring(0, 80) + '…' : text;
        // PERSO 4 : URL de votre API LanguageTool – À MODIFIER
        apiRequest.innerHTML = `<span class="js-keyword">fetch</span>(<span class="js-string">"https://languagetool.votredomaine.com/v2/check"</span>, { lang: <span class="js-string">"${lang}"</span>, text: <span class="js-string">"${escapeHtml(displayText)}"</span> });`;
    }

    function updateResponseBox(data) {
        apiResponse.innerHTML = JSON.stringify(data, null, 2)
            .replace(/"([^"]+)":/g, '<span class="json-key">"$1"</span>:')
            .replace(/: "([^"]+)"/g, ': <span class="json-str">"$1"</span>');
    }

    editor.addEventListener('keydown', e => { if (e.ctrlKey && e.key === 'Enter') checkText(); });
</script>
</body>
</html>
```

---

### Section Personnalisation

**Marqueurs PERSO** (à modifier selon vos besoins) :

- **PERSO 1** : Logo cliquable en bas à gauche → modifier le lien `href` ou supprimer toute la balise `<a id="custom-logo">`.
- **PERSO 2** : Lien "Mon Swagger" dans la navigation → modifier l’URL ou supprimer la ligne.
- **PERSO 3** : Message d’astuce sous le bouton → modifier le texte ou supprimer le bloc `<p>`.
- **PERSO 4** : URL de l’API LanguageTool (deux occurrences dans le script) → remplacer `https://languagetool.votredomaine.com/v2/check` par l’URL réelle de votre instance.

---

### Étape 4 : Redémarrage et vérification

```bash
systemctl restart nginx
```

L’interface est accessible via l’adresse IP de la VM/LXC en HTTP.

---

### Étape 5 : Configuration Nginx Proxy Manager

Créez un nouveau **Proxy Host** :

#### Onglet **Détails**
- **Noms de domaine** : `languagetool-web.votredomaine.com`
- **Schéma** : `http`
- **Nom d’hôte / IP** : `192.168.2.74` (IP de votre VM/LXC)
- **Port** : `80`
- **Liste d’accès** : Accessible au public

#### Onglet **SSL**
- Forcer SSL → **ON**
- HTTP/2 → **ON**
- HSTS actif → **ON**
- Sous-domaines HSTS → **ON**

#### Onglet **Emplacement personnalisé**
```nginx
proxy_hide_header X-Powered-By;
add_header Referrer-Policy "no-referrer" always;
add_header X-Frame-Options SAMEORIGIN always;
add_header X-XSS-Protection "1; mode=block" always;
```

#### Configuration Nginx personnalisée
```nginx
gzip on;
gzip_min_length 1000;
gzip_disable "msie6";
gzip_vary on;
gzip_proxied any;
gzip_comp_level 6;
gzip_buffers 16 8k;
gzip_http_version 1.1;
gzip_types text/plain text/css application/json application/javascript text/xml application/xml+rss text/javascript;

client_body_buffer_size 512k;
proxy_read_timeout 86400s;
client_max_body_size 0;

proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
```

Pour des headers de sécurité et une configuration gzip encore plus optimisée, consultez cette page de wiki :  
**[Les meilleurs headers pour NPM - Sécurité, Gzip et gestion du proxy NGINX](/fr/meilleurs-headers-npm-nginx-securite-gzip)**