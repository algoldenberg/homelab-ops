# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project
homelab-ops: read-only monitoring for a small homelab (Raspberry Pi + VPS), exposed
through an MCP server and a Telegram bot. Public repo, portfolio project.

## Commands

```bash
uv sync                                      # install all deps (including dev group)
uv run pytest                                # run all tests
uv run pytest tests/test_foo.py::test_name   # run a single test
uv run ruff check src tests                  # lint
uv run ruff format src tests                 # format
uv run pre-commit run --all-files            # run all hooks (ruff + gitleaks)
```

Before you say a task is done: tests and ruff must both pass.

## Architecture (src/homelab_ops/)
- `core/`       pure check logic (docker, disk, logs, nginx, systemd, ssl, ports).
                Returns typed results (dataclasses / pydantic). No I/O frameworks here.
- `agent/`      FastAPI app, one per host. Read-only endpoints over `core`.
                Bearer token auth. Binds ONLY to the VPN interface (WireGuard/Tailscale).
- `hub/`        async httpx client that polls all agents listed in `config.yaml`.
- `mcp_server/` FastMCP over `hub` (stdio + streamable HTTP).
- `tg_bot/`     router that plugs into an EXISTING bot. Don't make it a standalone bot.

Dependency direction: tg_bot / mcp_server -> hub -> agent (HTTP) -> core.
Nothing may import "upward".

## Hard rules (security)
- The VPS (`vps-main`) is LIVE PRODUCTION. Never run commands against it,
  never ssh, never suggest restarts as a fix. Work locally and in tests only.
- Read-only by default. Any mutating action (restart) needs ALL of:
  a service allowlist in config, Telegram inline-button confirmation, and an audit log entry.
- Never hardcode secrets, hostnames, IPs, user IDs or paths.
  Secrets go in `.env` and host data in `config.yaml`. Both are gitignored.
  Only `.env.example` / `config.example.yaml` with fake values are committed.
- Never read or print `.env` or `config.yaml`.
- Shell out only through `asyncio.create_subprocess_exec` with an argument list.
  Never use `shell=True` or build command strings from input.

## Tooling
- **Python 3.12** pinned via `.python-version`
- **ruff** for linting (`E,F,I,UP,B,SIM,ASYNC,S` rules) and formatting; line length 100
- **gitleaks** runs on staged files at commit time — never commit secrets
- Dev deps live in `[dependency-groups] dev` in `pyproject.toml` (uv native, not `[project.optional-dependencies]`)
- Machine-specific notes (local paths etc.) go in `CLAUDE.local.md` (gitignored)

## Code conventions
- Python 3.12, full type hints, `async` for I/O.
- Small pure functions in `core`. Parsing is kept separate from executing,
  so you can test parsers on fixture output without docker/systemd.
- Tests: pytest + pytest-asyncio. Mock subprocess calls and keep sample command
  output in `tests/fixtures/`. Tests must not need docker or network.
- Logging via `logging`, no `print`.
- Commit messages: Conventional Commits (`feat(core): ...`, `fix(agent): ...`).

## Workflow
Explore -> Plan -> Code -> Commit. For anything bigger than one function,
propose a plan first and wait for approval.