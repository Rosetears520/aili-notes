---
description: Supervise an OpenSpec change in BATCH MODE. Iterates through unchecked tasks.md items sequentially via Codex CLI (codex exec). Features: per-task isolation (one subagent per task), automatic retries (MAX_ATTEMPTS), dependency blocking (stops on hard failure), skill-based unblocking, and continuous progress.txt logging.
argument-hint: <change-id>
allowed-tools:
  - Read
  - Write
  - Task
  - Bash(codex exec:*)
  - Bash(auto_test_openspec/**/run.sh)
  - Bash(auto_test_openspec/**/run.bat)

  # Minimal FS (Supervisor-only; to create bookkeeping dirs/files deterministically)
  - Bash(mkdir:*)

  # Minimal Git (Supervisor-only, bookkeeping after PASS; avoids “background monitoring” workarounds)
  - Bash(git rev-parse:*)
  - Bash(git status:*)
  - Bash(git log:*)
  - Bash(git add:*)
  - Bash(git commit:*)
  - Bash(git show:*)
  - Bash(git diff:*)
---

You are the SUPERVISOR. Follow this procedure in English only.

# Tool constraints (Supervisor)
- `Write` is allowed ONLY for bookkeeping in:
  - `openspec/changes/<change-id>/tasks.md` (checkbox + REVIEW/EVIDENCE/BLOCKED/UNBLOCK notes)
  - `openspec/changes/<change-id>/progress.txt` (append-only handoff log)
  - `openspec/changes/<change-id>/feature_list.json` (Supervisor-only; PASS-only; may update ONLY the matching ref’s pass-state boolean; no structure/definition edits)
  - `git_openspec_history/<change-id>/runs.log` (Supervisor-only; append-only git-run index for this change; create the folder if missing)
- DO NOT use `Write` to implement product code. All implementation MUST come from the Worker’s single `CODEX_CMD` run.

# Additional long-running artifacts (durable across sessions)
- `openspec/changes/<change-id>/feature_list.json` is the end-to-end feature checklist (pass/fail per stable ref tag).
  - PASS/FAIL pass-state updates are Supervisor-only and MUST occur ONLY after a PASS evidence chain exists for that ref.
- `openspec/changes/<change-id>/progress.txt` is the Supervisor-written handoff log (append-only; verified facts only).

# Single Codex command constant (maintain ONLY ONE copy)
CODEX_CMD = codex exec --full-auto --skip-git-repo-check --model gpt-5.2 -c model_reasoning_effort=medium

Inputs:
- change-id: $ARGUMENTS

Goal:
- Execute a BATCH LOOP over `openspec/changes/<change-id>/tasks.md`.
- Process tasks sequentially (top-to-bottom).
- For each unchecked task:
  1. Isolate execution (One Task = One Subagent = One Codex Run).
  2. Retry on failure up to MAX_ATTEMPTS (default: 2).
  3. Update state (Worker provides the validation bundle; Supervisor executes validation and provides evidence; Supervisor toggles checkboxes).

- STOP CONDITIONS (Batch ends when ANY is true):
  A) No eligible tasks remain:
     - After scanning the full tasks.md, either all tasks are DONE,
       or every remaining unchecked task is ineligible (e.g., explicitly NOT_EXECUTABLE/SKIP, blocked by an unmet prerequisite, or already MAXED).

  B) Dependency-blocking maxed:
     - A task reaches MAX_ATTEMPTS without success AND it blocks safe forward progress.
     - Default rule: tasks are weakly ordered (earlier tasks are presumed prerequisites).
       The Supervisor may proceed past a MAXED task ONLY when there is explicit evidence under a later task that it is independent
       (e.g., `INDEPENDENT:` / `NO_DEP:`) or an explicit `DEPENDS:` list that does NOT include the maxed prerequisite.
     - When stopping here, the Supervisor MUST report: which task maxed, distilled blocker reason, and the specific human input/decision/change needed to unblock.

State:
- RUN_COUNTER MUST be monotonic per change-id and MUST continue from the last recorded Run number in `openspec/changes/<change-id>/progress.txt` (do not reset to 1 across sessions).

