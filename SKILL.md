---
name: python-dev
description: Python development standards and practices for zero-fabrication, test-driven development with strict quality gates. Use when working on Python projects that require rigorous testing, linting, and architecture standards with real integrations only.
---

# Python Development Standards

This skill provides comprehensive Python development standards focused on real implementations, rigorous testing, and zero fabrication.

## GUI Development

**If the task involves ANY GUI work** (desktop app, window, dialog, widget, display, visualization), you MUST read and follow `gui.md` in this skill directory BEFORE writing any GUI code. This includes:
- Creating a new GUI application
- Adding windows, dialogs, or visual components to an existing app
- Modifying styling, layout, colors, or fonts
- Adding icons or images to an application

The GUI standards ensure all desktop applications on this system share a consistent visual language, framework (PySide6), and architectural approach.

---

## Repository & Project Layout

**Required structure**
```
README.md
requirements.txt
.gitignore
run              # Python file with argparse (no .py), executable
src/             # all source code
output/          # gitignored runtime outputs
  testing/       # gitignored test output/logs/artifacts
local/           # gitignored large downloads/artifacts
```

**Directory constraints**
- Shallow structure preferred. Once you use subdirectories, place peer modules at the same nesting level.
- Max 20 files per directory.
- No experimental scripts or alt versions at root or anywhere else; use branches for iteration.
- **output/ and local/ must be accessed relative to the code, not the current working directory.** The caller's cwd is never a thing. Use `Path(__file__).parent` or similar to resolve paths relative to the script location.

**Example `.gitignore`**
```
output/
local/
.venv/
__pycache__/
*.pyc
```

---

## Virtual Environment

**Always use venv** as the environment manager. No conda, no poetry, no pipenv.

**Critical architecture: `run` is OUTSIDE the venv.** `run` itself runs with the ambient system Python. Its job is to:
1. **Create the venv** if it doesn't exist (`python3 -m venv .venv`)
2. **Install requirements** into the venv (`pip install -r requirements.txt`)
3. **Delegate everything** into the venv via `os.execv` into the venv Python

Callers never create, activate, or think about the venv. They just call `./run` or `~/bin/projectname` from anywhere. The venv is a local implementation detail that `run` manages transparently.

---

## Coding Standards

**Naming policy**
- Never use a name not in the dictionary
- Prefer snake_case for all identifiers - especially in the database

**Commenting policy**
- No docstrings. Use regular comments only.
- Every class/function must end with a **comment block** with line and two fields: a name and a description.
  - `# ##################################################################`
  - `# <short human name>`
  - `# <concise why + key intent in prose>`
- Keep functions tiny and obviously correct.
- A function with a loop should primarily loop and call a named helper.
- A multi-step function should delegate each step to named helpers.

**proc name**
there's a libary setproctitle, which should be pip installed - every app at the start
of main should do 
    setproctitle.setproctitle('meaningful title')

**Comment-block example**
```python
# ##################################################################
# clean repository
# restore the working tree to a fresh-clone state by
# removing untracked files, resetting changes, and ensuring only
# canonical files remain.
def cleanRepository() -> None:
    run_shell("git reset --hard")
    run_shell("git clean -xfd")
    ensure_only_expected_files()
```

**Static, namespaced utilities example**
```python
# ##################################################################
# text operations namespace
# stateless text helpers grouped for discoverability
class TextOperations:
    # ##################################################################
    # wrap
    # makes sure text is split into lines of less than this width, but
    # without breaking words unless there is no other choice
    @staticmethod
    def wrap(text: str, width: int) -> str:
        return textwrap.fill(text, width)
```

---

## DRY in Practice

- Centralize repeated patterns: logging, color output, constants, conversions.
- After writing code, extract commonality; then migrate other files to the new helper.

**Example: color output**
```python
# ##################################################################
# print header
# make sure headers are always cyan
def print_header(text: str) -> None:
    print(f"{Fore.CYAN}{text}{Style.RESET_ALL}")
    logger.info(f"----{text}")
```

---

## Error Handling

- Catch exceptions only where you can make a meaningful decision:
  - Entry points (fail the task, show the error)
  - Long processing loops (log context, continue to next item)
