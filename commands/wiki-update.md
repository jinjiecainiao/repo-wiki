---
description: "Incrementally update existing wiki documentation based on code changes"
---

Invoke the repo-wiki:generate-wiki skill in **update mode**.

The user may provide arguments:
- `--base <ref>`: Git ref to compare against (branch, tag, or commit). Default: `main`.
- `--lang <code>`: Documentation language (e.g., `zh-CN`, `en`, `ja`). Default: `en`.

Pass all arguments to the skill as context, and indicate this is an incremental update (not full generation).
