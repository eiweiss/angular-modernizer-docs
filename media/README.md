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

The platform exposes **41 MCP tools** to AI assistants across six categories.

**Analysis (10 tools):**

1. `analyze-project` - Framework detection and project structure analysis
2. `scan-all` - Run all scan plugins (architecture, SOLID, Angular best practices)
3. `scan-architecture` - Architecture violations (circular dependencies, layer violations, god objects)
4. `scan-solid` - SOLID principle violations (DIP, SRP, OCP, LSP, ISP)
5. `scan-angular` - Angular best practices (OnPush, RxJS optimization, lifecycle hooks)
6. `analyze-core-to-shared` - Core-to-shared dependency violations with impact assessment
7. `analyze-parser-performance` - Parser performance metrics from transformation feedback
8. `extract-libraries` - Library extraction candidate detection
9. `learn-patterns` - Pattern discovery across a codebase (recurring patterns, anti-patterns, transformation opportunities) with a persistent knowledge base and style-aware scoring
10. `compare-health` - Full scan compared with the previous run: new, fixed and unchanged violations per rule and file; stores the run as snapshot in `.angular-modernizer-cache/health/last-run.json`

**Planning and Transformation (3 tools):** 11. `plan-migration` - Multi-step migration strategy planning with constraints 12. `transform-code` - Atomic code transformations (26 transformation types); batch mode over several files via `files`, with rollback strategy 13. `orchestrate-migration` - End-to-end migration orchestration with checkpoint/resume

**Build and Validation (9 tools):** 14. `run-angular-cli` - Angular CLI command execution 15. `run-build` - Build system detection and execution (Angular CLI, Nx, Vite, Webpack) 16. `run-schematic` - Angular schematic execution for code generation and migrations 17. `validate-before-transform` - Pre-transformation validation (syntax, dependencies, types, schema) 18. `validate-transformation` - Post-transformation validation (build, tests, lint) 19. `check-incremental-compatibility` - Incremental build compatibility assessment 20. `analyze-build-impact` - Pre-transformation blast radius analysis 21. `validate-project-compatibility` - Parser compatibility validation 22. `get-operation-progress` - Deprecated: status of an operation started with the deprecated `streaming` parameter (native MCP progress and cancellation replace it)

**Rollback and Learning (4 tools):** 23. `rollback-changes` - Multi-strategy rollback system (auto, git-stash, file-backup) 24. `collect-feedback` - Transformation feedback collection for AI learning 25. `review-feedback` - Feedback analysis with success/failure pattern detection 26. `suggest-improvements` - Rule adjustment suggestions based on feedback

**Bug Investigation (14 tools):** 27. `parse-stack-trace` - Parse Node.js / Angular / Zone.js / browser V8 stack traces into structured frames with project-file classification 28. `find-angular-symbol` - Find Angular-decorated classes by CSS selector, attribute selector, pipe name, or class name; supports project-specific component decorators (`parser.componentDecorators`) 29. `resolve-type` - Resolve the TypeScript type at a file:line:col position; returns `isNullable`, `isAsync`, generic params, and declaration site 30. `get-call-graph` - Build a caller/callee tree for any class method; configurable direction (`up` / `down` / `both`) and depth 31. `find-usages` - Find all references to a named symbol across the project; filterable by `read` / `write` / `call` usage type 32. `search-codebase` - Search source files by literal text, regex pattern, or TypeScript named declaration (`symbol` mode) 33. `list-imports` - Imports of a file with where each one resolves, including path aliases 34. `inspect-template` - Structure of a component template: elements with bindings, matched project components/directives, pipes, control-flow depth, i18n keys 35. `find-component-hosts` - Every template host of a component or directive with bound, unbound and missing required inputs/outputs per host 36. `get-bootstrap-execution` - What runs at startup (initializers, bootstrap component, interceptors, provider factories) through calls and DI, and where a watched expression such as `AppConfig.settings` is read or written 37. `inspect-form-group` - Reactive Forms of a file as structure (controls, validators, initial state) with every enable/disable/validator change per control, ancestor cascades and unknown control paths 38. `trace-output-flow` - Payload flow of an `@Output` across the template boundary: emit sites with value, every host binding with its handler, what the handler method does with `$event` (writes, calls followed into callees, re-emits, conditions) and the hosts that do not bind it 39. `inspect-i18n` - Translation keys of a project: where a key is defined per language (file, line, value) and customer overlay, where templates and code use it, in which languages it is missing, which used keys are defined nowhere and which defined keys have no static usage; sources found without config (setTranslation, TranslateHttpLoader, TranslocoLoader, `assets/i18n`) 40. `verify-citations` - Check the `path:line` citations of Markdown plans against the current code: per citation ok, unverified, moved (with the new lines), anchor-missing, out-of-range, missing-file (with similar files) or ambiguous; run before executing a plan and after writing one

