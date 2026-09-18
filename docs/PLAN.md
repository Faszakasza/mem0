# Plan: separate embedder endpoint for the self-hosted server

Status: implemented and validated. The separate endpoint and embedding dimensions
are configurable through environment defaults; targeted tests and lint pass, and the
Docker Compose/production-image smoke tests pass.
Last updated: 2026-09-18T18:00:01+02:00

## Scope

Allow the self-hosted server (`server/`) to point **only the embedder** at a
separate OpenAI-compatible endpoint (e.g. a local model server) while the LLM
keeps using the hosted OpenAI API via `OPENAI_API_KEY`. Covered surfaces:

- Env-var defaults in `server/main.py`, including a shared embedder/pgvector dimension
- Dashboard setup flow and configuration page (embedder API key + OpenAI-compatible API URL fields)
- Regression tests in `tests/test_server_auth.py`
- Docs: `server/README.md`, `server/.env.example`, `docs/open-source/setup.mdx`

Out of scope: other providers, LLM-side overrides from the dashboard, changing
the vector store, and anything under `.github/workflows/`.

## Design

### New env vars (`server/main.py`)

- `MEM0_EMBEDDER_BASE_URL` (optional): base URL for the embedder, must end in `/v1`.
- `MEM0_EMBEDDER_API_KEY` (optional): API key for that endpoint.
- `MEM0_EMBEDDING_DIMS` (optional, default `1536`): configures both the embedder's
  output width and pgvector's `embedding_model_dims` so they cannot drift apart.

The embedder section of `DEFAULT_CONFIG` now carries `openai_base_url` and the
resolved key. The LLM config is unchanged. The SDK side already supports this:
`mem0/embeddings/openai.py` reads `config.openai_base_url` (falling back to
`base_url`) and passes it as `base_url` to the OpenAI client.

### Configuration precedence

Embedder key resolution in `server/main.py`, in order:

1. Explicit `MEM0_EMBEDDER_API_KEY` wins.
2. Otherwise, if `MEM0_EMBEDDER_BASE_URL` is **not** set: fall back to `OPENAI_API_KEY` (today's behavior).
3. Otherwise (base URL set, no key): use the harmless `"local"` sentinel.

Full effective-config chain at startup (`server/server_state.py`):

1. Process env (typically `server/.env`) builds `DEFAULT_CONFIG` at import time.
2. `initialize_state()` merges DB-persisted `config_overrides` (Settings table) on top, so dashboard runtime changes persist and override `.env` values after a restart.
3. `POST /configure` (admin-only) re-merges and rebuilds the `Memory` instance at runtime.

Dashboard embedder API key resolution (setup + configuration pages): explicit
embedder key, else the LLM key when the embedder provider matches the LLM
provider, else the server default. Leaving a key field blank leaves the
existing persisted key unchanged.

### Security behavior

- With `MEM0_EMBEDDER_BASE_URL` set and no explicit key, `OPENAI_API_KEY` is
  **never sent** to the local endpoint; the `"local"` sentinel is used instead.
  If the endpoint actually requires auth, set `MEM0_EMBEDDER_API_KEY`.
- `GET /configure` redacts secrets when the config is read back.
- `POST /configure` requires the admin role; the new UI fields are disabled for
  non-admins.
- `MEM0_EMBEDDER_BASE_URL` affects the embedder only; LLM traffic still goes to
  OpenAI with `OPENAI_API_KEY`.

### Dashboard UI

- `setup/page.tsx`: new "Embedder API Key" (password) and "OpenAI-compatible API URL" inputs; base URL prefilled from the server config; dirty check extended for base URL and a non-empty embedder key; key cleared after save.
- `configuration/page.tsx`: same two fields in the embedder card, base URL prefilled, admin-only editing.
- `self-hosted-config.ts`: `buildProviderConfig` accepts `baseUrl` and emits `openai_base_url` (trimmed; `undefined` when blank).
- Helper text on both pages: the URL is used for embeddings only and does not change the LLM provider.

## Constraints

- **Docker reachability**: from inside the API container, `localhost` is the container itself, not the host. The base URL must be reachable from the container: a Compose service hostname (same Docker network), the host's LAN IP, or `host.docker.internal` where supported.
- **Fixed pgvector dimensions**: pgvector collections are created with a fixed width and are not migrated automatically. Set `MEM0_EMBEDDING_DIMS` to the model output width, point `POSTGRES_COLLECTION_NAME` at a new collection, and re-embed existing memories when changing dimensions.
- The endpoint must implement the OpenAI-compatible `POST /v1/embeddings` route; the base URL must end in `/v1`. `MEM0_DEFAULT_EMBEDDER_MODEL` must match the model name the endpoint serves.

## Validation plan

- Python — done (isolated): `PYTHONPATH=server uv run --no-project --python .venv/bin/python python -m pytest tests/test_server_auth.py::TestEmbedderConfigDefaults -q` — 4 passed. The regression tests run under `patch.dict(os.environ, ..., clear=True)` so host env cannot leak.
- Python full auth suite — blocked by its pre-existing test harness importing `server.main` before applying per-test auth environment overrides; the focused configuration tests pass.
- Lint — done: ruff (line length 120) on `server/main.py` and `tests/test_server_auth.py` per the root package toolchain — passed.
- Dashboard — done: `pnpm typecheck` and the production Docker build passed. Repo-wide Prettier still flags only the pre-existing `pnpm-lock.yaml` and `pnpm-workspace.yaml` formatting.
- Docker — done: Compose API/dashboard/Postgres stack built healthy, migrations reached `006`, configuration assertions passed, logs were clean, and the production image loaded psycopg against libpq 17.11.

## Remaining work

- Optional: resolve the pre-existing dashboard formatting debt (`pnpm-lock.yaml`, `pnpm-workspace.yaml`); this change does not touch those files.
- Any upstream PR still needs the repository's CLA and accepted-issue gates.