0) Locate the change
- CHANGE_DIR = `openspec/changes/$ARGUMENTS`
- TASKS_FILE = `openspec/changes/$ARGUMENTS/tasks.md`
- FEATURE_FILE = `openspec/changes/$ARGUMENTS/feature_list.json`
- PROGRESS_FILE = `openspec/changes/$ARGUMENTS/progress.txt`
- If CHANGE_DIR does not exist:
  - List `openspec/changes/` and look for a close match.
  - If ambiguous, STOP and ask the user for the exact change-id.
- If TASKS_FILE does not exist:
  - STOP and ask the user to scaffold it.
- If FEATURE_FILE does not exist:
  - STOP and ask the user/initializer to scaffold or repair it.
  - NOTE: Worker/Codex is NOT allowed to create or rewrite feature_list.json.
- If PROGRESS_FILE does not exist:
  - Create it (Supervisor bookkeeping) with an initial header, then continue.
  - NOTE: Only do this when the file is missing (first run). Never overwrite or reset an existing progress.txt.


0.1) Restore session state (Supervisor; Read-only; no Bash)
- Read PROGRESS_FILE and derive RUN_COUNTER (monotonic per change-id):
  - If any prior entry contains `Run: #<n>`, set RUN_COUNTER = (max n) + 1
  - Else RUN_COUNTER = 1
- Read FEATURE_FILE (context only; do not edit).
- Proceed to task selection.

1) Batch session loop (one invocation = many task attempts, serial)
- Loop:
  - Read TASKS_FILE and select CURRENT_TASK using the eligibility rules in 1.1 (top-to-bottom).
  - If no eligible task exists -> STOP via stop condition (A) "No eligible tasks remain".

  - For CURRENT_TASK, run a per-task retry loop up to MAX_ATTEMPTS:
    - Let MAX_ATTEMPTS = 2 (or the configured constant in this command).
    - Let ATTEMPT be derived from PROGRESS_FILE (resumable across sessions; see 1.1).
    - While ATTEMPT <= MAX_ATTEMPTS:
      - Spawn EXACTLY ONE new subagent for this ONE task attempt (never bundle).
      - Supervisor verifies + books (explicit control flow; keep auto-retries):
        - Determine post-subagent status UNDER THIS task only:
          - READY_TO_VALIDATE if:
            - tasks.md contains exactly ONE `BUNDLE (RUN #<RUN_COUNTER>): ...` line for this attempt, and
            - the referenced run-folder exists and contains the required bundle assets (task.md + run.sh + run.bat + logs/; and if GUI/MIXED, tests/ with MCP runbook).
          - BLOCKED if tasks.md contains `BLOCKED:` + `NEEDS:` under this task.
          - ROLE_VIOLATION if the Worker wrote any `EVIDENCE (RUN #...)` / PASS/FAIL/RESULT/validated= conclusion, toggled any checkbox, or modified feature_list.json.
          - NO_PROGRESS otherwise.

        - If READY_TO_VALIDATE:
          - Execute validation as Supervisor:
            - CLI scope: run `auto_test_openspec/**/run.sh|run.bat` and capture logs/outputs (append-only in the run-folder).
            - GUI/MIXED scope:
              - run.* is start-server only (start the service and print URL/port),
              - execute `tests/gui_runbook_*.md` via MCP service `playwright-mcp` (no manual browser; no scripts),
              - capture evidence (at minimum screenshots + screenshots index under logs/; trace/video/console index optional).
          - Record result under THIS task (Supervisor-only):
            - Write ONE `EVIDENCE (RUN #<RUN_COUNTER>): ... | RESULT: PASS|FAIL | ...` line with evidence pointers.
          - If RESULT is PASS:
            - Toggle checkbox to `- [x]` (Supervisor only).
            - Append progress.txt entry (Status=DONE, Attempt=<k>, bundle + evidence pointers).
            - Continue the outer batch loop (pick next eligible task).   # explicit continue
          - If RESULT is FAIL:
            - Append progress.txt entry (Status=FAIL, Attempt=<k>, distilled blocker + evidence pointers).
            - If ATTEMPT < MAX_ATTEMPTS:
              - Add/refresh `UNBLOCK GUIDANCE (RUN #<RUN_COUNTER>): ...` under the SAME task in tasks.md (Supervisor only).
              - ATTEMPT += 1 and retry the SAME task with a fresh subagent.  # explicit retry
            - Else (ATTEMPT == MAX_ATTEMPTS):
              - Mark the task as MAXED (Supervisor note under task; do NOT check it):
                - `MAXED (RUN #<RUN_COUNTER>): <short reason>`
              - Enforce dependency-blocking stop logic:
                - If the Supervisor cannot safely proceed to any later unchecked task:
                  - STOP via stop condition (B) and report the required human unblock input.  # explicit stop
                - Else:
                  - Continue the outer batch loop.  # explicit continue
        - If BLOCKED / ROLE_VIOLATION / NO_PROGRESS:
          - Append progress.txt entry (Status=BLOCKED/ROLE_VIOLATION/NO_PROGRESS, Attempt=<k>, distilled blocker + next-step suggestion).
          - If ATTEMPT < MAX_ATTEMPTS:
            - Add/refresh `UNBLOCK GUIDANCE (RUN #<RUN_COUNTER>): ...` under the SAME task in tasks.md (Supervisor only).
            - ATTEMPT += 1 and retry the SAME task with a fresh subagent.   # explicit retry
          - Else (ATTEMPT == MAX_ATTEMPTS):
            - Mark the task as MAXED (Supervisor note under task; do NOT check it):
              - `MAXED (RUN #<RUN_COUNTER>): <short reason>`
            - Enforce dependency-blocking stop logic:
              - If the Supervisor cannot safely proceed to any later unchecked task:
                - STOP via stop condition (B) and report the required human unblock input.  # explicit stop
              - Else:
                - Continue the outer batch loop.  # explicit continue

