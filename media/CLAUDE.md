# @angular-modernizer/core

Domain-agnostic kernel and infrastructure. Contains orchestration, streaming, caching, config loading, parsers, and file system abstractions. Plugins must not depend on this package directly — they use `plugin-system` contracts. Tools in `adapter-mcp` depend on this package.

Usage: `README.md`. API reference: TypeDoc of the exports (`pnpm run docs:serve`).

## Commands

```bash
pnpm --filter @angular-modernizer/core build
pnpm --filter @angular-modernizer/core test
pnpm --filter @angular-modernizer/core bench:parser    # Parser benchmarks
pnpm --filter @angular-modernizer/core bench:all       # All benchmarks
```

## Key Classes

| Class | File | Purpose |
|-------|------|---------|
| `Kernel` | `kernel.ts` | Central orchestration engine; initializes plugins, exposes ts-morph Project |
| `createKernel` | `kernel-factory.ts` | Convenience factory: tsconfig discovery → Kernel construction → `initialize()` → `createPublicApi()` in one call; returns `{ kernel, api, config, tsConfigPath }`; throws if no tsconfig found; `findTsConfigPath(rootPath)` exported separately for discovery-only use |
| `FluentKernel` | `fluent/fluent-kernel.ts` | Chainable pipeline builder (analyze → filter → transform → buildCheck) |
| `StreamingKernelWrapper` | `streaming/streaming-kernel.ts` | Async generator wrapper for real-time progress |
| `ParserOrchestrator` | `parsers/parser-orchestrator.ts` | Selects standard vs enhanced parser based on project context |
| `ConfigLoader` | `config/config-loader.ts` | Loads and validates `.angular-modernizer.json` |
| `FeedbackCollector` | `feedback/feedback-collector.ts` | Privacy-first AI learning data collection; pipeline: extract learnings → validate → privacy filter → privacy audit → add metadata → store; confidence thresholds: 0.8 general, 0.9 failure, 0.7 degradation |
| `analyzePatterns` | `feedback/feedback-analyzer.ts` | Detects success/failure/quality-trend patterns in `TransformationFeedback[]`; params: `(feedback, analysisType: 'success-patterns'|'failure-patterns'|'quality-trends'|'all', minConfidence)`; returns `FeedbackInsight[]` sorted by confidence descending |
| `calculatePerformance` / `generateSuggestions` / `findCommonFailurePattern` | `feedback/suggestion-generator.ts` | `calculatePerformance` returns `FeedbackPerformance` `{ successRate, avgDuration, qualityImprovement }`; `generateSuggestions` yields `FeedbackSuggestion[]` with type/description/reasoning/confidence/requiresApproval; `findCommonFailurePattern` returns `string | null` |
| `IncrementalAnalyzer` | `orchestration/incremental-analyzer.ts` | Changed-file detection via file hashes; dependency-aware scope expansion; returns changeStats (cacheHitRate, skippedFiles, dependentFiles). `getBuildScope` enables template edges (third constructor argument `dependencyGraph` overrides): `templateDependentFiles` / `templateEdges`, counted for the strategy; `templateOnlyFiles` add no template hosts |
| `ParallelExecutor` | `orchestration/parallel-executor.ts` | Deprecated since 1.1 with the whole WorkerPool path (removal in 2.0). Dual-mode parallel analysis (Promise-based or worker threads); Phase 7b auto-mode-selection |
| `ParallelismModeSelector` | `orchestration/mode-selector.ts` | Deprecated since 1.1 with the whole WorkerPool path (removal in 2.0). Complexity scorer (file count 30%, LOC 35%, outliers 20%, memory 15%); selects Promise vs worker mode |
| `ProgressReporter` | `orchestration/progress-reporter.ts` | Real-time file/rule progress events with ETA and cancellation (extends EventEmitter) |
| `PromiseModeMonitor` | `orchestration/promise-monitor.ts` | Deprecated since 1.1 with the whole WorkerPool path (removal in 2.0). Heap monitoring every 5s in Promise mode; warning at 70%, abort at 85% projected peak |
| `ResourceManager` | `orchestration/resource-manager.ts` | Part of the deprecated WorkerPool path. Worker resource tracking; per-worker heap Map; crash budget (3/60s); canProceed() guard |
| `WorkerPool` | `orchestration/worker-pool.ts` | Deprecated since 1.1 with the whole WorkerPool path (removal in 2.0). Node.js worker_threads pool; task queue; round-robin dispatch; health monitoring; timeout per task. Workers start with `stdout: true` and their stdout is piped to stderr: by default Node forwards worker stdout into the process stdout, the JSON-RPC channel of the MCP server, and the `stdio-guard` of adapter-mcp only reroutes the main thread console |
| `DependencyGraphBuilder` | `analysis/dependency-graph-builder.ts` | Transitive dependency analysis; `getImpact` name-aware and type-only-aware. Past the first importer only exports whose type surface names an affected binding propagate (`analysis/export-surface.ts`: heritage, annotations, initializers without annotation, bodies without return type, `@NgModule` `exports`; not decorators or bodies with a stated return type; `export =` and augmentations count as all exports), so a pipe no longer reaches every importer of every module that imports its module (large app: yesNo.pipe 1689 -> 291 value dependents); side-effect imports keep all exports. Opt-in template edges (`templateEdges: true`, api `collectTemplateEdges`, lazy on first use) in a separate store: never in `dependencies`/`dependents`, cycles or transitive dependents; `getImpact` adds `templateDependents` one hop (hosts outside the impact whose template uses an affected class), `getTemplateDependencies` serves scan-plan. Tests inject edges via `collectTemplateEdges` because `@angular/compiler` is mocked in core |
| `applySuppressionComments` / `filterSuppressed` / `isSuppressedAt` / `suppressionMarkers` | `analysis/suppression.ts` | One suppression mechanism for every rule: markers from `rules['<id>'].suppressionComments` (schema: every registered rule id accepts it, unknown ids rejected; no marker built in), applied after `rule.analyze` by every runner (adapter scan engine and rule runner, extract-libraries, both `analyzeProject`, `ParallelExecutor` and `IncrementalAnalyzer` with `rulesConfig` from `ParallelOptions`/`IncrementalOptions`). TypeScript: comment before the outermost node starting on the finding's line (same line or ending on the line above, above decorators), after it on its last line, or anywhere on the finding's line. `.html`: `<!-- marker -->` on the line or ending on the line above. Worker threads of the WorkerPool path get no `rules` (plugins there are not initialized either) |
| `InMemoryFileSystemAdapter` | `file-system.ts` | In-memory FS for tests |
| `RealFsTestHelper` | `testing/real-fs-test-helper.ts` | Real FS test utility; `RealFsTestHelper.create()` factory creates isolated temp dir; methods: `createFile(path, content)`, `cleanup()`; use in beforeEach/afterEach when BackupManager or ArtifactMigrator require real fs paths |
| `InheritanceAnalyzer` | `pattern-analysis/inheritance-analyzer.ts` | Detects class inheritance hierarchies and composition opportunities. Bases resolve through the type checker (`getBaseClass()`: generic bases, import aliases, same-named classes); library bases and bases that are no class declaration go to `skipped` with the reason (before, the `extends` text was looked up by name and they vanished silently). Used base methods: read through `this`/`super` in the derived class (called or not; exported `thisOrSuperMemberNames`, shared with plugin-architecture's `architecture:inheritance-analysis`, which also counts overrides and template uses); `analyzeClass(derived, { alsoUsed })` also counts callers through the derived type (references) and names a caller saw elsewhere (templates). Overall confidence = average group confidence (0.5 without a group) less the deductions, rounded to 0.001: a pure method reaches 0.95 (before 0.5 + 0.3 x average capped it at 0.785, below the rule default 0.8) |
| `InheritanceToCompositionTransformer` | `pattern-analysis/inheritance-to-composition-transformer.ts` | Deprecated (stays exported in 1.x): its output does not compile in general (removes `extends` without moving state, copies state into a singleton, absolute `import("...")` types). The `inheritance-to-composition` transformation uses plugin-angular's `InheritanceToCompositionOrchestrator` |
| `PatternKnowledgeBase` | `pattern-analysis/pattern-knowledge-base.ts` | Self-learning pattern persistence (PatternEntry, StyleProfile, LearningEvolution) |
| `StyleConventionDetector` | `pattern-analysis/style-convention-detector.ts` | Detects codebase style conventions (RxJS, DI, state management, naming); returns StyleDetectionResult |
| `ContextAwareScorer` | `pattern-analysis/context-aware-scorer.ts` | Scores patterns with style-aware confidence (ScoringWeights, ConfidenceBreakdown) |
| `ContextAwareRuleGenerator` | `pattern-analysis/context-aware-rule-generator.ts` | Generates analysis rules aligned with codebase style conventions; filters by StyleProfile alignment threshold; returns GeneratedRule with styleAlignment, confidence, examples |
| `ContextAwareRoadmapGenerator` | `pattern-analysis/context-aware-roadmap-generator.ts` | Generates phased migration roadmaps respecting detected style conventions; includes StyleConsistencyCheck and PreparationTask items |
| `StyleAwareMigrationRiskAssessor` | `pattern-analysis/style-aware-migration-risk-assessor.ts` | Scores migration step risk across technical, organizational, timeline, and style-deviation dimensions; returns MigrationRiskAssessment with RiskBreakdown and MitigationStrategy |
| `ProjectMetricsCollector` | `orchestration/mode-selector.ts` | Deprecated since 1.1 with the whole WorkerPool path (removal in 2.0). Collects project-level metrics via fs.stat() (file count, bytes, large-file outliers, estimated LOC); used by ParallelismModeSelector for auto-mode selection |
| `PathAliasResolver` | `analysis/path-alias-resolver.ts` | Resolves tsconfig.json path aliases (@shared/*, @domain/*, etc.) to prevent parser "Cannot find module" errors; options: baseDir, customAliases, normalizePaths, enabled, tsconfigPath, preserveAliases; resolve cache for repeated imports |
| `StandardParserStrategy` | `parsers/standard-parser-strategy.ts` | Baseline ts-morph parser; `getReliability()` 0.95; use for projects without external decorators or complex path aliases |
| `EnhancedParserStrategy` | `parsers/enhanced-parser-strategy.ts` | Preprocessing pipeline on top of ts-morph; handles external decorators and complex path aliases (no production caller, see below) |
| `ExternalDecoratorHandler` | `parsers/external-decorator-handler.ts` | Auto-detects custom decorators (e.g. `@RegisteredComponent`) not known to ts-morph; used by EnhancedParserStrategy |
| `SharedProjectManager` | `parsers/shared-project-manager.ts` | Singleton ts-morph Project shared across parser invocations; file lifecycle management; delegates to ts-morph |
| `ASTFalsePositiveDetector` | `analysis/ast-false-positive-detector.ts` | Works directly on ts-morph Diagnostic objects instead of regex patterns; used to filter false-positive parser errors |
| `transformInFileStyle` | `transformation/file-style.ts` | Runs a transform rule with `project.manipulationSettings` set to the code style of `context.sourceFile` (api `codeStyleOf`: the file's quotes and indentation, else `.prettierrc` / `.editorconfig`, else single quotes and 2 spaces) and restores them when the rule settles; used by `TransformExecutionEngine` (imperative, fluent and batch paths) and `TransformRuleExecutor`. Without it ts-morph writes four spaces and double quotes |
| `TransformationResultValidator` | `transformation/transformation-result-validator.ts` | 4 validation checks: (1) silent failure (modified=true + changeCount=0), (2) success with syntax errors, (3) high-confidence failure (modified=false + AI confidence >0.9), (4) slow transformation (>30s); confidence penalties: silent failure −70%, syntax errors −50%, high-confidence failure −40%, slow −30% |
| `SyntaxTypeDetector` | `streaming/helpers/syntax-type-detector.ts` | Maps source code lines to TypeScript syntax types via ts-morph AST; used by structured diff generation; methods: getSyntaxType(sourceFile, lineNumber), analyzeFile(sourceFile), clearCache(); caches per file path |
| `CacheManager` | `cache/cache-manager.ts` | AST and analysis result caching with file change detection and automatic invalidation; `ensureGitignore()` writes a relative `.angular-modernizer-cache/` entry (not an absolute path) to the project root `.gitignore`; handles both base (`/project/.angular-modernizer-cache`) and subdirectory (`/project/.angular-modernizer-cache/solid`) cacheDir values. Entries live at `<cacheDir>/(ast|analysis)/<name>.<hash>.(ast|analysis).json`, independent of the date (the former `YYYY/MM/DD/` layout looked entries up under the current date, so they were lost at midnight; old date folders are ignored, not deleted), and `maxAge` (24 h) applies. Size limit eviction, `clear()` and the size statistics consider only these own entries, so a manager on the base directory leaves other tools' caches and `health/last-run.json` alone |
| `CacheCoordinator` | `validation/cache-coordinator.ts` | Coordinates Angular build cache (.angular/cache, read-only) with analyzer cache (.angular-modernizer/cache); interfaces: AngularCacheInfo, AnalyzerCacheInfo, CacheClearRecommendation; NEVER modifies Angular cache. Its only consumer is the deprecated `IncrementalBuildValidator`; the scan caches are `.angular-modernizer-cache/<tool>/` (`CacheManager`) |
| `PipelineStage` | `fluent/pipeline-stage.ts` | Abstract base for fluent pipeline stages; concrete implementations: AnalyzeStage, FilterStage, TransformStage, BuildValidationStage, ValidationStage; each stage transforms PipelineContext; TransformStage delegates to kernel.transformFile() and records the original texts (`PipelineContext.originalTexts`), BuildValidationStage saves the files, builds via `kernel.getBuildValidator()` and restores the originals on failure with rollback |
| `executePipelineAsObservable` | `fluent/rxjs-integration.ts` | Optional RxJS integration for FluentKernel; emits PipelineEvent union (stage-start, stage-complete, stage-error, progress, complete, error); throws if RxJS not installed; check with isRxJSAvailable() |
| `levenshteinDistance` / `findSimilarRules` / `MatchableRule` | `utils/rule-matcher.ts` | Fuzzy rule ID matching for "did you mean?" suggestions; `MatchableRule: { id, name, description? }`; `findSimilarRules<T extends MatchableRule>(query, rules, maxResults?)` returns closest matches by edit distance |
| `processGroupOptions` / `terminateProcessTree` / `terminateOnAbort` | `utils/process-tree.ts` | Child processes as a tree: `processGroupOptions()` spawns in an own process group on POSIX (`detached`), `terminateProcessTree` sends SIGTERM to the group and SIGKILL after 5 s (both to the group even when the shell already exited; before, an exited shell skipped the SIGTERM) and writes one stderr line with pid and reason (`timeout` by default, plus one when the SIGKILL still finds the group), `terminateOnAbort` does it when an `AbortSignal` aborts (reason `cancelled`). Used by `BuildValidator`, `PostTransformValidator`, the adapter's run-build, run-angular-cli and run-schematic and plugin-angular's library-extraction (`ng generate library`): builds run through a shell or `npx`, a signal to the direct child left `ng build` running. Windows has no process groups: `killWindowsTree` runs `taskkill /pid <pid> /T /F` (`windowsHide`, falls back to `child.kill()` when taskkill cannot start). Known limit: once the shell has exited, taskkill finds no process under its pid and the build it left in the background keeps running (only a build command that backgrounds its own work, such as `start /b`; `cmd /c` waits for the build). On both platforms the pipes are closed `PIPE_RELEASE_MS` (1 s) after the child's exit, so callers waiting for `'close'` settle even when a descendant escaped the kill and still holds them (before, such a descendant kept the call open until it ended) |
| `resolveBuildCommand` | `validation/build-command.ts` | Build command of a project: `build.command` of `.angular-modernizer.json` (read as plain JSON, the loader default `ng build` does not count), else `npx ng build` / `npx nx build` / `npx vite build` / `npx webpack --mode=production` by build system, else `npm run build` when `package.json` has a build script, else `undefined`. `PostTransformValidator` uses it unless `buildCommand` is passed and the adapter's run-build always (its only path since W24); both run it with `projectBinPath(cwd)` (`<cwd>/node_modules/.bin` first on `PATH`). `PostTransformValidator` appends `buildArgs` through `appendBuildArguments` (quoted, after `--` for an npm script) and reports `metadata.buildCommand` / `buildCommandSource`; before it always ran `npm run build`. Tests and lint run only when `package.json` has a `test` / `lint` script, and `metadata.testValidation` / `lintValidation` say whether they ran (before, `npm test` and `npm run lint` ran without a script and lint counted as validated) |
| `resolvePlatformRoot` | `utils/platform-root-resolver.ts` | Resolves the platform root path; used by tools that need an absolute base for config and file resolution |
| `ScanOrchestrationOptions` / `ToolScanScope` | `utils/scan-types.ts` | `ScanOrchestrationOptions` extends `ParallelOptions` with `parallel?`, `incremental?`, `baseBranch?`, `enableProgress?`, `resourceLimits?`, `excludePatterns?`; `ToolScanScope` narrows scan to `'all'|'applications'|'libraries'|'modules'|'features'|'folders'` with optional `targets[]` |

## Critical Gotchas

### FluentKernel analyze stage: bare context

`AnalyzeStage` looks files up with `project.getSourceFile(path)` (exact paths, no globs, files not in the project are skipped silently) and calls `rule.analyze({ sourceFile, project, filePath })` without `api`, `config` or `type`, and without `applySuppressionComments`. Rules that read `context.api` throw there. Programmatic analysis: `ContextFactory.createAnalysisContext` per file as in `README.md` (Load a project and run rules) and the adapter's `rule-runner.ts`.

### Kernel initialization order

`getProject()`, `getFileSystem()`, `getAllPlugins()`, etc. throw `"Kernel is not initialized"` before `kernel.initialize()` is called. The only safe pre-init call is `getProjectRoot()`.

### ProgressStore lives in adapter-mcp

`ProgressStore` is not part of core: it is in `packages/adapter-mcp/src/streaming/progress-store.ts`, and its ordering rule (`setResult` before `markComplete`) is in `packages/adapter-mcp/CLAUDE.md`. Core's streaming folder holds `StreamingKernelWrapper` and the diff helpers.

### ConfigLoader: schema path, validation, cache

`ConfigLoader.SCHEMA_PATH` is `../../schemas/angular-modernizer.schema.json` relative to `config/config-module-url.ts`, whose only content is `import.meta.url` (two levels below the package root in `src/` and `dist/`, so it also works installed from npm). Never resolve it from `process.cwd()`: the MCP server runs in the client project. Until 2026-10 the path came from `new Function('return import.meta.url')`, which throws in every module system, so the cwd fallbacks found the schema only inside this repo and every server in an app ran without validation. Jest runs the CommonJS runtime, which cannot parse `import.meta`: `jest.config.base.js` maps `config-module-url` to `test-support/config-module-url.ts` (packages with their own `moduleNameMapper` need the same entry if they load core sources). The schema compiles once per process (static, shared by all loaders); an unreadable schema makes `load()` of an existing config throw instead of skipping validation. Errors name unknown keys (`params.additionalProperty`) and use `(root)` as path for top-level errors.

The cache (per loader instance, `getConfigLoader()` is the process singleton) is keyed by project root and checked against the file's mtime and size (`missing` when there is no file) on every `load()`, so edits apply without a restart; `clearCache()` is only for tests.

`mergeWithDefaults()` gives defaults only to `build`, `validation` and `pathAliases`; `parser`, `rules`, `tools`, `layers`, `layersSource`, `projectRoot`, `projectContext` and `codingStandard` pass through as written and arrive as `undefined` when absent. A key missing from this list is dropped silently (until 2026-10 `projectRoot`, `projectContext` and `codingStandard` were).

Consumers must not swallow load errors: the Angular, Architecture and Solid plugins rethrow them from `initialize()` (the kernel reports `Failed to initialize plugin`), because a fallback to defaults drops every configured rule option without notice.

### Parser strategies and project context: no production caller

`ParserOrchestrator`, `StandardParserStrategy` and `EnhancedParserStrategy` exist, but no production code parses through them, and `parser.strategy` is deprecated. `Kernel.initialize` no longer runs `ProjectCompatibilityValidator` (it walked the whole project three times per initialization, 0.5 to 0.75 s on a large app, for a result nothing read); `getProjectContext()` is deprecated and returns a neutral context (every flag `false`, `angularVersion` empty). `ProjectCompatibilityValidator` and its report types are deprecated and stay exported in 1.x. Component decorators of a project are configured in `parser.componentDecorators`; no project decorator names are built in.

### Kernel transform path: api defaults to a project-bound PublicApi

When `KernelConfig.plugins` is set and `KernelConfig.api` is not, `initialize()` creates `createPublicApi(this.project)` and hands it to the `TransformationCoordinator`, so `context.api` is always defined for transform rules. An explicitly configured `api` wins. Before this fallback, every `new Kernel({...})` in `adapter-mcp` transform tools shipped `api: undefined` and six `context.api`-dependent transform rules (constructor-to-inject, constructor-injection-transform, container-presentational, static-class-to-functions, extract-static-from-stateful, service-injection-cleanup) failed at runtime. The adapter test suite mocks core and api entirely, so only a core integration test guards this.

### Transform path: abort signal, removed files, deleted backups

`TransformationConfig.signal` (also `FluentKernel.transform(..., { signal })` and `TransformStage`) reaches every rule as `TransformContext.signal`; a rule that starts a process ends it on abort (library-extraction runs `ng`). A rule may remove the requested file from the project (library-extraction moves it): the coordinator then runs no further rule of the call, the others would read a forgotten node; their ids are in `skippedRules` of the result and in a line of `message`. `RollbackManager.backupChanges` takes `deleted: true` for a file the transformation removed: the backup holds its text, `afterHash` is a marker no SHA-256 equals, so a rollback (file-backup or git-stash) writes it again only while it is still missing (`force` overrides).

### RealFileSystemAdapter as ts-morph file system: FileNotFoundError

ts-morph tells a missing file from a failed read by `instanceof FileNotFoundError` (`readFileIfExists`, `createSourceFile(..., { overwrite: true })`). `RealFileSystemAdapter.readFile` / `readFileSync` throw that class on `ENOENT` (other failures stay a plain `Failed to read file` error). ts-morph 27.0.2 exports the class at runtime but its typings only declare it, so `file-system.ts` reads it through the namespace import. Before, the generic error made `createSourceFile` with `overwrite` fail for a file no longer on disk (the envelope's type check of deleted files).

### Parallel analysis: ContextFactory required

Deprecated WorkerPool path (since 1.1, removal in 2.0); applies until it is removed. `executeWithWorkers()` must use `ContextFactory.createAnalysisContext()` - not a bare `{ sourceFile, project, config }` object. Rules that access `context.api` throw when given the bare object. `PublicApi` is created at batch level inside `ParallelExecutor` (not passed from `Kernel`).

### ParallelismModeSelector: auto-selection is default

Deprecated WorkerPool path. `autoSelectMode` defaults to `true`. Set `autoSelectMode: false` or explicitly set `useWorkerThreads` to opt out. Worker mode is chosen when the complexity score exceeds the threshold (file count 30%, LOC 35%, large file outliers 20%, memory pressure 15%). `minFilesForWorkers` defaults to 200 - below this, Promise mode is always used regardless of flags.

### ResourceManager: worker crash budget

Deprecated WorkerPool path. Workers are retired after 3 crashes within a 60-second window; crash counts reset outside that window. `canProceed()` returns `false` if `heapUsed > 90%` of `maxMemory` (default: 80% of available system memory). `PromiseModeMonitor` checks every 5s: warning at 70% projected heap, abort at 85%.

### Cache grouping: group by filePath before caching

After parallel analysis, results must be grouped by `filePath` via `Map<string, unknown[]>` before writing per-file cache entries. Flat distribution (e.g. `Math.floor`) poisons every file's cache with incorrect entries.

### InMemoryFileSystemAdapter in tests

Pass `InMemoryFileSystemAdapter` via `KernelConfig.fileSystem`. For ts-morph `Project` with `useInMemoryFileSystem: true`, set explicit compiler options — TypeScript 5.9 defaults to non-Node resolution for ES2020, which breaks `@angular/core` stub lookup.

### Kernel.tsConfigPath empty string

`tsConfigPath: ''` is valid in tests with `useInMemoryFileSystem: true`. An empty string is not the same as `undefined` — check for falsy rather than `=== undefined` when deciding whether to load tsconfig.

## Testing

All tests use `InMemoryFileSystemAdapter`:

```typescript
import { InMemoryFileSystemAdapter } from '@angular-modernizer/core';
const fs = new InMemoryFileSystemAdapter();
fs.writeFile('/test/component.ts', `...`);
const kernel = new Kernel({ fileSystem: fs, tsConfigPath: '' });
```

Worker-thread tests are gated behind `ENABLE_WORKER_TESTS=true`. Set this env var to run the full worker pool suite; without it, worker tests are skipped to keep CI fast.

## Dependencies

- `@angular-modernizer/plugin-system` — plugin contracts (AnalysisRule, TransformRule, etc.)

No plugin packages depend on core directly. They use plugin-system contracts and receive dependencies via context injection.
