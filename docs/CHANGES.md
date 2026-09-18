# Changes

Log of non-trivial changes to this workspace, newest first.

## 2026-09-18T15:26:39+02:00 — Separate embedder endpoint for the self-hosted server

Added `MEM0_EMBEDDER_BASE_URL` / `MEM0_EMBEDDER_API_KEY` so the self-hosted
server can point only the embedder at a separate OpenAI-compatible endpoint
(e.g. a local model server) while the LLM keeps using `OPENAI_API_KEY`.

- `server/main.py`: new env-var defaults with key resolution (explicit key wins;
  `OPENAI_API_KEY` fallback only when no base URL is set; `"local"` sentinel
  when a base URL is set without a key, so `OPENAI_API_KEY` is never sent to
  the local endpoint). Embedder section of `DEFAULT_CONFIG` now carries
  `openai_base_url`.
- Dashboard: `setup/page.tsx` and `configuration/page.tsx` gained an embedder
  API key + OpenAI-compatible API URL pair (admin-only, base URL prefilled,
  blank key preserves the persisted key); `self-hosted-config.ts`
  `buildProviderConfig` emits `openai_base_url`.
- `tests/test_server_auth.py`: `TestEmbedderConfigDefaults` regression tests
  covering the sentinel, explicit-key precedence, and the no-config fallback,
  run under a fully controlled environment (`patch.dict(..., clear=True)`).
- Docs: `server/README.md` and `docs/open-source/setup.mdx` gained a "Separate
  embedder endpoint" section (OpenAI-compatible `/v1/embeddings` requirement,
  Docker reachability, fixed pgvector dimensions); `server/.env.example`
  documents the new vars; setup.mdx step 2 updated for the editable
  configuration page.

Uncommitted in the working tree. Validation (all sandbox-runnable checks done as of
2026-09-18 15:49 +02:00): `TestEmbedderConfigDefaults` regression tests 3 passed in
isolation, ruff (line length 120) passed on `server/main.py` and
`tests/test_server_auth.py`, dashboard `pnpm typecheck` passed, and `git diff --check`
passed. `pnpm lint` (`prettier --check .`) fails only on pre-existing, unchanged
`pnpm-lock.yaml` and `pnpm-workspace.yaml`; the changed source files pass Prettier.
The full `tests/test_server_auth.py` suite is not runnable in this sandbox: without
`JWT_SECRET` it fails at a pre-existing import guard, and with `JWT_SECRET` it requires
Postgres, which is unavailable here, and exceeds 300s; a clean HEAD shows the same
environmental failure pattern. The production build (`pnpm build`) and the manual
embedding smoke test have not been run. Remaining: optional Docker/Postgres
full-suite smoke validation, optional existing dashboard formatting debt, commit.
See `docs/PLAN.md` and `docs/PROGRESS.md`.