- Elsewhere, catch only to add context and re-raise.
- For network work, use standard backoff utilities; keep these in one reusable module.
- Never log or raise a constant string; always add context.

**Processing-loop example**
```python
# ##################################################################
# process batch
# ensure one bad item doesn't abort a long batch while logging context for triage
def process_batch(items: list[str]) -> None:
    for item in items:
        try:
            process_one(item)
        except Exception as err:
            logger.error("process_one failed item=%r err=%s", item, err)
            continue
```

**Entry-point exception policy**
```python
# ##################################################################
# top-level entry
# single place to convert exceptions into exit codes and logs
def main() -> int:
    try:
        run_pipeline()
        return 0
    except Exception as err:
        logger.exception("Pipeline failed: %s", err)
        return 1
```

---

## Secrets & Configuration

**Secrets**
- Secrets come only from **keyring** or **AWS Parameter Store**.
- The literal word `keyring` must **never appear in tests**.
- If secret retrieval fails, allow natural failure; do not override or stub.

**Configuration**
- Internal apps: avoid config files. Encode base configuration in code.
- Environment-specific values via environment variables only.

**Example**
```python
# ##################################################################
# get api key
# centralized secret access with explicit failure; sane default via env override
def get_api_key(name: str) -> str:
    key = keyring.get_password("app", name)
    if not key:
        raise RuntimeError(f"Missing secret for {name}")
    return key

DEFAULT_TIMEOUT_SEC = int(os.getenv("APP_TIMEOUT_SEC", "15"))
```

---

## Testing Policy (Real, Integration-First, Pytest, Per-File Only)

- Each `x.py` has `x_test.py` beside it.
- Use **Pytest** only; no `__main__` blocks.
- All tests are real end-to-end or integration tests. No smoke checks.
- **Run only the specific test(s) for the file you are working on.**
  - Do **not** run the full suite with the tool. `dazpycheck` runs everything at the end.
  - Example: while developing `src/text/wrap.py`, run `pytest src/text/wrap_test.py::test_wrap_text`.
- Write all test logs and artifacts to `output/testing/`.
- Expensive external actions: design a plugin interface and provide at least two real implementations (e.g., Physical vs In-Memory). Run the same test suite on both. In production, you may wire either.
- For LLM calls or costly requests, use **memoization** keyed by full parameters. Identical calls hit cache, but single runs remain fully real.

**Plugin pattern example**
```python
# ##################################################################
# check printer
# this prints out an actual check, or cheque - we define it as an
# interface so we can have several different implementations
class CheckPrinter:

    # ##################################################################
    # check printer
    # this prints out an actual check, or cheque
    def print_check(self, data: dict) -> None: ...

# ##################################################################
# check printer physical
# for use when sending out cheques to clients
class CheckPrinterPhysical(CheckPrinter):

    # ##################################################################
    # check printer
    # this prints out an actual check, or cheque
    def print_check(self, data: dict) -> None:
        send_to_usb_printer(data)

# ##################################################################
# check printer memory
# for testing, uat and simulation, we don't want real checks to be
# printed out - so we just print them to memory and log them
class CheckPrinterMemory(CheckPrinter):
    def __init__(self) -> None:
        self.printed: list[dict] = []

    # ##################################################################
    # check printer
    # this prints out an actual check, or cheque
    def print_check(self, data: dict) -> None:
        self.printed.append(data)


# ##################################################################
# tests of CheckPrinter
# any time we have an interface we want tests that test each implementation, to make sure that
# they all work exactly the same
@pytest.mark.parametrize("impl_cls",[CheckPrinterPhysical, CheckPrinterMemory])
def test_prints_identically(impl_cls):
    impl = impl_cls()
    impl.print_check({"amount": 100})
    # assertions…
```

**LLM memoization example**
```python
from functools import lru_cache

# ##################################################################
# llm_complete
# we memoize this because calls to the llm are expensive - this is occasionally useful in production, but
# particularly helps with the speed of tests - if ever time we call the llm in a test we use the same prompt
# then the test suite only ever ends up calling the llm once
@lru_cache(maxsize=256)
def llm_complete(model: str, prompt: str) -> str:
    return call_llm(model=model, prompt=prompt)
```