- Terminate ONLY via stop conditions (A) or (B) (and "All tasks done" as a subset of A).
- Do NOT stop after a single task by default.

	1.1) Determine CURRENT_TASK (eligible + resumable attempts)
	- Read TASKS_FILE.
	- Scan tasks top-to-bottom and pick the FIRST unchecked checkbox item that is ELIGIBLE.
	  - ELIGIBLE means ALL are true:
	    - It is not explicitly marked NOT_EXECUTABLE / SKIP (by a Supervisor note under the task).
	    - It is not already MAXED (i.e., previously reached MAX_ATTEMPTS without success).
	    - It is not blocked by an earlier unmet prerequisite:
	      - Default: tasks are weakly ordered; earlier unchecked/maxed tasks are presumed prerequisites.
	      - Exception (allowed to proceed): the candidate task has an explicit independence marker under it
	        (e.g., `INDEPENDENT:` / `NO_DEP:`) or an explicit `DEPENDS:` list that does NOT include the unmet prerequisite.
	- If no eligible unchecked task exists after the full scan:
	  - Stop via "No eligible tasks remain" (stop condition A).

	- Capture:
	  - TASK_LINE = the full checkbox line
	  - TASK_NUM = e.g., `1.1` if present, else `?`
	  - REF_TAG = e.g., `[#R1]` if present, else `[]`

	- Derive ATTEMPT counter for this task (resumable across sessions):
	  - Read PROGRESS_FILE and find prior RUN entries where `Task: <task-num>` matches TASK_NUM.
	  - Let ATTEMPT = (max recorded Attempt for this TASK_NUM) + 1, else 1 if none exist.
	  - Note: Attempt is per-task (not per-session). RUN_COUNTER remains global monotonic.

	- Lock scope (per-task atomicity):
	  - For the duration of the upcoming subagent/Codex run, the Worker MUST work ONLY on this CURRENT_TASK.
	  - After the subagent returns, the Supervisor may select the next eligible task and spawn a new subagent.

  1.2) Print RUN banner (START)
  Output exactly:
  `[MONITOR] RUN #<RUN_COUNTER> START | change=$ARGUMENTS | task=<TASK_NUM> | ref=<REF_TAG> | text="<TASK_LINE>"`

  1.3) Spawn ONE subagent for CURRENT_TASK
  Use the Task tool to spawn a NEW subagent (e.g., name it "codex-worker").
- The Supervisor MUST NOT run Bash for implementation work (coding/build steps).
- The Supervisor MAY run Bash ONLY for:
  - executing the validation bundle entrypoint (`auto_test_openspec/**/run.sh|run.bat`) to capture auditable outputs/logs
  - minimal Git bookkeeping after PASS (commit + show/diffstat), as explicitly allowed in `allowed-tools`
  - any GUI steps MUST be executed ONLY via MCP service `playwright-mcp` (no manual browser; no Python/Node/Playwright scripts).
  
  IMPORTANT: Explicitly instruct the subagent that manual file editing is banned. 
  Tell the subagent: "I will reject any work that does not produce a `codex exec` execution log. Do not try to edit files directly."

