# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this role does

Ansible role `bsmeding.librechat_docker` — deploys [LibreChat](https://www.librechat.ai/) (ChatGPT-style multi-provider AI web UI) via Docker Compose. Requires the `bsmeding.docker` role to be applied first (provides Docker + Compose plugin, and exposes `docker_uid`/`docker_gid` facts).

Collections required: `community.docker`, `community.general`.

## Testing commands

```bash
# Lint YAML
pip3 install yamllint
yamllint .

# Run full Molecule test (creates container, converges, verifies, destroys)
molecule test

# Run against a specific distro
MOLECULE_DISTRO=debian12 molecule test
MOLECULE_DISTRO=ubuntu2404 molecule test

# Develop iteratively (faster — keeps container alive)
molecule converge
molecule verify
molecule destroy

# Available distros (from CI matrix)
# ubuntu2204, ubuntu2404, debian12, debian13
```

The Molecule driver is Docker. The test container mounts the host Docker socket so the role can launch real Docker Compose stacks inside it. The `prepare.yml` step installs the Compose v2 CLI plugin if missing.

## Architecture

```
defaults/main.yml        # All role variables with defaults
tasks/main.yml           # Entry point — just includes setup.yml
tasks/setup.yml          # All real logic (ordered): fact-building → dirs → templates → compose
handlers/main.yml        # "Restart LibreChat" — docker_compose_v2 state: restarted
templates/
  env.j2                 # Generates .env from librechat__env_merged
  docker-compose.yml.j2  # Full Compose stack (API, MongoDB, Meilisearch, pgvector, RAG API)
  librechat.yaml.j2      # LibreChat app config (MCP servers, endpoints, etc.)
  librechat.endpoints.block.j2  # Used only with librechat__manage_endpoints_in_place: true
molecule/default/        # Molecule test scenario
```

### Key data-flow in `setup.yml`

1. **Fact-building** — constructs `librechat__env_merged` (merge of `librechat__env` + domain override from `librechat__public_url` + `librechat__env_extra`), and normalises `librechat__mcp_servers` / `librechat__endpoints_custom` from either list or mapping format into canonical forms (`librechat__mcp_servers_map`, `librechat__endpoints_custom_list`).
2. Auto-appends `custom` to the `ENDPOINTS` env var when `librechat__endpoints_custom` is non-empty.
3. Templates → `docker-compose.yml`, `.env`, `librechat.yaml` — each change notifies the **Restart LibreChat** handler.
4. Compose stack brought up via `community.docker.docker_compose_v2`.

### Two modes for `librechat.yaml`

- **Default (`librechat__manage_endpoints_in_place: false`)** — fully re-templates `librechat.yaml` on every run.
- **In-place mode (`librechat__manage_endpoints_in_place: true`)** — creates the file once; subsequent runs only update the Ansible-managed `endpoints.custom` block using `blockinfile` (marker: `# {mark} ANSIBLE MANAGED BLOCK: LIBRECHAT ENDPOINTS`), leaving manual edits intact.

### Variable naming convention

All variables use the `librechat__` prefix (double underscore) to avoid conflicts.

### Secrets note

`librechat__env` ships with upstream example values for `JWT_SECRET`, `JWT_REFRESH_SECRET`, `CREDS_KEY`, `CREDS_IV`. These **must** be rotated for production (use Ansible Vault or `lookup('password', ...)`). The Meilisearch key `librechat__meili_master_key` also defaults to a placeholder.
