# Tutoriel PWA — NoteBox 📓 (v2, édition approfondie)

> Construire une application de carnet de notes **offline-first**, installable sur mobile et utilisable sur navigateur desktop. Version guidée : chaque concept est expliqué depuis sa raison d'être, avec le code complet commenté.

---

## Table des matières

- [Avant de commencer](#avant-de-commencer)
- [Module 0 — Mise en place : une vraie app dès le départ](#module-0)
- [Module 1 — Le manifest : pourquoi le web a voulu imiter le natif](#module-1)
- [Module 2 — Le Service Worker : l'histoire d'un proxy programmable](#module-2)
- [Module 3 — Les stratégies de cache](#module-3)
- [Module 4 — Workbox : ce que tout le monde utilise vraiment](#module-4)
- [Module 5 — IndexedDB : stocker des données, pas des fichiers](#module-5)
- [Module 6 — La synchronisation offline → online](#module-6)
- [Module 7 — Installation sur téléphone et déploiement](#module-7)
- [Récapitulatif et suite](#recap)

---

<a name="avant-de-commencer"></a>
## Avant de commencer

### Ce qu'on va construire

**NoteBox** : une app de prise de notes qui :

- s'affiche correctement sur desktop **et** mobile,
- fonctionne **sans connexion internet** après la première visite,
- **persiste** les notes localement (elles survivent à la fermeture du navigateur),
- se **synchronise** avec un serveur quand le réseau revient,
- s'**installe** sur l'écran d'accueil d'un téléphone comme une app native.

### Outils à installer

| Outil | Pourquoi | Lien |
|---|---|---|
| **VS Code** | Éditeur de code | [code.visualstudio.com](https://code.visualstudio.com) |
| **Live Server** (extension VS Code) | Serveur HTTP local avec rechargement auto | Marketplace VS Code → "Live Server" (Ritwick Dey) |
| **Chrome ou Edge** | Meilleur outillage DevTools pour les PWA | — |

Pas de Node.js, pas de npm, pas de bundler : tout passe par des CDN. C'est volontaire — on veut comprendre les mécanismes sans qu'un outil de build nous les cache.

> **Définition — CDN (Content Delivery Network)** : réseau de serveurs distribués géographiquement qui hébergent des fichiers statiques (bibliothèques JS, polices, images). Charger une lib via CDN évite de l'installer localement : le navigateur la télécharge directement depuis le réseau du fournisseur.

### Pourquoi un serveur local et pas un double-clic sur index.html ?

Quand tu ouvres un fichier par double-clic, l'URL commence par `file://`. Or les navigateurs appliquent la **Same-Origin Policy** : chaque page web appartient à une *origine* (protocole + domaine + port, ex: `https://example.com:443`), et de nombreuses API sont réservées aux origines "sûres". Une URL `file://` n'a pas d'origine exploitable, donc :

- les **modules ES** (`import` / `export`) sont bloqués,
- les **Service Workers** refusent de s'enregistrer,
- certains stockages sont restreints.

Un serveur local (`http://localhost:5500`) fournit une vraie origine. `localhost` bénéficie d'une exception : il est considéré comme sûr même sans HTTPS, précisément pour permettre le développement.

### Le panneau DevTools "Application"

Ton tableau de bord pour tout le tutoriel (F12 → onglet **Application**) :

| Section | Sert à |
|---|---|
| Manifest | Vérifier que le manifest est lu et valide |
| Service Workers | Statut du SW, forcer la mise à jour, simuler l'offline |
| Cache Storage | Inspecter ce qui est en cache |
| IndexedDB | Voir les données stockées |

---

<a name="module-0"></a>
## Module 0 — Mise en place : une vraie app dès le départ

### Pourquoi commencer par l'interface ?

Une PWA est avant tout… une page web. Tout ce qu'on ajoutera ensuite (manifest, Service Worker, IndexedDB) est invisible à l'œil nu. Autant partir d'une app agréable à regarder : ça rend chaque étape testable et motivante.

### Structure du projet

```
notebox/
├── index.html
├── manifest.json          ← Module 1
├── sw.js                  ← Module 2 (DOIT être à la racine, on verra pourquoi)
├── css/
│   └── style.css
├── js/
│   ├── app.js             ← logique UI
│   ├── db.js              ← Module 5 (IndexedDB)
│   └── sync.js            ← Module 6 (synchronisation)
└── icons/
    ├── icon-192.png
    └── icon-512.png
```

### Les icônes

Télécharge deux icônes PNG (clic droit → enregistrer dans `icons/`) :

- 192×192 : https://placehold.co/192x192/6c63ff/white.png?text=NB
- 512×512 : https://placehold.co/512x512/6c63ff/white.png?text=NB

> [placehold.co](https://placehold.co) génère des images de placeholder à la volée selon l'URL. Pour une vraie app, tu utiliserais [favicon.io](https://favicon.io) ou [maskable.app](https://maskable.app/editor) pour générer un jeu d'icônes propre depuis un logo.

### Le HTML

```html
<!-- index.html -->
<!DOCTYPE html>
<html lang="fr">
<head>
  <meta charset="UTF-8">
  <!-- viewport : sans cette ligne, un mobile affiche la page comme un écran
       de 980px de large puis dézoome → texte minuscule. Avec elle, le
       viewport CSS = largeur réelle de l'écran. -->
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>NoteBox</title>
  <link rel="stylesheet" href="css/style.css">
  <!-- Les deux lignes suivantes seront expliquées au Module 1 -->
  <link rel="manifest" href="manifest.json">
  <meta name="theme-color" content="#6c63ff">
</head>
<body>

  <header>
    <h1>📓 NoteBox</h1>
    <div class="header-actions">
      <button id="install-btn" hidden>Installer</button>
      <span id="status-badge">●</span>
    </div>
  </header>

  <main>
    <section id="editor">
      <textarea id="note-input" placeholder="Écris ta note ici…" rows="3"></textarea>
      <button id="add-btn">Ajouter</button>
    </section>

    <ul id="notes-list"></ul>

    <p id="empty-state">Aucune note pour l'instant. Écris ta première note ! ✍️</p>
  </main>

  <!-- type="module" active les imports ES. C'est aussi pour ça
       qu'on a besoin d'un serveur HTTP. -->
  <script type="module" src="js/app.js"></script>
</body>
</html>
```

### Le CSS — complet dès maintenant

Approche **mobile-first** : les styles par défaut visent les petits écrans, et une media query adapte le layout pour le desktop. C'est la pratique standard, car elle force à prioriser l'essentiel.

> **Définition — custom properties (variables CSS)** : valeurs nommées déclarées avec `--nom: valeur` et lues avec `var(--nom)`. Elles centralisent le design (couleurs, espacements) : changer le thème = modifier quelques lignes dans `:root`.

```css
/* css/style.css */

:root {
  --bg:       #1a1a2e;
  --surface:  #16213e;
  --surface2: #0f3460;
  --primary:  #6c63ff;
  --danger:   #e94560;
  --success:  #4ade80;
  --text:     #e0e0e0;
  --muted:    #8888aa;
  --radius:   10px;
  --gap:      1rem;
}

*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

body {
  font-family: system-ui, -apple-system, sans-serif;
  background: var(--bg);
  color: var(--text);
  min-height: 100dvh;   /* dvh = hauteur dynamique du viewport, gère la barre d'URL mobile */
  line-height: 1.6;
}

/* ---------- Header ---------- */
header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0.75rem var(--gap);
  background: var(--surface);
  border-bottom: 1px solid var(--surface2);
  position: sticky;       /* reste visible au scroll */
  top: 0;
  z-index: 10;
}

header h1 { font-size: 1.15rem; font-weight: 600; }

.header-actions { display: flex; gap: 0.5rem; align-items: center; }

#install-btn {
  font-size: 0.8rem;
  padding: 4px 12px;
  background: var(--primary);
  color: #fff;
  border: none;
  border-radius: 999px;
  cursor: pointer;
}

#status-badge {
  font-size: 0.72rem;
  padding: 3px 10px;
  border-radius: 999px;
  font-weight: 500;
}
#status-badge.online  { background: #14321f; color: var(--success); }
#status-badge.offline { background: #38141c; color: var(--danger); }

/* ---------- Layout principal ---------- */
main {
  padding: var(--gap);
  max-width: 640px;
  margin: 0 auto;
}

/* ---------- Éditeur ---------- */
#editor {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
  margin-bottom: var(--gap);
}

textarea {
  width: 100%;
  padding: 0.75rem;
  background: var(--surface);
  border: 1px solid var(--surface2);
  border-radius: var(--radius);
  color: var(--text);
  font: inherit;
  resize: vertical;
  transition: border-color 0.2s;
}
textarea:focus { outline: none; border-color: var(--primary); }

button {
  border: none;
  border-radius: var(--radius);
  cursor: pointer;
  font: inherit;
  font-weight: 500;
  /* touch-action: manipulation supprime le délai de ~300ms que les
     navigateurs mobiles ajoutent aux taps (ils attendaient un éventuel
     double-tap pour zoomer). */
  touch-action: manipulation;
  transition: opacity 0.15s;
}
button:active { opacity: 0.7; }

#add-btn {
  padding: 0.6rem 1.4rem;
  background: var(--primary);
  color: #fff;
  align-self: flex-end;
}

/* ---------- Liste de notes ---------- */
#notes-list {
  list-style: none;
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
}

.note-item {
  background: var(--surface);
  border: 1px solid var(--surface2);
  border-radius: var(--radius);
  padding: var(--gap);
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
  animation: slide-in 0.25s ease;
}

@keyframes slide-in {
  from { opacity: 0; transform: translateY(-6px); }
  to   { opacity: 1; transform: translateY(0); }
}

.note-content { white-space: pre-wrap; word-break: break-word; }

.note-meta {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.note-date { font-size: 0.75rem; color: var(--muted); }

.sync-dot { font-size: 0.7rem; }
.sync-dot.pending { color: var(--danger); }
.sync-dot.done    { color: var(--success); }

.delete-btn {
  background: transparent;
  color: var(--danger);
  border: 1px solid var(--danger);
  padding: 3px 10px;
  font-size: 0.78rem;
}

#empty-state {
  text-align: center;
  color: var(--muted);
  margin-top: 2rem;
}

/* ---------- Desktop ---------- */
@media (min-width: 640px) {
  main { padding: 2rem var(--gap); }
  #editor { flex-direction: row; align-items: stretch; }
  textarea { flex: 1; }
  #add-btn { align-self: auto; }
}
```

### Un premier app.js minimal (en mémoire pour l'instant)

Pour ce module, les notes vivent dans un simple tableau JS — elles disparaissent au rechargement. C'est voulu : on remplacera ce stockage par IndexedDB au Module 5 et la différence sera frappante.

```javascript
// js/app.js — version Module 0 (stockage en mémoire, temporaire)

const input  = document.getElementById('note-input');
const addBtn = document.getElementById('add-btn');
const list   = document.getElementById('notes-list');
const empty  = document.getElementById('empty-state');

let notes = [];   // ⚠ vide à chaque rechargement — corrigé au Module 5
let nextId = 1;

function render() {
  list.innerHTML = '';
  empty.hidden = notes.length > 0;

  for (const note of notes) {
    const li = document.createElement('li');
    li.className = 'note-item';
    li.innerHTML = `
      <p class="note-content"></p>
      <div class="note-meta">
        <span class="note-date">${new Date(note.createdAt).toLocaleString('fr-FR')}</span>
        <button class="delete-btn" data-id="${note.id}">Supprimer</button>
      </div>
    `;
    // textContent et non innerHTML pour le contenu : si la note contient
    // du HTML (<script>…), il sera affiché comme texte, pas exécuté.
    li.querySelector('.note-content').textContent = note.content;
    list.appendChild(li);
  }
}

addBtn.addEventListener('click', () => {
  const content = input.value.trim();
  if (!content) return;
  notes.unshift({ id: nextId++, content, createdAt: new Date().toISOString() });
  input.value = '';
  render();
});

list.addEventListener('click', e => {
  if (!e.target.matches('.delete-btn')) return;
  const id = Number(e.target.dataset.id);
  notes = notes.filter(n => n.id !== id);
  render();
});

render();
```

**Teste** : Live Server → l'app est jolie, les notes s'ajoutent et se suppriment. Recharge la page → tout disparaît. Coupe le réseau (DevTools → Network → Offline) et recharge → page d'erreur. Ces deux défauts sont exactement ce que les modules suivants vont corriger.

---

<a name="module-1"></a>
## Module 1 — Le manifest : pourquoi le web a voulu imiter le natif

### Partons du problème

Années 2010 : les smartphones explosent, et avec eux les app stores. Les apps natives ont des avantages écrasants sur les sites web mobiles :

- une **icône sur l'écran d'accueil** — l'utilisateur revient en un tap,
- un **lancement plein écran**, sans barre d'adresse,
- un **splash screen** pendant le chargement,
- une présence dans le **multitâche** du téléphone.

Pour les développeurs web, c'était frustrant : il fallait soit développer une app native par plateforme (coût ×2 ou ×3), soit accepter d'être un onglet parmi d'autres. Apple et Google ont d'abord proposé des bidouilles propriétaires (balises `<meta name="apple-mobile-web-app-capable">`…), chacune incompatible avec l'autre.

En 2015, le W3C standardise le **Web App Manifest** : *un* fichier JSON, lisible par tous les navigateurs, qui décrit comment la page doit se comporter une fois "installée". C'est l'acte de naissance officiel des PWA (terme inventé la même année par des ingénieurs de Google).

> **Définition — Web App Manifest** : fichier JSON déclaratif, lié à la page par `<link rel="manifest">`, qui fournit au navigateur et à l'OS les métadonnées d'installation : nom, icônes, couleurs, URL de démarrage, mode d'affichage.

### Sans manifest vs avec manifest

| | Sans manifest | Avec manifest valide (+ SW) |
|---|---|---|
| Écran d'accueil | Raccourci basique (favicon flou) | Vraie icône, vraie installation |
| Lancement | Ouvre le navigateur, barre d'URL visible | Mode standalone, plein écran |
| Splash screen | Aucun | Généré depuis `background_color` + icône |
| Multitâche Android | Fenêtre Chrome | Entrée séparée avec nom + icône de l'app |

### Le fichier

```json
{
  "name": "NoteBox — Carnet de notes",
  "short_name": "NoteBox",
  "description": "Carnet de notes offline-first",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#1a1a2e",
  "theme_color": "#6c63ff",
  "orientation": "portrait-primary",
  "icons": [
    { "src": "icons/icon-192.png", "sizes": "192x192", "type": "image/png", "purpose": "any maskable" },
    { "src": "icons/icon-512.png", "sizes": "512x512", "type": "image/png", "purpose": "any maskable" }
  ]
}
```

Décortiquons les champs non évidents :

- **`start_url`** : l'URL ouverte quand on lance l'app installée. `/` signifie "la racine du site". On pourrait mettre `/?source=pwa` pour mesurer dans les analytics combien d'utilisateurs passent par l'app installée.
- **`display: "standalone"`** : les quatre valeurs possibles forment un spectre du plus "web" au plus "app" : `browser` → `minimal-ui` → `standalone` → `fullscreen`. `standalone` retire toute l'interface du navigateur sauf la barre d'état système. `fullscreen` retire même celle-ci (utile pour les jeux).
- **`theme_color`** : colore la barre d'état Android et l'entête de la fenêtre installée sur desktop.
- **`purpose: "any maskable"`** : une icône **maskable** est conçue pour être rognée par l'OS (cercle, carré arrondi, "squircle"…) sans perdre son contenu. Concrètement : tout l'élément important de l'icône doit tenir dans un cercle central de 80% du canevas. Sans ça, Android colle ton icône carrée sur un fond blanc — moche. Tu peux tester tes icônes sur [maskable.app](https://maskable.app).

**Teste** : DevTools → Application → Manifest. Tous les champs doivent apparaître, les icônes doivent se charger. Les avertissements éventuels sont explicites.

---

<a name="module-2"></a>
## Module 2 — Le Service Worker : l'histoire d'un proxy programmable

### Partons du problème (et d'un échec célèbre)

Vers 2010, la question se pose : comment faire fonctionner un site web sans connexion ? La première réponse standardisée fut **AppCache** (HTML5 Application Cache) : un fichier *manifeste* listant les ressources à mettre en cache.

```
CACHE MANIFEST
index.html
style.css
app.js
```

Simple en apparence — et catastrophique en pratique. AppCache était **déclaratif et rigide** : impossible d'exprimer la moindre logique ("mets en cache sauf si…", "essaie le réseau d'abord pour l'API…"). Pire, ses règles implicites étaient pleines de pièges : la page qui référençait le manifeste était *toujours* mise en cache, les mises à jour ne se voyaient qu'au *deuxième* rechargement, et une seule ressource introuvable invalidait tout le cache. L'article culte de l'époque s'appelle ["Application Cache is a Douchebag"](https://alistapart.com/article/application-cache-is-a-douchebag/) (A List Apart, 2012) — c'est dire le traumatisme. AppCache est aujourd'hui supprimé des navigateurs.

La leçon retenue par les concepteurs des standards : **le cache offline a besoin de logique, pas de déclarations**. Plutôt que d'inventer un nouveau format de configuration, autant donner aux développeurs un endroit où exécuter *du code* entre la page et le réseau. Ce fut le **Service Worker** (2014, Chrome 40).

> **Définition — Service Worker** : script JavaScript enregistré par une page, exécuté par le navigateur dans un **thread** séparé (fil d'exécution indépendant du thread qui anime la page), sans accès au DOM, et dont le rôle est d'intercepter et de traiter les événements réseau et système de l'application : requêtes `fetch`, notifications push, synchronisations en arrière-plan.

> **Définition — proxy** : intermédiaire qui reçoit des requêtes à la place du destinataire final et décide de les transmettre, les modifier, ou y répondre lui-même. Le SW est un proxy *côté client*, *programmable en JS*.

### Les trois propriétés qui changent tout

**1. Il vit indépendamment de la page.** Le navigateur peut le réveiller même quand aucun onglet du site n'est ouvert (pour une notification push, par exemple), et l'endormir quand il n'a rien à faire. Conséquence importante : un SW ne peut pas compter sur des variables globales pour garder un état — il peut être tué et redémarré entre deux événements.

**2. Il n'a pas accès au DOM.** Il ne peut pas faire `document.querySelector`. Il communique avec les pages par messages (`postMessage`). C'est une séparation volontaire : le SW gère le *réseau et les données*, la page gère l'*affichage*.

**3. Il intercepte le réseau.** Une fois actif, **chaque** requête émise par les pages sous son contrôle (HTML, CSS, images, `fetch` d'API…) déclenche un événement `fetch` dans le SW, qui peut y répondre comme il veut.

### Comparaison : avant / après

Comment ferait-on de l'offline **sans** Service Worker ?

| Besoin | Sans SW | Avec SW |
|---|---|---|
| Charger l'app offline | Impossible : le premier fichier HTML doit venir du réseau | Le SW sert le HTML depuis le cache disque |
| Cache personnalisé | `localStorage` de strings + logique manuelle dans chaque fetch de l'app | Interception transparente, l'app ne change pas |
| Notification app fermée | Impossible (le JS de la page meurt avec l'onglet) | Le SW est réveillé par le push |
| Mise à jour du cache | AppCache rigide (déprécié) | Logique JS libre |

### Le cycle de vie — la partie qui piège tout le monde

```
       register() depuis la page
              ↓
        ┌──────────┐   event 'install'
        │ INSTALLING│   → on pré-remplit le cache ici
        └────┬─────┘
             ↓
        ┌──────────┐   Si un ANCIEN SW contrôle encore des pages,
        │  WAITING  │   le nouveau attend qu'elles soient toutes fermées
        └────┬─────┘
             ↓
        ┌──────────┐   event 'activate'
        │ ACTIVATED │   → on nettoie les vieux caches ici
        └────┬─────┘
             ↓
        Intercepte les events : fetch / push / sync / message
```

Deux subtilités à connaître absolument :

- **La phase WAITING.** Par défaut, un nouveau SW ne prend pas le contrôle tant que l'ancien sert encore des onglets ouverts. C'est une protection : deux versions du SW qui se partagent les mêmes pages, c'est l'incohérence assurée. En développement, c'est pénible — d'où `self.skipWaiting()` ("active-toi immédiatement") et `self.clients.claim()` ("prends le contrôle des pages déjà ouvertes").
- **Le scope.** Un SW ne peut contrôler que les URL situées dans son répertoire et en dessous. `/sw.js` contrôle tout le site ; `/js/sw.js` ne contrôlerait que `/js/…`. C'est pour ça que `sw.js` est à la racine.

### La Cache API

Le SW a besoin d'un endroit où ranger les réponses. C'est la **Cache API**.

> **Définition — Cache API** : stockage persistant sur disque, organisé en *caches* nommés, où les clés sont des objets `Request` et les valeurs des objets `Response` (corps + headers HTTP complets). À ne pas confondre avec le cache HTTP automatique du navigateur : la Cache API n'expire jamais seule et n'est modifiée que par ton code.

```
CacheStorage
└── "notebox-v1"
    ├── Request("/")            → Response(index.html)
    ├── Request("/css/style.css") → Response(…)
    └── Request("/js/app.js")     → Response(…)
```

Le nom versionné (`notebox-v1`) est la convention de mise à jour : quand tu modifies tes fichiers, tu passes à `notebox-v2`, le nouveau SW remplit un nouveau cache à l'`install` et supprime l'ancien à l'`activate`.

### Le code (version vanilla — Workbox arrive au Module 4)

```javascript
// sw.js

const CACHE_NAME = 'notebox-v1';

const STATIC_ASSETS = [
  '/',
  '/index.html',
  '/css/style.css',
  '/js/app.js',
  '/manifest.json',
  '/icons/icon-192.png',
  '/icons/icon-512.png',
];

// ---------- INSTALL : pré-remplir le cache ----------
self.addEventListener('install', event => {
  // event.waitUntil(promesse) dit au navigateur : "ne considère pas
  // l'installation terminée (et ne me tue pas) avant que cette promesse
  // soit résolue". Sans ça, le SW pourrait être endormi en plein addAll.
  event.waitUntil(
    caches.open(CACHE_NAME)
      .then(cache => cache.addAll(STATIC_ASSETS))
      .then(() => self.skipWaiting())
  );
});

// ---------- ACTIVATE : purger les anciens caches ----------
self.addEventListener('activate', event => {
  event.waitUntil(
    caches.keys()
      .then(keys => Promise.all(
        keys.filter(k => k !== CACHE_NAME).map(k => caches.delete(k))
      ))
      .then(() => self.clients.claim())
  );
});

// ---------- FETCH : répondre depuis le cache ----------
self.addEventListener('fetch', event => {
  // event.respondWith(promesseDeResponse) : "c'est MOI qui réponds à
  // cette requête, voici la Response (ou une promesse de Response)".
  event.respondWith(
    caches.match(event.request).then(cached => {
      if (cached) return cached;          // trouvé en cache → réponse immédiate
      return fetch(event.request).then(response => {
        // On clone car une Response est un flux lisible UNE seule fois :
        // un exemplaire part au navigateur, l'autre va au cache.
        const clone = response.clone();
        caches.open(CACHE_NAME).then(cache => cache.put(event.request, clone));
        return response;
      });
    })
  );
});
```

Et l'enregistrement, côté page :

```javascript
// js/app.js — à ajouter en haut du fichier

if ('serviceWorker' in navigator) {
  window.addEventListener('load', () => {
    navigator.serviceWorker.register('/sw.js')
      .then(reg => console.log('SW enregistré, scope :', reg.scope))
      .catch(err => console.error('SW :', err));
  });
}
```

On attend l'événement `load` pour ne pas concurrencer le chargement initial de la page : l'enregistrement du SW déclenche le téléchargement et le parsing de `sw.js`, autant le faire quand le reste est fini.

**Teste** : recharge deux fois (la 1ère installe le SW, la 2ème passe par lui). DevTools → Application → Service Workers → "activated and is running". Puis Network → Offline → recharge : **l'app s'affiche sans réseau**. Regarde aussi Cache Storage : tes fichiers y sont.

---

<a name="module-3"></a>
## Module 3 — Les stratégies de cache

### Partons du problème

Le `fetch` handler du Module 2 applique une seule politique à tout : *cache d'abord, réseau sinon*. Or toutes les ressources n'ont pas les mêmes besoins :

- `style.css` ne change presque jamais → le cache est parfait, le réseau est du gaspillage ;
- `/api/notes` change tout le temps → servir une vieille version en priorité serait un bug ;
- un avatar utilisateur change parfois → une version légèrement périmée est acceptable si elle s'affiche instantanément.

Une **stratégie de cache** est la règle qui ordonne cache et réseau pour une famille de requêtes. Quatre stratégies couvrent 99% des besoins :

### Les quatre stratégies

**Cache First** — *cache → réseau en secours*
Pour les assets statiques versionnés. Réponse instantanée, fonctionne offline, mais une ressource mise en cache n'est plus jamais rafraîchie (d'où le versioning du nom de cache).

```
Requête → [cache ?] ─oui→ réponse
              │ non
              └→ réseau → met en cache → réponse
```

**Network First** — *réseau → cache en secours*
Pour les données d'API. Toujours frais quand le réseau est là, dégradation gracieuse vers la dernière version connue quand il n'y est plus. Inconvénient : la latence réseau est subie à chaque requête.

```
Requête → réseau ─ok→ met en cache → réponse
              │ échec
              └→ [cache ?] → réponse (potentiellement périmée, mais mieux que rien)
```

**Stale While Revalidate** — *cache immédiat + rafraîchissement en arrière-plan*
Le compromis favori du web moderne. On répond tout de suite avec la version en cache ("stale" = périmée), et **en parallèle** on télécharge la version fraîche pour la *prochaine* fois. L'utilisateur a la vitesse du cache et une fraîcheur décalée d'une visite.

```
Requête → [cache] → réponse immédiate
      └──(en parallèle)──→ réseau → met à jour le cache (pour la prochaine fois)
```

**Network Only / Cache Only** — les cas extrêmes : analytics qui ne doivent jamais être cachées, ou ressources strictement offline.

### Le routeur

Reste à associer chaque requête à sa stratégie. On inspecte l'URL (ou `request.destination`, qui vaut `"image"`, `"script"`, `"style"`, `"document"`…) :

```javascript
// sw.js — fetch handler avec routing (version vanilla)

async function cacheFirst(request) {
  const cached = await caches.match(request);
  if (cached) return cached;
  const response = await fetch(request);
  const cache = await caches.open(CACHE_NAME);
  cache.put(request, response.clone());
  return response;
}

async function networkFirst(request) {
  const cache = await caches.open(CACHE_NAME);
  try {
    const response = await fetch(request);
    cache.put(request, response.clone());
    return response;
  } catch {
    const cached = await cache.match(request);
    // Si même le cache est vide, on répond une erreur JSON propre plutôt
    // que de laisser la requête planter brutalement.
    return cached ?? new Response(
      JSON.stringify({ error: 'offline' }),
      { status: 503, headers: { 'Content-Type': 'application/json' } }
    );
  }
}

self.addEventListener('fetch', event => {
  const url = new URL(event.request.url);

  if (url.pathname.startsWith('/api/')) {
    event.respondWith(networkFirst(event.request));
  } else {
    event.respondWith(cacheFirst(event.request));
  }
});
```

Tu remarques que ce code commence à ressembler à un mini-framework : des fonctions de stratégie + un routeur. C'est exactement le constat qui a mené à Workbox.

---

<a name="module-4"></a>
## Module 4 — Workbox : ce que tout le monde utilise vraiment

### Partons du problème

Le SW vanilla des Modules 2-3 fonctionne, mais en production les choses se corsent :

- gérer l'**expiration** (garder max 50 images, purger après 30 jours),
- gérer les **réponses opaques** (requêtes cross-origin dont on ne peut pas lire le statut),
- générer automatiquement la **liste des assets avec leurs hashes** pour invalider précisément le cache à chaque build,
- éviter les bugs subtils (oublier `clone()`, cacher une réponse 404…).

Chaque équipe réécrivait les mêmes ~300 lignes, avec les mêmes bugs. Google a fini par publier **Workbox** (2017) : la bibliothèque qui encapsule toutes les stratégies et les pièges connus. Aujourd'hui c'est l'outil par défaut — la plupart des SW que tu croiseras sur GitHub l'utilisent, directement ou via un plugin de framework (`vite-plugin-pwa`, `next-pwa`…).

> **Définition — Workbox** : ensemble de modules JS maintenus par Google qui fournissent des implémentations testées des stratégies de cache, un routeur de requêtes, la gestion d'expiration et le precaching versionné pour les Service Workers.

### Comparaison directe

La stratégie Stale While Revalidate, écrite à la main :

```javascript
// Vanilla — ~15 lignes, 2 pièges (clone, erreurs réseau non gérées)
self.addEventListener('fetch', event => {
  if (event.request.destination !== 'image') return;
  event.respondWith(
    caches.open('images').then(async cache => {
      const cached = await cache.match(event.request);
      const networkPromise = fetch(event.request).then(response => {
        cache.put(event.request, response.clone());
        return response;
      }).catch(() => undefined);
      return cached ?? networkPromise;
    })
  );
});
```

La même chose avec Workbox :

```javascript
// Workbox — déclaratif, expiration incluse
workbox.routing.registerRoute(
  ({ request }) => request.destination === 'image',
  new workbox.strategies.StaleWhileRevalidate({
    cacheName: 'images',
    plugins: [
      new workbox.expiration.ExpirationPlugin({ maxEntries: 50, maxAgeSeconds: 30 * 24 * 3600 })
    ]
  })
);
```

### Réécrire notre sw.js avec Workbox

Sans bundler, on charge Workbox via `importScripts` — la méthode d'import disponible dans les workers (l'équivalent de `<script src>` pour un contexte sans DOM) :

```javascript
// sw.js — version Workbox

importScripts('https://storage.googleapis.com/workbox-cdn/releases/7.0.0/workbox-sw.js');

const { precaching, routing, strategies, expiration } = workbox;

// ---------- PRECACHING ----------
// Équivalent de notre install + activate du Module 2, en mieux : chaque
// entrée a une "revision" — change la revision, et seule cette ressource
// est re-téléchargée (au lieu d'invalider tout le cache).
// (Avec un bundler, cette liste serait générée automatiquement avec des
// hashes de contenu — c'est le rôle de workbox-build / vite-plugin-pwa.)
precaching.precacheAndRoute([
  { url: '/',                   revision: '1' },
  { url: '/index.html',         revision: '1' },
  { url: '/css/style.css',      revision: '1' },
  { url: '/js/app.js',          revision: '1' },
  { url: '/js/db.js',           revision: '1' },
  { url: '/js/sync.js',         revision: '1' },
  { url: '/manifest.json',      revision: '1' },
  { url: '/icons/icon-192.png', revision: '1' },
  { url: '/icons/icon-512.png', revision: '1' },
]);

// ---------- API : Network First ----------
routing.registerRoute(
  ({ url }) => url.pathname.startsWith('/api/'),
  new strategies.NetworkFirst({
    cacheName: 'api-cache',
    networkTimeoutSeconds: 4,   // bascule sur le cache si le réseau traîne
  })
);

// ---------- Images : Stale While Revalidate + expiration ----------
routing.registerRoute(
  ({ request }) => request.destination === 'image',
  new strategies.StaleWhileRevalidate({
    cacheName: 'images',
    plugins: [
      new expiration.ExpirationPlugin({
        maxEntries: 50,
        maxAgeSeconds: 30 * 24 * 3600,
      })
    ]
  })
);

self.skipWaiting();
workbox.core.clientsClaim();
```

Trois choses que Workbox apporte ici qu'on n'avait pas :

1. **Invalidation fine** : la `revision` par fichier remplace le "tout ou rien" du nom de cache versionné.
2. **`networkTimeoutSeconds`** : un réseau *lent* est traité comme un réseau *absent* — détail crucial sur mobile, et pénible à coder à la main.
3. **Expiration automatique** des caches d'images.

Connaître la version vanilla reste indispensable : Workbox n'est qu'une couche au-dessus des mêmes événements `install`/`activate`/`fetch`, et quand quelque chose se passe mal, c'est le cycle de vie du Module 2 qu'il faut comprendre pour déboguer.

---

<a name="module-5"></a>
## Module 5 — IndexedDB : stocker des données, pas des fichiers

### Partons du problème

La Cache API stocke des *réponses HTTP* — parfait pour les fichiers, inadapté pour les *données applicatives* (nos notes). Pour celles-ci, l'histoire du stockage navigateur mérite d'être racontée :

- **Cookies** (1994) : ~4 Ko, renvoyés au serveur à *chaque* requête. Conçus pour les sessions, pas pour le stockage.
- **localStorage** (2009) : enfin un vrai stockage clé-valeur. Mais : uniquement des **strings**, API **synchrone** (chaque lecture/écriture bloque le thread principal — l'UI gèle), ~5 Mo, aucune requête possible autre que "donne-moi la clé X".
- **WebSQL** (2009) : une vraie base SQL dans le navigateur ! Abandonnée en 2010 : le standard exigeait plusieurs implémentations indépendantes, or tous les navigateurs embarquaient le même moteur (SQLite). Sans implémentations concurrentes, pas de standard — WebSQL est mort administrativement.
- **IndexedDB** (2010, standardisée 2015) : la réponse définitive. Une base de données **NoSQL orientée objets**, asynchrone, transactionnelle, avec index.

> **Définition — IndexedDB** : base de données embarquée dans le navigateur, stockant des objets JavaScript indexés par clé, avec accès asynchrone, transactions atomiques et index secondaires. Capacité de l'ordre de 50% de l'espace disque libre.

> **Définition — transaction** : groupe d'opérations traité comme un tout indivisible (*atomique*) : soit toutes réussissent, soit aucune n'est appliquée. Garantit qu'une coupure en plein milieu d'une écriture ne laisse pas la base dans un état incohérent.

### localStorage vs IndexedDB

| | localStorage | IndexedDB |
|---|---|---|
| Types | string uniquement (`JSON.stringify` obligatoire) | objets JS, blobs, fichiers |
| API | synchrone — **bloque l'UI** | asynchrone |
| Capacité | ~5 Mo | ~50% du disque libre |
| Requêtes | clé exacte | clé, index, plages, curseurs |
| Transactions | non | oui |
| Utilisable dans un SW | **non** (API synchrone interdite dans les workers) | oui |

La dernière ligne est décisive : si un jour ton Service Worker doit lire les données (Background Sync…), localStorage n'y est même pas accessible.

### Le modèle mental

```
IndexedDB
└── Database "notebox-db" (version 1)
    └── Object Store "notes"          ← l'équivalent d'une table
        ├── { id: 1, content: "…", createdAt: "…", synced: false }
        ├── { id: 2, … }
        └── Index "by-date" sur createdAt   ← pour requêter par date
```

- **keyPath** : le champ servant de clé primaire (`id` ici, avec `autoIncrement`).
- **Index** : structure auxiliaire permettant d'interroger par un autre champ que la clé. Comme un index SQL.
- **Versioning** : la structure (stores, index) ne peut être modifiée que dans le callback `upgrade`, déclenché quand le numéro de version augmente. C'est le mécanisme de migration de schéma d'IndexedDB.

### Pourquoi la lib idb

L'API native d'IndexedDB date d'avant les Promises : tout fonctionne par requêtes-objets avec callbacks `onsuccess`/`onerror`. Lire une note s'écrit en 8 lignes imbriquées. La bibliothèque **[idb](https://github.com/jakearchibald/idb)** (Jake Archibald, ~1 Ko) enveloppe la même API dans des Promises — c'est le standard de fait, on la croise partout sur GitHub.

```javascript
// API native — verbeux, callbacks
const request = indexedDB.open('notebox-db', 1);
request.onsuccess = e => {
  const db = e.target.result;
  const tx = db.transaction('notes', 'readonly');
  const getReq = tx.objectStore('notes').get(1);
  getReq.onsuccess = () => console.log(getReq.result);
};

// Avec idb — le même résultat
const db = await openDB('notebox-db', 1);
console.log(await db.get('notes', 1));
```

### Le code

```javascript
// js/db.js
import { openDB } from 'https://cdn.jsdelivr.net/npm/idb@8/build/index.js';

const DB_NAME = 'notebox-db';
const DB_VERSION = 1;
const STORE = 'notes';

let dbPromise;

export function initDB() {
  // On mémorise la promesse : tous les appels partagent la même connexion.
  if (!dbPromise) {
    dbPromise = openDB(DB_NAME, DB_VERSION, {
      // upgrade ne tourne QUE si la base n'existe pas encore, ou si
      // DB_VERSION a augmenté. C'est ici (et seulement ici) qu'on peut
      // créer/modifier les stores et les index.
      upgrade(db) {
        const store = db.createObjectStore(STORE, {
          keyPath: 'id',
          autoIncrement: true,
        });
        store.createIndex('by-date', 'createdAt');
      },
    });
  }
  return dbPromise;
}

export async function addNote(content) {
  const db = await initDB();
  return db.add(STORE, {
    content,
    createdAt: new Date().toISOString(),
    synced: false,   // ← flag pour le Module 6 : "pas encore envoyée au serveur"
  });
}

export async function getAllNotes() {
  const db = await initDB();
  // Lecture via l'index trié par date, puis inversion → plus récentes d'abord.
  const notes = await db.getAllFromIndex(STORE, 'by-date');
  return notes.reverse();
}

export async function getPendingNotes() {
  const db = await initDB();
  const all = await db.getAll(STORE);
  return all.filter(n => !n.synced);
}

export async function markSynced(note) {
  const db = await initDB();
  return db.put(STORE, { ...note, synced: true });  // put = upsert
}

export async function deleteNote(id) {
  const db = await initDB();
  return db.delete(STORE, id);
}
```

Et `app.js` branché sur cette couche (remplace la version Module 0) :

```javascript
// js/app.js — version finale (hors sync, ajoutée au Module 6)
import { initDB, addNote, getAllNotes, deleteNote } from './db.js';

if ('serviceWorker' in navigator) {
  window.addEventListener('load', () => {
    navigator.serviceWorker.register('/sw.js').catch(console.error);
  });
}

const input  = document.getElementById('note-input');
const addBtn = document.getElementById('add-btn');
const list   = document.getElementById('notes-list');
const empty  = document.getElementById('empty-state');

async function render() {
  const notes = await getAllNotes();
  list.innerHTML = '';
  empty.hidden = notes.length > 0;

  for (const note of notes) {
    const li = document.createElement('li');
    li.className = 'note-item';
    li.innerHTML = `
      <p class="note-content"></p>
      <div class="note-meta">
        <span class="note-date">
          ${new Date(note.createdAt).toLocaleString('fr-FR')}
          <span class="sync-dot ${note.synced ? 'done' : 'pending'}">●</span>
        </span>
        <button class="delete-btn" data-id="${note.id}">Supprimer</button>
      </div>
    `;
    li.querySelector('.note-content').textContent = note.content;
    list.appendChild(li);
  }
}

addBtn.addEventListener('click', async () => {
  const content = input.value.trim();
  if (!content) return;
  await addNote(content);
  input.value = '';
  render();
});

list.addEventListener('click', async e => {
  if (!e.target.matches('.delete-btn')) return;
  await deleteNote(Number(e.target.dataset.id));
  render();
});

initDB().then(render);
```

**Teste** : ajoute des notes, recharge → **elles sont toujours là**. Ferme le navigateur, rouvre → toujours là. DevTools → Application → IndexedDB → notebox-db → notes : tu vois tes objets, avec leur flag `synced: false` (pastille rouge dans l'UI — elle passera au vert au prochain module).

---

<a name="module-6"></a>
## Module 6 — La synchronisation offline → online

### Partons du problème

Notre app écrit désormais en local. Mais dans une vraie application, les données doivent finir sur un serveur (sauvegarde, multi-appareils, partage). Surgit alors le problème classique des systèmes distribués, version front-end :

> L'utilisateur crée une note dans le métro (offline). Comment garantir qu'elle arrivera au serveur, sans rien lui demander, et sans la dupliquer si l'envoi est retenté ?

La réponse standard est le pattern **outbox** (boîte d'envoi) : chaque modification locale est marquée "en attente d'envoi" (notre flag `synced: false`). Une routine de synchronisation parcourt les éléments en attente, les envoie, et les marque "envoyés" en cas de succès. En cas d'échec, ils restent en attente — la routine réessaiera.

### Détecter le retour du réseau

Le navigateur fournit deux mécanismes :

- **`navigator.onLine`** : booléen, état instantané. Fiabilité moyenne : `true` signifie "connecté à *un* réseau", pas "internet fonctionne" (un Wi-Fi captif sans accès renvoie `true`).
- **Événements `online` / `offline`** sur `window` : déclenchés aux changements d'état. Mêmes limites, mais suffisant pour déclencher une *tentative* de sync — si elle échoue, les notes restent en attente, rien n'est perdu.

### La Background Sync API

Le mécanisme prévu "officiellement" pour ce besoin est la **Background Sync API** : la page enregistre une étiquette (`sync-notes`), et le navigateur réveille le **Service Worker** avec un événement `sync` dès que la connectivité revient — **même si l'onglet a été fermé**. C'est le niveau de robustesse des apps natives.

```javascript
// Côté page : "préviens-moi quand le réseau revient"
const reg = await navigator.serviceWorker.ready;
await reg.sync.register('sync-notes');

// Côté sw.js : le navigateur nous réveille
self.addEventListener('sync', event => {
  if (event.tag === 'sync-notes') {
    event.waitUntil(syncNotes());   // syncNotes lirait IndexedDB depuis le SW
  }
});
```

Limite : support **Chrome/Edge/Android uniquement** (Firefox et Safari ne l'implémentent pas à ce jour). En pratique on code donc une **double détente** : Background Sync quand disponible, événement `online` en secours. Pour rester lisible, notre tutoriel implémente le fallback `online` (côté page) — la logique outbox est identique dans les deux cas.

### Le code

```javascript
// js/sync.js
import { getPendingNotes, markSynced } from './db.js';

// Envoie au serveur toutes les notes en attente.
// Le endpoint /api/notes n'existe pas dans ce tutoriel : les envois
// échoueront, et c'est OK — le pattern outbox est justement conçu pour
// que l'échec soit un état normal (on réessaiera).
export async function syncPendingNotes() {
  const pending = await getPendingNotes();
  if (pending.length === 0) return { sent: 0 };

  console.log(`[sync] ${pending.length} note(s) en attente`);
  let sent = 0;

  for (const note of pending) {
    try {
      const res = await fetch('/api/notes', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          content: note.content,
          createdAt: note.createdAt,
          // En production on enverrait aussi un identifiant unique généré
          // côté client (UUID) pour que le serveur déduplique si la même
          // note est envoyée deux fois (idempotence).
        }),
      });

      if (res.ok) {
        await markSynced(note);
        sent++;
      }
    } catch (err) {
      console.warn(`[sync] note ${note.id} : échec (${err.message}), on réessaiera`);
    }
  }
  return { sent };
}
```

```javascript
// js/app.js — ajouter ces lignes (imports + indicateur + déclencheurs)
import { syncPendingNotes } from './sync.js';

const badge = document.getElementById('status-badge');

function updateStatus() {
  const online = navigator.onLine;
  badge.textContent = online ? '● En ligne' : '● Hors ligne';
  badge.className = online ? 'online' : 'offline';
}

window.addEventListener('online', async () => {
  updateStatus();
  await syncPendingNotes();
  render();                 // rafraîchit les pastilles de sync
});
window.addEventListener('offline', updateStatus);
updateStatus();

// On tente aussi une sync au démarrage : des notes ont pu être créées
// offline lors de la session précédente.
syncPendingNotes().then(render);
```

**Teste** : DevTools → Network → Offline. Crée des notes (pastilles rouges, badge "Hors ligne"). Repasse en ligne : le badge devient vert et la console montre les tentatives d'envoi. Comme `/api/notes` n'existe pas, elles échouent proprement et les notes restent en attente — exactement le comportement attendu de l'outbox. Si tu veux voir le circuit complet réussir, n'importe quel endpoint de test fait l'affaire (par exemple remplacer l'URL par `https://httpbin.org/post`).

---

<a name="module-7"></a>
## Module 7 — Installation sur téléphone et déploiement

### Pourquoi HTTPS est non négociable

Un Service Worker peut intercepter et réécrire **tout le trafic** d'un site. Si on pouvait en installer un via une connexion HTTP non chiffrée, un attaquant en position d'intermédiaire (Wi-Fi public piégé…) pourrait injecter un SW malveillant qui persisterait *après* l'attaque — un piratage permanent du site dans ton navigateur. D'où la règle : **SW = HTTPS obligatoire** (exception : `localhost`).

> **Définition — HTTPS** : HTTP transporté dans un tunnel chiffré et authentifié (TLS). Le certificat prouve que tu parles bien au serveur du domaine affiché, et le chiffrement empêche la modification du trafic en route.

### Le prompt d'installation

Quand les critères sont réunis (manifest valide + SW avec handler `fetch` + HTTPS), Chrome émet l'événement **`beforeinstallprompt`**. Comportement par défaut : une discrète icône dans la barre d'adresse. Pour une vraie UX, on intercepte l'événement et on affiche notre propre bouton :

```javascript
// js/app.js — installation
let installPrompt = null;
const installBtn = document.getElementById('install-btn');

window.addEventListener('beforeinstallprompt', event => {
  event.preventDefault();      // bloque la mini-bannière par défaut
  installPrompt = event;       // on garde l'événement sous le coude
  installBtn.hidden = false;   // et on montre NOTRE bouton
});

installBtn.addEventListener('click', async () => {
  if (!installPrompt) return;
  installPrompt.prompt();                          // affiche le dialogue natif
  const { outcome } = await installPrompt.userChoice;
  console.log('Installation :', outcome);          // 'accepted' | 'dismissed'
  installPrompt = null;
  installBtn.hidden = true;
});

window.addEventListener('appinstalled', () => {
  installBtn.hidden = true;
});
```

**Cas iOS** : Safari n'implémente pas `beforeinstallprompt`. L'installation passe par le menu Partager → "Sur l'écran d'accueil". Le pattern courant : détecter iOS et afficher une petite infobulle d'instructions.

```javascript
const isIOS = /iphone|ipad|ipod/i.test(navigator.userAgent);
const isStandalone = window.matchMedia('(display-mode: standalone)').matches;
if (isIOS && !isStandalone) {
  // afficher : « Pour installer : bouton Partager → Sur l'écran d'accueil »
}
```

### Déployer

Deux options gratuites avec HTTPS automatique :

**Netlify** — [netlify.com](https://netlify.com) : glisse-dépose le dossier `notebox/` sur le dashboard ("Deploys" → zone de drop). URL en `https://xxx.netlify.app` en quelques secondes.

**GitHub Pages** : pousse le projet sur un repo, puis Settings → Pages → branche `main`. Attention au **scope** : l'URL sera `https://pseudo.github.io/notebox/` — ton site vit dans un *sous-chemin*, donc les chemins absolus (`/css/style.css`) cassent. Il faut soit passer en chemins relatifs (`./css/style.css`, `start_url: "./"`), soit préférer Netlify pour ce tutoriel.

**Teste sur ton téléphone** : ouvre l'URL déployée dans Chrome Android → le bouton "Installer" apparaît → installe → l'app a son icône, se lance en plein écran, figure dans le multitâche. Active le mode avion → elle fonctionne toujours. C'est une PWA complète.

---

<a name="recap"></a>
## Récapitulatif

Ce que chaque brique apporte, et le problème historique qu'elle résout :

| Brique | Problème d'origine | Solution |
|---|---|---|
| Manifest | Le web invisible face aux apps natives | Métadonnées d'installation standardisées (2015) |
| Service Worker | L'échec d'AppCache : l'offline a besoin de logique | Un proxy programmable en JS (2014) |
| Cache API | Où stocker des réponses HTTP complètes | Stockage Request→Response contrôlé par le code |
| Stratégies | Toutes les ressources n'ont pas la même fraîcheur | Cache First / Network First / SWR par famille d'URL |
| Workbox | Tout le monde réécrivait (mal) les mêmes 300 lignes | Stratégies + routing + expiration industrialisés (2017) |
| IndexedDB | localStorage : strings, synchrone, 5 Mo ; WebSQL : mort | Base objet asynchrone et transactionnelle |
| Outbox / sync | Modifications offline à fiabiliser | Flag `synced` + retry au retour du réseau |
| HTTPS | Un SW pirate = compromission persistante | Chiffrement et authentification obligatoires |

### Checklist finale

- [ ] Manifest valide (DevTools → Application → Manifest, zéro warning)
- [ ] SW "activated and running", `fetch` intercepté
- [ ] L'app charge en mode Offline
- [ ] Les notes survivent au redémarrage du navigateur
- [ ] Pastilles de sync cohérentes (rouge offline → verte après sync réussie)
- [ ] Déployé en HTTPS, installable sur Android

### Pour aller plus loin

- **Push Notifications** (`PushManager`) — notifications même app fermée ; nécessite un serveur de push
- **Periodic Background Sync** — rafraîchir les données périodiquement en arrière-plan (Chrome)
- **Web Share API** — `navigator.share()` pour partager une note via le menu natif du téléphone
- **vite-plugin-pwa** — quand tu passeras à un bundler : génération automatique du precache manifest avec hashes
- **Lighthouse** (DevTools → Lighthouse) — audit PWA automatisé avec score et recommandations
- **Conflits de sync** — que faire si la même note est modifiée sur deux appareils ? Mots-clés : *last-write-wins*, *CRDT*, *operational transformation*