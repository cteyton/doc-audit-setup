# doc-audit-setup

**Tired of outdated docs, broken links, and features nobody documented?** Stop guessing — audit it.

A Claude Code skill that generates a customized `/doc-audit` skill tailored to your codebase. It scans your repo, asks a few questions, then produces a complete documentation audit pipeline that detects broken links, stale references, missing coverage, and more.

**Focus:** end-user/functional documentation only — not developer docs.

## How it works

Run `/doc-audit-setup` and the skill walks through 6 steps:

1. **Detect documentation location** — Scans for doc directories, framework configs (Mintlify, Docusaurus, MkDocs, Sphinx, Hugo, etc.), file formats (`.md`, `.mdx`, `.rst`, `.adoc`), nav configs, and i18n structure. Asks you to confirm.

2. **Discover feature types** — Scans source code for user-facing features: API endpoints, CLI commands, SDK exports, configuration options, webhooks/events, UI pages, monorepo packages, and any project-specific feature areas. Asks you to confirm.

3. **Detect exclusions** — Identifies internal directories (test infra, build tooling, dev utilities) that should be skipped during the audit. Asks you to confirm.

4. **Clarifying questions** — Asks for the product name, redirect config location, and any custom code patterns your docs reference.

5. **Generate the skill** — Fills in templates with your project's specifics and writes 5 files to `.claude/skills/doc-audit/`:

   | File | Role |
   |---|---|
   | `SKILL.md` | Main 6-phase audit workflow |
   | `references/section-audit-instructions.md` | Detection rules for doc issues |
   | `references/section-audit-agent-prompt.md` | Prompt template for audit agents |
   | `references/reviewer-agent-prompt.md` | Prompt template for review agents |
   | `references/opportunities-agent-prompt.md` | Prompt template for coverage gap detection |

6. **Print summary** — Shows what was configured and how to run it.

## Usage

```
/doc-audit-setup    # one-time setup — generates the skill for your project
/doc-audit          # run the audit
```

Re-run `/doc-audit-setup` after major structural changes to regenerate the skill.

## Automate with GitHub Actions

Run `/doc-audit` on a weekly schedule so documentation issues never pile up. Copy [`weekly-doc-review.yml`](weekly-doc-review.yml) into your repo at `.github/workflows/weekly-doc-review.yml`:

```yaml
name: Weekly Documentation Audit

on:
  schedule:
    - cron: '0 7 * * 1' # Every Monday at 7:00 UTC
  workflow_dispatch:
    inputs:
      model:
        description: 'Claude model to use'
        required: false
        default: 'claude-sonnet-4-6'
        type: string

permissions:
  contents: read
  id-token: write

jobs:
  doc-audit:
    runs-on: ubuntu-latest
    timeout-minutes: 30
    steps:
      - uses: actions/checkout@v4

      - name: Run Documentation Audit
        uses: anthropics/claude-code-action@v1
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
          prompt: "/doc-audit"
          settings: |
            {
              "permissions": {
                "allow": [
                  "Bash(echo *)",
                  "Bash(git remote *)",
                  "Bash(mkdir *)",
                  "Agent",
                  "Read",
                  "Write",
                  "Glob",
                  "Grep"
                ]
              }
            }
          claude_args: "--model ${{ inputs.model || 'claude-opus-4-6' }} --max-turns 50"

      - name: Display reports
        if: always()
        run: |
          if [ -f "doc-audit-report.md" ]; then
            echo "=== Documentation Audit Report ==="
            cat "doc-audit-report.md"
            echo "## Documentation Audit Report" >> "$GITHUB_STEP_SUMMARY"
            cat "doc-audit-report.md" >> "$GITHUB_STEP_SUMMARY"
          else
            echo "No report file found."
            echo "No report file found." >> "$GITHUB_STEP_SUMMARY"
          fi

      - name: Upload reports artifact
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: doc-audit-report
          path: doc-audit-report.md
          if-no-files-found: warn
          retention-days: 30
```

**Prerequisites:**
- Add your `ANTHROPIC_API_KEY` as a [repository secret](https://docs.github.com/en/actions/security-for-github-actions/security-guides/using-secrets-in-github-actions)
- Run `/doc-audit-setup` at least once so the skill exists in your repo
- See [claude-code-action](https://github.com/anthropics/claude-code-action) for more options
