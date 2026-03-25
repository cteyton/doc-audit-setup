---
name: doc-audit-setup
description: 'Generate a customized /doc-audit skill for any codebase. Scans the repository to detect documentation location, user-facing feature types, and exclusions, then generates a complete doc-audit skill tailored to the project. Use for: documentation health checks, doc quality review, finding stale docs, detecting broken doc links, identifying documentation drift, auditing doc coverage. Invoke when setting up documentation auditing for a new project, reconfiguring after major structural changes, or whenever the user mentions doc quality, doc review, stale documentation, or broken links — even if they do not explicitly say "doc-audit-setup".'
---

# Documentation Audit Setup

This skill generates a customized `/doc-audit` skill for the current codebase. It produces 5 files that form a complete documentation audit pipeline — detecting broken links, outdated code references, non-existent concepts, misleading information, missing coverage, and undocumented feature areas.

**Focus: end-user/functional documentation only — not developer docs.**

## Step 1: Detect Documentation Location

Read `references/detection-patterns.md` for the full pattern list, then scan the codebase. This is a monorepo-aware scan — check inside `apps/`, `packages/`, and root-level directories.

### 1a. Auto-detect

1. **Find doc directories** — Glob for common doc directory patterns at multiple depths:
   - Root level: `docs/`, `doc/`, `documentation/`, `website/`, `content/`
   - Monorepo apps: `apps/doc/`, `apps/docs/`, `apps/documentation/`, `apps/website/`
   - Monorepo packages: `packages/doc/`, `packages/docs/`, `packages/documentation/`
   - Nested: `src/docs/`, `pages/docs/`

2. **Find framework config** — Glob for framework-specific config files anywhere in the repo:
   - `**/mint.json`, `**/docs.json`, `**/docusaurus.config.*`, `**/mkdocs.yml`, `**/.vitepress/config.*`, `**/book.toml`, `**/astro.config.*`, `**/conf.py`, `**/hugo.toml`, `**/hugo.yaml`, `**/antora.yml`, `**/antora-playbook.yml`, `**/redocly.yaml`
   - Framework configs are strong signals — if found, the parent directory is likely the doc root

3. **Find doc files** — Within each candidate directory, glob for `**/*.md`, `**/*.mdx`, `**/*.rst`, and `**/*.adoc` (exclude `CHANGELOG*`, `LICENSE*`, `CONTRIBUTING*`, `README.md`, `node_modules/`)

### 1b. Evaluate results

**If documentation directories or doc files are found:**

4. **Detect format** — Determine the primary doc format by file extension:
   - If any `.mdx` files found → MDX (`.mdx`)
   - If any `.rst` files found → reStructuredText (`.rst`)
   - If any `.adoc` files found → AsciiDoc (`.adoc`)
   - Otherwise → Markdown (`.md`)
   - When multiple formats coexist, prefer the dominant format (highest file count) and note the secondary format
5. **Detect nav config** — Look for sidebar/navigation config files in or near the doc root (`docs.json`, `sidebars.js`, `sidebars.ts`, `mkdocs.yml`, `mint.json`, `index.rst` toctrees, `nav.adoc`, `redocly.yaml`)

Present findings to user:
```
Documentation detected:
- Doc root: {path}
- Format: {.md or .mdx}
- Framework: {name or "unknown"}
- Nav config: {file or "none found"}
- Doc files found: {count}
```

**Ask user to confirm or correct.**

**If NO documentation directories or doc files are found:**

Ask the user directly:
```
No end-user documentation directory was detected in this repository.

Where is your end-user documentation located? Please provide the path relative to the project root (e.g., "docs/", "apps/doc/").

If this project does not have end-user documentation yet, this skill cannot proceed — it audits existing documentation against the codebase.
```

- If the user provides a path: verify it exists and contains `.md` or `.mdx` files. If it does, continue. If it doesn't contain doc files, inform the user and ask again.
- **If the user confirms there is no documentation: STOP the workflow.** Print:
  ```
  ⛔ No end-user documentation found. The /doc-audit skill requires existing documentation to audit.
  Consider creating documentation first, then re-run /doc-audit-setup.
  ```
  Do not proceed to Step 2 or generate any files.

### 1c. Detect i18n / multi-language structure

Check the confirmed doc root for internationalization patterns:

1. **Locale directories** — Glob for subdirectories matching common language codes within the doc root: `{DOC_ROOT}/en/`, `{DOC_ROOT}/fr/`, `{DOC_ROOT}/es/`, `{DOC_ROOT}/ja/`, etc. Also check for `i18n/`, `locales/`, or `translations/` directories.
2. **Framework i18n config** — If a framework config was found, check whether it references i18n settings, locale lists, or a default language.
3. **Parallel directory trees** — Look for parallel structures (e.g., `docs/en/getting-started/` and `docs/fr/getting-started/`) that indicate translated copies.

**If multiple languages detected:**

Present to the user:
```
Multiple documentation languages detected:
- {lang1} — {path}
- {lang2} — {path}

Which language should the audit target? (The audit runs against one language at a time.)
```

Adjust `DOC_ROOT` and `DOC_FILE_GLOB` to scope to the selected language's subtree if the languages live in separate directory trees.

**If no i18n structure detected:** skip — no adjustment needed.

### 1d. Record values

Once a valid doc root is confirmed, record:
- `DOC_ROOT` — path to doc root directory (e.g., `docs/`)
- `DOC_FILE_GLOB` — glob pattern for doc files (e.g., `docs/**/*.md`)
- `DOC_FILE_EXT` — file extension (`.md`, `.mdx`, `.rst`, or `.adoc`)
- `NAV_CONFIG_FILE` — nav/sidebar config file path (or empty)
- `DOC_FRAMEWORK` — framework name (Mintlify, Docusaurus, MkDocs, VitePress, Jekyll, mdBook, Astro, Sphinx, Hugo, Antora, Redocly, or "generic")
- `DOC_LANGUAGE` — target language code (e.g., `en`) if i18n detected, otherwise empty

## Step 2: Discover Feature Types

Scan the codebase to detect which **user-facing feature types** exist. Use `references/detection-patterns.md` as the scanning playbook.

For each feature type below, run the detection heuristics and record whether it's present:

### 2a. API Endpoints
- Glob for directories named `routes`, `handlers`, `controllers`, `views`, `api`, `endpoints`, `resources`, `routers` at any depth
- Read 2-3 files from candidate directories to confirm they define HTTP routes/handlers (look for HTTP method annotations, route path definitions, request/response patterns)
- Check the project's dependency files for web/API framework dependencies as additional signals
- If found: record the source directory and a glob pattern covering its files

### 2b. CLI Commands
- Glob for directories named `commands`, `cmd`, `cli`, `bin` at any depth
- Read 2-3 files from candidate directories to confirm they define CLI commands (look for argument parsing, subcommand registration, help strings)
- Check package manifests for CLI framework dependencies and `bin` entry points or console scripts
- If found: record the command directory path and a glob pattern for command source files

### 2c. SDK / Library Exports
- Read the project's package manifest to find the main entry point(s) (e.g., `main`, `exports`, `lib`, `entry_points` fields or equivalent)
- Look for entry point files at conventional locations (`src/`, `lib/`, root-level module files)
- Read the entry point file(s) — if they re-export multiple modules or declare a public API surface, this feature type is present
- If found: record the entry point locations

### 2d. Configuration Options
- Glob for `**/*.env.example`, `**/.env.template`, `**/*config*.schema.*`
- Glob for directories named `config` or `configuration` containing typed definitions or schemas
- Read candidate files to confirm they define user-facing configuration (environment variables, settings schemas, config keys with defaults)
- If found: record the config source locations

### 2e. Webhooks / Events
- Glob for directories named `webhooks`, `hooks`, `listeners`, `subscribers` at any depth
- Read 2-3 files from candidate directories to confirm they handle externally-facing webhooks or user-configurable events (not internal domain events or analytics)
- If found: record the source directories

### 2f. UI Pages / Features
- Glob for directories named `pages`, `views`, `screens`, `app` at any depth
- Read a few files from candidate directories to confirm they define page/route components or route configurations
- Look for route configuration or router setup files
- If found: record the routes/pages directories

### 2g. Packages (Monorepo)
- Check for workspace/monorepo config files: `nx.json`, `turbo.json`, `lerna.json`, `pnpm-workspace.yaml`, `go.work`, or root-level manifests with workspace definitions
- Glob for subdirectories containing their own package manifests: `packages/*/`, `libs/*/`, `crates/*/`, `modules/*/`, `services/*/`
- If found: record the packages/workspaces directory path

### 2h. Other Feature Areas (project-specific)

The 7 categories above cover common patterns, but every project is different. Scan for additional user-facing feature areas not yet captured:

1. **Read the project README** — look for "Features" sections, capability listings, or descriptions of functionality that don't map to any detected type above
2. **Read the package manifest** — check for entry points, scripts, or module declarations suggesting additional public surface (e.g., plugin systems, template engines, data export pipelines)
3. **Scan major source directories** — identify top-level source directories not already associated with a detected feature type; read a few files to assess whether they represent a user-facing capability

If additional feature areas are found, add them to the detection results with a descriptive label and their source path.

**Present findings to user:**
```
Feature types detected:
✓ API endpoints — {source path}
✓ CLI commands — {source path}
✗ SDK exports — not detected
✗ Configuration options — not detected
✗ Webhooks/events — not detected
✗ UI pages — not detected
✓ Packages (monorepo) — {source path}
✓ {custom feature} — {source path}
```

**Ask user to:**
1. Confirm which types are actually present (may need to correct false positives/negatives)
2. Provide any missed feature source paths
3. If CLI detected: provide the CLI tool name (e.g., `my-tool` as used in `my-tool <command>`)
4. Are there other user-facing feature areas not listed above? (e.g., plugins, templates, integrations, data models, auth scopes)

Record each confirmed feature type with its `{type, source_glob, label}`.

## Step 3: Detect Exclusions

Identify directories and packages that are internal infrastructure (not user-facing) and should be excluded from the audit. Use `references/detection-patterns.md` for the assessment approach.

1. **Scan project structure** — List all top-level and second-level directories. For monorepos, also list all packages/workspaces.

2. **Assess each directory** — For each directory, determine whether it is likely user-facing or internal by:
   - Reading its README or top-level files (if any) for descriptions of purpose
   - Checking if the directory is referenced in existing documentation (if so, likely user-facing)
   - Checking if the directory exports public API surface or defines user-visible features
   - Considering its role: test infrastructure, build tooling, internal utilities, dev-only tooling, and database migrations are typically internal

3. **API-level exclusions** — If API endpoints were detected in Step 2a, also flag health check endpoints and bootstrap/startup code.

**Present findings to user with reasoning:**
```
Proposed exclusions (internal/infrastructure, not user-facing):
- {directory} — {reason based on content analysis}
- {directory} — {reason based on content analysis}

These will be skipped during the audit. Items NOT in this list will be checked for documentation coverage.
```

**When uncertain about a directory, do not exclude it** — auditing an internal directory is less harmful than missing a user-facing feature area.

**Ask user to confirm/adjust.** Record as `EXCLUSION_LIST`.

## Step 4: Clarifying Questions

Ask the user these remaining questions in a single prompt:

1. **Product name** — What should we call this product in the audit prompts? (used for labels like "the {product} documentation")
2. **Redirects** — Does your doc framework handle redirects? If so, where is the redirects config? (This prevents false positives in broken link detection)
3. **Custom code patterns** — Are there specific code patterns your docs reference frequently? (e.g., config keys like `SETTING_NAME`, API path prefixes like `/api/v1/`)

Record:
- `PRODUCT_NAME`
- `HAS_REDIRECTS` (true/false) and `REDIRECTS_CONFIG` (file path or description)

## Step 5: Generate the Skill

### 5a. Determine output directory

Default: `.claude/skills/doc-audit/`. Ask user if they want a different location. If the directory already exists, warn and ask whether to overwrite.

### 5b. Build template values

From the collected data, prepare the following template substitution blocks:

**Ground truth sources block** (`GROUND_TRUTH_SOURCES`): For each confirmed feature type, generate the numbered ground truth collection instruction. Example for CLI:
```
2. **CLI commands** — List files in `src/commands/` to get current command files
```

**Feature type verification blocks** (`FEATURE_TYPE_VERIFICATION_BLOCKS`): For each confirmed feature type, generate a verification sub-section for Category B. Example for API endpoints:
```
**API endpoint references:**
- How to verify: Use Grep to search for the endpoint path in `src/routes/`. Read the route file to confirm the endpoint exists with the documented HTTP method and path.
- Valid finding: Doc references `POST /api/users/invite` but no matching route exists
- Not a finding: Generic references to "the API" without a specific endpoint path
```

**Coverage check blocks** (`COVERAGE_CHECK_BLOCKS`): For each confirmed feature type, generate a coverage check sub-section for Category E. Example for CLI:
```
**CLI commands:** Compare ground truth CLI command files against doc mentions. A command is "covered" if mentioned on any doc page.
```

