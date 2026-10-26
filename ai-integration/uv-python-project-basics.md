# uv: Python project and environment management — commands, model, gotchas

`uv` replaces pip + venv + pyenv + pip-tools + pipx with one binary: it resolves and locks
dependencies, creates the virtualenv, installs (or downloads) the interpreter, and runs
commands inside the environment. It's the tooling behind the Python side of this folder
(the FastAPI boundary in [fastapi-llm-service-boundary.md](fastapi-llm-service-boundary.md),
the OpenAI/Ollama clients). Verified on uv 0.12.10 and 0.13.0, macOS arm64.

## Mental model

Four files/dirs, and which command owns which:

| Thing | Who writes it | Commit? |
|---|---|---|
| `pyproject.toml` | you, or `uv add` / `uv remove` (edits `[project].dependencies` / `[dependency-groups]`) | yes |
| `uv.lock` | `uv lock`, and implicitly by `add`/`remove`/`sync`/`run` | **yes** — the cross-platform, exact-version resolution |
| `.python-version` | `uv init`, `uv python pin` | yes — which interpreter line the venv is built from |
| `.venv/` | `uv sync`/`uv run` (created on demand, never by hand) | no (`uv init` gitignores it) |

The flow is always *declare → lock → sync → run*, and `uv run` does the last three for you:
before executing it re-locks if `pyproject.toml` changed and syncs the venv, then runs the
command with `.venv/bin` first on `PATH`. You never activate the venv.

```bash
uv init myproj && cd myproj      # pyproject.toml, .python-version, README.md, .gitignore (.venv ignored), git init
uv add openai pydantic            # resolve, write pyproject + uv.lock, install into .venv (created now)
uv add --dev ruff pytest          # [dependency-groups].dev — installed by default, excluded by --no-dev
uv add "fastapi>=0.141"           # quote specifiers (shell would glob `>`)
uv remove httpx
uv run python script.py           # the only way you need to run things
uv run uvicorn api:app --reload   # console scripts of installed deps work the same
uv run -m pytest
uv sync                           # (re)create .venv exactly from uv.lock; what CI / a fresh clone runs
uv lock --check                   # fail if uv.lock is stale vs pyproject (CI gate)
uv lock --upgrade                 # bump everything within constraints; --upgrade-package openai for one
uv tree --depth 1                 # resolved versions incl. dev group
uv export --format requirements-txt --no-hashes > requirements.txt   # for tooling that can't read uv.lock
uv self update                    # upgrade uv itself (standalone install only — see below)
uv self version                   # = uv --version
```

## Keeping uv itself current: `uv self update`

uv moves fast (0.12.10 → 0.13.0 within a month here) and lockfile `revision`s, `uv init`
defaults and built-ins (`format`/`check`/`audit`) change between minors, so update
deliberately rather than never.

```bash
uv self update              # latest release from GitHub; prints old → new version
uv self update --dry-run    # show what it would do
uv self update 0.12.10      # pin / roll back to an exact version
uv self version             # current version + commit + platform
```

- **Only works for the standalone installer** (`curl -LsSf https://astral.sh/uv/install.sh | sh`,
  binaries in `~/.local/bin`, receipt in `~/.config/uv/uv-receipt.json` — that receipt is
  what `self update` reads). A Homebrew (`brew upgrade uv`) or `pip install uv` copy must
  be updated by that manager; `uv self update` refuses or updates the wrong binary.
  `which -a uv` tells you which copy you're running if both exist.
- Teams: pin the version in CI (`astral-sh/setup-uv` `version:` input) and bump it with
  the lockfile, so a developer's `self update` can't silently rewrite `uv.lock` to a newer
  `revision` than CI understands.

## Project discovery and layouts

- `uv run` walks **up** from the cwd to the nearest `pyproject.toml`. A repo with one root
  `pyproject.toml` and per-topic subfolders of scripts (no manifests inside them) works: run
  from any subfolder and it uses the root `.venv` (verified: `sys.prefix` from a nested
  week folder resolved to the repo-root venv). `--project DIR` / `--directory DIR` override.
