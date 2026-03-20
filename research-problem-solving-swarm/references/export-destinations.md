# Export Destinations

Read this file only when the user wants the research memo or decision note written somewhere specific.

## Default Behavior
- If the user does not name a destination, write a portable markdown file to the report directory.
- Offer an optional save to a project path or a notes tool in the final response.
- Preserve markdown as the source of truth even if the target system has richer formatting.

## Project Paths
- Prefer repo-local destinations such as `docs/research/`, `notes/`, or a path the user names.
- Create parent directories only inside the writable workspace.
- Use dated, slugged filenames such as `2026-03-17-ci-flakiness.md` unless the user names a specific file.

## Obsidian
- If the user provides a vault path with normal filesystem access, write a `.md` file directly into that vault.
- Preserve frontmatter, callouts, or wiki-links only if the vault already uses them or the user asked for them.
- If the vault is only reachable through MCP or a dedicated CLI, use that integration instead of guessing paths.
- Do not guess the vault location. Require a provided path, configured connector target, or explicit existing workspace location.

## Notion
- Prefer an MCP tool or existing integration that can create or update pages.
- Confirm the parent page or database when it is not obvious from the request.
- Keep the structure simple and portable: title, summary, headings, bullets, checklists, and tables only when the tool supports them cleanly.
- If direct write access is unavailable, save markdown locally and report that it can be pasted or imported.

## Notes Apps / Apple Notes
- Prefer MCP integration first.
- If no MCP tool exists, use platform automation or CLI only when it is present and the write is safe to perform.
- Treat note destinations as explicit targets. Do not guess a personal notebook, account, or folder.
- If the target strips markdown, convert headings and checklists into plain text equivalents rather than silently dropping structure.

## Failure Handling
- If an external write fails, do not lose the work.
- Save the markdown locally.
- Report the attempted destination, the blocker, and the fallback file path.

## Multi-Destination Writes
- If the user wants both a project artifact and an external note, write the local markdown first.
- Use the local file as the canonical artifact, then mirror it to the second destination.
- Mention every location that was successfully written.