**Per-file run example (tool behavior)**
```
# Good: focused
pytest -q src/text/wrap_test.py::test_wrap_text --maxfail=1 --disable-warnings > output/testing/wrap_test.log 2>&1

# Bad: full-suite duplication during development
pytest
```

---

## Linting, Warnings, and `dazpycheck`

- Treat all warnings as errors, including deprecations.
- Line length is 120.
- Run linter after each file, then run only the tests for that file (logging to `output/testing/`).
- At the end of the task, run `dazpycheck` which runs the full test suite and final gates.
- **No commits** without full `dazpycheck` pass.

**Command examples**
```bash
ruff check --line-length 120
pytest -W error -q src/text/wrap_test.py::test_wrap_text > output/testing/wrap_test.log 2>&1
./run check
```

---

## Run Facade (Minimal Delegation Only)

The `run` script is a **Python file named `run`** (no `.py` extension, executable). It is a **pure facade** — it does no work, no argument parsing, and contains no business logic. Its only jobs are:

1. Ensure the venv exists (create it if not)
2. Ensure requirements are installed
3. Pass ALL arguments through to the actual entry point inside the venv

**`run` must never:** parse arguments, implement subcommands, or contain any logic beyond bootstrapping. All of that lives in the real source code.

**`~/bin wrapper script`** - also create a wrapper in `~/bin/` for global access:

```bash
# ~/bin/myproject (executable, 2 lines)
#!/bin/bash
exec ~/src/myproject/run "$@"
```

**Example `run`:**
```python
#!/usr/bin/env python3
# run — pure bootstrap facade. No logic here. Everything delegates to src/main.py.
import os, subprocess, sys
from pathlib import Path

SCRIPT_DIR = Path(__file__).parent.resolve()
VENV = SCRIPT_DIR / ".venv"
REQS = SCRIPT_DIR / "requirements.txt"
PYTHON = VENV / "bin" / "python"

# Create venv if missing
if not VENV.exists():
    subprocess.run([sys.executable, "-m", "venv", str(VENV)], check=True)

# Install / sync requirements
if REQS.exists():
    subprocess.run([str(PYTHON), "-m", "pip", "install", "-q", "-r", str(REQS)], check=True)

# Hand off entirely — this process is replaced, no return
os.execv(str(PYTHON), [str(PYTHON), str(SCRIPT_DIR / "src" / "main.py")] + sys.argv[1:])
```

The actual `src/main.py` (or whatever the entry point is) handles all argparse, subcommands, and logic. `run` only ensures the venv is ready and then disappears via `execv`.

---

## Prohibited Patterns (Zero-Tolerance)

**Forbidden words in code/tests/comments:**
`simulate`, `mock`, `fake`, `pretend`, `placeholder`, `stub`, `dummy`, `sleep`, `todo`

**Forbidden method name fragments:**
`_simulate_`, `_mock_`, `_fake_`, `_stub_`, `_dummy_`, `_sleep_`

**Forbidden comments:**
`# TODO: replace with real implementation`

**Consequences:**
- Any appearance of the above indicates failure of the task.

**What to do instead:**
- If a dependency is missing or a system is unavailable, stop and report a blocker with exact requirements to proceed.

**Blocker example**
```
Cannot proceed: Postgres unreachable at $DATABASE_URL
Tried 3 retries with 1/2/4s backoff
Required next step: provide credentials and network access
```

---

## Architecture Heuristics

**Separation of concerns**
- Business rules are thin adapters over generic utilities.
- Cluster by domain (text/, net/, io/, llm/).

**Stateless first**
- Prefer pure, parameter-defined functions. Memoize when appropriate.

**File and function size signals**
- If a file exceeds ~200–300 lines or a function exceeds ~25 lines, refactor.

**Interfaces before conditionals**
- If you foresee two modes (physical/memory, online/offline), define an interface and two real implementations. Avoid branching inside one class.

**Examples that guide structure**
```python
# text/wrap.py
def wrap_text(text: str, width: int) -> str: ...
# loan_system/term_loan.py
def wrap_description(description: str) -> str: return wrap_text(description, 80)

# net/http_client.py
def get_json(url: str, timeout: int) -> dict: ...
# loan_system/catalog.py
def fetch_catalog() -> dict: return get_json(BASE_URL + "/catalog", DEFAULT_TIMEOUT_SEC)
```