**Server (1 tool):** 41. `server-status` - The running server itself: memory, uptime, cached ts-morph projects, response sizes, and whether a package `dist/` was rebuilt after the server started (then reconnect to load the new build)

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
AI Assistant → MCP Adapter (41 tools) → Core Kernel → Plugin Layer → Public API → ts-morph AST
```

### Packages

| Package                             | Purpose                                                       |
| ----------------------------------- | ------------------------------------------------------------- |
| `@angular-modernizer/core`          | Kernel, orchestration, streaming, validation, parsers, config |
| `@angular-modernizer/api`           | Analysis/transformation tools                                 |
| `@angular-modernizer/plugin-system` | Plugin contracts                                              |
| `@angular-modernizer/adapter-mcp`   | 41 MCP tools                                                  |
| `@angular-modernizer/worker`        | Worker thread parallelism                                     |
| `@angular-modernizer/orchestration` | High-level orchestration                                      |
| `@angular-modernizer/claude-plugin` | Claude Code plugin installer                                  |

### Analysis Plugins

- **`@angular-modernizer/plugin-analyzer`** - Project analysis: framework detection (one rule)
- **`@angular-modernizer/plugin-architecture`** - Architecture rules: 17 rules (circular dependencies, layer violations, god objects, import direction, inheritance patterns)
- **`@angular-modernizer/plugin-solid`** - SOLID principle violations: 7 rules (DIP, SRP, OCP, LSP, ISP)
- **`@angular-modernizer/plugin-angular`** - Angular best practices: 28 rules (OnPush, RxJS, lifecycle hooks, async pipe, component complexity, presentational patterns)

**Total: 52 analysis rules across 3 rule plugins** (plugin-analyzer adds the framework detector)

### Transformation Plugins

- **`@angular-modernizer/plugin-standalone`** - Standalone component migration
- **`@angular-modernizer/plugin-angular`** - 25 transformation types including constructor-to-inject, interface-extraction, form-modernization, facade-pattern, inheritance-to-composition, promise-to-observable cluster, subscription-transform, sequential-await-to-forkjoin, library-extraction, and more

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
    plugin-solid/            SOLID principle analysis (7 rules)
    plugin-angular/          Angular-specific rules (28 rules, 25 transforms)
    plugin-architecture/     Architecture and code quality (17 rules)
    worker/                  Worker thread parallelism
    orchestration/           High-level migration orchestration
    claude-plugin/           Claude Code plugin installer

  pnpm-workspace.yaml        Workspace configuration
  tsconfig.base.json         Shared TypeScript config
  package.json               Root configuration
```

## Documentation

- [Developer Usage](DEVELOPER_USAGE.md) - Using the platform programmatically
- [Architecture Overview](docs/architecture/overview.md) - System architecture and package structure
- [MCP Integration](docs/guides/mcp-integration.md) - How to connect AI assistants to the platform
- [Configuration](docs/CONFIGURATION.md) - Full `.angular-modernizer.json` reference
- [API Reference](packages/api/README.md) - Complete API documentation
- [Plugin Development](packages/plugin-standalone/README.md) - Building custom plugins
- [Changelog](CHANGELOG.md) - Recent changes and fixes

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
# Run all tests (1666+ passing)
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

**21 agents across five workflows:**

| Workflow | Agents |
|-|-|
| Migration | `ngm-init-project`, `ngm-health-check`, `ngm-architecture-scanner`, `ngm-migration-planner`, `ngm-migration-blueprint`, `ngm-build-guard`, `ngm-safe-transformer`, `ngm-batch-modernizer`, `ngm-rxjs-modernizer`, `ngm-service-bag-splitter`, `ngm-library-extractor`, `ngm-feedback-analyst` |
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
    "mode": "incremental",
    "timeout": 300000,
    "enforcementLevel": "warning"
  },
  "pathAliases": {
    "enabled": true,
    "tsconfigPath": "./tsconfig.json"
  },
  "parser": {
    "strategy": "auto"
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

- `build.mode`: `"fast"`, `"full"`, or `"incremental"` (10-30s with caching)
- `parser.strategy`: `"auto"` (recommended), `"standard"`, `"enhanced"`, or `"custom"`
- `rules`: Per-rule configuration overrides (thresholds, target lists, exclusions)
- `layers`: Layer topology for `architecture:layer-boundary-violation` detection

## Performance

- Small projects (under 100 files): Promise mode, 2-4x speedup
- Medium projects (100-500 files): Adaptive mode selection
- Large projects (500+ files): Worker threads, 2-4x to 10x speedup
- Incremental build validation: 2-4x faster (measured: 35s vs 2-4min full build)
- Parser optimization: 50% memory reduction, 10x faster path resolution
- Cache hit rate: over 90% on incremental analysis runs

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