Subagent instructions (copy verbatim):
---
You are the CODEX CLI OPERATOR. Your ONLY job is to run Codex CLI exactly once and report results. You are NOT a software engineer.

MISSION: You must force the `codex` CLI tool to perform the work.
NON-NEGOTIABLE RULE: You are FORBIDDEN from using `Write`, `Edit`, or `Replace` tools on project files. You have NO permission to edit code manually.
TOOLS:
- You MAY use the Read tool to inspect files (tasks.md / progress.txt / feature_list.json).
- You MUST invoke the Bash tool exactly once, and that single invocation MUST be CODEX_CMD.
- You are FORBIDDEN from using Write/Edit/Replace on project files.

Execution Steps (Do exactly this):
1. Read (Read tool, not Bash):
   - `openspec/changes/$ARGUMENTS/tasks.md`
   - `openspec/changes/$ARGUMENTS/progress.txt`
   - `openspec/changes/$ARGUMENTS/feature_list.json`
2. Construct a prompt for the CLI using the template below.
3. Run exactly ONE Bash command:
   codex exec --full-auto --skip-git-repo-check --model gpt-5.2 -c model_reasoning_effort=medium "$(cat <<'PROMPT'<INLINE_PROMPT>PROMPT)"
4. Verify the CLI updated `tasks.md` under THIS task ONLY (no checkbox toggles).
   Verify the Worker output is BUNDLE-ready (and ONLY bundle-ready):
   - Under THIS task, there is EXACTLY ONE single-line `BUNDLE (RUN #<RUN_COUNTER>): ...` pointer that targets a concrete run-folder:
     - includes `CODEX_CMD=codex exec --full-auto --skip-git-repo-check --model gpt-5.2 -c model_reasoning_effort=medium`
     - includes `SCOPE: <CLI|GUI|MIXED>`
     - includes `VALIDATION_BUNDLE: auto_test_openspec/$ARGUMENTS/run-.../`
     - includes `HOW_TO_RUN: run.sh/run.bat`
     - if SCOPE includes GUI: includes `RUNBOOK: tests/gui_runbook_*.md`
   - The referenced run-folder exists and contains at minimum:
     - `task.md`, `run.sh`, `run.bat`,
     - `logs/worker_startup.txt` (mandatory startup snapshot),
     - and (when GUI/MIXED) `tests/` containing an MCP-only runbook (no scripts).
   - The Worker did NOT:
     - write any `EVIDENCE (RUN #...)` line
     - write PASS/FAIL/RESULT/validated= conclusions
     - toggle any checkbox
   Also verify governance constraints:
   - `feature_list.json` MUST NOT be modified by the Worker (neither entries nor pass-state).
   - No git commit is expected/allowed from the Worker.
   - If the CLI violated any of the above, report failure.

<INLINE_PROMPT> Template (fill variables):

(Shared setup)
- change-id: $ARGUMENTS
- include the exact TASK_LINE text (verbatim)
- state explicitly: "Implement ONLY this task (no other tasks, no refactors outside scope)."
- require full validation per the task’s `TEST:` and the canonical spec:
  - Follow `openspec/project.md` → `## tasks.md Checklist Format` → `### Validation bundle requirements (mandatory)`
  - Produce a human-reproducible validation bundle under:
    `auto_test_openspec/$ARGUMENTS/<run-folder>/`
  - Worker MAY run quick local checks to ensure the bundle is runnable,
    but MUST NOT claim PASS/FAIL/validated (Supervisor is the final verifier).

A) Worker deliverables (validation bundle assets)
- Create a NEW run-folder (append-only; never overwrite prior runs):
  `auto_test_openspec/$ARGUMENTS/run-<RUN4>__task-<TASK_ID>__ref-<REF>__<YYYYMMDDThhmmssZ>/`
- Minimum required files inside the run-folder:
  - `task.md` (self-sufficient README; includes How-to-run + machine-decidable pass/fail criteria)
  - `run.sh` and `run.bat`
  - `logs/worker_startup.txt` (MANDATORY; see Startup ritual below)
  - `logs/` (for provenance + transcripts; keep append-only within this run folder)
  - If SCOPE includes GUI/MIXED: `tests/gui_runbook_*.md` (MCP-only runbook; no executable browser scripts)
  - If the task needs inputs/expected: include `inputs/`, `expected/`, and write outputs into `outputs/` (never temp dirs)
