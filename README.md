# standards

Shared coding conventions and AI agent instructions for all projects.

## Files

| File | Purpose |
|------|---------|
| `AGENTS.md` | Agent instructions for this repo (overview, structure, usage) |
| `AGENTS.universal.md` | Universal conventions — commit to every project root |
| `AGENTS.go.md` | Go-specific conventions — commit to every Go project root |
| `AGENTS.android.md` | Android conventions — builds, validation, versioning, signing and release-triggered F-Droid deployment |
| `.claude/skills/y-convert-to-standards/` | Skill to migrate a Go repo to these standards |

## Usage

To pull the latest conventions into a project:

```bash
make standards
```

To update conventions, edit the relevant file directly.

Android app repositories keep a copy of `AGENTS.android.md` at their root and
link it from `AGENTS.md`. Their `make standards` target refreshes it alongside
the universal conventions and any other applicable language files. This covers
both standalone Android apps and Android modules inside other projects.
