# Detection Patterns

Reference data for the doc-audit-setup skill. Use these patterns to scan a codebase and detect documentation locations, feature types, and exclusions.

## Documentation Locations

### Directory Patterns
Glob for these common documentation directories (order = priority):
- `docs/`
- `doc/`
- `documentation/`
- `apps/doc/`
- `apps/docs/`
- `website/docs/`
- `website/`
- `content/`
- `src/docs/`
- `pages/docs/`

### Framework Config Files
Look for these to identify the documentation framework:

| Config File | Framework | Supports Redirects | Nav Config Location | Link Resolution |
|---|---|---|---|---|
| `mint.json` | Mintlify | Yes (in `mint.json` redirects) | `docs.json` or `mint.json` navigation | Relative to doc root, no extension |
| `docusaurus.config.js` or `.ts` | Docusaurus | Yes (via plugin) | `sidebars.js` or `.ts` | Relative to doc root, no extension |
| `mkdocs.yml` | MkDocs / Material | Yes (via redirects plugin) | `mkdocs.yml` nav section | Relative to doc root, `.md` extension |
| `.vitepress/config.*` | VitePress | No built-in | `.vitepress/config.*` sidebar | Relative to doc root, `.md` extension |
| `gatsby-config.*` | Gatsby | No built-in | Custom (varies) | Varies |
| `next.config.*` + `pages/docs/` | Next.js docs | No built-in | Custom sidebar component | File-based routing |
| `book.toml` | mdBook (Rust) | No | `SUMMARY.md` | Relative, `.md` extension |
| `_config.yml` + `docs/` | Jekyll | Yes (via jekyll-redirect-from) | `_data/navigation.yml` | Relative, `.md` or `.html` |
| `conf.py` + `docs/` or `source/` | Sphinx | Yes (via sphinx-reredirects) | `index.rst` toctree directives | Relative to doc root, `.rst` extension |
| `hugo.toml` / `hugo.yaml` / `config.toml` (with Hugo markers) | Hugo | No built-in | `config.toml` menu or `_index.md` front matter | Content-based, section `_index` files |
| `antora.yml` / `antora-playbook.yml` | Antora | No built-in | `nav.adoc` per module | Module/page structure, `.adoc` extension |
| `redocly.yaml` | Redocly | Yes (in `redocly.yaml`) | `redocly.yaml` sidebar | Relative to doc root, no extension |

### Doc File Detection
Within the detected doc root:
- Glob `**/*.md`, `**/*.mdx`, `**/*.rst`, and `**/*.adoc`
- If any `.mdx` files found → format is MDX (`.mdx`)
- If any `.rst` files found → format is reStructuredText (`.rst`)
- If any `.adoc` files found → format is AsciiDoc (`.adoc`)
- Otherwise → format is Markdown (`.md`)
- When multiple formats coexist, prefer the dominant format (highest file count) and note the secondary format
- Exclude: `CHANGELOG*`, `LICENSE*`, `CONTRIBUTING*`, `CODE_OF_CONDUCT*`, `node_modules/`

## Feature Type Detection

For each feature type, scan using the heuristics below. These heuristics are language-agnostic — they rely on directory structure, file content sampling, and package manifest inspection rather than specific file extensions or frameworks.

A feature type is "detected" if at least one strong signal is found. When in doubt, include the candidate and let the user confirm or reject it.

### API Endpoints

**Directory signals** — Glob for directories named:
`**/routes/`, `**/handlers/`, `**/controllers/`, `**/views/`, `**/api/`, `**/endpoints/`, `**/resources/`, `**/routers/`

**File content signals** — Read 2-3 files from candidate directories and look for:
- HTTP method annotations or decorators (GET, POST, PUT, DELETE, PATCH)
- Route path definitions (string literals resembling URL paths like `/users`, `/api/v1/...`)
- Request/response handling patterns (reading request bodies, returning responses with status codes)

**Package manifest signals** — Check the project's dependency files for web/API framework dependencies. This list is non-exhaustive; the goal is to recognize any HTTP framework:
- Examples: Express, FastAPI, Django, Gin, Spring Boot, Rails, Phoenix, Actix, Flask, Koa, etc.

**Record**: the directory containing route/handler definitions and a glob pattern covering its source files.

### CLI Commands

**Directory signals** — Glob for directories named:
`**/commands/`, `**/cmd/`, `**/cli/`, `**/bin/`

**File content signals** — Read 2-3 files from candidate directories and look for:
- Argument/flag parsing (positional arguments, option flags, help strings)
- Subcommand registration or dispatch patterns
- Console/terminal output formatting