**Reviewer feature type blocks** (`REVIEWER_FEATURE_TYPE_BLOCKS`): For each confirmed feature type, generate Category B verification steps for the reviewer. Example for CLI:
```
**CLI command references:**
1. Read the cited doc file and locate the CLI command reference
2. Use Glob to check if a matching command file exists in `src/commands/`
3. If found, Read it and verify the command name/subcommand matches
4. If the doc references specific flags or options, verify they exist in the source
```

**Opportunity feature sources** (`OPPORTUNITY_FEATURE_SOURCES`): For each confirmed feature type, generate a scanning section. Example for API:
```
### API Handlers
- Glob `{API_SOURCE_PATH}` to list all route/handler files
- Group handlers by domain path
- Each domain group represents a functional area
```

**Exclusion list formatted** (`EXCLUSION_LIST_FORMATTED`): Format the exclusion list as a readable block based on what was confirmed in Step 3. Example:
```
**Internal directories**: {list of excluded directories with brief reasons}
**Internal API**: health checks, bootstrap
```

**Feature type working strategies block** (`FEATURE_TYPE_WORKING_STRATEGIES`): For each confirmed feature type beyond CLI and API (which have dedicated conditionals), generate a working strategy hint for the audit agents. Example for SDK:
```
   - **SDK exports** — check the entry point files for the referenced symbol using Grep across `{SDK_SOURCE_PATH}`.
```
Example for Config:
```
   - **Configuration keys** — grep for the key name in config schema files at `{CONFIG_SOURCE_PATH}`.
```
Example for Webhooks:
```
   - **Webhook references** — search webhook handler directories at `{WEBHOOK_SOURCE_PATH}` for the event type.
```
Example for UI:
```
   - **UI features** — check route/page directories at `{UI_SOURCE_PATH}` for the referenced page or component.
```
For custom feature types detected in Step 2h, generate a similar hint using the feature label and its source path.

**Doc framework link resolution** (`DOC_FRAMEWORK_LINK_RESOLUTION`): Based on the detected framework, write the link resolution rule. Examples:
- Mintlify: "Links resolve relative to the `{DOC_ROOT}` root — a link to `/concepts/foo` maps to `{DOC_ROOT}concepts/foo.mdx`."
- Docusaurus: "Links resolve relative to the `{DOC_ROOT}` root — a link to `/concepts/foo` maps to `{DOC_ROOT}concepts/foo.md`."
- Sphinx: "Links resolve via toctree references and `:ref:` / `:doc:` roles relative to the `{DOC_ROOT}` root — a `:doc:` link to `concepts/foo` maps to `{DOC_ROOT}concepts/foo.rst`."
- Hugo: "Links resolve relative to the `content/` directory — a link to `/concepts/foo/` maps to `{DOC_ROOT}concepts/foo/_index.md` or `{DOC_ROOT}concepts/foo.md`."
- Antora: "Links resolve within the module structure — an `xref:module:page.adoc` link maps to `modules/{module}/pages/page.adoc`."
- Redocly: "Links resolve relative to the `{DOC_ROOT}` root — a link to `/concepts/foo` maps to `{DOC_ROOT}concepts/foo.md` or `{DOC_ROOT}concepts/foo.mdx`."
- Generic: "Links resolve relative to the `{DOC_ROOT}` root — a link to `/concepts/foo` maps to `{DOC_ROOT}concepts/foo{DOC_FILE_EXT}`."

### 5c. Generate files

Read each template from `references/templates/`, substitute all `{{PLACEHOLDER}}` values, and handle conditional blocks:

1. **Read** the template file
2. **Replace** all `{{PLACEHOLDER}}` tokens with their collected values
3. **Process conditional blocks:**
   - If condition is true: remove only the `{{#IF_*}}` and `{{/IF_*}}` markers, keep the content between them
   - If condition is false: remove the entire block including markers and all content between them
4. **Process multi-line blocks** (`{{FEATURE_TYPE_VERIFICATION_BLOCKS}}`, `{{COVERAGE_CHECK_BLOCKS}}`, etc.): replace the placeholder with the generated block text from Step 5b
5. **Write** the final file to the output directory

#### Placeholder & Conditional Reference

**Conditionals** — each wraps content that should only appear when the condition is true:

| Conditional | True when | Source |
|---|---|---|
| `{{#IF_NAV_CONFIG}}` / `{{/IF_NAV_CONFIG}}` | `NAV_CONFIG_FILE` is non-empty | Step 1c |
| `{{#IF_CLI}}` / `{{/IF_CLI}}` | CLI Commands feature type confirmed | Step 2b |
| `{{#IF_API}}` / `{{/IF_API}}` | API Endpoints feature type confirmed | Step 2a |
| `{{#IF_MONOREPO}}` / `{{/IF_MONOREPO}}` | Packages (Monorepo) feature type confirmed | Step 2g |
| `{{#IF_REDIRECTS}}` / `{{/IF_REDIRECTS}}` | `HAS_REDIRECTS` is true | Step 4 |

**Simple placeholders** — replaced with a single value:

| Placeholder | Description | Source |
|---|---|---|
| `{{PRODUCT_NAME}}` | Product name for labels | Step 4 |
| `{{DOC_ROOT}}` | Path to documentation root directory | Step 1c |
| `{{DOC_FILE_GLOB}}` | Glob pattern for doc files (e.g., `docs/**/*.md`) | Step 1c |
| `{{DOC_FILE_EXT}}` | Doc file extension (`.md` or `.mdx`) | Step 1c |
| `{{NAV_CONFIG_FILE}}` | Nav/sidebar config file path (or empty) | Step 1c |
| `{{DOC_FRAMEWORK_LINK_RESOLUTION}}` | Framework-specific link resolution rule | Step 5b |
| `{{CLI_COMMANDS_PATH}}` | Directory containing CLI command source files | Step 2b |
| `{{PACKAGES_PATH}}` | Monorepo packages directory | Step 2g |
| `{{API_SOURCE_PATH}}` | Directory containing API route/controller source | Step 2a |
| `{{REDIRECTS_CONFIG}}` | Redirects config file path or description | Step 4 |
| `{{EXCLUSION_LIST}}` | Comma-separated exclusion list for inline references | Step 3 |

**Multi-line block placeholders** — replaced with generated text blocks from Step 5b:

| Placeholder | Content | Source |
|---|---|---|
| `{{ADDITIONAL_GROUND_TRUTH_SOURCES}}` | Extra ground truth collection steps per feature type | Step 5b |
| `{{FEATURE_TYPE_VERIFICATION_BLOCKS}}` | Category B verification instructions per feature type | Step 5b |
| `{{COVERAGE_CHECK_BLOCKS}}` | Category E coverage checks per feature type | Step 5b |
| `{{REVIEWER_FEATURE_TYPE_BLOCKS}}` | Reviewer verification steps per feature type | Step 5b |
| `{{FEATURE_TYPE_WORKING_STRATEGIES}}` | Audit agent working strategy hints per feature type (beyond CLI/API) | Step 5b |
| `{{OPPORTUNITY_FEATURE_SOURCES}}` | Feature source scanning sections for opportunity detection | Step 5b |
| `{{EXCLUSION_LIST_FORMATTED}}` | Multi-line formatted exclusion block with categories | Step 5b |

Generate these 5 files:
1. `{output}/SKILL.md` ← from `references/templates/SKILL.md.template`
2. `{output}/references/section-audit-instructions.md` ← from `references/templates/section-audit-instructions.md.template`
3. `{output}/references/section-audit-agent-prompt.md` ← from `references/templates/section-audit-agent-prompt.md.template`
4. `{output}/references/reviewer-agent-prompt.md` ← from `references/templates/reviewer-agent-prompt.md.template`
5. `{output}/references/opportunities-agent-prompt.md` ← from `references/templates/opportunities-agent-prompt.md.template`

### 5d. Validate

After writing all files:
1. Read each generated file and check for any remaining `{{` — if found, a placeholder was missed; fix it
2. Verify no placeholder markers (`{{#IF_`, `{{/IF_`) remain
3. Confirm the generated SKILL.md frontmatter has a valid `name` and `description`

## Step 6: Print Summary

Print a completion summary:

```
✅ doc-audit skill generated at {output_directory}/

Files created:
- SKILL.md (main 6-phase audit workflow)
- references/section-audit-instructions.md (detection rules)
- references/section-audit-agent-prompt.md (audit agent template)
- references/reviewer-agent-prompt.md (review agent template)
- references/opportunities-agent-prompt.md (opportunity detection template)

Configured for:
- Product: {PRODUCT_NAME}
- Doc root: {DOC_ROOT} ({count} files, {DOC_FILE_EXT} format)
- Feature types: {list of confirmed types}
- Exclusions: {count} directories/packages excluded

To run the audit: /doc-audit
```
