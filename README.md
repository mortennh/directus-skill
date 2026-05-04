# directus-skill

A collection of agent skills for working with [Directus](https://directus.io) — originally from [gumpen-app/directapp](https://github.com/gumpen-app/directapp), maintained here as `monh/directus-skill`.

## Skills

| Skill | Description |
|---|---|
| [Directus Backend Architecture](./skills/directus-backend-architecture/) | Custom API endpoints, hooks, flows, service layers, DB operations, auth, and performance |
| [Directus Development Workflow](./skills/directus-development-workflow/) | Project scaffolding, TypeScript, testing, Docker, CI/CD, and multi-environment management |
| [Directus UI Extensions Mastery](./skills/directus-ui-extensions-mastery/) | Production-ready Vue 3 panels, interfaces, displays, and layouts |
| [Directus AI Assistant Integration](./skills/directus-ai-assistant-integration/) | AI chat interfaces, RAG, embeddings, content generation, and moderation |

## Installation

```bash
npx skills add monh/directus-skill
```

Install a specific skill:

```bash
npx skills add monh/directus-skill --skill 'Directus Backend Architecture'
npx skills add monh/directus-skill --skill 'Directus Development Workflow'
npx skills add monh/directus-skill --skill 'Directus UI Extensions Mastery'
npx skills add monh/directus-skill --skill 'Directus AI Assistant Integration'
```

Install for a specific agent (e.g. Claude Code):

```bash
npx skills add monh/directus-skill -a claude-code
```
