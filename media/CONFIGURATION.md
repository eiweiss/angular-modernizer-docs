# Configuration Reference

Angular Modernizer is configured via `.angular-modernizer.json` in the root of the target Angular project being analyzed. Only `build` is required; all other sections are optional.

The file is validated against the JSON Schema of `@angular-modernizer/core` (see [Schema and IDE autocompletion](#schema-and-ide-autocompletion)). A file that is not valid JSON or does not match the schema is an error: scans and transformations answer with the loader's message instead of running with defaults, the investigation tools that read `parser.componentDecorators` fall back to `Component` and report the message as a warning, and unknown keys are named (`must NOT have additional properties ('scan-angular')`). The MCP server rereads the file when its modification time or size changes; no restart is needed.

## Complete Example

```json
{
  "$schema": "./node_modules/@angular-modernizer/core/schemas/angular-modernizer.schema.json",
  "projectRoot": "/home/user/projects/my-angular-app",
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
    },
    "angular:presentational-component-violation": {
      "dialogDetection": {
        "baseClassNames": ["BaseDialogComponent"]
      }
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

---

## `$schema`

```json
"$schema": "./node_modules/@angular-modernizer/core/schemas/angular-modernizer.schema.json"
```

Optional but recommended. Points to the bundled JSON Schema for IDE autocompletion and inline validation. See the note at the bottom of this page.

---

## `projectRoot`

```json
"projectRoot": "/absolute/path/to/project"
```

Absolute path to the Angular project root. Read by the feedback tools (`collect-feedback`, `review-feedback`, `suggest-improvements`, `analyze-parser-performance`) when they are called without a `projectRoot` argument: they load the config from the server's working directory and then use this path for their storage. All other tools take the project from their own arguments (`rootPath`, `projectRoot`, `filePath`).

---

## `build`

Required. The build that run-build, validate-transformation and transform-code with `validateBuild` run.

| Field | Type | Default | Description |
|-|-|-|-|
| `command` | string | - | Build command to run (`ng build`, `npx nx build my-app`, `pnpm run build`, etc.). Required |
| `timeout` | number (ms) | `300000` | Maximum time run-build waits for the build command (after its `options.timeout` and `tools['run-build'].timeout`) |

The build always runs `build.command` as a full build: the Angular CLI has no file-scoped build. analyze-build-impact lists the files a change affects.

### Deprecated `build` keys

Still accepted so existing configs load; they have no effect and the loader warns when one is set (removal in 2.0):

- `mode` (`fast`, `full`, `incremental`): no tool chooses a build by it. No longer required.
- `enforcementLevel`: nothing reads it.
- `fastModeCommands`: there is no fast mode.
- `incremental`: there is no incremental build path. To clear the build caches before a build, pass `cacheStrategy: 'clear'` to run-build or validate-transformation (or set it in their `tools` section): it removes `.angular/cache` (and `.nx/cache` in an Nx workspace) inside the project, never under `node_modules`. The same warning covers `incrementalMode`, `forceFullBuild` and `cacheStrategy: 'skip'` in `tools['run-build']` and `tools['validate-transformation']`.

---

## `validation` (deprecated)

Every key of this section has no effect; the loader warns when one is set (removal in 2.0). transform-code type-checks every transformation, builds with `validateBuild` and rolls back on failure itself; validate-transformation has its own arguments (`validateTypes`, `validateBuild`, `buildTimeout`).

| Field | Former meaning |
|-|-|
| `typescript` | TypeScript compilation check |
| `eslint` | ESLint check |
| `build` | Build of the service-bag transformation |
| `buildTimeout` | Timeout of that build |
| `rollbackOnFailure` | Rollback of the service-bag transformation on a failed check |

---

## `parser`

### `parser.componentDecorators`

Decorator names that mark a class as an Angular component in addition to `Component`. The platform has no project decorator names built in: a class with a project decorator such as `@PageComponent({ selector, template })` counts as a component only when its decorator is listed here. Default: `[]`.

The list adds to `Component`; an empty list and a missing key mean the same thing (`Component` only). It is read by the template analysis rules of plugin-angular, by `solid:srp-single-responsibility-violation` (components are skipped) and `solid:god-object-violation` (constructor parameters of components count as injected dependencies), and by the investigation tools `find-angular-symbol`, `find-component-hosts`, `inspect-template`, `trace-output-flow` and `find-usages` (template usages). The MCP server rereads the file when it changes.

```json
"parser": {
  "componentDecorators": ["PageComponent"]
}
```

Decorators that only sit on methods or properties, or that mark a class as something other than a component, need no configuration.

### Deprecated `parser` keys

The parser is chosen automatically. `strategy`, `timeout`, `falsePositiveDetection`, `externalDecorators` and `pathAliasResolution` are still accepted so existing configs load; they have no effect and the loader warns when one is set (removal in 2.0).

---

## `pathAliases` (deprecated)

No effect: nothing reads `enabled`, `tsconfigPath` or `preserveAliases`. Imports are resolved through `compilerOptions.paths` of the project's tsconfig. The loader warns when the section is set (removal in 2.0).

---

## `rules`

Per-rule configuration overrides, keyed by rule ID. Every rule accepts `suppressionComments` (below); a subset of
rules expose further options. Unknown rule IDs are rejected.

### Suppression comments (every rule)

`rules['<rule-id>'].suppressionComments` lists comment markers that hide a finding of that rule. No marker is built in;
without the option nothing is suppressed. A finding is hidden when a comment starting with a marker

- sits on the finding's line, or
- ends on the line directly above the declaration or statement the finding is reported at (for a class reported at its
  decorator: above the decorators), or follows that declaration on its last line (multi-line imports),
- in a template (`.html` findings such as missing-trackby): `<!-- marker -->` on the line or ending on the line above.

A comment two lines up or trailing the previous statement does not count, nor does a marker in the middle of a comment
(`// see static-ignore` does not match `static-ignore`).

```json
"rules": {
  "angular:static-class-detection": { "suppressionComments": ["static-ignore"] },
  "angular:missing-trackby": { "suppressionComments": ["trackby-ignore"] },
  "angular:type-safety": { "suppressionComments": ["any-ignore"] },
  "angular:any-to-interface": { "suppressionComments": ["any-ignore"] }
}
```

An inline comment after the type counts too, since it sits on the finding's line: `item: any /* any-ignore: untyped
DTO */`. Each rule needs its own entry; an `any` node is reported by either type-safety or any-to-interface, never by
both. The `any-to-interface` transformation keeps an `any` marked `any-ignore` whatever the configuration says.

```typescript
// static-ignore: kept for the public API
export class JsonHelper {
  static parse(text: string): unknown { return JSON.parse(text); }
}
```

The scans (scan-all, scan-angular, scan-solid, scan-architecture), analyze-project, extract-libraries and the
`analyzeProject` functions apply the markers after each rule, also on the parallel and incremental paths. Analysis in
worker threads of the deprecated WorkerPool path does not read `rules` at all.

### `angular:component-wrapper-instantiation`

Detects components/wrappers being instantiated directly (via `new`) instead of through Angular's DI.

| Field | Type | Description |
|-|-|-|
| `classNamePatterns` | string[] | Regex patterns matched against class names |
| `explicitClassNames` | string[] | Exact class names always treated as targets |
| `baseClassNames` | string[] | Any class extending one of these is a target |
| `decoratorNames` | string[] | Any class decorated with one of these is a target |
| `excludedClassNames` | string[] | Class names never reported, even when a pattern matches. Replaces the rule's default list entirely (the defaults are not merged) |

Example:
```json
"angular:component-wrapper-instantiation": {
  "classNamePatterns": [".*Wrapper$", ".*Adapter$"],
  "explicitClassNames": ["SpecialComponent"]
}
```

### `angular:dto-direct-instantiation`

Reports `new X()` of a project class that is plain data: public properties without defaults and nothing else (no base class, decorator, constructor parameters or code, method, accessor, non-public property, and no `instanceof` check against it in the project). Names do not make a class a DTO; library classes, unresolved callees and a `new` inside the class itself are not reported. A typed object literal builds the same value; the `dto-object-literal` transform rewrites the sites where every required property is assigned right after the `new` and names the reason for the others. Findings are warnings.

| Field | Type | Description |
|-|-|-|
| `classNamePatterns` | string[] | Regex patterns: when set, only classes whose name matches one of them or `explicitClassNames` are reported. Default: none, no name filter |
| `explicitClassNames` | string[] | Class names: when set, only these or names matching `classNamePatterns` are reported. Default: none |
| `excludedClassNames` | string[] | Class names never reported. Replaces the default list (`HttpResponse`, `HttpErrorResponse`, `HttpHeaderResponse`) entirely |
| `baseClassNames`, `decoratorNames` | string[] | Deprecated, no effect (removed in 2.0): a class with a base class or decorator is no plain data class. The loader warns when one is set |

### `angular:magic-string-selector`

Detects magic string selectors: a string or template literal passed as the first argument of a configured method whose first parameter is a string (type checker; a parameter typed `any`, a constructor type or an unresolved callee is not inspected). A property access (`this.config.selector`) is reported only when its name is listed in `selectorPropertyNames`. Findings are warnings.

| Field | Type | Description |
|-|-|-|
| `targetMethodPatterns` | string[] | Names of the project methods that look up a component by its selector string; their first argument is inspected (exact match). Default: none, so the rule reports nothing until the project lists its methods |
| `selectorPropertyNames` | string[] | Property names that count as a selector when a property access is the first argument; other property accesses are not reported. Unset: no property access is reported |

```json
"angular:magic-string-selector": {
  "targetMethodPatterns": ["openBySelector"],
  "selectorPropertyNames": ["selector", "componentSelector"]
}
```

### `angular:too-many-inputs`

Reports a component with more own inputs than `maxInputs`: `@Input()` properties and setters, signal `input()` / `model()` fields and `inputs:` metadata; inputs of base classes do not count. Findings are warnings.

| Field | Type | Description |
|-|-|-|
| `maxInputs` | integer | Most own inputs a component may declare before it is reported. Default: 5 |

```json
"angular:too-many-inputs": {
  "maxInputs": 8
}
```

### `angular:presentational-component-violation`

Reports, at severity info, a component that another component's template uses by selector, that has inputs or
outputs, and that loads data itself (a dependency reaching HttpClient). Routed components, components created in code
and dialogs are never reported; a registry entry whose lookup selector the code does not tie to the component does not
count as created in code (`metadata.registryCandidates`). The library dialog keys (`MatDialogRef`, `MAT_DIALOG_DATA`, `DialogRef`,
`DIALOG_DATA`, `MatBottomSheetRef`, `MAT_BOTTOM_SHEET_DATA`) always mark a dialog; `dialogDetection` adds project
dialogs.

```json
"angular:presentational-component-violation": {
  "dialogDetection": {
    "injectedClassNames": ["MyDialogRef"],
    "baseClassNames": ["BaseDialogComponent"]
  }
}
```

| Field | Type | Description |
|-|-|-|
| `dialogDetection.injectedClassNames` | string[] | Injection keys (constructor parameter type or `@Inject()` token, `inject()` argument) that mark a component as a dialog, in addition to the library dialog keys. Default: `[]` |
| `dialogDetection.baseClassNames` | string[] | Base classes (any level) that mark a component as a dialog. Default: `[]` |
| `actionTilePatterns` | string[] | Deprecated, no effect (removed in 2.0); ConfigLoader warns when it is set |

### `architecture:content-placement`

Checks that each content type lives in a layer allowed for it. Layer membership comes from [`layers`](#layers) or
[`layersSource`](#layerssource), never from folder names. Without layers or without `placement` the rule reports
nothing; content types missing from `placement` and files outside every layer are not checked.

```json
"architecture:content-placement": {
  "placement": {
    "service": ["@core", "@shared", "@features"],
    "guard": ["@core"],
    "model": ["@shared"]
  }
}
```

| Field | Type | Description |
|-|-|-|
| `placement` | object | Allowed layers (name or alias) per content type. Keys: `service`, `component`, `guard`, `interceptor`, `directive`, `pipe`, `model`, `utility` |

### `architecture:layer-boundary-violation`

The layer topology itself comes from [`layers`](#layers) or [`layersSource`](#layerssource). This entry only holds
[suppression markers](#suppression-comments-every-rule): an import is not reported when a comment on its line, on the
line directly above or after its last line starts with one of them. No marker is built in.

```json
"architecture:layer-boundary-violation": {
  "suppressionComments": ["arch-ignore"]
}
```

```typescript
import { LegacyBridge } from '@main/legacy/bridge'; // arch-ignore: removed with the legacy shell
```

| Field | Type | Description |
|-|-|-|
| `suppressionComments` | string[] | Comment markers that suppress a finding (`// arch-ignore: reason` matches `arch-ignore`). Default: `[]` |

### `architecture:library-extraction-opportunity`

Finds cohesive groups of at least 3 artifacts in shared folders that could become an Angular library. Files inside an existing library (a folder with `ng-package.json` or `tsconfig.lib.json` at or above the file) are skipped. The extract-libraries tool uses the same thresholds by default but takes its options from its call arguments, not from this section (only `suppressionComments` applies there); its `targetFolders` default is the same `["shared", "common", "core", "libs"]` (`LIBRARY_EXTRACTION_DEFAULTS`); `models`, `interfaces` or other folders are passed as `targetFolders`.

```json
"architecture:library-extraction-opportunity": {
  "targetFolders": ["shared", "ui-kit"],
  "minCohesionScore": 0.5,
  "categoryOverrides": { "ButtonComponent": "ui" }
}
```

| Field | Type | Description |
|-|-|-|
| `minCohesionScore` | number | Minimum cohesion score (0-1) of a cluster. Default: `0.5` |
| `minReusabilityScore` | number | Minimum reusability score (0-100) of an artifact. Default: `40` |
| `targetFolders` | string[] | Folder names scanned for candidates, matched as whole path segments. Default: `["shared", "common", "core", "libs"]` |
| `categoryOverrides` | object | Fixed category per class name: `ui`, `forms`, `layout`, `utils`, `data`, `core` or `feature` |
| `excludePatterns` | string[] | Glob patterns of project-relative paths to exclude (`**/generated/**` also matches `generated/x.ts`). Default: `[]` |

### `angular:interface-duplication`

Three kinds of findings. Interface families (severity `info`): 3 or more interfaces sharing a suffix such as `DialogRequest`, `Config`, `Data`, `Dto` or `Payload` that all declare some properties themselves (inherited ones do not count); only maximal families, once per project at the first member, and the message says whether the `interface-duplication` transform can consolidate the family or why not. Same-name copies (severity `warning`): an exported interface declared with an identical shape (by the type checker) in 2 or more files, once per group at the first declaration. ID parameters (severity `warning`): public methods and top-level functions with 2 or more `*Id` parameters written as `string`. The options below apply to the families; `scope` applies to all three. The `interface-duplication` transform reads `minSimilarInstances`, `minCommonProperties` and `minSimilarityScore` from this section too (transform-code passes them as defaults under the call `options`, which win key by key), so the rule's fixable claim and the transform use the same thresholds; `scope` and `suppressionComments` stay with the rule.

```json
"angular:interface-duplication": {
  "minSimilarInstances": 4,
  "minCommonProperties": 3
}
```

| Field | Type | Description |
|-|-|-|
| `minSimilarInstances` | integer | Minimum number of interfaces in a family (also the minimum group the transform consolidates). Default: `3` |
| `minCommonProperties` | integer | Minimum number of own properties all interfaces of a family declare. Default: `2` |
| `minSimilarityScore` | number | Minimum Jaccard score (0-1) of a family: the properties all members declare, divided by all own properties of the members. Default: `0` |
| `scope` | object | `{ "type": "folder", "paths": ["/abs/src/app/shared"] }` compares only files below `paths`; other types compare the whole project. Default: the whole project |

### `solid:god-switch`

Switch statements with too many clauses that do more than return or assign a value.

```json
"solid:god-switch": { "maxCases": 15 }
```

| Field | Type | Description |
|-|-|-|
| `maxCases` | integer | A switch with more non-lookup clauses than this is reported. Clauses are counted by body: grouped labels without statements are one clause with the next, `default` with a body counts, a body that is only `break` is not counted. A clause whose body (without `break`, blocks unwrapped) is a single `return` or assignment with no call or `new` in it or one call with only constant arguments (literals, enum members, `const` variables, such as `translate.instant('KEY')`) is a lookup and not counted. Default: `10` |

### Deprecated `rules` entries

`rules['angular:performance-violation']` is still accepted so existing configs load; the rule reports nothing, the entry has no effect and the loader warns when it is set (removal in 2.0). Subscriptions without cleanup are reported by `architecture:rxjs-memory-leak`.

`rules['angular:lifecycle-hook-violation']` is still accepted so existing configs load; the rule reports nothing, the entry has no effect and the loader warns when it is set (removal in 2.0). Unhandled async errors are reported by `architecture:error-handling-pattern`, subscriptions without cleanup by `architecture:rxjs-memory-leak`.

`rules['angular:service-without-injectable']` is still accepted so existing configs load; the rule reports nothing, the entry has no effect and the loader warns when it is set (removal in 2.0). The Angular compiler rejects an undecorated class that dependency injection cannot build where it sees the provider (NG2005, NG2006); providers outside decorators fail at run time (NG0204).

`rules['solid:lsp-violation']` is still accepted so existing configs load; the rule reports nothing, the entry has no effect and the loader warns when it is set (removal in 2.0). A class that misses a required member of an implemented interface or an abstract method of its base is a compile error, which the TypeScript compiler reports.

`rules['solid:service-dip-violation']` is still accepted so existing configs load; the rule reports nothing, the entry has no effect and the loader warns when it is set (removal in 2.0). An injection against the layer direction is always an upward import, which `architecture:layer-boundary-violation` reports for the configured `layers` / `layersSource`. The `service-dip-violation-transform` is not affected.

`rules['architecture:service-architecture']` is still accepted so existing configs load; the rule reports nothing, the entry has no effect and the loader warns when it is set (removal in 2.0). An exported interface, type, enum, constant or function next to a service that the service does not use in its public members and that other files import is reported by `architecture:model-separation`; more than 10 injected dependencies by `solid:god-object-violation`.

---

## `tools`

Per-tool defaults for the MCP tools, keyed by tool name. A tool merges its section under its call arguments (`{ ...config, ...args }`): a call argument always wins over the section, the section wins over the built-in default. Unknown tool names or fields make the file invalid. Deprecated fields are still accepted so existing configs load; they have no effect, the loader warns when one is set, and they are removed in 2.0.

```json
"tools": {
  "scan-all": { "excludePatterns": ["**/node_modules/**", "**/*.spec.ts", "**/legacy/**"] },
  "transform-code": { "validateBuild": true },
  "run-build": { "configuration": "development", "timeout": 600000 }
}
```

### `tools['scan-architecture']`

| Field | Type | Default | Description |
|-|-|-|-|
| `scope.type` | `all` \| `applications` \| `libraries` \| `modules` \| `features` \| `folders` | `all` | What to scan: the whole project, Angular apps, libraries, NgModules, feature modules or folders |
| `scope.targets` | string[] | - | App, library, module or feature names, or folder paths, matching `scope.type` |
| `rules` | string[] | all | Architecture rules that run |
| `includePatterns` | string[] | - | Glob patterns of the files to scan |
| `excludePatterns` | string[] | - | Added to the built-in list (`**/node_modules/**`, `**/dist/**`, `**/build/**`, `**/coverage/**`, `**/e2e/**`, `**/.*/**`, `**/*.spec.ts`, `**/*.test.ts`) |
| `maxFiles` | number | `100` | Cap on the files analyzed; a cut sets `execution.incomplete` |
| `maxResults` | number | `1000` | Violations returned per page |
| `resultOffset` | number | `0` | Violations skipped before the page |
| `includeAutoFixes` | boolean | - | Deprecated, no effect: scan-architecture always returns suggested fixes |
| `orchestration` | object | - | Deprecated: read only with `apiStyle: 'imperative'`, where `parallel` and `incremental` select the old WorkerPool path (no faster than the default scan worker, about four times the memory). Leave unset |

### `tools['scan-all']`, `tools['scan-angular']`, `tools['scan-solid']`

The three scans share one field set.

| Field | Type | Default | Description |
|-|-|-|-|
| `rules` | string[] | all | Rules that run |
| `includePatterns` | string[] | - | Glob patterns of the files to scan |
| `excludePatterns` | string[] | built-in list | Replaces the built-in list (the one of scan-architecture above); repeat its entries you want to keep |
| `maxFiles` | number | `100` | Batch size; every file in scope is analyzed |
| `maxResults` | number | `1000` | Violations returned per page |
| `resultOffset` | number | `0` | Violations skipped before the page |
| `includeAutoFixes` | boolean | `false` | Include suggested fixes in the results |
| `scope` | object | - | Deprecated, no effect: these scans have no scope, use `includePatterns` and `excludePatterns` |
| `orchestration` | object | - | Deprecated, no effect: the scan runs in the scan worker |

### `tools['analyze-project']`

| Field | Type | Default | Description |
|-|-|-|-|
| `includeDetails` | boolean | `false` | Return the rule findings (paged by `maxResults` and `resultOffset`) |
| `maxDepth` | number (1-10) | `3` | Depth of the dependency tree |
| `excludePatterns` | string[] | - | Added to the built-in ignores (`node_modules/**`, `.git/**`, `dist/**`, `build/**`, `coverage/**`) |
| `maxFiles` | integer (at least 1) | `50` | Cap on the TypeScript files the rules analyze, in path order; a cut sets `counts.truncated` and the summary names the files left out |
| `maxResults` | number | `1000` | Findings returned per page |
| `resultOffset` | number | `0` | Findings skipped before the page |
| `orchestration` | object | - | Deprecated, no effect: analyze-project reads `orchestration` only from the call argument |

### `tools['transform-code']`

Read by single-file calls and by batches (transform-code with `files`).

| Field | Type | Default | Description |
|-|-|-|-|
| `validateBeforeTransform` | boolean | `true` | Validate the file before the transformation |
| `createBackup` | boolean | `true` | Back up the file (and take a git snapshot) before the transformation; rollback-changes restores it |
| `validateTransformation` | boolean | `true` | Type check of every changed file and the files that import one (new errors only), plus build and lint when enabled |
| `autoRollback` | boolean | `true` | Roll back on a failed validation |
| `backupDir` | string | - | Custom backup directory |
| `rollbackStrategy` | `all-or-nothing` \| `selective` \| `none` | `all-or-nothing` | Batch only, when a file fails: restore every file, only the failed ones, or nothing |
| `apiStyle` | `imperative` \| `fluent` | `fluent` | Kernel API used for the transformation |
| `buildCheck` | boolean | `false` | Build validation after the transformation (fluent API only) |
| `validateBuild` | boolean | `false` | Run the project build (`build.command` or the detected build system) after the transformation |
| `validateLint` | boolean | `false` | Run the project's lint script after the transformation (lints the whole project) |
| `rollbackOnBuildFailure` | boolean | `true` | Roll back (true) or keep (false) a transformation whose only failure is the build |
| `failOnTestErrors` | boolean | `false` | Fail on type errors the transformation causes in test files; by default they are reported in `validation.testTypeErrors` |
| `orchestration` | object | - | Batch only: options of the batch run, under the call's `orchestration`; `parallel` defaults to `false` and is switched on for more than 100 files when neither the call nor the config sets it. Keys: `parallel`, `autoSelectMode`, `enableProgress`, `resourceLimits` (`enableMonitoring`, `incremental` and `baseBranch` are deprecated, no effect) |
| `streaming` | boolean | - | Deprecated, no effect: read from the call arguments only |
| `diffFormat` | `unified` \| `structured` \| `both` | - | Deprecated, no effect: streaming emits no diff events |

### `tools['transform-code-batch']` (deprecated)

No effect: a batch (transform-code with `files`) reads `tools['transform-code']`. The fields `parallel`, `enableProgress`, `rollbackStrategy`, `useWorkerThreads`, `autoSelectMode`, `maxWorkers`, `timeout` and `resourceLimits` are still accepted; the loader warns when the section is set (removal in 2.0).

### `tools['run-build']`

| Field | Type | Default | Description |
|-|-|-|-|
| `buildSystem` | `auto-detect` \| `angular-cli` \| `nx` \| `webpack` \| `vite` | `auto-detect` | Build system to use |
| `cacheStrategy` | `use` \| `clear` \| `skip` | `use` | `clear` removes `.angular/cache` (and `.nx/cache` in an Nx workspace) inside the project before the build, never under `node_modules`; `use` leaves the caches alone; `skip` is deprecated and has no effect |
| `configuration` | string | - | Passed as `--configuration` (after `--` for an npm script); a call's `options.configuration` wins |
| `watch` | boolean | `false` | Watch mode |
| `timeout` | number (ms) | `300000` | Build timeout |
| `incrementalMode` | boolean | - | Deprecated, no effect: run-build always runs the full build command |
| `forceFullBuild` | boolean | - | Deprecated, no effect: every build is a full build |

### `tools['validate-transformation']`

| Field | Type | Default | Description |
|-|-|-|-|
| `cacheStrategy` | `use` \| `clear` \| `skip` | `use` | As in `tools['run-build']` |
| `buildConfig` | string | - | Passed as `--configuration` when set; unset, `build.command` runs as configured |
| `incrementalMode` | boolean | - | Deprecated, no effect: validate-transformation always runs the full build command |
| `forceFullBuild` | boolean | - | Deprecated, no effect: every validation is a full build |

### `tools['collect-feedback']`

| Field | Type | Default | Description |
|-|-|-|-|
| `storageLocation` | `project` \| `platform` \| `both` | `both` | `project`: `<projectRoot>/.claude-feedback`; `platform`: the platform store (requires `ANGULAR_MODERNIZER_PLATFORM_ROOT`); `both`: the project, plus the platform store when that variable is set |
| `minCompletenessScore` | number (0-100) | - | Feedback with a lower completeness score is reported as not valid (`validationResult.valid`). Unset, the score is only reported |
| `strictMode` | boolean | `false` | Feedback with validation warnings is reported as not valid (default: only errors) |

Neither check affects storage: the feedback is stored either way.

### `tools['review-feedback']`

| Field | Type | Default | Description |
|-|-|-|-|
| `source` | `project` \| `platform` \| `both` | `both` | Where feedback is read, as `storageLocation` of collect-feedback |
| `analysisType` | `success-patterns` \| `failure-patterns` \| `quality-trends` \| `all` | `all` | Used when the call passes no `analysisType` |
| `minConfidence` | number (0-1) | `0.7` | Minimum confidence of a pattern |
| `minCompleteness` | number (0-100) | `40` | Minimum completeness score of feedback that is included. A successful transformation recorded by collect-feedback with execution and validation results scores 40, without validation 20 |

### `tools['suggest-improvements']`

| Field | Type | Default | Description |
|-|-|-|-|
| `source` | `project` \| `platform` \| `both` | `both` | As in `tools['review-feedback']` |
| `minConfidence` | number (0-1) | `0.7` | Minimum confidence of a suggestion's pattern |

### `tools['analyze-parser-performance']`

The tool is deprecated (removal in 2.0): no MCP tool records the parser metadata it reads, so feedback collected through MCP gives 0 sessions analyzed. The keys still apply.

| Field | Type | Default | Description |
|-|-|-|-|
| `source` | `project` \| `platform` \| `both` | `both` | As in `tools['review-feedback']` |
| `sessionIds` | string[] | all sessions | Feedback sessions to analyze |

### `tools['refresh-state']`

Retention of transform backup groups and git snapshot refs in a report. `mode`, `planId` and `parts` are call arguments only and rejected here. An apply uses the retention of its report and refuses `retentionDays` and `keepNewest` as arguments.

| Field | Type | Default | Description |
|-|-|-|-|
| `retentionDays` | integer (at least 0) | `30` | Backup groups and snapshot refs older than this many days are listed |
| `keepNewest` | integer (at least 1) | `5` | Newest backup groups that are never listed, whatever their age; at least 1, because the newest group is what rollback-changes restores |

---

## `projectContext`

```json
"projectContext": {
  "migrationGoal": "angular-cli-projects",
  "migrationPhase": "core-services"
}
```

Free-form notes for agents and humans: migration goal, current phase, extraction targets, anything the team wants next to the config. The platform does not read this block; it exists so such notes do not make the file invalid.

---

## `layers`

Defines the architectural layer topology used by the `architecture:layer-boundary-violation` rule. When this array is absent or empty the rule is silently disabled.

With `layers` or `layersSource` set, `solid:import-direction-violation` reports nothing: its built-in folder
conventions (`app`, `features`, `shared`, `core`, `libs`) would contradict the configured topology, which
layer-boundary-violation checks instead. `architecture:advanced-content-placement` does not read `layers`; it matches
fixed folders (`src/domains/`, `src/features/`, `src/shared/`, `src/core/`) relative to the project root (the root
the scan runs on, else the tsconfig's folder), not to the server's working directory.

Each layer entry:

| Field | Required | Description |
|-|-|-|
| `name` | yes | Human-readable label, e.g. `"core"` |
| `alias` | yes | Path alias prefix as declared in `tsconfig.json`, e.g. `"@core"`. Fallback for aliased imports that cannot be resolved to a file; also used to find the layer directory when `paths` is omitted. May be `""`. |
| `paths` | no | Glob patterns (relative to project root) identifying files that belong to this layer. Used to classify both the importing file and the imported file. When omitted, the directory the alias maps to in `tsconfig.json` `paths` is used. |
| `canImportFrom` | yes | Aliases (or names) of layers this layer may import from. Empty array means no imports from other layers are allowed. Imports within the same layer are always allowed. |
| `level` | no | Position in the hierarchy, lower = more foundational. When the importing and the imported layer both have a level, `canImportFrom` only lists the allowed exceptions upwards: imports into the same or a lower level are always allowed. Without levels `canImportFrom` is the complete list. |

How imports are classified:
- Every `import` and `export ... from` declaration is checked, whether the specifier is a path alias (`@features/x`) or relative (`../../features/x`).
- The specifier is resolved to a file with TypeScript module resolution (tsconfig `paths` and `baseUrl`, directory `index.ts` probing), then classified by path.
- `paths` globs are anchored at the project root and matched segment by segment. If several layers match, the most specific (longest) pattern wins, regardless of declaration order.
- Aliased imports that cannot be resolved fall back to matching the `alias` prefix.
- With `level` on both layers an import is reported only when it goes to a higher level that is not in `canImportFrom`; otherwise it is reported when the target layer is not in `canImportFrom`.
- Imports marked with one of the rule's [`suppressionComments`](#architecturelayer-boundary-violation) are skipped.
- Violation metadata includes `importKind` (`aliased` or `relative`), `typeOnly` (`import type`, or every named specifier is type-only), `declarationKind` (`import` or `export`) and `resolvedPath`.

Example — standard three-layer setup:

```json
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
```

With this configuration:
- `core` cannot import from any other layer.
- `shared` can import from `core` only.
- `features` can import from `core` and `shared`.
- Any import that violates these rules is reported as a `architecture:layer-boundary-violation`.

---

## `layersSource`

Loads the layer topology from an existing file instead of duplicating it in `layers`. The architecture plugin imports the module with a dynamic `import()` during initialization and maps it to the `layers` model.

| Field | Required | Description |
|-|-|-|
| `format` | yes | Source format. Currently only `"arch-validator"`. |
| `path` | yes | Path to the module, relative to the project root or absolute. Must be loadable by Node as ESM (`.mjs`, or `.js` inside a `"type": "module"` package). |
| `exportName` | no | Named export holding the config object. Default `"CONFIG"`. |

```json
"layersSource": {
  "format": "arch-validator",
  "path": "tools/arch-validator/config.js"
}
```

For an arch-validator config such as:

```js
export const CONFIG = {
  layers: {
    'projects/shared': { level: 1, allowedImports: [], pathAlias: '@app/shared' },
    'src/app/app-shared': { level: 2, allowedImports: ['projects/shared'] },
    'src/app/features': { level: 3, allowedImports: ['src/app/app-shared'], pathAlias: '@features' },
  },
};
```

each entry is mapped as follows:

| arch-validator | `layers` field |
|-|-|
| layer key (`'src/app/features'`) | `name` |
| `key + '/**'` | `paths` |
| `pathAlias` | `alias` (`""` when absent) |
| `allowedImports` (other layer keys) | `canImportFrom` (the referenced layer's alias, or its key when it has no alias) |
| `level` (number) | `level` |

`description` is ignored. As in the arch-validator, an import into the same or a lower level is allowed and an import into a higher level only when the target is in `allowedImports`. Suppression comments of the validator are not taken over; set them with [`suppressionComments`](#architecturelayer-boundary-violation).

If both `layers` and `layersSource` are set, `layersSource` wins. The plugin logs a drift warning listing layers that exist on only one side or differ in `paths` / `canImportFrom` / `level` (layers are paired by alias, else by name).

---

## `codingStandard`

Points agents to the project's coding conventions.

```json
"codingStandard": {
  "path": "docs/coding-standard.md",
  "enforceOn": ["feature-impl", "bug-fix"]
}
```

| Field | Type | Required | Description |
|-|-|-|-|
| `path` | string | yes | Markdown or text file with the conventions, relative to the project root or absolute |
| `enforceOn` | (`transform` \| `generate` \| `feature-impl` \| `bug-fix`)[] | no | Operations the standard applies to; all of them when omitted |

The platform code (MCP tools, rules, transformations) does not read this section; the loader only passes it through. It is read by:

- the agent `ngm-feature-planner`: takes the path from this section (asks for a standard when it is absent), writes it into the execution plan header and refers to it at each code-writing step;
- the agent `ngm-feature-implementer`: reads the file before writing code when the plan names it;
- the skill `ngm-feature-blueprint`: tells the planner to add the path to the plan header, note constraints at the relevant steps and list them in `overview.md`.

No agent or skill evaluates `enforceOn`; the bug-fix agents (`ngm-bug-fixer`, `ngm-bug-blueprint`) do not read the section.

---

## Schema and IDE autocompletion

The full JSON Schema is at `packages/core/schemas/angular-modernizer.schema.json`. Add the following to your `.angular-modernizer.json` to get IDE autocompletion and inline validation:

```json
"$schema": "./node_modules/@angular-modernizer/core/schemas/angular-modernizer.schema.json"
```