- **`uv init` defaults changed:** on 0.12 `uv init NAME` produces a *packaged* layout —
  `src/NAME/__init__.py`, `[build-system] uv_build`, `[project.scripts]` — and `uv sync`
  installs the project itself editable (`source = { editable = "." }` in `uv.lock`). For a
  folder of standalone scripts you want a **virtual project** instead: `uv init --bare`
  (or `--no-package`, or just delete `[build-system]`) → `source = { virtual = "." }`,
  nothing of your own gets installed, imports are plain sibling-file imports from the cwd.
  The sandbox repo is virtual; a packaged project would make `from contracts import …`
  depend on the `src/` layout and on running from the right place.
- Scripts with no project: `uv init --script s.py` / `uv add --script s.py httpx` writes
  PEP 723 inline metadata (`# /// script … dependencies = ["httpx>=0.28.1"] … ///`) at the
  top of the file and `uv run s.py` builds a throwaway cached env for it — a single-file
  deliverable with pinned deps. `uv run --no-project` ignores any enclosing pyproject.

## Gotchas verified here

- **`uv run` syncs *inexactly*; `uv sync` syncs *exactly*.** After deleting a dependency
  from `pyproject.toml`, `uv run` re-locked (the package left `uv.lock`) and reported a
  sync, yet the package was **still importable** — `uv run` only adds/updates, it does not
  remove extraneous packages unless `--exact`. `uv sync` removed it. So "I deleted it from
  pyproject and the import still works" is not proof of a stale cache; it's proof you ran
  via `uv run`. Same reason a fresh clone (`uv sync`) can fail where your machine passes.
- **Which Python backs `.venv` is whatever matched first on `PATH`**, not necessarily the
  newest or the uv-managed one. With `.python-version` = `3.13` and three 3.13 builds on
  the test machine (Homebrew 3.13.3, python.org 3.13.2, miniconda 3.13.2), the venv was built on
  **miniconda's** (`home = /opt/miniconda3/bin` in `.venv/pyvenv.cfg`). Pin a full version
  (`uv python pin 3.13.2`), or `uv python install 3.13` + `UV_MANAGED_PYTHON=1` /
  `--managed-python` to force uv's own builds. Check with `uv python find` / `uv python list`.
  Changing the pin after the venv exists: `uv sync` rebuilds the venv on the new interpreter.
- **An activated venv from another project is ignored**, with a warning
  (`VIRTUAL_ENV=… does not match the project environment path .venv and will be ignored;
  use --active`). `uv run` targets the project's `.venv`, full stop — a leftover
  `source other/.venv/bin/activate` in the shell is harmless but confusing.
- `uv run --with rich python …` injects a package for one invocation without touching the
  project (temporary layered env). `uvx ruff --version` (= `uv tool run`) runs a CLI from a
  cached, isolated env — nothing lands in the project; `uv tool install ruff` makes it a
  persistent command in `~/.local/share/uv/tools` (uv's pipx).
- `--frozen` runs with the existing `uv.lock` without checking it against pyproject (fast,
  CI/Docker); `--locked` errors if the lock is stale (the safety version). `--no-sync`
  skips installing entirely.
- 0.12 ships `uv format` (ruff format), `uv check` (ruff check) and `uv audit`
  (vulnerability scan of the lock) as built-ins, no dev dependency needed for the first two.

## Environment and housekeeping

- Cache at `~/.cache/uv` (`uv cache dir`; 1.1 GB here after a few projects) — wheels and
  built packages are **hard-linked** into every venv, so a second project pulling the same
  versions costs milliseconds and ~0 bytes. `uv cache prune` drops entries no venv uses;
  `uv cache clean` nukes it. `.venv` dirs are disposable: delete and `uv sync`.
- `uv run` loads a `.env` only when asked: `--env-file .env` or `UV_ENV_FILE=.env`
  (`OPENAI_API_KEY` stays out of the repo and out of `pyproject.toml`).
- Useful env vars: `UV_PROJECT_ENVIRONMENT` (venv path, e.g. a shared one in Docker),
  `UV_PYTHON`, `UV_MANAGED_PYTHON`, `UV_FROZEN`/`UV_LOCKED`, `UV_NO_SYNC`, `UV_OFFLINE`,
  `UV_INDEX_URL`/`UV_EXTRA_INDEX_URL` (private indexes), `UV_NO_CACHE`.
- `uv venv` and `uv pip install …` exist as a drop-in pip/venv for repos that aren't uv projects — don't
  mix them with `uv add` in the same project, the lock won't know about pip-installed packages.
