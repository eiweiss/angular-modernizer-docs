# Worker Threads & Intelligent Mode Selection

> **Last Updated:** 2025-01-28
> **Back to:** [CLAUDE.md](../CLAUDE.md)

## Overview

The platform supports dual-mode parallelism with intelligent automatic mode selection:

1. **Intelligent Auto-Selection** (DEFAULT) - Automatically chooses optimal mode based on project complexity
2. **Promise-based parallelism** - 3-4x speedup, no build required, good for small-medium projects
3. **Worker thread parallelism** - 5-10x speedup, requires `pnpm build`, optimal for large projects (500+ files)

## Usage

```typescript
import { ParallelExecutor, ResourceManager } from '@angular-modernizer/core';

const executor = new ParallelExecutor(project, resourceManager, 4);

// Intelligent auto-selection (default)
const results1 = await executor.executeParallel(files, rules, {
  // autoSelectMode: true,  // DEFAULT: Enabled by default
  tsConfigPath: './tsconfig.json',
  workerScriptPath: './dist/workers/analysis-worker.js',
});

// Manual Promise-based (opt-out of auto-selection)
const results2 = await executor.executeParallel(files, rules, {
  autoSelectMode: false,
  maxWorkers: 4,
});

// Manual Worker threads (opt-out of auto-selection)
const results3 = await executor.executeParallel(files, rules, {
  autoSelectMode: false,
  useWorkerThreads: true,
  maxWorkers: 4,
  tsConfigPath: './tsconfig.json',
  workerScriptPath: './dist/workers/analysis-worker.js',
  pluginIds: ['architecture'],
  timeout: 30000,
});
```

## Performance Characteristics

**Auto-selection (DEFAULT):**

- 1.14x faster than manual worker mode (251.81s vs 279.83s on 1,500 files)
- Immediate correct decision eliminates warmup overhead

**Project Size Thresholds:**

- **Small projects (<100 files):** Auto-selects Promise mode (complexity score <35)
- **Medium projects (100-500 files):** Auto-selects Promise with monitoring (score 35-50) or Workers (score 50-75)
- **Large projects (500+ files):** Auto-selects Worker threads (complexity score >50)

**Memory Management:**

- Memory-aware worker count (2-8 workers based on system RAM)
- 2 workers (<4GB), 3 workers (4-8GB), 4 workers (8-16GB), 8 workers (16GB+)
- Isolated heaps prevent fragmentation
- Automatic recycling at 80% memory threshold
- Real-time monitoring aborts at 85% heap in Promise mode

**Real-time Monitoring:**

- Projection deferral: Wait 50 files before projecting peak memory (avoids JIT warmup false alarms)
- Automatic fallback to workers if Promise mode detects memory pressure
- Zero false alarms in validation testing

## Worker Thread Integration

### Key Architectural Constraints

**1. ts-morph Serialization**

- Project instances cannot be transferred between threads
- Each worker creates its own isolated Project instance
- Communication uses file paths, not AST objects
- Workers load plugins independently

**2. Plugin Loading in Workers**

- Add plugin cases to `packages/worker/src/analysis-worker.ts` `loadPluginRules()` function
- Plugins are cached per worker for efficiency
- PublicApi is created per-worker instance
- Worker package exports `getWorkerScriptPath()` helper

**3. Feature Flag Pattern**
Use `useWorkerThreads` for safe rollout:

```typescript
const results = await executor.executeParallel(files, rules, {
  useWorkerThreads: process.env.USE_WORKER_THREADS === 'true',
  maxWorkers: 4,
  tsConfigPath: './tsconfig.json',
});
```

**4. ResourceManager Integration**
Automatic worker lifecycle management:

- Memory tracking per worker (80% threshold triggers recycling)
- Crash recovery (max 3 crashes per worker in 60s window)
- Graceful shutdown with cleanup

**5. Testing Strategy**
Tests adapt to environment:

- Unit tests validate orchestration logic with in-memory FS
- Integration tests validate Promise-based mode (always available)
- Worker-specific tests skip in dev (require compiled worker script)
- Real-world validation requires `pnpm build` first

## Adding New Plugin to Workers

```typescript
// In packages/worker/src/analysis-worker.ts
import { YourPlugin } from '@angular-modernizer/plugin-your-plugin';

function loadPluginRules(pluginIds: string[]): AnalysisRule[] {
  // ...
  switch (pluginId) {
    case 'your-plugin':
    case '@angular-modernizer/plugin-your-plugin': {
      const plugin = new YourPlugin();
      rules = plugin.getAnalysisRules();
      break;
    }
    // ... other plugins
  }
  // ...
}
```

## Testing

```bash
# Build required for worker threads
pnpm run build
pnpm --filter @angular-modernizer/worker build

# Test intelligent mode selection and worker performance
ENABLE_WORKER_TESTS=true pnpm --filter @angular-modernizer/worker test
```

## Components

### ProjectMetricsCollector

Fast file statistics via fs.stat() without parsing:

- File count
- Lines of code (LOC)
- File sizes
- Memory pressure estimation

### ComplexityScorer

Multi-factor scoring producing 0-100 scale:

- File count: 30%
- LOC: 35%
- Outliers (large files): 20%
- Memory pressure: 15%

### ParallelismModeSelector

Intelligent decision logic with conservative thresholds:

- `promiseModeSafe`: 35 (Promise mode safe below this)
- `workerModePreferred`: 50 (Workers preferred above this)
- `workerModeRequired`: 75 (Workers required above this)

### PromiseModeMonitor

Real-time memory tracking:

- Projection deferral: `MIN_FILES_FOR_PROJECTION = 50`
- Critical threshold: 85% heap usage
- Automatic abort and fallback to workers

## Design Notes

**Conservative threshold tuning:** Lowered from 65/80 to 50/75 to catch medium-large projects earlier.

**Projection deferral:** Wait 50 files before projecting peak memory to avoid JIT warmup false alarms.

**Memory-aware worker count:** Scale with system RAM (2-8 workers) instead of CPU cores (which would default to 7+ workers and exhaust memory on large codebases).
