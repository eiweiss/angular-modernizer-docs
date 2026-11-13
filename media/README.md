# Angular Modernization Platform

> An AI-powered, plugin-based platform for analyzing, modernizing, and transforming Angular codebases

[![TypeScript](https://img.shields.io/badge/TypeScript-5.x-blue.svg)](https://www.typescriptlang.org/)
[![Node.js](https://img.shields.io/badge/Node.js-20.19%2B-green.svg)](https://nodejs.org/)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

**Note:** This is an analysis and transformation tool for Angular applications, not an Angular application itself.

## Overview

The Angular Modernization Platform is an extensible system for analyzing and transforming Angular codebases. It exposes its capabilities to AI assistants via MCP (Model Context Protocol), enabling automated analysis, migration planning, and code transformation at scale.

**The AI is the architect, our code is the universal toolbox.**

## Capabilities

- **AI-First Design**: Built specifically for AI assistants via MCP
- **Plugin Architecture**: Extensible system for custom analysis and transformations
- **API-Driven Design**: Stable public API for AST manipulation and analysis
- **Test-First**: In-memory file system for fast, isolated tests
- **Architecture Analysis**: Comprehensive cross-file dependency analysis and violation detection
- **Type-Safe**: Strict TypeScript with no `any` types

## Quick Start

### For AI Assistants (MCP Integration)

The platform exposes **<!-- facts:tools -->42<!-- /facts --> MCP tools** to AI assistants: scans and health
comparison, migration planning, code transformation with backup and rollback, build and validation, feedback, bug
investigation (stack traces, usages, call graphs, types, templates, forms, i18n, plan citations) and server status.
The generated tool table with each tool's access and purpose is in the
[adapter-mcp README](packages/adapter-mcp/README.md#available-tools); arguments and results are in each tool's input
and output schema (`tools/list`).

**Parallelism and Orchestration:**

- Intelligent automatic mode selection (Promise-based vs worker threads) based on project complexity
- Real-time memory monitoring with projection-based abort to prevent crashes
- Memory-aware worker scaling (2-8 workers based on system RAM)
- Automatic fallback from Promise to Worker mode under memory pressure
- `analyze-project`, `scan-architecture` and `transform-code` select the mode automatically by default (`orchestration.autoSelectMode`)

### For Developers (Direct API Usage)

```typescript
import { Kernel, RealFileSystemAdapter } from '@angular-modernizer/core';
import { StandalonePlugin } from '@angular-modernizer/plugin-standalone';
import { SolidPlugin } from '@angular-modernizer/plugin-solid';
import { AngularPlugin } from '@angular-modernizer/plugin-angular';
import { ArchitecturePlugin } from '@angular-modernizer/plugin-architecture';
import { createPublicApi } from '@angular-modernizer/api';
import { ContextFactory } from '@angular-modernizer/plugin-system';

const kernel = new Kernel({
  tsConfigPath: './tsconfig.json',
  fileSystem: new RealFileSystemAdapter(),
  plugins: [
    new StandalonePlugin(),
    new SolidPlugin(),
    new AngularPlugin(),
    new ArchitecturePlugin(),
  ],
  cacheConfig: {
    enabled: true,
    cacheDir: '.angular-modernizer-cache',
    maxSize: 100 * 1024 * 1024,
    maxAge: 24 * 60 * 60 * 1000,
    compression: true,
    version: '1.0.0',
  },
});

await kernel.initialize();

const project = kernel.getProject();
const api = createPublicApi(project);
const plugin = kernel.getPlugin('@angular-modernizer/plugin-standalone');
const rule = plugin.getTransformRules()[0];

const sourceFile = project.addSourceFileAtPath('app.component.ts');
const context = ContextFactory.createTransformContext({
  sourceFile,
  project,
  api,
  config: {},
});

const result = await rule.transform(context);
if (result.modified) {
  await sourceFile.save();
}
```

### Fluent API

```typescript
import { FluentKernel } from '@angular-modernizer/core';

const result = await FluentKernel.create(kernel)
  .analyze(['src/**/*.ts'], rules)
  .filter((v) => v.severity === 'high')
  .transform('standalone-migration')
  .buildCheck()
  .execute();
```

## Architecture

```
AI Assistant → MCP Adapter (MCP tools) → Core Kernel → Plugin Layer → Public API → ts-morph AST
```

### Packages

| Package                             | Purpose                                                       |
| ----------------------------------- | ------------------------------------------------------------- |
| `@angular-modernizer/core`          | Kernel, orchestration, streaming, validation, parsers, config |
| `@angular-modernizer/api`           | Analysis/transformation tools                                 |
| `@angular-modernizer/plugin-system` | Plugin contracts                                              |
| `@angular-modernizer/adapter-mcp`   | MCP server with all MCP tools                                 |
| `@angular-modernizer/worker`        | Worker thread parallelism                                     |
| `@angular-modernizer/orchestration` | High-level orchestration                                      |
| `@angular-modernizer/claude-plugin` | Claude Code plugin installer                                  |

### Analysis Plugins

- **`@angular-modernizer/plugin-analyzer`** - Project analysis: framework detection (one rule)
- **`@angular-modernizer/plugin-architecture`** - Architecture rules: <!-- facts:analysis-rules-architecture -->17<!-- /facts --> rules (circular dependencies, layer violations, god objects, import direction, inheritance patterns)
- **`@angular-modernizer/plugin-solid`** - SOLID principle violations: <!-- facts:analysis-rules-solid -->7<!-- /facts --> rules (DIP, SRP, OCP, LSP, ISP)
- **`@angular-modernizer/plugin-angular`** - Angular best practices: <!-- facts:analysis-rules-angular -->28<!-- /facts --> rules (OnPush, RxJS, lifecycle hooks, async pipe, component complexity, presentational patterns)

**Total: <!-- facts:analysis-rules -->52<!-- /facts --> analysis rules across 3 rule plugins** (plugin-analyzer adds the framework detector)

### Transformation Plugins

- **`@angular-modernizer/plugin-standalone`** - Standalone component migration
- **`@angular-modernizer/plugin-angular`** - most transformation types, including constructor-to-inject, interface-extraction, form-modernization, facade-pattern, inheritance-to-composition, promise-to-observable cluster, subscription-transform, sequential-await-to-forkjoin, library-extraction, and more

### Design Principles

1. **AI-First Architecture**: Every component designed for AI consumption and orchestration
2. **API-Driven Plugins**: Plugins consume stable APIs, not low-level AST manipulation
3. **Context-Based DI**: Dependencies injected via context objects, not constructors
4. **Separation of Concerns**: Clear boundaries between infrastructure, API, and business logic
5. **Testability First**: In-memory file system for fast, isolated testing

## Project Structure

```
tsanalyzer/
  packages/
    core/                    Kernel and plugin infrastructure
    plugin-system/           Plugin interfaces and contexts
    api/                     Public API (analysis tools)
    plugin-standalone/       Standalone component migration
    adapter-mcp/             MCP server for AI integration
    plugin-analyzer/         Project analysis rules
    plugin-solid/            SOLID principle analysis
    plugin-angular/          Angular-specific analysis and transform rules
    plugin-architecture/     Architecture and code quality
    worker/                  Worker thread parallelism
    orchestration/           High-level migration orchestration
    claude-plugin/           Claude Code plugin installer

  pnpm-workspace.yaml        Workspace configuration
  tsconfig.base.json         Shared TypeScript config
  package.json               Root configuration
```

## Documentation

- [Core package](packages/core/README.md) - Using the platform programmatically (kernel, rules, cache)
- [Architecture Overview](docs/architecture/overview.md) - System architecture and package structure
- [MCP server](packages/adapter-mcp/README.md) - How to connect AI assistants to the platform, tool table
- [Configuration](docs/CONFIGURATION.md) - Full `.angular-modernizer.json` reference
- [API Reference](packages/api/README.md) - Complete API documentation
- [Plugin Development](packages/plugin-system/README.md) - Rule interfaces, contexts and the transformation type gate
- [Testing](docs/testing/strategy.md) - Testing strategy, ESLint rules, strict TypeScript
- [All docs](docs/README.md) - Index of the guides and architecture pages
- Changelogs: one per package (`packages/*/CHANGELOG.md`, generated by Changesets), for example [core](packages/core/CHANGELOG.md) and [adapter-mcp](packages/adapter-mcp/CHANGELOG.md)

## Development

### Prerequisites

Node.js `^20.19.0 || ^22.12.0 || >=24.0.0` (the range of Angular 21) and pnpm.

```bash
pnpm install
pnpm run build
```

### Build Troubleshooting

```bash
# Clean all build artifacts
find packages -name "tsconfig.tsbuildinfo" -delete
find packages -type d -name "dist" -exec rm -rf {} + 2>/dev/null

# Rebuild everything
pnpm run build
```

This project uses TypeScript composite project references. All packages must use `tsc --build` in their build scripts.

### Testing

```bash
# Run all tests
pnpm run test

# Single package
pnpm --filter @angular-modernizer/core test

# Watch mode
pnpm test:watch

# Performance benchmarks
pnpm --filter @angular-modernizer/core bench:all

# Real Angular CLI build tests
pnpm --filter @angular-modernizer/adapter-mcp test:real-build
```

### Code Quality

```bash
pnpm run lint        # Type-aware ESLint
pnpm run type-check  # TypeScript type checking
pnpm run format      # Prettier formatting
```

### MCP Server

```bash
# Start MCP server (stdio)
pnpm run mcp:start

# Register with Claude Code
claude mcp add angular-modernizer npx @angular-modernizer/adapter-mcp

# Local dev registration
claude mcp add angular-modernizer -- node /absolute/path/to/packages/adapter-mcp/dist/cli.js
```

### Claude Code Plugin

```bash
# Install agents and skills into ~/.claude/
npx @angular-modernizer/claude-plugin
```

This installs agent definitions and skill files to both `~/.claude/` (global) and `<cwd>/.claude/` (project-local).

**<!-- facts:agents -->22<!-- /facts --> agents across five workflows:**

| Workflow | Agents |
|-|-|
| Migration | `ngm-init-project`, `ngm-health-check`, `ngm-refresh`, `ngm-architecture-scanner`, `ngm-migration-planner`, `ngm-migration-blueprint`, `ngm-build-guard`, `ngm-safe-transformer`, `ngm-batch-modernizer`, `ngm-rxjs-modernizer`, `ngm-service-bag-splitter`, `ngm-library-extractor`, `ngm-feedback-analyst` |
| Bug fix | `ngm-reproduce-bug`, `ngm-bug-investigator`, `ngm-bug-blueprint`, `ngm-bug-fixer` |
| Feature | `ngm-feature-planner`, `ngm-feature-implementer` |
| Testing | `ngm-baseline-tester`, `ngm-visual-comparator` |
| Design | `ngm-architect` |

**Bug fix workflow:**
```
ngm-reproduce-bug (browser evidence)
  → ngm-bug-blueprint (investigate + write plan.md and investigation.md to the plan folder)
  → ngm-bug-fixer (apply minimal fix with safety envelope)
```

**Feature workflow:**
```
ngm-architect (design blueprint)
  → ngm-feature-planner (investigate codebase + write .feature-plan/ to disk)
  → ngm-feature-implementer (ng generate + edit + validate)
```

**Audience levels** — pass to any plan-writing agent or skill to control report detail:
- `junior` — pattern explanations, annotated examples, glossary
- `regular` — standard detail (default)
- `senior` — decisions and reasoning only
- `architect` — structure, risks, and layer implications only

## Configuration

The platform supports project-specific configuration via `.angular-modernizer.json`:

```json
{
  "build": {
    "command": "ng build",
    "timeout": 300000
  },
  "parser": {
    "componentDecorators": ["PageComponent"]
  },
  "rules": {
    "angular:component-wrapper-instantiation": {
      "classNamePatterns": [".*Wrapper$", ".*Adapter$"]
    }
  },
  "layers": [
    {
      "name": "core",
      "alias": "@core",
      "paths": ["src/app/core/**"],
      "canImportFrom": []
    },
    {
      "name": "shared",
      "alias": "@shared",
      "paths": ["src/app/shared/**"],
      "canImportFrom": ["@core"]
    },
    {
      "name": "features",
      "alias": "@features",
      "paths": ["src/app/features/**"],
      "canImportFrom": ["@core", "@shared"]
    }
  ]
}
```

**Key options:**

- `build.command`: the only required key; the build tools always run it as a full build
- `parser.componentDecorators`: project decorators that mark a class as an Angular component in addition to `Component` (the parser itself is chosen automatically)
- `rules`: Per-rule configuration overrides (thresholds, target lists, exclusions)
- `layers`: Layer topology for `architecture:layer-boundary-violation` detection
- Deprecated, no effect (still accepted, the loader warns, removal in 2.0): `build.mode`, `build.enforcementLevel`, `build.fastModeCommands`, `build.incremental`, the `validation` and `pathAliases` sections, and every `parser` key except `componentDecorators`. Full reference: [docs/CONFIGURATION.md](docs/CONFIGURATION.md)

## Performance

- The MCP scans run their analysis in a scan worker thread, so the server keeps answering during a scan
- The WorkerPool analysis path of core (`ParallelExecutor`, mode selection) is deprecated since 1.1 and will be removed in 2.0

## Orchestration API

The analysis, planning and transform tools (`analyze-project`, `scan-all`, `scan-architecture`, `scan-solid`, `scan-angular`, `analyze-core-to-shared`, `learn-patterns`, `plan-migration`, `transform-code`) take an optional `orchestration` object. The accepted keys differ per tool (see the tool's input schema; unknown keys are rejected). `analyze-project` accepts, for example:

```typescript
{
  parallel?: boolean;
  incremental?: boolean;
  baseBranch?: string;
  maxWorkers?: number;
  enableProgress?: boolean;
  useWorkerThreads?: boolean;   // default: true
  timeout?: number;             // default: 30000
  autoSelectMode?: boolean;     // default: true
  enableMonitoring?: boolean;   // default: true
  fallbackToWorkers?: boolean;  // default: true
}
```

Zero-configuration usage is supported — intelligent mode selection handles execution strategy automatically:

```typescript
await callTool('scan-architecture', {
  rootPath: '/workspace/angular-app',
  // orchestration handled automatically
});
```

## Contributing

To add new capabilities:

1. **For Analysis**: Create a new plugin in `packages/plugin-*` implementing `AnalysisRule`
2. **For Transformations**: Implement `TransformRule` with access to the Public API
3. **For AI Tools**: Add new tool definitions in `packages/adapter-mcp`
4. **Write Tests**: Every component needs comprehensive test coverage
5. **Update Docs**: Keep the READMEs and `docs/` current

### Development Workflow

```bash
# 1. Create feature branch
git checkout -b feature/new-analysis-rule

# 2. Implement and test
pnpm --filter @your-package test

# 3. Build and verify
pnpm run build

# 4. Update documentation
# 5. Create PR
```

Coverage reports are generated during testing but excluded from version control. Use `pnpm run test:coverage` to generate locally.

## License

MIT

## Built With

- [ts-morph](https://ts-morph.com/) - TypeScript AST manipulation
- [TypeScript](https://www.typescriptlang.org/) - Language and compiler
- [pnpm](https://pnpm.io/) - Fast, efficient package manager
- [Jest](https://jestjs.io/) - Testing framework

---

**Built with care and architectural precision**
