# Brancher un modèle sur Trésor Studio (MCP & API REST)

Trésor Studio expose deux surfaces locales, sans compte ni clé d'API :

- un **serveur MCP** (Model Context Protocol) en stdio, utilisable par Claude Desktop, Cursor, Zed, Continue, etc. ;
- une **API REST** sur `http://localhost:8787`, utilisable par n'importe quel script.

Tout reste sur votre machine. Le serveur écrit dans `server/data/projects.json`.

---

## 1. Préparer le serveur

```bash
npm install
npm run build:server   # génère server/lib.bundle.mjs (modules partagés du studio)
```

## 2. Serveur MCP (stdio)

```bash
npm run mcp
```

Déclaration dans le client (exemple Claude Desktop, `claude_desktop_config.json`) :

```json
{
  "mcpServers": {
    "tresor": {
      "command": "node",
      "args": ["/chemin/absolu/tresor-studio/server/mcp.js"]
    }
  }
}
```

### Les 18 outils

| Outil | Rôle |
| --- | --- |
| `lister_composants` | Catalogue des composants disponibles |
| `lister_modeles` | Modèles d'écran prêts à l'emploi |
| `lister_themes` | Palettes et thèmes |
| `lister_glass` | Préréglages de verre liquide |
| `lister_projets` | Projets enregistrés côté serveur |
| `creer_projet` | Crée un projet et son premier écran |
| `lister_ecrans` | Écrans d'un projet |
| `lire_ecran` | Arbre complet d'un écran |
| `creer_ecran` | Ajoute un écran (parfois depuis un modèle) |
| `ajouter_element` | Ajoute un composant, éventuellement dans un conteneur |
| `exporter_code` | Génère le code (html, react, python, php, node, compose) |
| `inspecter_code` | Statistiques du code généré |
| `adapter_responsive` | Recalcule la géométrie d'un point de rupture ou replie une navbar |
| `lister_widgets` | Composants riches (vecteurs, données, navigation, retours…) |
| `exporter_fichier_unique` | Un seul HTML avec CSS et JavaScript intégrés |
| `generer_logique` | Graphe de nœuds → JavaScript |
| `creer_montage_motion` | Montage animé → page HTML autonome |
| `lister_presets_motion` | Formats et animations de montage |

### Nom des paramètres

Les outils parlent français : `nom`, `modele`, `projet_id`, `ecran_id`, `type`, `x`, `y`, `largeur`, `hauteur`,
`texte`, `parent_id`, `verre`, `preset_verre`, `couleur`, `taille` (`mobile` | `tablet` | `desktop`),
`langage` (`html` | `react` | `python` | `php` | `node` | `compose`).

### Exemple de conversation

> « Crée un projet “Landing Nébula”, ajoute une barre de navigation, rends-la responsive puis exporte le HTML. »

Le modèle enchaîne `creer_projet` → `ajouter_element` → `adapter_responsive` → `exporter_code` et renvoie le code.
Les identifiants renvoyés par un appel (`id`, `ecrans[0]`, `element_id`) servent aux appels suivants ; aucun
identifiant ne doit être inventé.

## 3. API REST

```bash
npm run server    # http://localhost:8787
```

| Méthode | Route | Rôle |
| --- | --- | --- |
| GET | `/api/health` | État du serveur |
| GET | `/api/components` | Catalogue des composants |
| GET | `/api/themes` | Thèmes et palettes |
| GET | `/api/glass` | Préréglages de verre |
| GET | `/api/mcp/info` | Nom, version et liste des outils MCP |
| GET | `/api/projects` | Projets |
| POST | `/api/projects` | Crée un projet |
| GET | `/api/projects/:id` | Détail d'un projet |
| GET | `/api/projects/:id/screens` | Écrans |
| POST | `/api/projects/:id/screens` | Crée un écran (corps : `{ name, template }`) |
| POST | `/api/projects/:id/screens/:sid/elements` | Ajoute un élément (corps : `{ type, overrides, parentId }`) |
| GET | `/api/projects/:id/screens/:sid/export/:lang` | Exporte le code |
| GET | `/api/projects/:id/screens/:sid/single-file` | HTML + CSS + JS en un fichier |
| GET | `/api/widgets` | Composants riches disponibles |
| POST | `/api/blueprint/compile` | `{ nodes, edges }` → `{ javascript }` |
| GET | `/api/motion/presets` | Formats et animations de montage |
| POST | `/api/motion/export` | `{ project }` → page HTML animée |

