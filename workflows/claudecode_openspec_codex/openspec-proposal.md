- When drafting `openspec/changes/<id>/tasks.md`, you MUST follow:
  - `openspec/project.md` → `## tasks.md Checklist Format` (canonical; do not invent a parallel format).

- Hard gate reminders (do not expand here; see canonical spec above):
  - Every task MUST include `ACCEPT:` and `TEST:`.
  - Every checkbox task line MUST include EXACTLY ONE `[#R<n>]` token, unique across the file.
  - `TEST:` MUST include `SCOPE: CLI|GUI|MIXED` and MUST enable a human-reproducible validation bundle
    (all bundle rules + role split + evidence rules live ONLY in `openspec/project.md`).

  - Role split (mandatory; see `openspec/project.md` → “Validation bundle requirements”):
    - Worker produces bundle assets only; Supervisor executes and records PASS/FAIL evidence.

  - GUI/MIXED constraint (mandatory; see `openspec/project.md` → “CLI/GUI/MIXED validation requirements”):
    - GUI verification must be driven via MCP service `playwright-mcp` and evidence must be archived; do NOT use any browser automation scripts (Python/Node/Playwright test runner).