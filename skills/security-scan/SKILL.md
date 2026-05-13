---
name: security-scan
description: "Four-part security audit — dependency vulnerabilities, secret detection, code pattern analysis, and config file audit. Supports --fix to auto-remediate safe issues and --review to flag what remains."
allowed-tools: [Bash, Read, Grep]
---

# /security-scan

Run a four-part security audit against the current repository. Auto-detects project type. Only scans git-tracked files — gitignored files are never touched.

Usage:
- `/security-scan` — audit only, no changes
- `/security-scan --fix` — audit + auto-fix Critical/High where safe
- `/security-scan --review` — audit + flag unfixed issues for human review
- `/security-scan --fix --review` — full pipeline: fix what's safe, flag the rest
- `/security-scan --json` — machine-readable JSON output for CI/CD pipelines

**Exit codes** (when `--json` is set, suitable for CI gate use):
- `0` — PASS: no issues found
- `1` — WARN: issues found, none blocking
- `2` — FAIL: Critical/High issues requiring immediate action

---

## Check 1 — Dependency Audit
> **OWASP A06: Vulnerable and Outdated Components**

Detect the package manager and run the appropriate audit tool:

```bash
# Node / npm
if [ -f package.json ]; then
  npm audit --audit-level=moderate 2>&1
fi

# Python
if [ -f requirements.txt ] || [ -f pyproject.toml ]; then
  pip-audit 2>/dev/null || echo "pip-audit not installed — run: pip install pip-audit"
fi

# PHP / Composer
if [ -f composer.json ]; then
  composer audit 2>/dev/null || echo "composer not found"
fi
```

If `--fix` is set and Node is detected, apply safe fixes then report what remains unfixed:

```bash
npm audit fix 2>&1

# List remaining Critical/High that could not be auto-fixed
npm audit --json 2>/dev/null | python3 -c "
import json, sys
data = json.load(sys.stdin)
for name, v in data.get('vulnerabilities', {}).items():
    sev = v.get('severity', '')
    if sev in ('critical', 'high'):
        fix = v.get('fixAvailable', False)
        print(sev.upper() + ': ' + name + (' (fix available)' if fix else ' (manual fix required)'))
" 2>/dev/null
```

Report severity breakdown:
- **Critical:** N
- **High:** N
- **Moderate:** N
- **Low:** N

---

## Check 2 — Secret Detection
> **OWASP A02: Cryptographic Failures · A07: Identification and Authentication Failures**

**Scope: git-tracked files only.** `git ls-files` returns only files known to git — gitignored files (`.env`, build artifacts, credentials) are automatically excluded. This eliminates the entire class of false positives from local-only files.

```bash
# Scan tracked source files for credential patterns
git ls-files | grep -E '\.(ts|tsx|js|jsx|py|php|rb|go|sh|conf)$' | xargs grep -n \
  -e "password\s*=\s*['\"][^'\"]\+" \
  -e "api[_-]key\s*=\s*['\"][^'\"]\+" \
  -e "secret\s*=\s*['\"][^'\"]\+" \
  -e "private[_-]key\s*=\s*['\"][^'\"]\+" \
  -e "Bearer [A-Za-z0-9_\-\.]\{20,\}" \
  -e "-----BEGIN.*PRIVATE KEY-----" \
  -e "mongodb+srv://[^\"' ]\+:[^\"' ]\+" \
  -e "postgres://[^\"' ]\+:[^\"' ]\+" \
  -e "mysql://[^\"' ]\+:[^\"' ]\+" \
  2>/dev/null

# Scan tracked data/config files
git ls-files | grep -E '\.(json|yaml|yml|toml|ini|xml)$' | \
  grep -v -e "package-lock.json" -e "yarn.lock" -e "pnpm-lock.yaml" | \
  xargs grep -n \
  -e "\"password\"\s*:\s*\"[^\"]\+" \
  -e "\"secret\"\s*:\s*\"[^\"]\+" \
  -e "\"api_key\"\s*:\s*\"[^\"]\+" \
  2>/dev/null
```

