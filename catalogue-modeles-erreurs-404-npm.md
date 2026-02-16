---
title: [Catalogue] 7 modèles de pages d'erreur 404 pour Nginx Proxy Manager
description: Découvrez 7 modèles de pages d'erreur 404 personnalisés pour Nginx Proxy Manager. Du style minimaliste au mode Matrix glitché, améliorez l'expérience de vos utilisateurs avec des designs prêts à l'emploi.
published: false
date: 2026-02-16T14:51:40.405Z
tags: npm, html, css, 404, auto-hébergement, personnalisation, nginx proxy manager
editor: markdown
dateCreated: 2026-02-16T14:03:15.558Z
---

Ce catalogue regroupe différentes esthétiques pour vos pages d'erreur. Avant de choisir un modèle, assurez-vous d'avoir suivi l'un de mes deux guides techniques selon votre besoin :

1. **[Guide : Personnalisation par HTML injecté](/fr/page-erreur-npm-hote-inconnu)** : Pour une mise en place rapide par hôte.
2. **[Guide : Personnalisation statique avec SSL](/fr/page-erreur-404-statique-ssl-npm)** : Ma méthode recommandée pour un rendu professionnel et sécurisé.

---

## 1. La minimaliste (Zen)

Un design inspiré du style Apple : pur, calme et professionnel.

* **Idéal pour :** Un portfolio ou un site vitrine sobre.
* **Difficulté :** ⭐
* **À personnaliser :** Lignes 23 (lien bouton) et 25-26 (lien logo).

```html
<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>404 - Page non trouvée</title>
    <style>
        :root { --bg-color: #f8f9fa; --text-main: #2d3436; --text-secondary: #636e72; --accent-color: #2ecc71; --container-bg: #ffffff; }
        body { margin: 0; padding: 0; font-family: -apple-system, sans-serif; background-color: var(--bg-color); color: var(--text-main); display: flex; justify-content: center; align-items: center; height: 100vh; text-align: center; }
        .container { padding: 40px; background: var(--container-bg); border-radius: 24px; box-shadow: 0 10px 30px rgba(0, 0, 0, 0.05); max-width: 400px; width: 90%; }
        h1 { font-size: 80px; margin: 0; font-weight: 800; background: linear-gradient(135deg, var(--text-main), var(--accent-color)); -webkit-background-clip: text; -webkit-text-fill-color: transparent; }
        p { font-size: 1.1em; color: var(--text-secondary); margin: 20px 0 30px; line-height: 1.5; }
        .btn { display: inline-block; padding: 12px 28px; background-color: var(--accent-color); color: white; text-decoration: none; border-radius: 12px; font-weight: 600; transition: transform 0.2s; }
        .btn:hover { transform: translateY(-2px); }
        .footer-logo { margin-top: 30px; opacity: 0.5; }
        .footer-logo img { width: 40px; filter: grayscale(100%); }
    </style>
</head>
<body>
    <div class="container">
        <h1>404</h1>
        <p>Il semble que vous ayez pris un chemin inexistant. Pas de panique, cela arrive même aux meilleurs.</p>
        <a href="https://blablalinux.be/mes-services-publics/" class="btn">Retour à l'accueil</a>
        <div class="footer-logo">
            <a href="https://blablalinux.be/mes-services-publics/">
                <img src="https://blablalinux.be/wp-content/uploads/2025/11/logo-npm-02.png" alt="Logo">
            </a>
        </div>
    </div>
</body>
</html>

```

![minimaliste.png](/catalogue-modeles-erreurs-404-npm/minimaliste.png)

---

## 2. La technique (Terminal bash)

Simule une console Linux avec un effet d'auto-typage du texte.

* **Idéal pour :** Les homelabs et serveurs d'administration.
* **Difficulté :** ⭐⭐
* **À personnaliser :** Ligne 33 (URL du lien `curl`).

