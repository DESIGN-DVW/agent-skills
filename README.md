# DVW Design — Agent Skills

Open-source Claude Code skills from [DVW Design](https://dvw.design).

Each skill is a composable, installable unit that extends Claude Code with specialized workflows. Install any skill with the [Skills CLI](https://skills.sh/):

```bash
npx skills add design-dvw/agent-skills@<skill-name>
```

---

## Available Skills

| Skill | Description | Install |
| ----- | ----------- | ------- |
| [security-scan](skills/security-scan/) | Four-part security audit — CVE detection, secret scanning, code pattern analysis, config file audit. OWASP-mapped. | `npx skills add design-dvw/agent-skills@security-scan` |

---

## What These Skills Do

Skills in this collection are built for real-world engineering workflows. They run inside Claude Code, use only the tools they need (`Bash`, `Read`, `Grep`), and produce actionable output — not boilerplate reports.

### Design Principles

- **Git-scoped** — scans cover only git-tracked files; gitignored secrets are structurally excluded
- **OWASP-mapped** — findings link to the relevant Top 10 category
- **CI/CD ready** — `--json` flag + exit codes for pipeline gating
- **Self-contained** — no external services, no cloud calls, no keys required
- **Auto-fixable** — `--fix` applies safe remediations; `--review` flags what remains for humans

---

## More Skills Coming

This collection grows as we extract and generalise workflows from the DVW Design ecosystem. Watch this repo for updates.

---

## License

MIT — [DVW Design](https://dvw.design)
