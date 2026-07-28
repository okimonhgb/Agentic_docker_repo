---
description: Run lat.md commands to search, validate, and update the project knowledge graph
---

# lat.md Knowledge Graph

This project uses [lat.md](https://www.npmjs.com/package/lat.md) to maintain a structured knowledge graph in `lat.md/`.

## Before starting work

1. Run `lat search "<task description>"` to find sections relevant to your task. Read them to understand design intent before writing code.
2. Run `lat expand "<user prompt>"` to expand any `[[refs]]` in the prompt to resolved file locations.

## Post-task checklist (REQUIRED — do not skip)

After EVERY task, before responding to the user:

1. Update `lat.md/` if you added or changed any functionality, architecture, tests, or behavior.
2. Run `lat check` — all wiki links and code refs must pass.
3. Do not skip these steps. Do not consider your task done until both are complete.

## Useful commands

```bash
lat locate "Section Name"      # find a section by name (exact, fuzzy)
lat refs "file#Section"        # find what references a section
lat search "natural language"  # semantic search across all sections
lat expand "user prompt text"  # expand [[refs]] to resolved locations
lat check                      # validate all links and code refs
lat --help                     # show all available commands
```

If `lat search` fails because no API key is configured, use `lat locate` for direct lookups instead.