---

## Examples Library (Ready-to-Copy)

**Memoized expensive call**
```python
# ##################################################################
# memoized embedding
# cut repeated external cost without faking calls; cache key is full parameter tuple
@lru_cache(maxsize=1024)
def embed(model: str, text: str) -> list[float]:
    return call_embed(model=model, text=text)
```

**Loop + helper**
```python
# ##################################################################
# normalise all
# go through and normalise each item in the list
def normalize_all(rows: list[str]) -> list[str]:
    return [normalize(r) for r in rows]

# ##################################################################
# normalise
# make sure there's exactly one space between words
def normalize(s: str) -> str:
    return " ".join(s.strip().split())
```

---

## Workflow Integration

When Python code needs to execute n8n workflows, use the following pattern:

```python
import requests

def execute_n8n_workflow(workflow_id: str, data: dict = None) -> dict:
    payload = {
        "authKey": "c9543cecb08a4f84644110bedf91b4b04493d95e21b528508d94107c99178b28",
        "workflowId": workflow_id
    }
    if data:
        payload["data"] = data

    response = requests.post(
        "https://BFC910259E30AA1A89A40802CF16112CE.asuscomm.com:11133/webhook/run/workflow",
        json=payload
    )
    response.raise_for_status()
    return response.json()

# Example usage:
# execute_n8n_workflow("ppfQrpbtEExtanYe")  # Without data
# execute_n8n_workflow("WORKFLOW_ID", {"key": "value"})  # With data
```

---

## Development Workflow

**For each file change:**
1. Write or modify code following the standards above
2. Run linter: `ruff check --line-length 120`
3. Run tests for that specific file: `pytest src/path/file_test.py`
4. Write test output to `output/testing/`
5. Fix any issues and repeat steps 2-4

**At task completion:**
1. Run `./run check` (which executes `dazpycheck`)
2. Ensure all tests pass
3. Ensure no warnings or lint errors
4. Only then commit

**Key Principles:**
- Verify success through **real, individual tests** for the file being worked on
- Write test output to `output/testing/`
- Clean the repository and re-run checks until zero errors
- `dazpycheck` must be green before any commit

---

## Gotchas & Known Pitfalls

### FastAPI
- `{path:path}` route parameters are greedy — they swallow sub-paths like `/history`. Register specific sub-routes BEFORE catch-all `{path:path}` routes, or they'll never match.
- `TestClient` streaming (SSE): `iter_bytes()`/`iter_lines()` block indefinitely on infinite SSE generators. Use a real uvicorn server in a `threading.Thread` with `server.should_exit = True` for cleanup. Don't use multiprocessing for test servers on macOS (spawn context can't pickle local functions).
- `async def` endpoints with synchronous DB calls block the event loop. Use plain `def` instead — FastAPI auto-runs them in a threadpool. Critical when SSE streaming coexists with slow queries: one blocked query stalls ALL concurrent SSE clients.
- `StaticFiles` mount doesn't set cache headers. For immutable content-hash caching, replace with a custom route that sets `Cache-Control: public, max-age=31536000, immutable` and use `resolve()` + prefix check for path traversal.
- httpx (used by `TestClient`) deprecated per-request `cookies=` parameter. With `-W error`, tests setting `cookies={"key": "val"}` on individual `client.get()` calls will fail as `DeprecationWarning`. Fix: set cookies on the client instance via `client.cookies.set("key", "val")` before making requests.
- `@app.on_event("startup")` is deprecated in favor of `lifespan` context manager: `@asynccontextmanager async def lifespan(app): ... yield ...` passed to `FastAPI(lifespan=lifespan)`.

### SQLAlchemy
- SQLAlchemy + Lambda + PostgreSQL: always use `NullPool` (`from sqlalchemy.pool import NullPool; create_engine(url, poolclass=NullPool)`). Lambda concurrency creates many processes each with their own pool — exhausting `max_connections`. NullPool opens/closes per request.
- SQLAlchemy dual-DB pattern (SQLite for tests, PostgreSQL for prod): guard DB-specific code with `if engine.url.drivername.startswith("sqlite")` (not `"postgresql"`) for robustness to driver changes.