```html
<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>bash: 404: command not found</title>
    <style>
        body { background-color: #0c0c0c; color: #00ff00; font-family: 'Courier New', monospace; display: flex; justify-content: center; align-items: center; height: 100vh; margin: 0; padding: 20px; }
        .terminal { width: 100%; max-width: 700px; background: #1e1e1e; border-radius: 8px; box-shadow: 0 15px 35px rgba(0,0,0,0.8); overflow: hidden; border: 1px solid #333; }
        .terminal-header { background: #333; padding: 8px 15px; display: flex; gap: 8px; }
        .dot { width: 12px; height: 12px; border-radius: 50%; }
        .red { background: #ff5f56; } .yellow { background: #ffbd2e; } .green { background: #27c93f; }
        .terminal-window { padding: 20px; line-height: 1.6; min-height: 300px; }
        .prompt { color: #2ecc71; font-weight: bold; }
        .path { color: #3498db; } .error { color: #e74c3c; font-weight: bold; }
        .cursor { display: inline-block; width: 10px; height: 18px; background-color: #00ff00; margin-left: 5px; animation: blink 1s infinite; vertical-align: middle; }
        @keyframes blink { 50% { opacity: 0; } }
        a { color: #00ff00; text-decoration: underline; }
        .typing { overflow: hidden; white-space: nowrap; border-right: 3px solid transparent; width: 0; animation: typing 2s steps(40, end) forwards; }
        @keyframes typing { from { width: 0 } to { width: 100% } }
        @keyframes fadeIn { to { opacity: 1; } }
    </style>
</head>
<body>
<div class="terminal">
    <div class="terminal-header"><div class="dot red"></div><div class="dot yellow"></div><div class="dot green"></div></div>
    <div class="terminal-window">
        <div><span class="prompt">user@linux</span>:<span class="path">~</span>$ curl -I /page-perdue</div>
        <div class="typing" style="animation-delay: 1s;">HTTP/1.1 <span class="error">404 Not Found</span></div>
        <br>
        <div style="opacity: 0; animation: fadeIn 0.1s forwards; animation-delay: 3s;">
            [INFO] Analyse du secteur... Hôte introuvable.<br>
            [SUGGESTION] Retourner au <a href="https://blablalinux.be/mes-services-publics/">Point d'entrée principal</a>.
        </div>
        <br>
        <div><span class="prompt">user@linux</span>:<span class="path">~</span>$ <span class="cursor"></span></div>
    </div>
</div>
</body>
</html>

```

![technique.gif](/catalogue-modeles-erreurs-404-npm/technique.gif)

---

## 3. L'humoristique (Pause café)

Un ton décalé pour déstresser l'utilisateur égaré.

* **Idéal pour :** Créer un lien sympathique avec vos visiteurs.
* **Difficulté :** ⭐⭐
* **À personnaliser :** Lignes 25 (bouton) et 26-27 (lien logo).

```html
<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>404 - Petite pause café ?</title>
    <style>
        :root { --mocha: #6f4e37; --cream: #f5f5dc; --accent: #e67e22; }
        body { background-color: var(--cream); color: var(--mocha); font-family: sans-serif; display: flex; justify-content: center; align-items: center; height: 100vh; margin: 0; text-align: center; }
        .container { display: flex; flex-direction: column; align-items: center; gap: 20px; padding: 20px; }
        .coffee-icon { font-size: 80px; position: relative; animation: float 3s infinite; }
        .steam { position: absolute; top: -20px; left: 50%; opacity: 0; animation: steamUp 2s infinite; }
        @keyframes steamUp { 0% { opacity: 0; } 50% { opacity: 0.6; } 100% { transform: translateY(-40px); opacity: 0; } }
        @keyframes float { 50% { transform: translateY(-10px); } }
        .btn-coffee { display: inline-block; padding: 15px 30px; background: var(--mocha); color: var(--cream); text-decoration: none; border-radius: 50px; font-weight: bold; }
        .logo-footer { margin-top: 30px; display: block; }
        .logo-footer img { width: 60px; height: auto; border-radius: 10px; filter: sepia(0.5); }
    </style>
</head>
<body>
    <div class="container">
        <div class="coffee-icon"><span class="steam">☁️</span>☕</div>
        <h1>Erreur 404... ou pause café ?</h1>
        <p>On dirait que cette page n'est pas encore moulue.</p>
        <a href="https://blablalinux.be/mes-services-publics/" class="btn-coffee">Retourner au comptoir</a>
        <a href="https://blablalinux.be/mes-services-publics/" class="logo-footer">
            <img src="https://blablalinux.be/wp-content/uploads/2025/11/logo-npm-02.png" alt="Logo">
        </a>
    </div>
</body>
</html>

```

![humoristique.gif](/catalogue-modeles-erreurs-404-npm/humoristique.gif)

---

## 4. La visuelle (Plein écran)

Utilise une image immersive avec effet de verre dépoli.

* **Idéal pour :** Un rendu "Premium" et technologique.
* **Difficulté :** ⭐⭐⭐
* **À personnaliser :** Ligne 23 (bouton de retour).

