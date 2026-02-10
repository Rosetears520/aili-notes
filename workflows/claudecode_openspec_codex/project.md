## tasks.md Checklist Format

This section is the SINGLE canonical spec for tasks.md format and validation bundles.
Do not duplicate this spec elsewhere; other docs must link here.

### Task Line Format (required)

Each checkbox task line MUST follow:
- `- [ ] <task-id> <task summary> [#R<n>]`
- `<task-id>` MUST be dot-numbered (e.g. `1.1`, `2.3`).
- Each checkbox line MUST include EXACTLY ONE `[#R<n>]` token (e.g. `[#R1]`).
  - `[#R<n>]` MUST be unique across the entire tasks.md (never reuse).
- Every task MUST include both `ACCEPT:` and `TEST:` blocks.
- `TEST:` MUST include `SCOPE: CLI|GUI|MIXED` and MUST be implementable into a validation bundle
  per `### Validation bundle requirements (mandatory)` below.

### Example (copy/paste)

- [ ] 1.1 Do X and produce Y [#R1]
  - ACCEPT: ...
  - TEST: SCOPE: CLI
    - When done, generate validation bundle under:
      auto_test_openspec/<change-id>/<run-folder>/
    - run-folder MUST be:
      run-<RUN4>__task-<task-id>__ref-<ref-id>__<YYYYMMDDThhmmssZ>/
    - Run: auto_test_openspec/<change-id>/<run-folder>/run.sh (macOS/Linux) or run.bat (Windows)
	- run-folder MUST be:
	  run-<RUN4>__task-<task-id>__ref-<ref-id>__<YYYYMMDDThhmmssZ>/
	- Run: auto_test_openspec/<change-id>/<run-folder>/run.sh (macOS/Linux) or run.bat (Windows)
	- Inputs: inputs/sample.json
	  Outputs: outputs/result.json
	- Verify: compare against expected/result.json (or rule-based assertions)

### Validation bundle requirements (mandatory)

For every task, `TEST:` MUST be written so:
- the Worker can produce a **human one-click reproducible** validation bundle (assets + scripts for CLI checks; GUI checks are MCP-driven and MUST NOT use any browser automation scripts),
- AND the Supervisor can execute it and record the final PASS/FAIL evidence chain
  (each run-folder is immutable; evidence pointers are written after execution).

0) Roles & responsibilities (mandatory)
- Worker (produces artifacts; not the final verifier):
  - Implement product code + write tests (CLI). For GUI/MIXED, produce an MCP runbook only (no executable browser automation scripts).
  - Produce the validation bundle assets under the run-folder:
    `task.md`, `run.sh`, `run.bat`, `tests/` (CLI tests and/or GUI MCP runbook; no executable browser scripts), and (when applicable) `inputs/`, `expected/`.
  - MUST NOT declare PASS/FAIL.
  - MUST NOT overwrite/edit prior run-folders (append-only history).

- Supervisor (executes validation; forms the evidence chain):
  - MUST create a brand-new run-folder for every validation attempt (never overwrite).
  - Executes `run.sh` / `run.bat`, captures `outputs/` + `logs/` + GUI evidence when applicable.
  - MUST write the final PASS/FAIL result + evidence pointers (this is the DONE hard gate).

1) Canonical on-disk location (repo root; append-only)
- Root folder (fixed):
  - `auto_test_openspec/<change-id>/`
- Each validation attempt MUST create a brand-new run folder (never overwrite; keep ALL history forever):
  - `auto_test_openspec/<change-id>/<run-folder>/`
- Once created, a run folder MUST be treated as immutable evidence:
  - do not edit prior runs; create a new run folder instead.

2) Run folder naming (required; MUST include run#, task-id, ref-id; timestamp recommended)
- `<run-folder>` MUST follow this exact pattern:
  - `run-<RUN4>__task-<task-id>__ref-<ref-id>__<YYYYMMDDThhmmssZ>/`
- Example:
  - `run-0007__task-1.1__ref-R1__20260111T031500Z/`
- Rules:
  - `<RUN4>`: zero-padded, monotonic run counter (e.g. 0001, 0002, ...).
    - MUST match the Supervisor workflow RUN_COUNTER / `EVIDENCE (RUN #n)` numbering for audit alignment.
    - Mapping rule: `RUN #7` => `run-0007`, `RUN #12` => `run-0012`.
  - `<task-id>`: dot-numbered task id from the checkbox line (e.g. `1.1`).
  - `<ref-id>`: stable ref id derived from the task tag (e.g. `[#R1]` → `R1`).
  - `<YYYYMMDDThhmmssZ>`: UTC timestamp to guarantee uniqueness and ease auditing.

3) Minimum required contents inside EVERY run folder
Each run folder MUST contain at least:

A) `task.md` (this run’s readme; MUST be self-sufficient)
task.md MUST include:
- change-id, run#, task-id, ref-id
- SCOPE covered (CLI / GUI / MIXED)
- How to run (Windows + macOS/Linux)
  - CLI: run.sh/run.bat executes CLI checks.
  - GUI/MIXED: run.sh/run.bat starts the service only; GUI steps are executed via the MCP runbook under tests/.
- Test inputs (if any): input file paths, params, sample data
- Test outputs (if any): what files/stdout/stderr/screenshots/logs will be produced and where
- Expected results (machine-decidable): pass/fail criteria
  - exit code checks
  - stdout/stderr assertions (required when relevant)
  - file existence/content assertions (required when outputs exist)
  - GUI assertion points (when GUI/MIXED): which screenshots/states prove correctness
- Hard rules (GUI/MIXED):
  - task.md MUST NOT contain manual browser steps (no “open Chrome/click buttons” prose).
  - task.md MUST point to the MCP-only runbook under tests/ (e.g., tests/gui_runbook_<topic>.md).
  - Any required “copy/seed/prepare input/state” steps MUST be written as exact commands/steps here (and referenced by the runbook). run.sh/run.bat MUST NOT perform them.
- Provenance of expected/assumptions:
  - If inputs/expected are not provided by a human, the Worker MUST generate them and document where they came from
    (e.g., derived from ACCEPT, or an explicit reasonable assumption).


B) One-click scripts (both required; GUI/MIXED = start-server only)
- run.sh (macOS/Linux)
- run.bat (Windows)

Script requirements (all bundles):
- Must assume the default dev machine environment is ready.
- Non-destructive:
  - MUST NOT modify global environment
  - MUST NOT globally install dependencies
  - MUST NOT write to system directories
- Must be runnable from ANY working directory:
  - the script MUST cd/pushd to its own directory first, then resolve paths from there.

Hard rule (when SCOPE includes GUI):
- run.sh/run.bat MUST be start-server only:
  - MUST: start the local service and print the access URL/port (e.g., http://127.0.0.1:<PORT>/)
  - MUST NOT: copy/overwrite data files, mutate state/inputs, generate exports/outputs, run tests, run exports, probe/install dependencies, or perform environment probes (python/uv version checks do NOT belong in GUI start scripts)
  - Any required “copy/seed/prepare input/state” steps MUST be documented as exact commands/steps in task.md (and referenced by tests/gui_runbook_*.md) for the Supervisor to execute and record in EVIDENCE.

For CLI bundles (or the CLI portion of MIXED):
- run.sh/run.bat SHOULD print key results to console and SHOULD write logs to logs/.
- Environment provenance SHOULD be documented as optional preflight commands in task.md (not forced into GUI start scripts), e.g.:
  - interpreter path + version (Python/Node if used)
  - uv --version when Python/uv is involved
- When provenance is executed, it SHOULD be recorded to logs/.

C) Test asset folders (create the ones that apply)

- `logs/` MUST exist (always):
  - run logs, env/version info, command transcript, GUI screenshot index, etc.
- `tests/` MUST exist when:
  - SCOPE includes GUI (MCP-driven via `playwright-mcp`), OR
  - validation is not fully expressible as simple CLI assertions.
- `inputs/` MUST exist when the task involves file input (see I/O hard rule below).
- `outputs/` MUST exist when the validation produces file outputs (see I/O hard rule below).
- `expected/` SHOULD exist when golden-file comparison is used; otherwise rule-based assertions are acceptable.

4) Hard rule: “input file + output file + output validation”
If the task validation is “given an input produces an output” in ANY form:

- `inputs/` MUST contain at least one reproducible input sample.
- `run.*` MUST write the real produced outputs into `outputs/` (never into random temp/system dirs).
- The bundle MUST include at least one machine-decidable verification method (pass/fail), typically:
  - (A) golden file compare against `expected/` (exact match OR documented allowed-diff rules), and/or
  - (B) rule-based assertions (e.g. JSON schema, key fields, row counts, regex match, exit code, forbidden strings).

