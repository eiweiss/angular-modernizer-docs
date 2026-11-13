# Fluent API Guide

`FluentKernel` (`@angular-modernizer/core`) builds a pipeline of stages on top of an initialized `Kernel` and runs
them in order with `execute()`. The MCP tool `transform-code` uses it on its default path (`apiStyle: 'fluent'`):
transform stage plus an optional build check. Signatures of every method: the TypeDoc of `FluentKernel`.

## A transformation with build check

```typescript
import { createKernel, FluentKernel } from '@angular-modernizer/core';
import { AngularPlugin } from '@angular-modernizer/plugin-angular';

const { kernel } = await createKernel({
  projectPath: '/path/to/angular-project',
  plugins: [new AngularPlugin()],
});
const filePath = '/path/to/angular-project/src/app/user.service.ts';
kernel.getProject().addSourceFileAtPath(filePath);

const result = await FluentKernel.create(kernel)
  .transform('constructor-to-inject', [filePath], {
    // the rules of plugin-angular run only for their transformationType
    '@angular-modernizer/plugin-angular': { transformationType: 'constructor-to-inject' },
  })
  .buildCheck(true) // save, build, restore the original texts when the build fails
  .withMetadata({ ticket: 'WI-1234' })
  .execute();

if (!result.success) {
  console.error(result.errors);
}
```

`PipelineResult` holds `success` (no stage recorded an error), `files`, `violations`, `transformations`, `errors`
and `metadata`.

## Stages

| Method | Stage | What it does |
|-|-|-|
| `.analyze(files, rules)` | `AnalyzeStage` | Runs `rules` on the listed files |
| `.filter(predicate)` | `FilterStage` | Keeps the violations the predicate accepts |
| `.transform(type, files?, pluginConfig?, { signal }?)` | `TransformStage` | Calls `kernel.transformFile` per file; records the original texts |
| `.buildCheck(rollbackOnFailure = false)` | `BuildValidationStage` | Saves the transformed files, builds with `kernel.getBuildValidator()`, restores the originals in memory and on disk on failure when rollback is on; metadata `buildValidation`, `rollback`, `rolledBackFiles` |
| `.preValidate()`, `.postValidate()` | `ValidationStage` | See limits below |

Options: `.rollbackOnFailure()` stops at the first error and turns rollback on for every build check of the
pipeline; `.verbose()` logs each stage; `.withMetadata(obj)` adds to `result.metadata`; `.onProgress(cb)` reports
stage progress. `.getStages()`, `.clear()` and `.getKernel()` inspect or reset the builder.

## Limits

- File lists are exact paths of files already in the kernel's project (`project.getSourceFile(path)`), not globs;
  a path that is not in the project is skipped without an error. When earlier stages already set the files,
  `.transform(type, patterns)` keeps the files whose path contains one of the patterns.
- `.analyze` calls `rule.analyze({ sourceFile, project, filePath })`: no `api`, no `config`, no suppression
  comments. Rules that read `context.api` (most plugin rules) fail there. To run plugin rules, build the context with
  `ContextFactory.createAnalysisContext` as in the [core README](../packages/core/README.md#load-a-project-and-run-rules).
- `.preValidate()` and `.postValidate()` look for `getValidationRules()` on the kernel, which the `Kernel` does not
  have, so both stages are always skipped (`metadata.validation: 'skipped'`).
- The full safety envelope of transform-code (backups with a git snapshot, type check of the changed files and their
  importers) is not part of the pipeline; only `buildCheck` restores files.

## RxJS

`await fluent.toObservable()` runs the pipeline as an Observable of pipeline events (`stage-start`,
`stage-complete`, `stage-error`, `progress`, `complete`, `error`). It needs `rxjs` installed and throws without it.
It is typed `Promise<unknown>`, and neither `PipelineEvent` nor `isRxJSAvailable()` is exported from the package
root, so cast the result yourself. Unsubscribing stops the pipeline before the next stage; a stage that already runs
finishes.

## Tests

Pipelines run against a kernel on `InMemoryFileSystemAdapter` like any other kernel test; see the core README
(Tests with the in-memory file system) and the suites in `packages/core/__tests__/fluent/`.
