# AI Story Writer Agent

Enterprise tool that converts refined Product Requirements Documents (PRDs) into JIRA-ready Scrum artifacts — Epics, User Stories, and Sub-Tasks — with a review-and-approve workflow before JIRA creation.

Part of the AI PDLC pipeline:

```
PRD Agent → Discovery Agent → AI Story Writer Agent → JIRA (PI-ready)
```

## Goal

Reduce PO manual effort by 50–70% for story writing, standardize JIRA items across the organization, and produce PI-planning-ready work items.

## Three Implementations

| Package | Description | Deployment |
|---------|-------------|------------|
| `packages/core` | Shared library: types, schemas, prompts, engine, mappers | npm package |
| `packages/skill` | Claude Code skill with MCP integrations | Org catalog (VS Code) |
| `packages/service` | Standalone Express + Anthropic SDK service | GCP Cloud Run |
| `packages/hybrid` | Core + both wrappers | Catalog + Cloud Run |

## Quick Start

```bash
# Install dependencies
npm install

# Build all packages
npm run build

# Run tests
npm test

# Start the service (dev mode)
npm run -w packages/service dev
```

## Configuration

Organization-specific configuration lives in `config/org/`. See `config/org/default.json` for the full schema. Key settings:

- **JIRA field mappings** — project key, issue type IDs, custom fields
- **Story defaults** — labels, components, Definition of Done template
- **Prompt overrides** — terminology, example stories, system prompt addendum
- **Workflow** — approval required, auto-sprint assignment, notifications

## MCP Setup (for Claude Code Skill)

Set environment variables for MCP authentication:

```
JIRA_API_TOKEN=<your-jira-pat>
JIRA_MCP_URL=<your-jira-mcp-endpoint>
CONFLUENCE_API_TOKEN=<your-confluence-pat>
CONFLUENCE_MCP_URL=<your-confluence-mcp-endpoint>
GITHUB_PAT=<your-github-pat>
```

## Documentation

- [Architecture Overview](docs/architecture.md)
- [Developer Setup](docs/setup.md)
- [Skill Usage Guide](docs/skill-usage.md)
- [REST API Reference](docs/api-reference.md)
- [JIRA Field Mapping](docs/jira-field-mapping.md)
- [Prompt Engineering](docs/prompt-engineering.md)

## Tech Stack

- **Runtime**: Node.js 22, TypeScript 5.7, ESM
- **LLM**: Claude (via Anthropic SDK or MCP)
- **Validation**: Zod schemas
- **Templates**: Handlebars
- **API**: Express
- **Cloud**: GCP (Cloud Run, Firestore, Pub/Sub, Secret Manager)
- **IaC**: Pulumi
- **Testing**: Vitest
- **CI/CD**: GitHub Actions

## License

UNLICENSED — Internal enterprise tool.