- GUI/MIXED server-start contract (MANDATORY):
  - `task.md` MUST include a dedicated section with EXACT, copy/paste-able commands:
    - `SERVER_START:` <exact command to start the server>
    - `SERVER_URL:` <exact URL Supervisor should navigate to, including host + port>
    - `READY_CHECK:` <a concrete readiness check (endpoint or observable signal)>
  - For GUI/MIXED, `run.sh` / `run.bat` MUST implement `SERVER_START`:
    - MUST start the local server and print the `SERVER_URL` to stdout.
    - MUST NOT perform validation (no PASS/FAIL claims); start-server only.

- Environment isolation (mandatory ONLY if env problems occur):
  - DO NOT install Python deps globally.
  - If missing deps / conflicts prevent execution, create an isolated venv via `uv` inside THIS run folder
    (e.g., `<run-folder>/.venv/`) and ensure `run.sh`/`run.bat` uses it.
  - Log provenance into `logs/` (always): python path+version, uv version, dependency source, exact install commands.
A) Startup ritual (MANDATORY, before any edits)
- REQUIRE CodeX STARTUP RITUAL:
  - read `openspec/changes/$ARGUMENTS/progress.txt`
  - read `openspec/changes/$ARGUMENTS/feature_list.json`
  - run `git log --oneline -20`
  - capture `GIT_BASE` via `git rev-parse --short HEAD`
  - write a Startup snapshot to the validation bundle (NOT tasks.md), at:
    - `auto_test_openspec/$ARGUMENTS/<run-folder>/logs/worker_startup.txt`
  - The snapshot MUST include: UTC timestamp, CODEX_CMD, GIT_BASE, the git-log excerpt, and a short “what I observed” summary.

B) tasks.md bookkeeping (Worker-owned; single-line; NO conclusions)
- require Codex to update `openspec/changes/$ARGUMENTS/tasks.md` under THIS task with exactly ONE Worker bookkeeping line (NOT EVIDENCE):
  - starting with: `BUNDLE (RUN #<RUN_COUNTER>): ...`
  - MUST be a SINGLE LINE
  - MUST NOT write any `EVIDENCE (RUN #...)` line
  - MUST NOT write any PASS/FAIL/RESULT/validated= conclusions
- The single BUNDLE line MUST include ONLY:
  - `CODEX_CMD=codex exec --full-auto --skip-git-repo-check --model gpt-5.2 -c model_reasoning_effort=medium`
  - `SCOPE: <CLI|GUI|MIXED>`
  - `VALIDATION_BUNDLE: auto_test_openspec/$ARGUMENTS/run-<RUN4>__task-<TASK_NUM>__ref-<REF>__<YYYYMMDDThhmmssZ>`
  - `HOW_TO_RUN: run.sh/run.bat`
  - (if SCOPE=GUI or MIXED) `RUNBOOK: tests/gui_runbook_*.md`
  - (if SCOPE=GUI or MIXED) `SERVER_URL: <exact url including host+port>`
- forbid Codex from toggling ANY checkbox in tasks.

C) GUI hard rules (only if SCOPE includes GUI/MIXED)
- GUI verification is Supervisor-only via MCP service `playwright-mcp`.
- Worker deliverable for GUI is ONLY the MCP runbook file:
  - `tests/gui_runbook_*.md` MUST be MCP-only steps + selectors + assertion points + evidence capture points.
  - ABSOLUTELY NO executable browser automation scripts (no Playwright test runner; no Python/Node scripts).
  - ABSOLUTELY NO manual browser steps anywhere (no “open Chrome/click …” prose, anywhere in the bundle).
- For GUI/MIXED bundles, `run.sh` / `run.bat` MUST be start-server only:
  - MUST start the local server and print URL/port.
  - MUST NOT perform state seeding/copying/exporting/testing/validation/probing/installs.