### SQLite
- Multiple connection factories: if you have more than one function that creates connections (e.g., thread-local vs fresh), ensure ALL of them set critical PRAGMAs (`busy_timeout`, `journal_mode`). Missing `busy_timeout` causes immediate "database is locked" errors instead of waiting.
- Per-request `connect()`/`close()` in web servers adds ~500ms overhead. Use thread-local connections (`threading.local()`) for the server process.
- `CREATE TABLE IF NOT EXISTS` does NOT modify existing tables — won't add new columns. Use `ALTER TABLE ADD COLUMN` in a migration function.
- Computed/normalized columns: when the normalization function changes, recompute ALL rows (not just NULLs). Migration that only fills `WHERE normalized_col IS NULL` silently leaves stale values from the old algorithm.
- `UPDATE ... SET x = ?` on a table with `UNIQUE(name, x)` will fail if the target already has that name. When merging parent entities, handle child UNIQUE constraints: find-or-merge children first, then reassign remaining, then delete the source parent.

### Playwright / Testing
- `page.goto()` defaults to `wait_until="load"` which waits for ALL resources (images). On pages with many lazy-loaded images, this times out. Use `wait_until="domcontentloaded"` for heavy image grids.
- `expect()` has its own timeout separate from `page.set_default_timeout()`. Use `expect.set_options(timeout=N)` to configure assertion timeouts globally. Without this, `expect(locator).to_be_visible()` uses the default 5s.
- E2E: when a UI click triggers an API fetch that updates the DOM, use `page.expect_response(lambda r: ...)` as a context manager around the click to wait for the specific response before asserting DOM state.
- `Path.rglob()` on network volumes (NFS/SMB) is extremely slow. For tests, iterate top-level dirs and rglob within each subdirectory with an early-exit limit instead of scanning the entire tree.
- Tests spawning tmux sessions: user shell init files (.zshrc, .bashrc) can block indefinitely. Fix: set `HOME` to a temp dir and `SHELL=/bin/bash` in TestMain to skip init files entirely.
- Tests for fire-and-forget launchers (`subprocess.Popen(["open", url])`) must NEVER pass valid URLs — they actually open in the browser on every test run. Only test rejection of invalid inputs.
- Pytest `scope="module"` fixtures that bind ports stay alive for the entire module. Standalone tests in the same file that start their own servers MUST use a different port, or the health check will hit the fixture's server while the new server silently fails to bind.

### Python stdlib / Runtime
- `os.walk()` on NFS can hang indefinitely on a single stalled directory. Replace with manual BFS + `os.listdir()` in a daemon thread with `queue.get(timeout=N)` so stalled dirs are skipped.
- `datetime.fromisoformat()` on RFC3339 strings returns tz-aware datetimes; `datetime.now()` returns naive. Comparing them raises "can't compare offset-naive and offset-aware datetimes". Fix at the ingestion point: `.astimezone().replace(tzinfo=None)`.
- `run` scripts using `os.execv` to activate a venv: MUST compare `Path(sys.executable).resolve()` to the venv python path before re-exec. Without this check the script re-execs forever (infinite loop) because `VIRTUAL_ENV` env var is not set by execv.
- Optional-module try/except import pattern (`try: import foo; except ImportError: foo = None`) causes Pyright to flag all `foo.bar()` calls as `union-attr` errors. Fix: add `# type: ignore[union-attr]` on each usage site.
- Python 3.14 + Homebrew: `cffi` and `pycparser` may have broken RECORD files preventing `pip install --force-reinstall`. Use `pip install --ignore-installed <pkg>` to work around.
- Python wrapper scripts importing from deep internal package paths are fragile to version upgrades — packages reorganize internal structure between versions. Prefer importing from stable public APIs.
- Python strings embedded in JS templates: `\uD83D\uDE80` (UTF-16 surrogate pairs) are invalid in Python UTF-8 strings and cause `UnicodeEncodeError: surrogates not allowed` on file write. Use full 8-digit escapes: `\U0001F680` (🚀).
- `dict.get("key", "")` returns `None` (not the default) when the key IS present but its value is `None`. Use `(d.get("key") or "")` to guard against `None` values when chaining `.strip()` or other string methods.
- Python 3.14 `sqlite3.Cursor` does NOT support context manager protocol (`with conn.cursor() as cur:` raises TypeError). Use `cur = conn.cursor(); try: ... finally: cur.close()` instead.
- `asyncio.run()` in a sync pytest test fails with "cannot be called from a running event loop" when pytest-asyncio is active. Fix: run the coroutine in a `ThreadPoolExecutor` thread (each thread gets its own event loop): `pool.submit(asyncio.run, coro()).result(timeout=N)`.
- Bulk Python module rename via string `.replace("old_name.", "new_name.")` will double-hit strings that already contain `new_name.old_name.`. Do the rename in a single pass with exact `from old_name` / `import old_name` patterns first, then run a cleanup pass for doubled prefixes.

