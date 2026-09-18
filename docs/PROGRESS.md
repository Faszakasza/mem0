# Progress

Session log for workspace work, newest first.

## 2026-09-18 (session 15:49 +02:00) — Validation status update

**Validation completed in this sandbox** (all against the current working tree):

- `PYTHONPATH=server /tmp/mem0srv-venv/bin/python -m pytest tests/test_server_auth.py::TestEmbedderConfigDefaults -q` — 3 passed (isolated run).
- Ruff (line length 120) on `server/main.py` and `tests/test_server_auth.py` — passed.
- Dashboard `pnpm typecheck` (tsc --noEmit) — passed.
- Dashboard `pnpm lint` (`prettier --check .`) — run; fails only on pre-existing, unchanged `pnpm-lock.yaml` and `pnpm-workspace.yaml`; the changed source files pass Prettier.
- `git diff --check` — passed.

**Not runnable in this sandbox** (environmental, not a regression):

- Full `pytest tests/test_server_auth.py` — without `JWT_SECRET` it fails at a pre-existing import guard; with `JWT_SECRET` it requires Postgres, which is unavailable here, and the run exceeds 300s. A clean HEAD shows the same environmental failure pattern.
- Dashboard production build (`pnpm build`) — not run.
- Manual embedding smoke test (local OpenAI-compatible embedder) — not run.

**Remaining work** (updated; supersedes the previous entry's remaining-work list):

- Optional Docker/Postgres full-suite smoke validation: full `tests/test_server_auth.py` and the manual stack check with a local OpenAI-compatible embedder, in an environment that has Postgres.
- Pre-existing dashboard formatting debt (`pnpm-lock.yaml`, `pnpm-workspace.yaml`) — only if desired; not touched by this change.
- Commit the working tree (explicitly not committed per task instructions).

Working tree unchanged apart from the tracking docs: 8 modified files +
`docs/PLAN.md`, `docs/CHANGES.md`, `docs/PROGRESS.md` untracked.

## 2026-09-18 (session 15:24–15:27 +02:00) — Separate embedder endpoint

**Record corrected (15:31 +02:00):** an earlier version of this entry claimed the
targeted regression tests and dashboard checks were "not yet run." Completed worker
reports and parent verification confirm they ran and passed; the validation section
below reflects that.

**Cancelled worker run.** An earlier worker run for this task was cancelled
mid-flight. Its working tree was cleaned up before the retry: no stray or
untracked files remained, and `git status --porcelain` shows exactly the eight
intended modified files (see below). The work was then retried and completed.

**Completed steps** (all in the working tree, uncommitted):

- `server/main.py` — `MEM0_EMBEDDER_BASE_URL` / `MEM0_EMBEDDER_API_KEY` defaults
  with key resolution (explicit key wins; `OPENAI_API_KEY` fallback only when no
  base URL is set; `"local"` sentinel otherwise) and `openai_base_url` in the
  embedder section of `DEFAULT_CONFIG`.
- Dashboard — embedder API key + OpenAI-compatible API URL fields in
  `setup/page.tsx` and `configuration/page.tsx`; `buildProviderConfig` in
  `self-hosted-config.ts` emits `openai_base_url`.
- `tests/test_server_auth.py` — `TestEmbedderConfigDefaults` (3 regression tests)
  covering sentinel, explicit-key precedence, and no-config fallback, under a
  fully controlled environment.
- Docs — "Separate embedder endpoint" sections in `server/README.md` and
  `docs/open-source/setup.mdx`, `server/.env.example` vars, setup.mdx step 2
  rewritten for the editable configuration page.
- Tracking docs created: `docs/PLAN.md`, `docs/CHANGES.md`, this file.

Working tree at this point (8 modified files, plus the three untracked tracking
docs listed above):

```
 M docs/open-source/setup.mdx
 M server/.env.example
 M server/README.md
 M server/dashboard/src/app/(root)/dashboard/configuration/page.tsx
 M server/dashboard/src/app/setup/page.tsx
 M server/dashboard/src/utils/self-hosted-config.ts
 M server/main.py
 M tests/test_server_auth.py
?? docs/CHANGES.md
?? docs/PLAN.md
?? docs/PROGRESS.md
```

**Validation** (run in the completed worker run; re-verified against the working
tree at record-correction time):

- `PYTHONPATH=server /tmp/mem0srv-venv/bin/python -m pytest tests/test_server_auth.py::TestEmbedderConfigDefaults -q` — 3 passed (2 pre-existing deprecation warnings, unrelated: anyio `BlockingPortal` alias, `crypt` module).
- Dashboard (`server/dashboard`; deps installed with `corepack pnpm install --frozen-lockfile`): `corepack pnpm typecheck` (tsc --noEmit) passed; Prettier check passed on the three changed dashboard files. Production build (`pnpm build`) **not** run. Note: repo-wide `pnpm lint` (`prettier --check .`) additionally flags pre-existing formatting in the unmodified `pnpm-lock.yaml` / `pnpm-workspace.yaml`, unrelated to this change.
- `git diff --check` — passed.

**Remaining work** (not done as of this entry — nothing below has been run or
committed yet):

- Full `pytest tests/test_server_auth.py` suite (the targeted `TestEmbedderConfigDefaults` class already passed).
- Final lint/build/manual smoke checks as applicable: ruff (line length 120) on `server/main.py` and `tests/test_server_auth.py`; dashboard production build (`pnpm build`); optional manual check with a local OpenAI-compatible embedder.
- Commit the working tree (explicitly not committed per task instructions).