### Exemple complet

```bash
# 1. créer un projet
PROJ=$(curl -sX POST localhost:8787/api/projects \
  -H 'content-type: application/json' \
  -d '{"name":"Landing Nébula"}' | jq -r .id)

# 2. créer un écran depuis un modèle
SCR=$(curl -sX POST localhost:8787/api/projects/$PROJ/screens \
  -H 'content-type: application/json' \
  -d '{"name":"Accueil","template":"hero-saas"}' | jq -r .id)

# 3. ajouter un bouton
curl -sX POST localhost:8787/api/projects/$PROJ/screens/$SCR/elements \
  -H 'content-type: application/json' \
  -d '{"type":"button","overrides":{"props":{"text":"Démarrer","x":24,"y":520,"w":180,"h":50}}}'

# 4. exporter le HTML
curl -s localhost:8787/api/projects/$PROJ/screens/$SCR/export/html -o accueil.html
```

## 4. Dans la page IA & MCP

La page `#/ia` du site vérifie la disponibilité de l'API (`/api/health`), affiche la configuration à copier et
détaille chaque outil. Aucune requête ne part vers l'extérieur : un badge indique seulement si le serveur local
répond.

## 5. Nouveaux appels, exemples

### Fichier unique exécutable

```bash
curl -s localhost:8787/api/projects/$PROJ/screens/$SCR/single-file \
  | jq -r .html > page-autonome.html
```

### Compiler un graphe de logique

```bash
curl -sX POST localhost:8787/api/blueprint/compile \
  -H 'content-type: application/json' \
  -d '{
        "nodes": [
          { "id": "a", "type": "chargement" },
          { "id": "b", "type": "texte", "data": { "target": "el_123", "text": "Bonjour" } }
        ],
        "edges": [{ "from": "a", "to": "b" }]
      }' | jq -r .javascript
```

Types de nœuds acceptés : `clic`, `chargement`, `requete`, `lecture`, `base`, `variable`, `afficher`,
`texte`, `notification`, `naviguer`, `script`, `condition`, `delai`. Une entrée n'accepte qu'un seul lien ;
les sorties nommées `out`, `vrai`, `faux` viennent du nœud condition.

### Créer un montage motion

```bash
curl -sX POST localhost:8787/api/motion/export \
  -H 'content-type: application/json' \
  -d '{
        "project": {
          "name": "Teaser",
          "format": "paysage",
          "scenes": [
            {
              "id": "s1", "name": "Plan 1", "duration": 4, "background": "#0b0b12",
              "layers": [
                { "id": "l1", "type": "texte", "text": "Votre marque mérite mieux",
                  "x": 140, "y": 300, "w": 1000, "h": 120, "fontSize": 64,
                  "color": "#ffffff", "start": 0, "duration": 3.5, "animation": "fade-up" }
              ]
            }
          ]
        }
      }' | jq -r .html > montage.html
```

Côté MCP, `creer_montage_motion` construit le projet depuis une description courte
(`{ nom, format, plans: [{ duree, fond, voix_off, calques: [...] }] }`) et renvoie la page complète.

## 6. Clés d'API tierces (voix, images)

Le studio ne stocke aucune clé et n'appelle aucun service payant. Pour une voix off ElevenLabs ou Gemini :

1. générez le fichier audio de votre côté ;
2. hébergez-le (ou convertissez-le en data-URI) ;
3. renseignez son URL dans le champ « Piste audio » du plan, ou dans le nœud « Requête API » du blueprint.

L'export contient alors un `<audio>` joué au début du plan, sans dépendance à votre clé.