```html
<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>404 - Perdu ?</title>
    <style>
        body, html { margin: 0; height: 100%; font-family: sans-serif; overflow: hidden; }
        .bg-image { position: fixed; width: 100%; height: 100%; background: url('https://images.unsplash.com/photo-1451187580459-43490279c0fa?q=80&w=2072&auto=format&fit=crop') center/cover; animation: kenburns 20s infinite alternate; z-index: -1; }
        @keyframes kenburns { to { transform: scale(1.1); } }
        .overlay { height: 100%; display: flex; justify-content: center; align-items: center; background: rgba(0,0,0,0.5); }
        .glass-card { background: rgba(255,255,255,0.1); backdrop-filter: blur(15px); padding: 50px; border-radius: 30px; text-align: center; color: white; border: 1px solid rgba(255,255,255,0.2); }
        h1 { font-size: 100px; margin: 0; }
        .btn-action { display: inline-block; padding: 15px 40px; background: white; color: black; text-decoration: none; border-radius: 50px; font-weight: bold; margin-top: 20px; }
    </style>
</head>
<body>
    <div class="bg-image"></div>
    <div class="overlay">
        <div class="glass-card">
            <h1>404</h1>
            <p>Le serveur n'a détecté aucune forme de vie à cette adresse.</p>
            <a href="https://blablalinux.be/mes-services-publics/" class="btn-action">Téléportation</a>
        </div>
    </div>
</body>
</html>

```

![visuelle.gif](/catalogue-modeles-erreurs-404-npm/visuelle.gif)

---

## 5. L'interactive (Le jeu Pong)

Transforme l'erreur en moment de divertissement avec un clone de Pong.

* **Idéal pour :** Faire oublier la frustration de la page manquante.
* **Difficulté :** ⭐⭐⭐⭐
* **À personnaliser :** Ligne 16 (bouton pour quitter le jeu).

```html
<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>404 - Pong !</title>
    <style>
        body { margin: 0; background: #111; color: #fff; font-family: monospace; display: flex; flex-direction: column; align-items: center; justify-content: center; height: 100vh; }
        #pong { border: 4px solid #fff; background: #000; }
        .btn-home { padding: 10px 20px; border: 2px solid #2ecc71; color: #2ecc71; text-decoration: none; margin-top: 20px; }
    </style>
</head>
<body>
    <h1>404 - Perdu ? Jouez un peu !</h1>
    <canvas id="pong" width="600" height="400"></canvas>
    <a href="https://blablalinux.be/mes-services-publics/" class="btn-home">Quitter le jeu</a>
    <script>
        const canvas = document.getElementById("pong"); const ctx = canvas.getContext("2d");
        const ball = { x: 300, y: 200, r: 10, s: 5, dx: 5, dy: 5 };
        const user = { y: 150, h: 100, s: 0 }; const com = { y: 150, h: 100, s: 0 };
        canvas.addEventListener("mousemove", e => { user.y = e.clientY - canvas.getBoundingClientRect().top - 50; });
        function update() {
            ball.x += ball.dx; ball.y += ball.dy;
            com.y += (ball.y - (com.y + 50)) * 0.1;
            if(ball.y < 0 || ball.y > 400) ball.dy *= -1;
            let p = ball.x < 300 ? user : com;
            if(ball.x < 20 || ball.x > 580) { if(ball.y > p.y && ball.y < p.y+100) ball.dx *= -1.1; else { ball.x < 300 ? com.s++ : user.s++; ball.x=300; ball.dx=5; } }
        }
        function draw() {
            ctx.fillStyle="#000"; ctx.fillRect(0,0,600,400); ctx.fillStyle="#fff";
            ctx.fillText(user.s, 150, 50); ctx.fillText(com.s, 450, 50);
            ctx.fillRect(0, user.y, 10, 100); ctx.fillRect(590, com.y, 10, 100);
            ctx.beginPath(); ctx.arc(ball.x, ball.y, 10, 0, Math.PI*2); ctx.fill();
        }
        setInterval(() => { update(); draw(); }, 20);
    </script>
</body>
</html>

```

![intercative.gif](/catalogue-modeles-erreurs-404-npm/intercative.gif)

---

## 6. La dynamique (Particules)

Un réseau de neurones interactif qui réagit à la souris.

* **Idéal pour :** Les sites axés sur l'IA et le développement.
* **Difficulté :** ⭐⭐⭐⭐
* **À personnaliser :** Ligne 19 (lien bouton).