`task.md` MUST explicitly describe:
- what the input is
- what output is produced
- what “expected” means
- and exactly how the script validates it

5) CLI / GUI / MIXED validation requirements
- If SCOPE includes CLI:
  - MUST run the real CLI command(s) in `run.*`
  - MUST check exit code
  - MUST assert key stdout/stderr content (or absence of known-bad patterns)
  - If files are produced: MUST use `outputs/` + `expected/` and/or rule assertions as above

- If SCOPE includes GUI:
  - The validation bundle MUST provide an MCP-only GUI verification runbook
    (stored under tests/ and executed by the Supervisor via playwright-mcp; do NOT use any scripts to drive the browser).
  - Hard rule: run.sh/run.bat MUST be start-server only for GUI/MIXED bundles:
    - MUST: only start the service and print URL/port
    - MUST NOT: copy/seed/prepare input/state, generate exports/outputs, run tests, or perform environment probes
    - Any required data prep steps MUST be written as exact commands/steps in task.md (and referenced by the runbook).
  - Supervisor execution constraint (mandatory):
    - GUI verification MUST be driven via MCP service playwright-mcp
      - no manual browser interaction
      - no Python/Node/Playwright scripts to drive the browser
  - Must archive auditable evidence artifacts (append-only; never overwrite):
    - at minimum: screenshots (e.g., outputs/screenshots/ plus a screenshots index file in logs/)
    - recommended: trace/video and a console log index when available from MCP (paths recorded in logs/)

- If SCOPE is MIXED:
  - The bundle MUST cover both CLI and GUI checks (either in one test file or split; see “two test files” rule below).

6) Allowing two test files (when needed; organization rule)
Default: one test file should cover key acceptance points.

Two test files are allowed / recommended when:
- CLI + GUI are both involved:
  - one test focuses on CLI
  - one runbook focuses on GUI (MCP steps + assertions; no executable browser scripts)
- Same entrypoint but two distinct paths must be covered:
  - happy path + error/edge path (e.g., valid vs invalid args)
- GUI needs both “functional flow” and “render/state”:
  - split into two smaller, more stable tests

Suggested naming under the run folder:
- `tests/test_cli_<topic>.*`
- `tests/gui_runbook_<topic>.md` (MCP-only steps + assertion points; no executable browser scripts)

Note:
- “two test files” refers to validation assets under `tests/` (CLI test scripts and/or GUI MCP runbook).
- The “input/output two files + validation” rule refers to runtime data under `inputs/outputs/expected` and is additive, not conflicting.

7) Environment isolation (uv venv rule; mandatory when env problems occur)
- Under no circumstances may the Worker “pollute global Python env” to make validation pass (e.g., global `pip install`).
- If the Worker encounters environment problems (missing deps, conflicts, cannot run):
  - MUST create an isolated venv using `uv`
  - Recommended location: inside THIS run folder (e.g. `<run-folder>/.venv/` or `<run-folder>/venv/`)
  - All installs/runs must occur inside that venv
- `run.*` and/or `logs/` MUST clearly record:
  - which interpreter is used
  - uv version
  - where dependencies came from (lockfile / pyproject / etc.)
- Note:
  - Creating a venv is conditional (only when env problems occur),
    but running the full validation bundle is unconditional (always required).
    

8) tasks.md bookkeeping lines (mandatory; role split; no duplicated rules elsewhere)
- Under the task entry in `openspec/changes/<change-id>/tasks.md`, TWO lines are mandatory:
  - Worker-written (bundle-ready; NO PASS/FAIL):
    - `BUNDLE (RUN #n): ... | VALIDATION_BUNDLE: auto_test_openspec/<change-id>/<run-folder> | HOW_TO_RUN: run.sh/run.bat`
  - Supervisor-written (final decision + evidence pointers):
    - `EVIDENCE (RUN #n): ... | VALIDATED: <exact commands + exit code> | RESULT: PASS|FAIL | GUI_EVIDENCE: <paths when applicable>`
- Worker MUST NOT claim PASS/FAIL anywhere; Supervisor is the only role that records PASS/FAIL after running the bundle.