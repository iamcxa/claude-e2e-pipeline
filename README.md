# e2e-pipeline

Browser E2E testing pipeline for Claude Code. Maps UI elements, runs test flows, and walks through apps interactively — all with context-isolating subagents that keep browser data out of your main conversation.

## Prerequisites

- [agent-browser](https://github.com/anthropics/agent-browser) CLI installed globally

## Quick Start

### 1. Map your app's UI

```
/e2e-map
```

Creates a YAML mapping of pages, elements, and selectors in `.claude/e2e/mappings/<app>.yaml`.

### 2. Run a test flow

```
/e2e-test <flow-name>
```

Executes a flow file from `.claude/e2e/flows/` against the mapped UI.

### 3. Walk through interactively

```
/e2e-walkthrough
```

Human-guided browser exploration with trace recording and auto-generated flow output.

## Commands

| Command | What it does |
|---------|-------------|
| `/e2e-map` | Create or update UI element mappings |
| `/e2e-test <flow>` | Run a specific E2E test flow |
| `/e2e-test --tag smoke` | Run all flows tagged with `smoke` |
| `/e2e-test --suite <name>` | Run a named test suite |
| `/e2e-walkthrough` | Interactive browser walkthrough |
| `/e2e-walkthrough --smoke` | Auto-generated smoke walkthrough from mapping |
| `/e2e-dispatch` | Unified entry point (routes to the right skill) |
| `/e2e-skill-ops` | Debug, maintain, or evaluate E2E skills |

## File Structure

```
.claude/e2e/
├── mappings/          # UI element mappings (YAML)
│   ├── my-app.yaml
│   └── my-admin.yaml
├── flows/             # Test flow definitions (YAML)
│   └── login-flow.yaml
└── suites/            # Test suite definitions (YAML)
    └── smoke.yaml
```

## Architecture

Skills run in main context as thin orchestrators. Heavy browser work is delegated to subagents:

| Skill | Agent | Role |
|-------|-------|------|
| `e2e-map` | `e2e-mapper` | Explore pages, extract selectors |
| `e2e-test` | `e2e-test-runner` | Execute flow steps, collect results |
| `e2e-walkthrough` | `e2e-trace-analyzer` | Parse trace.zip for API/console errors |

This keeps browser screenshots, accessibility trees, and trace data out of the main conversation context.

## Multi-Site Testing

Map multiple apps, then test across them:

```
/e2e-map                              # map each app separately
/e2e-test --all-sites                  # run flows on all mapped sites
/e2e-walkthrough --sites admin,portal  # cross-site walkthrough
```

## Troubleshooting

| Issue | Fix |
|-------|-----|
| `agent-browser not found` | Install globally: see [agent-browser](https://github.com/anthropics/agent-browser) |
| Auth expired during test | Delete `~/.agent-browser/<app>/` and re-login |
| Selectors stale | Re-run `/e2e-map --page <page>` to refresh |
| Flow uses v1 format | Migrate: `app:` → `mapping:`, step `name:` → `id:` |

For deeper diagnostics: `/e2e-skill-ops --debug`