**Package manifest signals** — Check for CLI framework dependencies. Non-exhaustive examples:
- commander, yargs, oclif, click, typer, cobra, clap, argparse, etc.
- Also check if the package manifest declares a `bin` entry point or console scripts

**Record**: command directory path and a glob pattern for command source files.

### SDK / Library Exports

**Directory signals** — Look for main entry point files by reading the project's package manifest:
- Check fields like `main`, `exports`, `lib`, `entry_points`, or equivalent for the project's language
- Look for files at conventional entry point locations (`src/`, `lib/`, root-level module files)

**File content signals** — Read the entry point file(s) and look for:
- Multiple re-exports or public API declarations
- Symbols explicitly marked as public (`export`, `pub`, `__all__`, etc.)
- A clear public API surface (not just internal wiring)

**Record**: entry point file locations and the public symbols or modules they expose.

### Configuration Options

**Directory signals** — Glob for:
- `**/*.env.example`, `**/.env.template`
- `**/*config*.schema.*` (config schema files in any format)
- Directories named `config/` or `configuration/` containing typed definitions

**File content signals** — Read candidate files and look for:
- Environment variable definitions or documentation
- Typed configuration interfaces, structs, or schemas
- Settings objects with documented keys and default values

**Record**: config source file locations.

### Webhooks / Events

**Directory signals** — Glob for directories named:
`**/webhooks/`, `**/hooks/`, `**/listeners/`, `**/subscribers/`

**File content signals** — Read 2-3 files from candidate directories and look for:
- Webhook payload handling (receiving HTTP callbacks, signature verification)
- Event type definitions or event handler registration
- Patterns distinguishing user-facing webhooks/events from internal pub/sub or domain events

**Important**: Internal event buses, domain events, and analytics events are common false positives. Only flag directories where files handle externally-facing webhooks or events that users configure.

**Record**: source directories containing webhook/event handler definitions.

### UI Pages / Features

**Directory signals** — Glob for directories named:
`**/pages/`, `**/views/`, `**/screens/`, `**/app/`

**File content signals** — Read a few files from candidate directories and look for:
- Page or route component definitions (rendering UI for a specific URL path)
- File-based routing conventions (filename maps to URL)
- Route configuration or router setup files

**Record**: routes/pages directories and any route configuration files.

### Packages (Monorepo)

**Detection** — Check for workspace/monorepo configuration:
- Config files: `nx.json`, `turbo.json`, `lerna.json`, `pnpm-workspace.yaml`, `go.work`, or a root `Cargo.toml` with `[workspace]`
- Package manifest fields: `workspaces` in `package.json`, `members` in `Cargo.toml`, etc.
- Glob for subdirectories containing their own package manifests: `packages/*/`, `libs/*/`, `crates/*/`, `modules/*/`, `services/*/`

**Record**: packages/workspaces directory path.

## Exclusion Heuristics

Identify directories and packages that are **internal infrastructure, not user-facing** and should be excluded from the audit. Use content-based analysis rather than directory name matching — project structures vary widely.

### Content-Based Exclusion Assessment

For each top-level and second-level directory (and each monorepo package if applicable), assess whether it is internal infrastructure by examining:

1. **Purpose indicators** — Read the directory's README, main entry file, or package manifest description. Does it describe internal tooling, test infrastructure, build processes, or developer utilities?

2. **Documentation references** — Grep the doc files for mentions of this directory or package. If the docs reference it, it is likely user-facing and should NOT be excluded.

3. **Public API surface** — Does the directory export symbols or features that end users interact with? If it defines part of the product's public interface, do not exclude it.

4. **Common internal roles** (use as guidance when assessing purpose, not as automatic exclusion rules):
   - Test suites and test infrastructure
   - Build output and cache directories
   - Developer scripts and tooling
   - Database migrations and seeds
   - Internal shared utilities with no public-facing interface
   - Mock/fixture/stub data for testing
   - Benchmark and performance test suites

### API-Level Exclusions

When API endpoints are detected, also exclude these from the audit:
- Health check endpoints (`/health`, `/healthz`, `/ready`, `/ping`)
- Bootstrap/startup code
- Infrastructure middleware (unless it implements user-facing behavior that should be documented)

### Confidence Check

Before excluding any directory or package:
- If it has its own README or doc page → probably user-facing, don't exclude
- If referenced in existing documentation → don't exclude
- If it exports public API surface → don't exclude
- **When uncertain, do not exclude.** Auditing an internal directory is less harmful than missing a user-facing feature area.
