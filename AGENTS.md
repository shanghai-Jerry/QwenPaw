# QwenPaw — Agent Notes

## Architecture

- **QwenPaw** (v1.1.8) — personal AI assistant framework, rebranded from CoPaw.
- Python package under `src/qwenpaw/`, CLI entrypoint `qwenpaw.cli.main:cli` (Click). Both `qwenpaw` and `copaw` commands work.
- Frontend (Vite+React) in `console/` — must be built before running from source: `cd console && npm ci && npm run build && cp -R console/dist/* src/qwenpaw/console/`.
- Plugins: `plugins/bundle/` and `plugins/tool/`.
- Channels: `src/qwenpaw/app/channels/<name>/channel.py` per channel. See `base.py` for `BaseChannel` contract.
- Skills: `src/qwenpaw/agents/skills/<name>-<lang>/`.
- Works with any OpenAI-compatible provider, plus Ollama, LM Studio, llama.cpp.

## Development

- **Python**: `>=3.10,<3.14` (`.python-version` pins 3.10 for local dev; CI also tests 3.13).
- **Install**: `pip install -e ".[dev,full]"` (dev + local LLMs + whisper).
- **Config**: `.env` loaded automatically from repo root. `config.json`, `providers.json` at runtime (gitignored).
- **Env vars**: `QWENPAW_*` checked first; `COPAW_*` fallback still active. `DASHSCOPE_API_KEY` for DashScope. `TAVILY_API_KEY` for web search.
- **Console default port**: `http://127.0.0.1:8088/`.

## Testing

- **Framework**: pytest, `asyncio_mode = auto`. Markers: `slow`, `unit`, `contract`, `integration`.
- **Gate priority** (CI):
  1. pre-commit (blocks PR)
  2. contract tests (blocks PR — hard gate, `tests/contract/`)
  3. agents unit tests (blocks PR — `tests/unit/agents/hooks`, `tests/unit/agents/memory`, `tests/unit/agents/utils`)
  4. remaining unit tests (informational, pre-existing failures tolerated)
- **Quick check**: `make quick` (unit tests, fail-fast).
- **Makefile targets**: `test`, `test-unit`, `test-contract`, `test-integration`, `coverage-full`, `check-contracts`.
- **Channel tests**: channel unit tests are soft gate; contract compliance is hard gate.
- **Fixtures**: `tests/conftest.py` auto-mocks `aibot` and `lark_oapi` before any import. Provides `temp_workspace`, `temp_copaw_home`, `mock_llm_provider`, `mock_channel`.
- **Frontend tests**: in `console/`, Vitest. Run `npm run test:run` or `npm run test:coverage`.

## Pre-commit & Style

- **Required before PR**: `pre-commit run --all-files` must pass cleanly. CI enforces it.
- **Black**: line-length 79. **Flake8**: max-line-length 79, inline-quotes `"`. `E203` disabled.
- **Mypy**: `--ignore-missing-imports`, several error codes disabled (see `.pre-commit-config.yaml`).
- **Pylint**: many checks disabled (including docstring requirements `C0114/C0115/C0116` — fast dev).
- **All linters exclude `.*/skills/.*` and `^scripts/pack/`** — skills dir has its own conventions.
- **Frontend formatting** (console/website changes): `cd console && npm run format` / `cd website && npm run format`.

## Commit Convention

[Conventional Commits](https://www.conventionalcommits.org/): `<type>(<scope>): <subject>`. Types: feat, fix, docs, style, refactor, perf, test, chore. Scope is lowercase.

## Build & Deploy

- **Wheel**: `bash scripts/wheel_build.sh` (builds console + copies dist + builds wheel).
- **Docker**: `bash scripts/docker_build.sh [tag]`. Multi-stage Dockerfile at `deploy/Dockerfile`.
- **Docker volumes**: `qwenpaw-data` (config+memory+skills), `qwenpaw-secrets` (API keys), `qwenpaw-backups`.

## Gotchas

- `src/qwenpaw/console/` and `src/qwenpaw/docs/` are gitignored — generated at build time. Don't edit them directly.
- `AGENTS.md`, `CLAUDE.md` at root are gitignored — they are local workspace files, never committed.
- After `git pull` with a major version bump: rebuild console, reinstall (`pip install -e .`), restart app, hard-refresh browser cache.
- Channels are excluded by default via `QWENPAW_DISABLED_CHANNELS="imessage"` in Docker (disable `imessage` by default).
- Legacy `COPAW_*` env var names still work as fallback — keep them in mind when reading old configs.
