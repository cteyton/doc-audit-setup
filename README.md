# Skill `doc-audit-setup` to maintain user docs in your monorepo

**Tired of outdated docs, broken links, and features nobody documented?** Stop guessing — audit it.

If your repo looks something like this, this skill is for you:

```
my-project/
├── src/            # application code
├── docs/           # user-facing documentation (Markdown, MDX, RST…)
├── api/
└── ...
```

Your code and your docs live side by side, but nothing ensures they stay in sync. This skill cross-references both to surface gaps, stale references, and inconsistencies automatically.

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

Run `/doc-audit` on a weekly schedule so documentation issues never pile up. Copy [`weekly-doc-review.yml`](weekly-doc-review.yml) into your repo at `.github/workflows/weekly-doc-review.yml`.

**Prerequisites:**
- Add your `ANTHROPIC_API_KEY` as a [repository secret](https://docs.github.com/en/actions/security-for-github-actions/security-guides/using-secrets-in-github-actions)
- Run `/doc-audit-setup` at least once so the skill exists in your repo
- See [claude-code-action](https://github.com/anthropics/claude-code-action) for more options
