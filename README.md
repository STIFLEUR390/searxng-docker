# SearXNG local + Dokploy + MCP pour agents IA

Déploiement Docker de **SearXNG** (moteur de recherche métafédéré, respectueux de la
vie privée) utilisé par les agents IA (Claude, Codex, …) via
**[`@aplix39/searxng-mcp`](https://www.npmjs.com/package/@aplix39/searxng-mcp)**,
notre serveur [MCP](https://modelcontextprotocol.io).

- Repo du MCP : <https://github.com/STIFLEUR390/searxng-mcp>
- Paquet npm : <https://www.npmjs.com/package/@aplix39/searxng-mcp>
- Interface web : <http://localhost:8888>

> **Pourquoi aucun service MCP dans la stack ?** Le paquet n'expose que le transport
> **stdio** (aucun mode HTTP/SSE — vérifié sur la v1.0.4 : `--help` : « The server
> communicates over stdio »). Un conteneur ne peut donc pas être joint **en HTTP** par
> Codex ou Claude Code. C'est l'agent qui lance le MCP en stdio (`npx`), voir
> [Utiliser le MCP](#utiliser-le-mcp).

## Services

| Service | Rôle |
|---|---|
| `searxng` | Le moteur. Port **127.0.0.1:8888** → 8080 dans le conteneur (exposé uniquement en local) |
| `valkey` | Cache/rate-limiting interne de SearXNG (volume `valkey-data`) |

## Démarrer

```bash
cd /home/herold/docker/searxng
docker compose up -d          # searxng + valkey
docker compose ps
curl -s 'http://localhost:8888/search?q=test&format=json' | head -c 200
```

L'UI web est sur <http://localhost:8888>. Le port étant lié à `127.0.0.1`, l'instance
n'est pas joignable depuis le LAN : c'est voulu (usage agent/local uniquement).

## Configuration requise par le MCP (`searxng/settings.yml`)

Le MCP interroge l'instance en **JSON** (`/search?format=json`), pas en HTML.
Trois réglages sont donc indispensables — ils sont **déjà configurés** ici :

```yaml
search:
  formats:
    - html    # pour le navigateur
    - json    # REQUIS par le MCP (?format=json renverrait 403 sans lui)

server:
  limiter: false      # usage local uniquement ; un limiter public bloquerait les appels agents
  image_proxy: true   # les résultats d'images passent par l'instance (vie privée)
```

Sans `json` dans `formats`, chaque recherche MCP échoue avec une erreur qui explique
justement qu'il faut ajouter ce réglage.

## Utiliser le MCP

Le MCP parle **stdio** : aucun service HTTP dans la stack, c'est l'agent qui le lance.

### 1. Directement depuis la machine hôte (npx / bunx)

```bash
npx -y @aplix39/searxng-mcp --url http://localhost:8888
# ou, avec Bun :
bunx @aplix39/searxng-mcp --url http://localhost:8888
```

### 2. Configuration agent (Claude Desktop / Claude Code / Codex…)

```json
{
  "mcpServers": {
    "searxng": {
      "command": "npx",
      "args": ["-y", "@aplix39/searxng-mcp", "--url", "http://localhost:8888"]
    }
  }
}
```

### Variables d'environnement

| Variable | Défaut | Rôle |
|---|---|---|
| `SEARXNG_URL` | `http://localhost:8888` | URL de base de l'instance ; le drapeau `--url` l'écrase |
| `SEARXNG_TIMEOUT_MS` | `30000` | Timeout HTTP vers SearXNG (500–300000 ms) |

## Les 7 outils exposés à l'agent

| Outil | Usage |
|---|---|
| `searxng_search` | Recherche web. Args : `q` (requis), `categories`, `language`, `time_range` (`day\|week\|month\|year`), `pageno`, `safesearch`, `engines`, `limit`, `include_domains`, `exclude_domains` |
| `searxng_search_news` | Actualités (category `news`), idéal avec `time_range` |
| `searxng_search_images` | Images (URL image + vignette) |
| `searxng_search_videos` | Vidéos (URL page + vignette) |
| `searxng_autocomplete` | Suggestions de saisie (`q` partiel, `language`) |
| `searxng_config` | Catégories/moteurs de l'instance (`include_engines: false` = mode compact) |
| `searxng_extract_content` | Extrait le contenu d'une page URL, chrome (header/nav/footer) retiré. Args : `url` (requis), `max_length`, `selector`, `headings_only` (plan h1–h6), `start_char` (fenêtrage), `include_links/images/code` |

Exemple d'appel agent :

> `searxng_search { "q": "bun javascript runtime", "time_range": "week", "limit": 5 }`
> `searxng_extract_content { "url": "https://bun.sh/", "headings_only": true }`

## Déployer sur Dokploy

Variante dédiée : **`docker-compose.dokploy.yml`** (Compose Path
`./docker-compose.dokploy.yml`), adaptée aux contraintes Dokploy :

- **pas de `container_name`** (Dokploy : provoque des soucis de logs et de métriques)
- **`expose: 8080`** au lieu de `ports` : routage par Traefik, rien d'exposé sur le host
- **labels Traefik manuels** (Méthode 2) : `Host(${SEARXNG_DOMAIN})`, entrypoint
  `websecure` + certificat Let's Encrypt
- **`dokploy-network` (externe)** pour que Traefik joigne `searxng` ; `valkey` reste sur
  le réseau interne `searxng-net` (pas exposé aux autres apps)
- **config via File Mount** (`../files/settings.yml`) : AutoDeploy re-claonne le dépôt à
  chaque déploiement, un bind mount `./searxng/…` venant du repo serait vidé

### Étapes

1. **Service → Compose** : Compose Type *Docker Compose*, Compose Path
   `./docker-compose.dokploy.yml`.
2. **Onglet Environment** (écrit dans `.env`, interpolé par `${…}` dans le fichier) :

   ```
   SEARXNG_DOMAIN=search.exemple.org
   SEARXNG_BASE_URL=https://search.exemple.org/
   ```

3. **Advanced → Volumes** (c'est l'UI *Mounts* de Dokploy, celle qui affiche
   « No volumes/mounts configured ») : créer un *File Mount* `settings.yml`
   (contenu ci-dessous, identique à `searxng/settings.yml`) :

   ```yaml
   # SearXNG — instance pour recherche web des agents IA
   use_default_settings: true

   server:
     # secret géré ici (pas de var d'env) ; régénérer: openssl rand -hex 32
     secret_key: "2ca1a576d97e2bc493f473ad647c7b6b0c1b0758e43a9ff1de736d3b0b9ca036"
     # instance publique sur Dokploy : passer à true si abuse (voir avertissement ci-dessous)
     limiter: false
     image_proxy: true

   search:
     # json REQUIS pour le MCP (?format=json renverrait 403 sans lui)
     formats:
       - html
       - json

   valkey:
     url: redis://valkey:6379/0

   ui:
     static_use_cdn: false
   ```

   ⚠️ **Ordre obligatoire** : Mounts **avant** le premier Deploy. Sinon Docker crée un
   **dossier** `settings.yml` à la place du fichier (ce qui provoque exactement
   `cp: … is a directory` → `is not a valid file, exiting…` en boucle).

4. **DNS** : enregistrement A `search.exemple.org` → IP du serveur.
5. **Deploy** — Traefik génère le certificat Let's Encrypt.

### Advanced — réglages Dokploy (Réglages → Advanced)

| Élément de l'UI | Action |
|---|---|
| **Run Command** | Laisser la commande par défaut (ne pas override) |
| **Volumes** | L'UI des mounts — doit afficher **1 mount** (`settings.yml`) après l'étape 3 |
| **Import** | ⚠️ **Ne jamais cliquer** : efface env vars, mounts et domains existants |
| **Networks** | `searxng` : **laisser attaché** à `dokploy-network` (Traefik en a besoin) ; `valkey` : **Detach** (sinon les autres apps du réseau joignent Valkey, sans auth) |
| **Enable Isolated Deployment** | **Désactivé** (obsolète selon l'UI) — le fichier gère déjà ses réseaux : `searxng-net` interne + `dokploy-network` externe |

### Récupération si `settings.yml` existe en dossier

**Diagnostic** : le banner du déploiement doit afficher `Detected: 1 mounts 📂`.
`0 mounts` = le File Mount n'a **pas** été créé (cause racine de `is a directory`).
Les variables d'env sont séparées : si `${SEARXNG_BASE_URL:?…}` manquait, l'`up`
échouerait avant même de créer les conteneurs.

Sur le serveur Dokploy — le volume `../files/settings.yml` se résout par rapport à
`code/` (là où Dokploy clone le dépôt) :

```bash
# chemin exact pour cette app :
ls -la /etc/dokploy/compose/ia-searxng-xyud7f/files/settings.yml
# drwx… = dossier créé par Docker → le supprimer :
rm -rf /etc/dokploy/compose/ia-searxng-xyud7f/files/settings.yml

# (ou recherche générique)
find /etc/dokploy -type d -name settings.yml 2>/dev/null
```

Puis : créer le File Mount (étape 3 ci-dessus) **avant** de Redeployer, et vérifier
que le banner affiche bien **`Detected: 1 mounts 📂`**.

> **Alternative (Méthode 1, recommandée par Dokploy)** : retirer les `labels` du fichier
> et déclarer le domaine dans l'onglet **Domains** de Dokploy — il injecte les labels
> Traefik lui-même. Ne pas cumuler les deux (routers dupliqués).

> ⚠️ **Sécurité** : sur Dokploy l'instance est **publique**. `server.limiter: false`
> convient en local uniquement ; pour une instance exposée, envisager
> `server.limiter: true` (au risque de limiter aussi les appels JSON des agents) ou
> restreindre l'accès au niveau de Traefik.

## Mise à jour du MCP

Le MCP n'est plus épinglé dans `docker-compose.yml` (plus de service compose) : la
version vit dans la configuration de l'agent. Épinglez-la explicitement dans les `args` :

```json
"args": ["-y", "@aplix39/searxng-mcp@1.0.4", "--url", "http://localhost:8888"]
```

Vérification : `npx -y @aplix39/searxng-mcp@1.0.4 --version` doit afficher la même
version (npx met le téléchargement en cache).

## Dépannage

| Symptôme | Cause / fix |
|---|---|
| `403 … JSON format is probably disabled` | `search.formats` ne contient pas `json` dans `settings.yml` → ajouter, puis `docker compose restart searxng` |
| `429 … rate limiter` | `server.limiter: true` alors que l'usage est local → `limiter: false` |
| MCP : `getaddrinfo ENOTFOUND searxng` | `SEARXNG_URL` pointe vers `http://searxng:8080` (DNS interne compose) alors que l'agent tourne sur l'hôte → forcer `--url http://localhost:8888` |
| Recherche lente / moteurs en timeout | Normal sur instance locale : les moteurs tiers filent. Réglable via `SEARXNG_TIMEOUT_MS` |
| `--version` du MCP ≠ version npm | Ne pas arriver : `npm version` rebuild `dist/` et un test garantit la synchro |
| Dokploy : `SEARXNG_DOMAIN manquant …` | Erreur voulue (`${VAR:?…}`) : définir les variables dans l'onglet **Environment** avant Deploy |
| Dokploy : `settings.yml is not a valid file` | File Mount absent → Docker a créé un **dossier** à sa place → Advanced → Mounts → créer le fichier `settings.yml` |
