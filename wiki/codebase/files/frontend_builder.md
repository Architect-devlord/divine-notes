---
type: file
status: ingested
---
# frontend_builder.py

💡 **Role**: `FrontendBuilder.build_frontend(frontend_dir, output_dir)` — runs `npm install`/`npm run build` against the React/Vite frontend and copies the result into whatever directory [[wiki/codebase/files/packager|packager.py]] bundles into the agent exe. Genuinely standalone: no internal project imports at all, stdlib only — the only coupling to the rest of the codebase is being called by `packager.py` and, per its own header comment, sharing a naming convention with it.

## Imports

`logging`, `os`, `shutil`, `subprocess`, `sys`, `pathlib.Path`, `typing.Optional` — all stdlib, nothing internal.

## Module-level

- `NPM_CMD = shutil.which("npm") or "npm"` — resolved once at import time. **The header comment claims this exists "so the module-level constant can be reused by packager.py (which also declares NPM_CMD itself)" — checked directly: `packager.py` does declare its own, separate `NPM_CMD` with identical logic, but doesn't import this one.** Harmless in practice (both resolve to the same value the same way), but the comment describes a reuse that isn't actually happening — worth being precise that the intent and the reality don't match here, distinct from an actual bug.
- Timeout/polling constants (`_INSTALL_TIMEOUT`/`_BUILD_TIMEOUT` = 600s each, `_DIST_POLL_SECS` = 3, `_DIST_WAIT_EXTRA` = 10) and `_BUILD_OUTPUT_DIRS = ("dist", "build", "out")`, checked in that priority order to support Vite, CRA, or other tooling without hardcoding one.

## Functions

- `_make_npm_env()` — copies `os.environ` and sets `NODE_ENV=production`/`NO_COLOR=1`. **A clean, self-contained docstring-vs-code mismatch, worth noting as a precise example of the pattern**: the function's own docstring describes setting `CI=true` ("Vite/CRA treats this as a non-interactive build, turning warnings into errors so we catch problems early"). The actual code does the opposite — `env.pop("CI", None)` — with an in-line comment directly explaining why: `CI=true` turns chunk-size warnings into hard failures on slow machines, so it's deliberately _not_ set, and failures are caught through `CalledProcessError` instead. The docstring describes an earlier design the code moved away from; the comment right next to the code explains the reversal. Trusting the code over the docstring here isn't a judgment call — the code contradicts its own doc directly.

## Classes

### `FrontendBuilder`

- `__init__(npm_cmd=None)` — resolves `npm_cmd` or falls back to the module constant; warns (doesn't fail) if `npm` isn't found on `PATH`.
- `build_frontend(frontend_dir, output_dir) -> bool` — the public entry point, confirmed to match exactly how [[wiki/codebase/files/packager|packager.py]] calls it (`frontend_builder.build_frontend(self.frontend_dir, frontend_dest)`). Validates `package.json` exists → `npm install` only if `node_modules` is missing → `npm run build`, with one specific, named recovery path: if the build fails with an esbuild-related error, installs `esbuild` explicitly as a dev dependency and retries once — the in-line comment explains this is because rolldown-vite v7+ removed its bundled minifier, a real, specific version-compatibility fix rather than a generic retry — → polls for the output directory to actually appear on disk (see `_poll_for_build_output`) → `shutil.rmtree` + `shutil.copytree` into `output_dir`. Returns `False` at any failure point rather than raising, so a caller can treat "no frontend" as a soft condition — matching [[wiki/codebase/files/packager|packager.py]]'s own `include_frontend` optionality.
- `_run(cmd, cwd, env, timeout, label)` _(static)_ — subprocess wrapper returning `(success, stderr_text)` rather than raising; distinguishes timeout, non-zero exit (combining stdout+stderr, since CRA writes errors to stdout and Vite to stderr — different tools, different conventions, handled explicitly rather than assumed), and command-not-found as three distinct failure messages.
- `_find_build_output(frontend_dir)` _(static)_ — checks `dist`/`build`/`out` in order, and — a real sanity check, not just a directory-existence check — requires at least one `.html` file inside before accepting a candidate.
- `_poll_for_build_output(frontend_dir)` _(classmethod)_ — the reason this file polls rather than trusting `npm run build`'s exit alone: on a slow disk, the build process can exit before every file is actually flushed. Polls every 3s up to the same 600s ceiling as the build itself, logs progress every 15s so a long build doesn't look hung, and waits an extra 10s after first finding output before declaring success, to let remaining assets land.

## `if __name__ == "__main__":`

A standalone CLI test harness: `python frontend_builder.py [frontend_dir [output_dir]]`, defaulting to `dw_agent/electron/react-app` → `test_build`, printing a clear success/failure line and exiting with the matching status code.

## Problems (faced by traditional AI systems / LLMs)

Not really an AI/ML-specific problem — this file's problem is a general build-tooling one: a subprocess-based build step can exit successfully while the filesystem hasn't caught up yet, and different frontend toolchains (Vite vs. CRA vs. whatever comes next) disagree on where output lands and which stream carries errors.

## Solutions

Polling for the actual output directory rather than trusting the subprocess's exit code alone is a real, specific answer to the flush-timing problem, not a generic sleep-and-hope. Checking three candidate output directories in priority order, each validated by actually containing HTML rather than just existing, means this file doesn't need to know in advance which tool produced the build. The esbuild recovery path is narrower and more interesting than a general retry-on-failure: it pattern-matches the specific error text of one known, named tooling regression (rolldown-vite v7 dropping its bundled minifier) and applies the specific fix, rather than blindly retrying every failure once.

## Files Required

None — stdlib only.

## Files Used In

- [[wiki/codebase/files/packager|packager.py]] — `_build_frontend`/`package_agent()`'s frontend step imports `FrontendBuilder` directly (`from frontend_builder import FrontendBuilder`) and calls `build_frontend()` with exactly the signature this file defines. Also independently (and redundantly, see above) declares its own copy of `NPM_CMD` rather than importing this file's.