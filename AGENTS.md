# standards

Universal conventions and language-specific extensions.

```
AGENTS.universal.md          Commit to every project root
AGENTS.go.md                 Commit to every Go project root
CLAUDE.md                    One line: @AGENTS.md — lets Claude Code read the same instructions
prompts/
  convert-to-standards.md    Paste into any agent to convert a repo
```

To refresh in a project:
```bash
make standards
```

To update conventions: edit the relevant file, update `prompts/convert-to-standards.md` if migration steps changed.
