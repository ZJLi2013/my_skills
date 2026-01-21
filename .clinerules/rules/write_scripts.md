---
description: |
  Prevent terminal hangs / missing stdout: when a Bash workflow is more than a simple one-liner,
  write it to a .sh file first, then execute the script.
author: updated by assistant
version: 1.0
tags: ["bash","reliability","tooling"]
globs: ["*"]
---

# Write Bash scripts to a `.sh` file, then execute

## Intent
Some multi-line Bash sequences can be hard to capture reliably when sent directly to the terminal.
To reduce the chance of hangs / lost stdout, prefer writing a shell script file first.

## Compatibility with other `.clinerules`
- This rule is **additive** and **must not override** or disable other rules in `.clinerules/`.
- If another rule requires a specific workflow, follow both rules. When in doubt: keep the workflow, but run the Bash via a `.sh` file.

## When to apply
Use a script file when:
- The command is multi-line, uses loops/functions/heredocs, or has complex quoting.
- The command is long-running or produces lots of output.
- The command chains many sub-commands with `&&`, pipes, redirects, `set -euo pipefail`, etc.

Direct `execute_command` is still fine for short, simple, read-only one-liners (e.g., `ls`, `cat`, `rg`).

## How to apply (preferred)
1. Create a file like `scripts/_cline_tmp/<name>.sh` (any temp folder is OK).
2. Start with:
   ```bash
   #!/usr/bin/env bash
   set -euo pipefail
   ```
3. Execute it via `execute_command` using:
   ```bash
   bash scripts/_cline_tmp/<name>.sh
   ```
   (No need to `chmod +x` unless you want to.)

## Notes
- Keep scripts deterministic and avoid interactive prompts.
- If the script is destructive (deletes files, overwrites outputs, etc.), ensure the `execute_command` call is marked as requiring approval.