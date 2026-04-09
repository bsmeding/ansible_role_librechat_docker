# LibreChat Docker Ansible Role

Deploys [LibreChat](https://www.librechat.ai/) (ChatGPT-style web UI with multi-provider AI support) via Docker Compose.

Based on the [official docker-compose.yml](https://github.com/danny-avila/LibreChat/blob/main/docker-compose.yml).

## Requirements

- Docker with Docker Compose plugin (via `bsmeding.docker` role)
- `community.docker` and `community.general` collections

## Role Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `librechat__name` | `librechat` | Service name; data path prefix |
| `librechat__home` | `/opt/librechat` | Persistent data directory |
| `librechat__port` | `3080` | Web UI port |
| `librechat__public_url` | `""` | If set, `DOMAIN_CLIENT` / `DOMAIN_SERVER` use this URL (required when not browsing via `localhost`) |
| `librechat__rag_port` | `8000` | RAG API port |
| `librechat__image_api` | `registry.librechat.ai/.../librechat-dev:latest` | Main API image |
| `librechat__image_rag_api` | `registry.librechat.ai/.../librechat-rag-api-dev-lite:latest` | RAG API image |
| `librechat__meili_master_key` | `changeme-minimum-32-chars-required` | Meilisearch master key (**must change** in production) |
| `librechat__config_version` | `1.3.6` | `version` in `librechat.yaml` (see [librechat_yaml](https://www.librechat.ai/docs/configuration/librechat_yaml)) |
| `librechat__env` | (see `defaults/main.yml`) | Includes `JWT_*`, `CREDS_*` — **rotate for production** (Vault recommended) |
| `librechat__env_extra` | `{}` | Extra `.env` entries merged **after** `librechat__env` (see below) |
| `librechat__state` | `started` | `started` or `stopped` |
| `librechat__firewall_allow` | `false` | Allow port in UFW |
| `librechat__remove_existing` | `false` | Remove Compose project before deploy |
| `librechat__remove_existing_home_dir` | `false` | **DANGER** Remove data dir (tests only) |
| `librechat__mcp_servers` | `[]` | MCP server definitions → `mcpServers` in `librechat.yaml` (prefer list format; mapping also supported for compatibility) ([object structure](https://www.librechat.ai/docs/configuration/librechat_yaml/object_structure/mcp_servers)) |
| `librechat__mcp_settings` | `{}` | `mcpSettings` (e.g. `allowedDomains` for remote MCP / Docker host) |
| `librechat__interface_mcp_permissions` | `{}` | `interface.mcpServers` UI flags (`use`, `create`, `share`, `public`) |
| `librechat__endpoints_custom` | default includes `Ollama` | Custom AI endpoints → `endpoints.custom` in `librechat.yaml` (prefer list format; mapping also supported for compatibility) |
| `librechat__manage_endpoints_in_place` | `false` | Create `librechat.yaml` once if missing, then only in-file edit a managed `endpoints.custom` block |
| `librechat__librechat_yaml_tail` | `""` | Raw YAML appended to `librechat.yaml` (e.g. paste blocks from [Smithery](https://smithery.ai)) |

See [LibreChat env configuration](https://www.librechat.ai/docs/configuration/dotenv).

**All variables from upstream [`.env.example`](https://github.com/danny-avila/LibreChat/blob/main/.env.example)** (and any other key–value pairs LibreChat reads from `.env`) can be set via **`librechat__env_extra`**: each dict key is the variable name, each value is the string written to `.env`. Values in `librechat__env_extra` override the same keys from `librechat__env` when merged. Use this for provider API keys, `ENDPOINTS=`, Redis, LDAP, MCP-related `MCP_*` tunables, UI vars like `APP_TITLE`, etc.

### Secrets and config file

The API needs **`JWT_SECRET`**, **`JWT_REFRESH_SECRET`**, **`CREDS_KEY`**, and **`CREDS_IV`** in `.env` or it crashes with `JwtStrategy requires a secret or key`. Defaults match the upstream [`.env.example`](https://github.com/danny-avila/LibreChat/blob/main/.env.example); **replace them** (e.g. `openssl rand -hex 32`) before production.

A minimal **`librechat.yaml`** is deployed to `{{ librechat__home }}/librechat.yaml` and mounted at `/app/librechat.yaml`. Extend the template in the role or add endpoints/models per the [config guide](https://www.librechat.ai/docs/configuration/librechat_yaml).

### First account — no default login

Per [LibreChat authentication](https://www.librechat.ai/docs/user_guides/authentication), **there is no default username/password**. The **first user** is created by choosing **Sign up** on the login page; that account becomes the **admin**.

- The role sets **`ALLOW_REGISTRATION=true`** and **`ALLOW_EMAIL_LOGIN=true`** in `.env` so sign-up is allowed (unless you override them in `librechat__env_extra`).
- **`DOMAIN_CLIENT`** and **`DOMAIN_SERVER`** must match the URL you use in the browser. Defaults are `http://localhost:{{ librechat__port }}`. If you open the site as **`http://192.168.x.x:3080`** or another hostname, set **`librechat__public_url`** to that full URL, then redeploy — otherwise cookies / auth can break and **Sign up may not appear or work**.
- You can also open the register route directly, e.g. `http://<host>:3080/register` (if your build exposes it).

### MCP (Model Context Protocol) servers

Configure MCPs **in Ansible** via `librechat__mcp_servers` (and optionally `librechat__mcp_settings` / `librechat__interface_mcp_permissions`), or **manually** by editing the generated `librechat.yaml` on the host.

- **[Smithery](https://smithery.ai)** can help you find MCP servers and produce YAML snippets. Paste entries into **`librechat__mcp_servers`** (preferred: list format), or paste a larger fragment into **`librechat__librechat_yaml_tail`** (appended as raw YAML).
- **Restart:** Any add/edit to MCP config requires a LibreChat restart so connections initialize. Re-running this role after a variable change redeploys `librechat.yaml` and notifies a stack restart (same as `docker compose restart` for the API).

Example (SSE + stdio, Docker host MCP URL):

```yaml
librechat__mcp_settings:
  allowedDomains:
    - "host.docker.internal"
    - "localhost"

librechat__interface_mcp_permissions:
  use: true
  create: true
  share: false
  public: false

librechat__mcp_servers:
  - name: "local-sse"
    type: sse
    url: "http://host.docker.internal:3001/sse"
  - name: "puppeteer"
    type: stdio
    command: npx
    args:
      - "-y"
      - "@modelcontextprotocol/server-puppeteer"
```

### Custom endpoints (`endpoints.custom`)

Configure custom model/API endpoints via `librechat__endpoints_custom`.
Preferred format is a list (same as LibreChat `endpoints.custom`). For backward compatibility, mapping format is also supported.
When one or more custom endpoints are configured, the role also ensures `.env` `ENDPOINTS` contains `custom` so the endpoint is visible in the LibreChat UI.

Default value includes a local Ollama endpoint (`host.docker.internal:11434`). Example override (OpenRouter, list format):

```yaml
librechat__endpoints_custom:
  - name: "OpenRouter"
    apiKey: "${OPENROUTER_KEY}"
    baseURL: "https://openrouter.ai/api/v1"
    models:
      default: ["openai/gpt-3.5-turbo"]
      fetch: true
    titleConvo: true
    titleModel: "openai/gpt-3.5-turbo"
    modelDisplayLabel: "OpenRouter"
```

### In-place endpoint edits (keep manual YAML changes)

Enable this if you want Ansible to only manage `endpoints.custom` in `librechat.yaml` and leave all other manual YAML edits untouched:

```yaml
librechat__manage_endpoints_in_place: true
```

Behavior:
- If `librechat.yaml` does not exist: role creates it.
- If it exists: role updates only the Ansible-managed endpoints block in-file.

## Example Playbook

```yaml
- hosts: chat_servers
  roles:
    - bsmeding.docker
    - bsmeding.librechat_docker
```

## Example host_vars

```yaml
librechat__port: 3080
# Match the URL you type in the browser (not localhost when using LAN IP):
# librechat__public_url: "http://192.168.1.50:3080"
librechat__meili_master_key: "your-secure-32-char-key-here"
librechat__firewall_allow: true
# Strong unique secrets (example generation):
# librechat__env_extra:
#   JWT_SECRET: "{{ lookup('password', '/dev/null length=64 chars=hex') }}"
#   JWT_REFRESH_SECRET: "{{ lookup('password', '/dev/null length=64 chars=hex') }}"
#   CREDS_KEY: "{{ lookup('password', '/dev/null length=64 chars=hex') }}"
#   CREDS_IV: "{{ lookup('password', '/dev/null length=32 chars=hex') }}"
librechat__env_extra:
  OPENAI_API_KEY: "sk-..."
  ANTHROPIC_API_KEY: "sk-ant-..."
```

## Production / release images

The main app is on **Docker Hub** as `librechat/librechat`. The **RAG API is not** on Docker Hub under `librechat/librechat-rag-api` (that name returns 404). Use **GHCR** or the project registry for RAG.

**Typical pairing:**

```yaml
# Main UI/API — Docker Hub
librechat__image_api: librechat/librechat:latest
# RAG API — GitHub Container Registry (lite build, matches upstream compose)
librechat__image_rag_api: ghcr.io/danny-avila/librechat-rag-api-dev-lite:latest
```

Alternatives: `registry.librechat.ai/danny-avila/librechat-rag-api-dev-lite:latest` (same stack as [upstream compose](https://github.com/danny-avila/LibreChat/blob/main/docker-compose.yml)), or the full RAG image per [RAG API docs](https://www.librechat.ai/docs/configuration/rag_api).

## License

MIT