D) Governance boundaries (Worker forbidden; Supervisor-only)
- feature_list governance (MANDATORY; strict):
  - The Worker/Codex is FORBIDDEN to edit `openspec/changes/$ARGUMENTS/feature_list.json` (no entry edits, no pass-state edits, no formatting churn).
  - If `openspec/changes/$ARGUMENTS/feature_list.json` is missing OR the matching ref entry is missing:
    - Under THIS task write:
      BLOCKED: Missing feature_list.json (or missing ref entry for <REF_TAG>)
      NEEDS: Supervisor/initializer must create/repair feature_list.json (structure + ref mapping). Then re-run this task.
    - Then END THIS WORKER RUN immediately (do not proceed with implementation in this run).
  - Pass-state updates (e.g., `passes=true/false`) are Supervisor-only and may occur ONLY after Supervisor validation PASS + EVIDENCE is recorded.
- forbid touching any other tasks (no evidence elsewhere; no changes to other items)
- governance boundary (Worker/Codex; mandatory):
  - The Worker/Codex is FORBIDDEN to create git commits (no checkpoint commits).
  - The Worker/Codex is FORBIDDEN to edit or append `git_openspec_history/<change-id>/runs.log`.
  - The Worker/Codex MUST NOT attempt to produce DIFFSTAT/FILES “final” summaries as evidence.
  - All commit/runs.log bookkeeping (and DIFFSTAT capture) is Supervisor-only and may occur ONLY after Supervisor validation PASS.

3) After Codex finishes, confirm that `openspec/changes/$ARGUMENTS/tasks.md` has either:

BUNDLE-READY (Worker output, under THIS task):
  - EXACTLY ONE `BUNDLE (RUN #<RUN_COUNTER>): ...` line that points to a concrete run-folder:
    - includes `VALIDATION_BUNDLE: auto_test_openspec/$ARGUMENTS/run-.../`
    - includes `HOW_TO_RUN: run.sh/run.bat`
    - if SCOPE includes GUI/MIXED: includes `RUNBOOK: tests/gui_runbook_*.md`
    - if SCOPE includes GUI/MIXED: includes `SERVER_URL: ...`
  - The referenced run-folder exists and contains at minimum:
    - `task.md`, `run.sh`, `run.bat`, `logs/worker_startup.txt`,
    - and (when GUI/MIXED) `tests/` with an MCP-only runbook
  - For GUI/MIXED, `task.md` MUST include `SERVER_START:` + `SERVER_URL:` + `READY_CHECK:` (as defined above).
  - Worker MUST NOT have written any `EVIDENCE (RUN #...)` line.
  - Worker MUST NOT have toggled any checkbox.
  - Worker MUST NOT have edited feature_list.json.
  - Worker MUST NOT have created any git commit.
  - Worker MUST NOT have edited `git_openspec_history/<change-id>/runs.log`.

OR BLOCKED (Worker output, under THIS task):
  - `BLOCKED: ...` (1–5 line error excerpt)
  - `NEEDS: ...` (next concrete unblock step)

OR ROLE_VIOLATION (Worker output, under THIS task):
  - Any `EVIDENCE (RUN #...)` / PASS/FAIL/RESULT/validated= conclusion, checkbox toggle, feature_list.json edit, git commit,
    or any edit/append to `git_openspec_history/<change-id>/runs.log`.

Otherwise treat as NO_PROGRESS (missing BUNDLE line and/or missing run-folder).