Verify `.gitignore` covers the critical sensitive file patterns:

```bash
for pattern in ".env" ".env.local" ".env.production" "*.pem" "*.key" "*.p12" "*.pfx"; do
  grep -q "$pattern" .gitignore 2>/dev/null \
    && echo "OK: $pattern covered" \
    || echo "MISSING: $pattern not in .gitignore"
done
```

If `gitleaks` is installed, run a full scan with redaction:

```bash
which gitleaks > /dev/null 2>&1 && gitleaks detect --source . --redact 2>&1
```

Distinguish false positives (placeholder values: `"your-key-here"`, `"changeme"`, `"example"`, `"<secret>"`) from genuine exposures. Only flag values that look like real credentials.

---

## Check 3 — Code Pattern Analysis
> **OWASP A03: Injection · A01: Broken Access Control**

**Scope: git-tracked source files only.**

```bash
git ls-files | grep -E '\.(ts|tsx|js|jsx)$' | xargs grep -n \
  -e "eval(" \
  -e "innerHTML\s*=" \
  -e "outerHTML\s*=" \
  -e "document\.write(" \
  -e "dangerouslySetInnerHTML" \
  -e "child_process\.exec(" \
  -e "execSync(" \
  -e "\.query(\`" \
  -e "shell:\s*true" \
  -e "\.readFileSync([^)]*\.\." \
  2>/dev/null
```

Risk reference for findings:

| Pattern                   | Risk                                           | Exploitable if…                       | OWASP       |
| ------------------------- | ---------------------------------------------- | ------------------------------------- | ----------- |
| `eval(`                   | Remote code execution                          | Any user input reaches it             | A03         |
| `innerHTML =`             | XSS                                            | User-controlled string assigned       | A03         |
| `dangerouslySetInnerHTML` | XSS (React)                                    | Value derived from user input         | A03         |
| `document.write(`         | XSS / DOM injection                            | User input in argument                | A03         |
| `child_process.exec(`     | Command injection                              | Args include user-controlled values   | A03         |
| `execSync(`               | Command injection (sync)                       | Args include user-controlled values   | A03         |
| `.query(\``               | SQL injection via template literal             | Variables in query string not escaped | A03         |
| `shell: true`             | Enables shell injection in child_process       | Any user input in the command         | A03         |
| `readFileSync(..`         | Path traversal                                 | Path derived from user input          | A01         |

---

## Check 4 — Config File Audit
> **OWASP A02: Cryptographic Failures · A05: Security Misconfiguration**

Scan configuration files that are commonly committed with secrets overlooked.

**Docker Compose — hardcoded service credentials:**

```bash
git ls-files | grep -E 'docker-compose.*\.ya?ml' | xargs grep -n \
  -e "POSTGRES_PASSWORD\s*:" \
  -e "MYSQL_ROOT_PASSWORD\s*:" \
  -e "REDIS_PASSWORD\s*:" \
  -e "password\s*:" \
  2>/dev/null | grep -iv "your-\|changeme\|example\|placeholder\|CHANGE_ME\|<"
```

**Kubernetes — plaintext Secret objects (should use sealed-secrets or external-secrets):**

```bash
git ls-files | grep -E '\.ya?ml' | xargs grep -ln "kind:\s*Secret" 2>/dev/null | while read f; do
  echo "K8s Secret found: $f — verify no plaintext values (use stringData with external secrets)"
done
```

**GitHub Actions — hardcoded values instead of `${{ secrets.* }}`:**

```bash
git ls-files | grep -E '\.github/workflows/.*\.ya?ml' | xargs grep -n \
  -e "password\s*:" \
  -e "api[_-]key\s*:" \
  -e "token\s*:" \
  2>/dev/null | grep -v '\${{ secrets\.'
```

