# Plan: separate embedder endpoint for the self-hosted server

Status: implemented in the working tree (uncommitted); all sandbox-runnable validation
passed (`TestEmbedderConfigDefaults` 3 passed in isolation, ruff on the two changed
Python files, dashboard `pnpm typecheck`, Prettier on changed files, `git diff --check`).
Full auth suite is not runnable in this sandbox (environmental: needs Postgres; same
failure pattern at clean HEAD). Remaining: optional Docker/Postgres full-suite smoke
validation, optional existing dashboard formatting debt, commit.
Last updated: 2026-09-18T15:49:27+02:00

## Scope

Allow the self-hosted server (`server/`) to point **only the embedder** at a
separate OpenAI-compatible endpoint (e.g. a local model server) while the LLM
keeps using the hosted OpenAI API via `OPENAI_API_KEY`. Covered surfaces:

- Env-var defaults in `server/main.py`
- Dashboard setup flow and configuration page (embedder API key + OpenAI-compatible API URL fields)
- Regression tests in `tests/test_server_auth.py`
- Docs: `server/README.md`, `server/.env.example`, `docs/open-source/setup.mdx`

Out of scope: other providers, LLM-side overrides from the dashboard, changing
the vector store, and anything under `.github/workflows/`.

## Design

### New env vars (`server/main.py`)

- `MEM0_EMBEDDER_BASE_URL` (optional): base URL for the embedder, must end in `/v1`.
- `MEM0_EMBEDDER_API_KEY` (optional): API key for that endpoint.

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
- **Fixed pgvector dimensions**: pgvector collections are created with a fixed width and are not migrated automatically. If the new endpoint's model produces a different embedding dimension, point `POSTGRES_COLLECTION_NAME` at a new collection and re-embed existing memories.
- The endpoint must implement the OpenAI-compatible `POST /v1/embeddings` route; the base URL must end in `/v1`. `MEM0_DEFAULT_EMBEDDER_MODEL` must match the model name the endpoint serves.

## Validation plan

- Python — done (isolated): `PYTHONPATH=server /tmp/mem0srv-venv/bin/python -m pytest tests/test_server_auth.py::TestEmbedderConfigDefaults -q` — 3 passed (2 pre-existing deprecation warnings, unrelated: anyio `BlockingPortal` alias, `crypt` module). The 3 new regression tests run under `patch.dict(os.environ, ..., clear=True)` so host env cannot leak.
- Python — not runnable in this sandbox: the full `pytest tests/test_server_auth.py` file. Without `JWT_SECRET` it fails at a pre-existing import guard; with `JWT_SECRET` it requires Postgres, which is unavailable here, and the run exceeds 300s. A clean HEAD shows the same environmental failure pattern. Defer to an environment with Docker/Postgres (optional full-suite smoke validation).
- Lint — done: ruff (line length 120) on `server/main.py` and `tests/test_server_auth.py` per the root package toolchain — passed.
- Dashboard — done: deps installed with `corepack pnpm install --frozen-lockfile`; `corepack pnpm typecheck` (tsc --noEmit) passed; `pnpm lint` (`prettier --check .`) run and fails only on pre-existing, unchanged `pnpm-lock.yaml` and `pnpm-workspace.yaml` — the changed source files pass Prettier. — not run: production build (`pnpm build`).
- Manual — pending: bring up the stack with a local OpenAI-compatible embedder, confirm embeddings flow to it, the LLM still hits OpenAI, and the dashboard setup/configuration pages round-trip the new fields.

## Remaining work

- Optional Docker/Postgres full-suite smoke validation: full `pytest tests/test_server_auth.py` (the targeted `TestEmbedderConfigDefaults` class already passed in isolation) plus the manual stack smoke test with a local OpenAI-compatible embedder. Both are blocked environmentally in this sandbox (see Validation plan) — environmental, not a regression.
- Optional, only if desired: resolve the pre-existing dashboard formatting debt (`pnpm-lock.yaml`, `pnpm-workspace.yaml`) that repo-wide `pnpm lint` flags; this change does not touch those files.
- Commit the working tree (explicitly not committed per task instructions).
- Any follow-up: PR hygiene (CLA, accepted-issue gate) if this goes upstream.
