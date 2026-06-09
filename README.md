<div align="center">

# 🔍 TenderFinder

**Agrégateur d'appels d'offres publics IT européens**

[![Live](https://img.shields.io/badge/Live-GitHub%20Pages-00c853?style=for-the-badge&logo=github)](https://m-arlaud.github.io/tender-finder/)
[![License](https://img.shields.io/badge/License-MIT-blue?style=for-the-badge)](LICENSE)
[![HTML](https://img.shields.io/badge/Stack-HTML%20%7C%20CSS%20%7C%20JS-f5a623?style=for-the-badge)](#-tech-stack)

<br/>

> Application web mono-fichier qui agrège les appels d'offres publics IT de plusieurs sources européennes en temps réel, avec filtrage par code CPV, recherche plein texte, tri par date de publication, et interface bilingue FR/EN.

</div>

---

## 📋 Table des matières

- [Fonctionnalités](#-fonctionnalités)
- [Sources de données](#-sources-de-données)
- [Architecture](#-architecture)
- [Installation & Déploiement](#-installation--déploiement)
- [Configuration](#-configuration)
- [Notes techniques par source](#-notes-techniques-par-source)
- [Tech Stack](#-tech-stack)
- [Structure du projet](#-structure-du-projet)
- [Contribuer](#-contribuer)

---

## ✨ Fonctionnalités

- **Agrégation multi-sources** : BOAMP FR, TenderNed NL, Vergabe NRW DE, TED EU
- **Tri par date de publication** : toutes les sources sont mélangées et triées par date (plus récentes d'abord), pas regroupées par origine
- **Filtrage CPV** : sélection des codes CPV IT (72xxx, 48xxx) avec interface de gestion et reset aux valeurs par défaut
- **Recherche plein texte** : recherche dans les titres, acheteurs et descriptions
- **Tri** : par montant ou par date limite de soumission
- **Favoris** : marquer et filtrer les offres favorites (localStorage), bordure dorée `#f5c518` sur les cards favorites
- **Interface bilingue** : FR / EN avec basculement par drapeau collé au logo
- **Mode sombre** : détection automatique des préférences système
- **Toolbar sticky** : barre de recherche/filtres qui se colle en haut (pleine largeur) quand le header sort de l'écran
- **Responsive** : adapté mobile, tablette et desktop
- **Mono-fichier** : un seul `index.html` autonome (~140 KB)

---

## 🌐 Sources de données

| Source | Pays | Méthode | Statut |
|--------|------|---------|--------|
| **BOAMP FR** | 🇫🇷 France | API REST directe (pas de proxy) | ✅ Live |
| **TenderNed NL** | 🇳🇱 Pays-Bas | API REST via Cloudflare Worker | ✅ Live |
| **Vergabe NRW DE** | 🇩🇪 Allemagne | ZIP Open Data via Cloudflare Worker + JSZip | ✅ Live |
| **TED EU** | 🇪🇺 Union Européenne | API v3 expert search via Cloudflare Worker | ✅ Live |

---

## 🏗 Architecture

```
┌─────────────────────────────────────────────────────┐
│  GitHub Pages (m-arlaud.github.io/tender-finder)     │
│  index.html — app mono-fichier HTML/CSS/JS           │
└───┬─────────┬──────────────┬──────────────┬──────────┘
    │         │              │              │
    ▼         ▼              ▼              ▼
 BOAMP FR  CF Worker      CF Worker      CF Worker
 (direct)  TenderNed NL   Vergabe NRW    TED EU
              │              │              │
              ▼              ▼              ▼
        tenderned.nl   evergabe.nrw.de  api.ted.europa.eu
        (API REST)     (ZIP Open Data)  (API v3 expert)
```

### Workers Cloudflare

| Worker | URL | Rôle |
|--------|-----|------|
| `lt-tenderned-proxy` | `lt-tenderned-proxy.malcolm-arlaud.workers.dev` | Proxy TenderNed avec CORS |
| `lt-vergabe-proxy` | `lt-vergabe-proxy.malcolm-arlaud.workers.dev` | Proxy ZIP Vergabe NRW avec CORS + cache 12h |
| `lt-tender-finder` | `lt-tender-finder.malcolm-arlaud.workers.dev` | Proxy TED EU — transmet le body POST de l'app + clé API |

> **Important** : le worker TED transmet désormais le body POST envoyé par l'app (query, tri, champs, limit). Il ne doit PAS coder la requête en dur, sinon l'app perd tout contrôle sur le tri et le filtrage.

---

## 🚀 Installation & Déploiement

### Prérequis

- Un compte GitHub (pour GitHub Pages)
- Un compte Cloudflare (pour les Workers proxy)

### Déploiement de l'app

```bash
git clone https://github.com/m-arlaud/tender-finder.git
cd tender-finder
# index.html EST l'application ; GitHub Pages le sert depuis main
```

### Déploiement d'un worker

Coller le contenu du fichier `*-proxy-worker.js` correspondant dans un Worker Cloudflare via le dashboard, puis Deploy. Le worker TED nécessite la clé API TED (variable `TED_API_KEY` dans le worker).

---

## ⚙️ Configuration

La modal Settings (icône ⚙ en haut à droite) permet de configurer :

- **Codes CPV** : liste des codes CPV à surveiller, avec bouton « Réinitialiser » (reset aux valeurs par défaut)
- **URLs des Workers** : endpoints des proxies Cloudflare (TED, TenderNed, Vergabe) + URL directe BOAMP
- **Résultats par page** : de 10 à 100, par incréments de 10

La modal applique automatiquement les changements de CPV à la fermeture (refetch + vidage du cache).

### Codes CPV par défaut

```
72000000  Services TI
72200000  Programmation et conseil
72210000  Programmation de progiciels
72220000  Conseil en systèmes et technique
72250000  Maintenance de systèmes
72260000  Services liés aux logiciels
72300000  Services de données
72600000  Assistance et conseil informatiques
62000000  (conservé pour BOAMP/TenderNed — voir note TED ci-dessous)
48000000  Logiciels et systèmes d'information
```

---

## 📝 Notes techniques par source

### BOAMP FR
Accès direct à l'API publique OpenData. Filtre par codes CPV côté serveur via le paramètre `select`. **Ne jamais ajouter de champs non vérifiés** dans `select` (ex: `procedure_libelle`) — l'API renvoie 400.

### TenderNed NL
- L'API nécessite un proxy pour les headers CORS.
- L'API limite `size` à **100 max** (400 au-delà) → pagination en 2 requêtes parallèles de 100.
- Ne pas utiliser `URLSearchParams` pour le `sort` : la virgule encodée `%2C` est refusée (400). Construire l'URL manuellement.

### Vergabe NRW DE
- Source atypique : **ZIP quotidien** (régénéré à 23h) contenant un `index.xml` + un XML par annonce.
- Le worker proxifie le ZIP avec CORS ; le navigateur le décompresse via **JSZip**, filtre les annonces SERVICES/DELIVERY, puis filtre par mots-clés IT allemands.
- Formats XML supportés : national (`NOTICE_NAT`) et eForms (UBL).
- URL des annonces : `https://www.evergabe.nrw.de/VMPSatellite/public/company/project/{ID}/de/overview` (format cosinex VMP).
- Badge type : règle de passation (VgV → « Above EU threshold », UVgO → « Below EU thresholds »).

### TED EU — ⚠️ pièges importants
L'intégration TED a nécessité de résoudre **quatre problèmes empilés** :

1. **CORS** : `api.ted.europa.eu` ne renvoie pas de headers CORS → passage obligatoire par le worker Cloudflare.
2. **Worker transparent** : le worker doit **transmettre le body POST de l'app**, pas coder la requête en dur (sinon tri/filtrage ignorés).
3. **Syntaxe de l'opérateur `IN`** : les valeurs CPV se séparent par un **ESPACE**, pas une virgule. `IN (72000000 72200000)` ✅ — `IN (72000000,72200000)` ❌ (toute la chaîne lue comme une seule valeur invalide).
4. **Codes CPV invalides** : `62000000` n'existe pas dans la nomenclature CPV. TED rejette **toute** la query si un seul code est invalide (contrairement à BOAMP/TenderNed qui les ignorent). Ces codes sont filtrés côté TED via `TED_INVALID_CPV` dans `fetchTED`.

Le tri se fait **dans la query expert** via `SORT BY publication-date DESC` (pas de paramètre `sortField` séparé). Garder `checkQuerySyntax: false` (le mode `true` valide la syntaxe mais renvoie une liste vide).

---

## 🛠 Tech Stack

| Technologie | Usage |
|-------------|-------|
| **HTML/CSS/JS** | Application mono-fichier, aucun framework |
| **Roboto Flex** | Fonte variable (axes `wdth`, `wght`) |
| **Material Symbols** | Icônes Google Material |
| **Flag Icons** | Drapeaux pays (fi-fr, fi-nl, fi-de, fi-eu) |
| **JSZip** | Décompression ZIP côté client (Vergabe NRW) |
| **Cloudflare Workers** | Proxies CORS pour APIs tierces |
| **GitHub Pages** | Hébergement statique |

---

## 📁 Structure du projet

```
tender-finder/
├── index.html                    # Application complète (HTML + CSS + JS)
├── README.md                     # Ce fichier
├── ted-proxy-worker.js           # Worker CF — proxy TED EU (transmet le body app)
├── tenderned-proxy-worker.js     # Worker CF — proxy TenderNed NL
└── vergabe-proxy-worker.js       # Worker CF — proxy Vergabe NRW DE
```

---

## 🎨 Conventions de code

- **Commentaires en français** partout dans le code.
- **Espacements** : multiples de 2/4/8 ; **hauteurs/sizings en multiples de 8px**.
- **Roboto Flex** partout (`var(--sans)`, axes `wdth`/`wght`).
- **Tooltips** : préférer les tooltips CSS (`data-tip` + `::after`) au `title` natif (peu fiable).
- **Accessibilité** : `lang`, `aria-label`, `aria-hidden`, `role`, `aria-live`.

---

## 🤝 Contribuer

1. Fork le repo
2. Crée une branche (`git checkout -b feature/ma-feature`)
3. Commit (`git commit -m 'Add ma feature'`)
4. Push (`git push origin feature/ma-feature`)
5. Ouvre une Pull Request

---

## 📝 License

Distribué sous licence MIT. Voir `LICENSE` pour plus d'informations.

---

<div align="center">

**Conçu par [Malcolm Arlaud](https://github.com/m-arlaud) @ [Lunatech Labs](https://lunatech.com)**

</div>