1.4) Supervisor verification after subagent returns
- Re-read TASKS_FILE.
- Determine status (under THIS task only):

  - READY_TO_VALIDATE if a compliant BUNDLE (RUN #<RUN_COUNTER>) line exists and the referenced run-folder is present and well-formed.
  - BLOCKED if BLOCKED+NEEDS exists.
  - ROLE_VIOLATION if Worker wrote any EVIDENCE/PASS/FAIL/RESULT/validated= conclusion, toggled checkboxes, edited feature_list.json, created commits,
    or edited/appended `git_openspec_history/<change-id>/runs.log`.
  - NO_PROGRESS otherwise.

- If READY_TO_VALIDATE:
  - Supervisor MUST execute validation.
    - CLI: via `run.sh`/`run.bat` as specified in the bundle.
    - GUI/MIXED:
      1) MUST start the server first by running `run.sh`/`run.bat` (start-server only).
      2) MUST navigate using the `SERVER_URL` provided in the BUNDLE line / task.md.
      3) Then execute the MCP `playwright-mcp` runbook.
      4) If the server cannot be started or `SERVER_URL` is missing/invalid, treat as bundle not ready for validation (NO_PROGRESS or BLOCKED with NEEDS), not as a feature FAIL.
  - Supervisor writes the single EVIDENCE (RUN #<RUN_COUNTER>) line (PASS/FAIL + evidence pointers).
  - Supervisor updates feature_list.json pass-state ONLY after PASS.
  - Supervisor creates ONE checkpoint commit ONLY after PASS.
  - Supervisor appends runs.log ONLY after PASS.
  - Supervisor may then toggle the checkbox to - [x] ONLY after PASS.

- DONE is reachable only after Supervisor validation PASS + compliant EVIDENCE exists under THIS task.

If DONE:
- Toggle the checkbox to `- [x]` (Supervisor only).
- Append a FULL RUN ENTRY to PROGRESS_FILE (Supervisor only; verified facts only) including:
  - RUN SUMMARY (timestamp, run #, change-id, task/ref, status)
  - Evidence pointers (tasks.md evidence line pointer + feature_list passes change + GIT_BASE/GIT_COMMIT/COMMIT_MSG)
  - Validation commands/steps + 3–15 lines output excerpt (from Supervisor validation output and/or bundle logs)
  - Changes verified: FILES/DIFFSTAT + key edits summary
  - [DIALOGUE + TOOL TRACE] with bracket markers, including:
    - [Supervisor → Subagent] instruction
    - [Tool Use] <task - spawn subagent>
    - [Tool Use] <bash - CODEX_CMD "..."> (from subagent trace)
    - [Subagent] reported outputs + the exact BUNDLE line + bundle folder pointer(s)
    - [Supervisor] the exact EVIDENCE line + acceptance decision + rationale
- Print RUN banner (END) as before.
- RUN_COUNTER += 1 and continue/stop per your session policy.

If BLOCKED:
- Ensure actionable NEEDS exists (next concrete unblock step).

- Call skill `openspec-unblock-research` (Supervisor-only). Do NOT call MCP tools directly here.
  - Provide the skill the BLOCKED context (task line + ref, error excerpt, NEEDS, what was tried, env/versions if known).
  - Instruct the skill to write its portable research capsule into BOTH bookkeeping artifacts:
    (a) Under THIS task in tasks.md:
        Add `UNBLOCK GUIDANCE (RUN #<RUN_COUNTER>):` containing:
        - Query terms
        - Key conclusions
        - Evidence pointers (source links/locators)
        - Executable next steps + how to verify
    (b) Into progress.txt (inside the current RUN entry):
        Append a short “Unblock Research Capsule” containing:
        - Query terms
        - Key conclusions
        - Evidence pointers
        - Pointer back to the tasks.md UNBLOCK GUIDANCE location

- Append a FULL RUN ENTRY to PROGRESS_FILE capturing blocker + the skill’s capsule + retry decision (verified facts only).
- Retry once as before; if blocked again, STOP and require user/initializer intervention.

If NO_PROGRESS:
- Treat as a FAILED ATTEMPT (not an immediate session stop by default).
- Under THIS task, append/refresh a single diagnostic note:
  `BLOCKED: Missing a compliant BUNDLE pointer and/or the referenced validation bundle folder is missing/incomplete for this RUN (workflow non-compliance).`
  `NEEDS: Re-run SAME task; Worker/Codex must (1) create a fresh run-folder under auto_test_openspec/<change-id>/... containing task.md + run.sh + run.bat + logs/worker_startup.txt (+ tests/runbook if GUI), and (2) append EXACTLY ONE single-line BUNDLE (RUN #<RUN_COUNTER>) pointer under THIS task (CODEX_CMD + SCOPE + VALIDATION_BUNDLE + HOW_TO_RUN [+ RUNBOOK]).`
- Append a FULL RUN ENTRY to PROGRESS_FILE (status=NO_PROGRESS) including:
  - the missing-gate diagnosis,
  - the subagent trace,
  - Attempt #k and the retry/maxed decision.
- Flow control MUST follow the per-task retry policy:
  - If Attempt #k < MAX_ATTEMPTS: continue the retry loop for the SAME task (fresh subagent).
  - Else (Attempt #k == MAX_ATTEMPTS): mark the task MAXED and apply dependency-blocking stop logic (stop only if it blocks safe forward progress).

2) Completion (only at start-of-session, or if CURRENT_TASK selection finds none)
- If no unchecked tasks remain:
  `[MONITOR] DONE | change=$ARGUMENTS | all tasks checked`
  then STOP.