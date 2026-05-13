# security-scan

A Claude Code skill that runs a four-part security audit on any git repository. Covers dependency vulnerabilities, secret detection, code pattern analysis, and config file audit — all mapped to OWASP Top 10 (2021).

Works on Node.js, Python, and PHP projects. Only scans git-tracked files, so gitignored files (`.env`, build artifacts, credentials) are structurally excluded.

---

## Installation

```bash
npx skills add dvwdesign/security-scan
```

Or clone and install manually:

```bash
git clone https://github.com/dvwdesign/security-scan
npx skills add ./security-scan
```

---

## Usage

```txt
/security-scan                  Audit only — no changes made
/security-scan --fix            Audit + auto-fix Critical/High where safe
/security-scan --review         Audit + triage remaining issues for human review
/security-scan --fix --review   Full pipeline: fix what's safe, flag the rest
/security-scan --json           Machine-readable JSON output for CI/CD
```

---

## What it checks

| # | Check                  | What it finds                                         | OWASP          |
| - | ---------------------- | ----------------------------------------------------- | -------------- |
| 1 | Dependency Audit       | Known CVEs in npm, pip, or composer packages          | A06            |
| 2 | Secret Detection       | Credentials, API keys, connection strings in source   | A02 · A07      |
| 3 | Code Pattern Analysis  | eval, innerHTML, command injection, SQL injection, path traversal | A03 · A01 |
| 4 | Config File Audit      | Hardcoded secrets in Docker Compose, k8s, GitHub Actions, .env.example | A02 · A05 |

---

## Sample output

```md
Security Scan Report — my-project — 2026-05-13
===============================================
Check 1 (Dependency Audit):  WARN   — 2 moderate CVEs
Check 2 (Secret Detection):  PASS
Check 3 (Code Patterns):     FAIL   — eval() at src/utils/parser.ts:84
Check 4 (Config Files):      WARN   — .env.example line 12

Overall: FAIL

Findings:
- [HIGH]     [Check 1] lodash < 4.17.21 — Prototype Pollution (CVE-2021-23337)
- [CRITICAL] [Check 3] eval() with user-controlled input — src/utils/parser.ts:84
- [MODERATE] [Check 4] .env.example contains real-looking 40-char token — line 12

Recommended Actions:
- Upgrade lodash to >= 4.17.21
- Replace eval() with JSON.parse() or a safe parser
- Replace token value with placeholder: YOUR_TOKEN_HERE
```

### JSON output (`--json`)

```json
{
  "repo": "my-project",
  "date": "2026-05-13",
  "overall": "FAIL",
  "checks": {
    "dependency_audit": { "verdict": "WARN", "count": 2 },
    "secret_detection": { "verdict": "PASS", "count": 0 },
    "code_patterns":    { "verdict": "FAIL", "count": 1 },
    "config_files":     { "verdict": "WARN", "count": 1 }
  },
  "findings": [
    {
      "severity": "CRITICAL",
      "check": "code_patterns",
      "owasp": "A03",
      "description": "eval() with user-controlled input",
      "file": "src/utils/parser.ts",
      "line": 84,
      "action": "Replace eval() with JSON.parse() or a safe parser"
    }
  ],
  "auto_fixed": []
}
```

---

## CI/CD integration

Exit codes when using `--json`:

| Code | Meaning |
| ---- | ------- |
| `0`  | PASS — no issues |
| `1`  | WARN — issues found, none blocking |
| `2`  | FAIL — Critical/High issues requiring action |

### GitHub Actions example

```yaml
- name: Security scan
  uses: anthropics/claude-code-action@beta
  with:
    prompt: /security-scan --fix --json
  env:
    ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

---

## Requirements

- `git` — required (scan scope uses `git ls-files`)
- `node` + `npm` — for Node.js dependency audit and `--fix`
- `python3` — for JSON output formatting; `pip-audit` optional for Python projects
- `gitleaks` — optional, enhances secret detection if installed
- `composer` — optional, for PHP dependency audit

---

## OWASP coverage

| Code | Category                                   | Covered                                       |
| ---- | ------------------------------------------ | --------------------------------------------- |
| A01  | Broken Access Control                      | Partial (path traversal via readFileSync)     |
| A02  | Cryptographic Failures                     | Yes — Check 2, Check 4                        |
| A03  | Injection                                  | Yes — Check 3                                 |
| A04  | Insecure Design                            | No — architectural, not statically detectable |
| A05  | Security Misconfiguration                  | Yes — Check 4                                 |
| A06  | Vulnerable and Outdated Components         | Yes — Check 1                                 |
| A07  | Identification and Authentication Failures | Yes — Check 2                                 |
| A08  | Software and Data Integrity Failures       | No — requires runtime analysis                |
| A09  | Security Logging and Monitoring Failures   | No — requires runtime analysis                |
| A10  | Server-Side Request Forgery (SSRF)         | No — requires data-flow analysis              |

A04, A08, A09, A10 are architectural or runtime concerns that static pattern scanning cannot reliably surface. Use a dedicated DAST tool or manual review for those.

---

## Scheduling

Run daily via Claude Code's built-in scheduler:

```txt
CronCreate({ cron: "7 9 * * *", prompt: "/security-scan --fix --review" })
```

Or add to your system crontab for persistence across sessions:

```bash
# runs at 9:07am daily
7 9 * * * cd /path/to/project && claude -p "/security-scan --fix --review"
```

---

## License

MIT — DVW Design
