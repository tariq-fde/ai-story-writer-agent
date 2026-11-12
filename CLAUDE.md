# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Project Is

AI Story Writer Agent — an enterprise tool that converts refined PRDs (Markdown) into JIRA-ready Scrum artifacts (Epics → Stories → Sub-tasks). Part of the AI PDLC pipeline: **PRD Agent → Discovery Agent → AI Story Writer Agent**.

Three implementations live in one npm-workspaces monorepo:
- **`packages/core`** — Shared library: types, Zod schemas, Handlebars prompt templates, story engine, JIRA field mapper, validators
- **`packages/skill`** — Claude Code skill wrapper (publishable to org catalog, uses MCP for JIRA/Confluence/GitHub)
- **`packages/service`** — Standalone Express + Anthropic SDK service (deployable to GCP Cloud Run)
- **`packages/hybrid`** — Thin adapters importing core for both skill and service contexts

## Build and Development Commands

```bash
npm install                          # Install all workspace dependencies
npm run build                        # Build all packages (tsc --build)
npm run typecheck                    # Type-check without emitting
npm run lint                         # ESLint across all packages
npm run lint:fix                     # Auto-fix lint issues
npm run format                       # Prettier format
npm run format:check                 # Check formatting
npm test                             # Run all tests (vitest)
npm run test:coverage                # Tests with coverage report
```

### Per-package commands

```bash
npm run -w packages/core test        # Test core only
npm run -w packages/service dev      # Start service with tsx watch
npm run -w packages/service start    # Start service (production)
npm run -w packages/core build       # Build core only
```

### Single test file

```bash
npx vitest run packages/core/__tests__/unit/schemas/story-schema.test.ts
```

### Infrastructure

```bash
cd infra/pulumi && npm install
pulumi up -s dev                     # Deploy dev stack
pulumi up -s staging                 # Deploy staging
```

### Docker (service)

```bash
docker build -t story-writer-service -f packages/service/Dockerfile .
docker build -t story-writer-hybrid -f packages/hybrid/Dockerfile .
```

## Architecture

### Package Dependency Graph

```
packages/core        (zod, handlebars, pino — no internal deps)
    ↑
    ├── packages/skill    (core)
    ├── packages/service  (core, @anthropic-ai/sdk, express, @google-cloud/*)
    └── packages/hybrid   (core — re-exports for both contexts)
```

### Core Engine Flow

The story engine (`packages/core/src/engine/story-engine.ts`) orchestrates a multi-step LLM pipeline:

1. **Parse PRD** — `prd-parser.ts` converts Markdown (with optional YAML frontmatter) into a `PRDDocument`
2. **Decompose** — `decomposer.ts` sends PRD to LLM with `epic.hbs` template, validates output against `epicGenerationOutputSchema`
3. **Generate Stories** — `generator.ts` runs per-epic with `story.hbs` template, validates against `storyGenerationOutputSchema`
4. **Generate Sub-tasks** — runs per-story with `subtask.hbs` template
5. **Estimate & Plan** — `estimator.ts` validates story points, `sprint-planner.ts` allocates to sprints using topological sort + team velocity

The engine takes an `LlmCaller` function parameter — the skill and service each provide their own implementation (MCP-based vs Anthropic SDK).

### Review-First Workflow

Stories are **never pushed to JIRA automatically**. The engine produces a Markdown preview (`story-formatter.ts`), the PO reviews and approves, then the approved set is pushed via JIRA field mapper → JIRA API/MCP.

### JIRA Field Mapping

All JIRA field mapping is configuration-driven via `config/org/*.json` and `config/jira/*.json`. The `JiraFieldMapper` class (`packages/core/src/mappers/jira-field-mapper.ts`) handles custom field mapping, value transforms, and Markdown → Atlassian Document Format (ADF) conversion. Each org provides their own field IDs since every JIRA instance differs.

### MCP Integrations

Configured in `.mcp.json` (project root). The skill uses JIRA MCP, Confluence MCP, and GitHub MCP — credentials come from environment variables (`JIRA_API_TOKEN`, `CONFLUENCE_API_TOKEN`, `GITHUB_PAT`). The service calls REST APIs directly instead.

**GitHub constraint**: The agent never clones repos locally. It uses GitHub API/MCP/GraphQL for code search and file exploration.

### Prompt Templates

Handlebars templates in `packages/core/src/prompts/templates/*.hbs`. Each has a TypeScript builder in the parent directory. Templates are parameterized by org config (terminology overrides, example stories for few-shot, custom system prompt addendum).

### GCP Deployment

- **Cloud Run** — service and hybrid packages (0-10 instances, IAM auth)
- **Firestore** — PRDs, generation runs, story previews, review sessions
- **Pub/Sub** — events: story-generated, story-approved, jira-pushed
- **Secret Manager** — API tokens
- **Artifact Registry** — Docker images

IaC via Pulumi in `infra/pulumi/` with dev/staging/prod stacks.

## Key Type Definitions

All in `packages/core/src/types/`:
- `PRDDocument` / `PRDSection` / `PRDMetadata` — PRD input model
- `Epic` / `EpicEstimate` / `ReviewStatus` — Epic output model
- `UserStory` / `UserStoryStatement` / `AcceptanceCriteria` — Story model with Given/When/Then
- `SubTask` / `SubTaskCategory` — Implementation task model
- `JiraFieldMapping` / `JiraCreateIssuePayload` — JIRA integration model
- `OrgConfig` / `StoryDefaults` / `PromptOverrides` — Per-org configuration

## Testing

- **Unit tests** in `__tests__/unit/` — schemas, mappers, validators, prompt builders
- **Integration tests** in `__tests__/integration/` — engine flows, API routes (supertest)
- **Fixtures** in `__tests__/fixtures/` — sample PRDs, expected outputs, org configs
- Coverage thresholds: 80% lines, 75% branches, 80% functions

## Claude Code Skills

- `/story-writer <prd-path>` — generate stories from a PRD
- `/review-stories` — review generated stories
- `/push-to-jira` — push approved stories to JIRA

Skill definitions in `.claude/skills/story-writer/` and `.claude/skills/story-reviewer/`.
