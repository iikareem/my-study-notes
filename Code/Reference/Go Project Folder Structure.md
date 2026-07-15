---
tags:
  - code
  - go
  - project-structure
  - reference
aliases:
  - Folder structure Reference
  - Go folder structure
---

# Go Project Folder Structure

Short reference for organizing a Go codebase.

## Starter reading

- [Organize Like a Pro — Go project folder structures (Medium)](https://medium.com/@smart_byte_labs/organize-like-a-pro-a-simple-guide-to-go-project-folder-structures-e85e9c1769c2)

## Common layout ideas

Many Go projects settle on a variant of:

```
cmd/           # main entrypoints (one folder per binary)
internal/      # private application code (not importable by others)
pkg/           # optional: library code safe for external import
api/           # OpenAPI / proto / HTTP contract defs
configs/       # config samples
scripts/       # build / migrate helpers
test/          # extra test utilities (or keep tests next to code)
```

**Rules of thumb**

- Prefer **`internal/`** for app code you do not want others to import.
- Keep **`cmd/`** thin: wire dependencies, then call into `internal/`.
- Match package names to **responsibility**, not to layers alone (`user`, `billing` vs only `controllers` / `services` when the domain is rich).

## See also

- [[Code MOC]]
- [[Treat Third-Party Dependencies as Untrusted]] — keep vendor details at the edges
- [[Enforce the "Single Level of Abstraction" Principle (SLAP)]] — keep `cmd` and handlers thin