### Claude / AI SDK
- Claude Code SDK `async for msg in query()` crashes on unknown message types (e.g., `rate_limit_event`) with `MessageParseError`. Fix: use manual `__anext__()` in a while loop with try/except, catching and skipping unknown types. Also: `ResultMessage` has `.result` (str) not `.content` — skip it when collecting response text.
- `claude_agent_sdk` from sync code: wrap with `asyncio.run(_ask_claude(prompt))` where `_ask_claude` is `async def`. Never use `subprocess.run(["claude", ...])` — it fails inside Claude Code sessions.
- `claude_agent_sdk.ProcessError` is raised when the claude CLI exits non-zero — can happen during teardown even after tools executed successfully. Catch it in the `async for` loop and verify operation success via DB/API rather than treating it as a deployment failure.
- Claude Agent SDK `query()` from a FastAPI background thread spawned from a sync endpoint: use `ThreadPoolExecutor` + `asyncio.run()` inside the thread (can't use `asyncio.run()` directly if already in an async context).

### Pyright / Type Checking
- Pyright + async generators: an ABC method returning `AsyncIterator[str]` must be declared as `def stream(...)` (NOT `async def`). Subclasses implement as `async def stream(...)` with `yield`. If the base is `async def`, pyright sees a type mismatch between `Coroutine[..., AsyncIterator[str]]` and `AsyncIterator[str]`.
- Pyright `reportMissingImports` warnings for installed packages: caused by missing `py.typed` marker. Fix with `# pyright: ignore[reportMissingImports]` on the import line. Don't touch `site-packages/` — it's fragile and gets blown away on reinstall.

### macOS / PyObjC
- `pyobjc-framework-Quartz` is NOT installed by default. Needed for `CGWindowListCopyWindowInfo` native window capture: `pip install pyobjc-framework-Quartz`.
- PyObjC bridge calls are expensive in bulk. When processing large collections of ObjC objects (e.g., EventKit reminders), do lightweight filtering/sorting on raw ObjC objects first, then convert only the needed subset to Python dataclasses. Also use direct ID lookups (`calendarItemWithIdentifier_`) instead of fetching all items and iterating.
- macOS screen sleep/wake: use PyObjC `NSWorkspace.sharedWorkspace().notificationCenter().addObserverForName_object_queue_usingBlock_("NSWorkspaceScreensDidSleepNotification", None, None, callback)`. Requires `pyobjc-framework-Cocoa`. Callbacks run on the main thread — safe to set flags and call `QTimer.singleShot`. Wrap in try/except for graceful degradation.
- EventKit EKEventStore must be a process-level singleton. Creating new instances per poll cycle causes EventKit to throttle and deny access after ~2 minutes. Cache as module-level variable; create and authorize once at startup.
- PySide6 blocking operations in QTimer callbacks freeze the UI. Use `threading.Thread(daemon=True)` + `Signal(object)` to run work off-thread and deliver results via signal emission.
- `keyring` package in macOS venvs requires `keyrings.alt` backend installed in the same venv — otherwise `keyring.get_password()` returns None silently (no error).
- Homebrew Python code-signing breakage: `brew upgrade` can leave Python dylibs with invalid Team IDs, causing `dyld` load failures. Fix: `brew reinstall python@3.XX`.

### ML / AI Models
- Neural network integration: always check the original training code's input normalization (e.g., `Normalize((0.5,0.5,0.5),(0.5,0.5,0.5))` for [-1,1]). Mismatched input ranges cause systematic brightness/contrast corruption. Compare per-channel stats (mean, std, min, max) against the source image to verify.
- Long-running ML generation (video/audio): use subprocess-per-chunk for memory isolation. MLX/PyTorch models leak memory across iterations even with `del` + `gc.collect()`. Save intermediate results as `.npy` files for resume support (check existence before generating).
- HuggingFace MLX Community Whisper repos use `-mlx` suffix: `mlx-community/whisper-base-mlx`, `whisper-small-mlx`. Exception: `whisper-large-v3-turbo` has no suffix. Always verify repo exists before hardcoding.
- fastembed (BAAI/bge-small-en-v1.5, 384-dim): first call per process downloads/loads model (~30s if not cached, ~3s if cached). In daemon/watcher processes, expect an apparent stall on the first chunk indexed.
- When chaining sequential large ML model calls (e.g. generation then background removal), call `gc.collect()` between them to release the first model's memory before loading the second. Wrap the second call in a retry loop (3 attempts, ~10s delay) to handle transient OOM kills — OOM kills skip Python `finally` blocks so temp files persist and can be retried.

### Web / APIs
- Browser `getUserMedia()` requires a user gesture context (click/tap handler). Calling it on page load or outside a gesture silently returns "Permission denied" without ever prompting. Always init mic inside a click handler. Web Audio nodes (MediaStreamSource, AnalyserNode, ScriptProcessorNode) must be stored at module/global scope — local variables get garbage collected, causing `onaudioprocess` callbacks to silently stop firing.
- OAuth token refresh: use proactive refresh (10 minutes before expiry) with background checking (every 5 minutes) and minimum refresh interval (30 seconds) to prevent loops. Store access_token, refresh_token, and expiry separately with last_refresh timestamp.
- Wikidata SPARQL endpoint: for bulk queries (100K+ results), request CSV format (`Accept: text/csv`) instead of JSON — more reliable and faster to parse with Python's `csv` module.
- YouTube playlists: yt-dlp `--flat-playlist --dump-json` returns titles in the flat listing — pre-filter by title before fetching full metadata to avoid downloading data for irrelevant videos.
- Open Library author search API can return wrong authors. Always validate the returned name matches the searched name before using bio/photo.

### Services / Processes
- uvicorn `--reload` on macOS uses `multiprocessing.spawn` to create parent/child process pairs. After extended runtime (hours/days), the worker subprocess can hang (bound to port but not serving). Never use `--reload` for daemon-managed services — the daemon handles restarts.
- JSONL watcher daemons: always cap lines processed per file per poll cycle (e.g., 150 max). Without a cap, a single large file starves all other files from ever being processed. Store byte offset after each batch so next poll resumes from where it stopped.
- SSE streaming with snapshot-then-live: when connection drops and reconnects, the UI MUST clear stale data before the new snapshot arrives. Wire a `reconnecting` callback that clears state, fired just before the retry delay.
- Process group killing: when processes are started with `start_new_session=True`, killing the parent leaves orphaned children. Use `os.killpg(os.getpgid(pid), signal)` to kill the entire group. Critical for services that spawn workers (uvicorn, gunicorn, etc).
- Port pre-flight checks: before starting any service that binds a port, check availability with a socket bind attempt. Include `lsof -i :<port>` in error messages for diagnosis.

### Media (ffmpeg)
- When concatenating audio with `-c copy`, ALL input files MUST have identical sample rates. Mismatched rates produce silence/garble. Either generate all inputs at the same rate, or use re-encoding (`-ar <rate>`) in the concat step.
- Segment cutting with floating-point `-ss`/`-t` accumulates frame-rounding drift when concatenating many segments. Fix: snap timestamps to frame boundaries (`round(t * fps) / fps`) and use `-frames:v N` for exact frame counts instead of `-t duration`.
- Playwright screenshots of WebAssembly-heavy pages (OpenCascade.js, etc.): use fresh browser per capture to avoid memory leak crashes. Use `wait_until="domcontentloaded"` not `"networkidle"` (WASM keeps connections alive, causing timeout).