**`.env.example` — real-looking values committed as examples:**

```bash
git ls-files | grep -E '\.env\.(example|sample|template)$' | xargs grep -nE \
  "[A-Za-z0-9_-]{32,}" \
  2>/dev/null | grep -iv "your-\|changeme\|example\|placeholder\|xxxxxxxx"
```

Flag any hit as **WARN** — example files with real-looking values risk being copied and used directly.

---

## --review Pass (runs after --fix, or standalone)

After fixes are applied (or if `--fix` was not set), produce an actionable triage of remaining issues:

- List unresolved Critical/High dependency vulnerabilities with CVE IDs and affected package version
- List genuine secret exposures with the file and line, and recommended action (rotate credential, add to .gitignore, remove from history with `git filter-repo`)
- List code patterns that require manual refactor, with the specific line and the safer alternative
- List config file findings with the file and recommended fix
- Assign each finding: **CRITICAL / HIGH / MODERATE / LOW**

---

## Report

```
Security Scan Report — {repo name} — {date}
============================================
Check 1 (Dependency Audit):  PASS | WARN | FAIL | skipped
Check 2 (Secret Detection):  PASS | WARN | FAIL
Check 3 (Code Patterns):     PASS | WARN | FAIL
Check 4 (Config Files):      PASS | WARN | FAIL

Overall: PASS | WARN | FAIL

Findings:
- [SEVERITY] [Check N] description — file:line

Recommended Actions:
- [concrete remediation step]

Auto-fixed this run:
- [packages updated, if --fix was set]
```

| Verdict | Condition |
| ------- | ---------------------------------------------------------------------------- |
| PASS    | No issues found across all checks                                            |
| WARN    | Issues found, none immediately blocking (low-severity deps, false positives) |
| FAIL    | Critical/High vulnerability, confirmed secret exposure, or dangerous pattern in production code |

---

## --json Output (CI/CD mode)

When `--json` is set, suppress all prose and emit a single JSON object to stdout. Set exit code based on overall verdict (0/1/2). This allows CI pipelines to gate on the result and parse findings programmatically.

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
      "severity": "HIGH",
      "check": "dependency_audit",
      "owasp": "A06",
      "description": "lodash < 4.17.21 — Prototype Pollution",
      "cve": "CVE-2021-23337",
      "action": "Upgrade lodash to >= 4.17.21"
    },
    {
      "severity": "CRITICAL",
      "check": "code_patterns",
      "owasp": "A03",
      "description": "eval() with user-controlled input",
      "file": "src/utils/parser.ts",
      "line": 84,
      "action": "Replace eval() with JSON.parse() or a safe parser"
    },
    {
      "severity": "MODERATE",
      "check": "config_files",
      "owasp": "A05",
      "description": ".env.example contains real-looking 40-char token",
      "file": ".env.example",
      "line": 12,
      "action": "Replace with placeholder value e.g. YOUR_TOKEN_HERE"
    }
  ],
  "auto_fixed": []
}
```

Produce this structure by collecting all findings during the four checks, then serialising at the end with `python3 -c "import json, sys; ..."` or equivalent. When `--fix` is also set, populate `auto_fixed` with a list of `{ "package": "x", "from": "1.0.0", "to": "1.0.1" }` entries.

---

## OWASP Top 10 (2021) Reference

| Code | Category                                    | Checks that surface it |
| ---- | ------------------------------------------- | ---------------------- |
| A01  | Broken Access Control                       | Check 3 (path traversal) |
| A02  | Cryptographic Failures                      | Check 2, Check 4       |
| A03  | Injection                                   | Check 3                |
| A05  | Security Misconfiguration                   | Check 4                |
| A06  | Vulnerable and Outdated Components          | Check 1                |
| A07  | Identification and Authentication Failures  | Check 2                |

Full reference: https://owasp.org/Top10