```html
<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>404 - Connexion perdue</title>
    <style>
        body, html { margin: 0; padding: 0; height: 100%; background: #0f172a; color: white; overflow: hidden; display: flex; justify-content: center; align-items: center; font-family: sans-serif; }
        #particles { position: absolute; width: 100%; height: 100%; top: 0; left: 0; }
        .content { z-index: 2; text-align: center; }
        .btn { pointer-events: auto; padding: 12px 30px; border: 1px solid white; color: white; text-decoration: none; border-radius: 5px; }
    </style>
</head>
<body>
    <canvas id="particles"></canvas>
    <div class="content">
        <h1 style="font-size: 100px; margin: 0;">404</h1>
        <p>Connexion interrompue dans le réseau.</p>
        <a href="https://blablalinux.be/mes-services-publics/" class="btn">Réparer la connexion</a>
    </div>
    <script>
        const canvas = document.getElementById('particles'); const ctx = canvas.getContext('2d');
        canvas.width = window.innerWidth; canvas.height = window.innerHeight;
        let p = [];
        for(let i=0; i<100; i++) p.push({ x: Math.random()*canvas.width, y: Math.random()*canvas.height, vx: Math.random()*2-1, vy: Math.random()*2-1 });
        function draw() {
            ctx.clearRect(0,0,canvas.width,canvas.height); ctx.fillStyle="white"; ctx.strokeStyle="rgba(255,255,255,0.1)";
            p.forEach(i => {
                i.x += i.vx; i.y += i.vy;
                if(i.x<0 || i.x>canvas.width) i.vx*=-1; if(i.y<0 || i.y>canvas.height) i.vy*=-1;
                ctx.beginPath(); ctx.arc(i.x, i.y, 2, 0, Math.PI*2); ctx.fill();
                p.forEach(j => { let d = Math.hypot(i.x-j.x, i.y-j.y); if(d<100) { ctx.beginPath(); ctx.moveTo(i.x,i.y); ctx.lineTo(j.x,j.y); ctx.stroke(); } });
            });
            requestAnimationFrame(draw);
        }
        draw();
    </script>
</body>
</html>

```

![dynamique.gif](/catalogue-modeles-erreurs-404-npm/dynamique.gif)

---

## 7. L'ultime (Matrix glitch edition)

La signature visuelle de BlablaLinux. Ce modèle utilise un canvas pour la pluie digitale, des animations de bordures néon et un texte immersif.

* **Idéal pour :** Un homelab Proxmox ou un serveur technique.
* **Difficulté :** ⭐⭐⭐⭐⭐
* **À personnaliser :** Ligne 141 (URL du logo) et 140 (Lien de retour).

