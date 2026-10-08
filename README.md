# SearXNG local + MCP pour agents IA

Déploiement Docker de **SearXNG** (moteeur de recherche métafédéré, respectueux de la
vie privée) branché sur **[`@aplix39/searxng-mcp`](https://www.npmjs.com/package/@aplix39/searxng-mcp)**,
notre serveur [MCP](https://modelcontextprotocol.io) qui expose l'instance aux agents IA
(Claude, Codex, etc.).

- Repo du MCP : <https://github.com/STIFLEUR390/searxng-mcp>
- Paquet npm : <https://www.npmjs.com/package/@aplix39/searxng-mcp>
- Interface web : <http://localhost:8888>

## Services

| Service | Rôle |
|---|---|
| `searxng` | Le moteur. Port **127.0.0.1:8888** → 8080 dans le conteneur (exposé uniquement en local) |
| `valkey` | Cache/rate-limiting interne de SearXNG (volume `valkey-data`) |
| `mcp-searxng` | Serveur MCP stdio : `npx @aplix39/searxng-mcp@1.0.4`, joignable via `http://searxng:8080` (DNS du réseau compose) |

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

### 1. Depuis Docker (stdio, recommandé si l'agent tourne sur cette machine)

```bash
docker compose run --rm -i mcp-searxng
```

Le service tourne en stdio MCP : l'agent attache son stdin/stdout au conteneur,
`SEARXNG_URL` pointe déjà vers `http://searxng:8080` (réseau interne).

### 2. Directement depuis la machine hôte (npx / bunx)

```bash
npx -y @aplix39/searxng-mcp --url http://localhost:8888
# ou, avec Bun :
bunx @aplix39/searxng-mcp --url http://localhost:8888
```

### 3. Configuration agent (Claude Desktop / Claude Code / Codex…)

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

Variante Docker (l'agent appelle le conteneur) :

```json
{
  "mcpServers": {
    "searxng": {
      "command": "docker",
      "args": ["compose", "-f", "/home/herold/docker/searxng/docker-compose.yml", "run", "--rm", "-i", "mcp-searxng"]
    }
  }
}
```

### Variables d'environnement

| Variable | Défaut | Rôle |
|---|---|---|
| `SEARXNG_URL` | `http://localhost:8888` (hôte) / `http://searxng:8080` (compose) | URL de base de l'instance ; le drapeau `--url` l'écrase |
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

## Mise à jour du MCP

Le service `mcp-searxng` est épinglé sur une version npm. Pour passer à la suivante :

```bash
# 1. docker-compose.yml : mettre à jour @aplix39/searxng-mcp@X.Y.Z dans `command:`
# 2. relancer (npx re-télécharge la nouvelle version dans le cache npm du volume)
docker compose run --rm -i mcp-searxng
```

Le cache npm du conteneur est un volume (`npm-cache`), les démarrages suivants sont rapides.

## Dépannage

| Symptôme | Cause / fix |
|---|---|
| `403 … JSON format is probably disabled` | `search.formats` ne contient pas `json` dans `settings.yml` → ajouter, puis `docker compose restart searxng` |
| `429 … rate limiter` | `server.limiter: true` alors que l'usage est local → `limiter: false` |
| MCP : `getaddrinfo ENOTFOUND searxng` | Lancement hors du réseau compose → utiliser `--url http://localhost:8888` |
| Recherche lente / moteurs en timeout | Normal sur instance locale : les moteurs tiers filent. Réglable via `SEARXNG_TIMEOUT_MS` |
| `--version` du MCP ≠ version npm | Ne pas arriver : `npm version` rebuild `dist/` et un test garantit la synchro |
