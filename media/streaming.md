# Streaming

> MCP clients: the `streaming` parameter (transform-code, scan-architecture) and `get-operation-progress` are
> deprecated and stay in 1.x. Call the tools synchronously: they send `notifications/progress` when the request has a
> `progressToken` and stop on `notifications/cancelled`. See the
> [adapter-mcp README](../../packages/adapter-mcp/README.md#progress-and-cancellation). This guide covers the core
> streaming classes, which stay.

## StreamingKernelWrapper

`StreamingKernelWrapper` (`@angular-modernizer/core`) wraps an initialized `Kernel` and yields
`StreamingProgressEvent`s from async generators. Options (`StreamingOptions`): `includeDiffs` (default `true`),
`diffFormat` (`'unified'` default, `'structured'`, `'both'`), `includeMemory` (`true`), `validateSyntax` (`true`),
`diffContext` (3 lines), `collectFeedback` (`false`; needs a `FeedbackCollector` as third constructor argument).

```typescript
import { StreamingKernelWrapper } from '@angular-modernizer/core';

const wrapper = new StreamingKernelWrapper(kernel, { diffFormat: 'structured' });
const stream = wrapper.transformStreaming([], ['src/app/**/*.service.ts'], { dryRun: true });

for await (const event of stream) {
  console.log(`[${event.phase}] ${event.percentage.toFixed(0)}% ${event.message}`);
}
```

| Method | What it does |
|-|-|
| `transformStreaming(rules, filePatterns, config?)` | Runs `kernel.transformFile` on every project file matching the ts-morph globs, with `config` as `TransformationConfig`; one event per file with diff, syntax errors and memory as configured, then `validating`, `building` and `complete`; returns a `StreamingTransformResult`. `rules` is not read: all transform rules of the loaded plugins run, selected by their `pluginConfig` |
| `analyzeStreaming(rules, filePatterns, config?)` | Emits one `analyzing` event per matching file and `complete`; it runs no rules (`rules` and `config` are not read) |

Phases: `analyzing`, `transforming`, `validating`, `building`, `complete`.

## Diff helpers

`generateUnifiedDiff`, `generateDiffSummary` and `detectSyntaxErrors` are exported from the package root and work
without the wrapper. The diff types (`FileDiff`, `DiffHunk`, `LineChange`, `StructuredDiff`,
`SemanticAnnotation`) are exported as types. Signatures: TypeDoc of `@angular-modernizer/core`.

## ProgressStore

The operation store behind `get-operation-progress` lives in adapter-mcp
(`packages/adapter-mcp/src/streaming/progress-store.ts`); its ordering rule (`setResult` before `markComplete`) and
TTLs are in `CLAUDE.md` (Streaming Infrastructure).