```html
<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=yes, maximum-scale=3.0">
    <title>🛸 Destination Inconnue !</title>
    <style>
        *, *::before, *::after { box-sizing: border-box; }
        html, body { margin: 0; padding: 0; height: 100%; width: 100%; overflow: hidden; background-color: transparent; }
        #matrix-canvas { position: fixed; top: 0; left: 0; width: 100vw; height: 100vh; z-index: -1; background-color: #000; }
        :root { --text-color: #ecf0f1; --neon-green: #2ecc71; --neon-glow: #00ff41; --container-bg: rgba(10, 20, 15, 0.9); --shadow-color: rgba(46, 204, 113, 0.5); }
        body { font-family: 'Segoe UI', sans-serif; text-align: center; color: var(--text-color); display: flex; justify-content: center; align-items: center; min-height: 100vh; padding: 10px; }
        .container { max-width: 650px; width: 95%; background: var(--container-bg); padding: 30px 20px; border-radius: 20px; box-shadow: 0 0 15px var(--shadow-color), 0 0 30px var(--shadow-color); border: 2px solid var(--neon-green); animation: bounceIn 0.8s cubic-bezier(0.68, -0.55, 0.265, 1.55), glitch-border 6s infinite; backdrop-filter: blur(10px); position: relative; }
        @keyframes glitch-border { 0%, 95%, 100% { border-color: var(--neon-green); box-shadow: 0 0 15px var(--shadow-color); } 96% { border-color: #fff; box-shadow: 0 0 30px var(--neon-glow); } 98% { border-color: var(--neon-glow); box-shadow: 0 0 40px var(--neon-glow); } }
        @keyframes bounceIn { 0% { opacity: 0; transform: scale(0.3); } 100% { opacity: 1; transform: scale(1); } }
        .glitch-title { font-size: 2.2em; margin: 0 0 10px; color: var(--neon-green); text-shadow: 0 0 10px var(--neon-green); position: relative; }
        .glitch-title:hover::before { content: "💊 Erreur de Matrice"; position: absolute; left: 2px; text-shadow: -2px 0 var(--neon-glow); clip: rect(44px, 450px, 56px, 0); animation: glitch-anim 0.2s infinite linear alternate-reverse; }
        @keyframes glitch-anim { 0% { clip: rect(10px, 9999px, 20px, 0); transform: skew(0.5deg); } 100% { clip: rect(80px, 9999px, 90px, 0); transform: skew(-0.5deg); } }
        h2 { font-size: 1.2em; color: #ff4757; margin-bottom: 20px; text-shadow: 0 0 10px rgba(255, 71, 87, 0.5); }
        .code-box { display: inline-block; background: rgba(0, 0, 0, 0.8); padding: 8px 15px; border-radius: 8px; border: 1px solid var(--neon-green); margin: 10px 0; }
        .code { font-family: monospace; font-weight: bold; color: var(--neon-green); text-shadow: 0 0 5px var(--neon-green); }
        hr { border: none; height: 1px; background: var(--neon-green); opacity: 0.3; margin: 20px 0; }
        #custom-logo { position: fixed; bottom: 20px; right: 20px; z-index: 10000; transition: all 0.4s ease; transform: scaleX(-1); }
        #custom-logo:hover { transform: scaleX(-1) scale(1.2) rotate(10deg); filter: drop-shadow(0 0 15px var(--neon-glow)); }
        #custom-logo img { width: 60px; height: 60px; border-radius: 12px; box-shadow: 0 5px 15px rgba(0,0,0,0.8); display: block; border: 1px solid var(--neon-green); }
        @media (min-width: 768px) { .container { padding: 40px; } .glitch-title { font-size: 3em; } h2 { font-size: 1.5em; } }
    </style>
</head>
<body>
    <canvas id="matrix-canvas"></canvas>
    <div class="container">
        <h1 class="glitch-title">💊 Erreur de Matrice</h1>
        <h2>Destination introuvable... 🛰️</h2>
        <p>Le Proxy a fouillé le code source de l'univers, il a même réveillé l'Oracle 🤝, mais ce domaine n'est pas répertorié.</p>
        <p>Soit la pilule était trop forte 🔵, soit cet hôte manque à l'appel 📝.</p>
        <div class="code-box"><span class="code">ERROR_404: GLITCH_IN_SIMULATION 🌌</span></div>
        <hr>
        <p style="font-size: 0.85em; opacity: 0.9;">
            💡 <strong>Conseil de Néo :</strong> Vérifie l'URL ou va prendre un café ☕. Si ça persiste, c'est un bug dans la simulation !
        </p>
    </div>
    <a href="https://blablalinux.be/mes-services-publics/" id="custom-logo" target="_blank">
        <img src="https://blablalinux.be/wp-content/uploads/2025/11/logo-npm-02.png" alt="BlablaLinux">
    </a>
    <script>
        const canvas = document.getElementById('matrix-canvas'); const ctx = canvas.getContext('2d');
        function resizeCanvas() { canvas.width = window.innerWidth; canvas.height = window.innerHeight; }
        window.addEventListener('resize', resizeCanvas); resizeCanvas();
        const chars = '01ABCDEFGHIJKLMNOPQRSTUVWXYZｦｱｲｳｴｵｶｷｸｹｺｻｼｽｾｿﾀﾁﾂﾃﾄﾅﾆﾇﾈﾉﾊﾋﾌﾍﾎﾏﾐﾑﾒﾓﾔﾕﾖﾗﾘﾙﾚﾛﾜﾝ';
        const fontSize = 16; const columns = Math.floor(canvas.width / fontSize);
        const drops = new Array(columns).fill(1);
        function drawMatrix() {
            ctx.fillStyle = 'rgba(0, 0, 0, 0.05)'; ctx.fillRect(0, 0, canvas.width, canvas.height);
            ctx.shadowBlur = 8; ctx.shadowColor = '#0F0'; ctx.fillStyle = '#0F0'; ctx.font = fontSize + 'px monospace';
            for (let i = 0; i < drops.length; i++) {
                const text = chars[Math.floor(Math.random() * chars.length)];
                ctx.fillText(text, i * fontSize, drops[i] * fontSize);
                if (drops[i] * fontSize > canvas.height && Math.random() > 0.975) drops[i] = 0;
                drops[i]++;
            }
            ctx.shadowBlur = 0;
        }
        function triggerRandomGlitch() {
            if (Math.random() > 0.985) { 
                document.body.style.filter = "contrast(1.5) brightness(1.2)";
                setTimeout(() => { document.body.style.filter = "none"; }, 40);
            }
        }
        setInterval(() => { drawMatrix(); triggerRandomGlitch(); }, 35);
    </script>
</body>
</html>

```

![ultime.gif](/catalogue-modeles-erreurs-404-npm/ultime.gif)