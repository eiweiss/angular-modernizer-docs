# CacheManager - Performance Optimization Guide

**Last Updated:** 2025-01-28

The `CacheManager` provides intelligent caching for AST parsing and analysis results with automatic invalidation based on file changes. It delivers **4.90x speedup** on real-world codebases with nearly 3,000 files.

## Features

- **SHA-256 Content Hashing** - Detects actual file changes beyond modification time
- **Automatic Invalidation** - Invalidates cache when files change
- **LRU Eviction** - Least Recently Used eviction when cache size limits are reached
- **Gzip Compression** - Optional compression to reduce disk usage
- **Dependency Tracking** - Invalidates analysis results when dependencies change
- **Statistics** - Real-time hit rate and performance metrics
- **Per-Tool Caches** - Each scan tool (scan-architecture, scan-solid, scan-angular, scan-all) keeps its own cache directory
- **Orchestration Integration** - Seamless integration with parallel and incremental analysis
- **Intelligent Mode Selection** - Automatic parallelism mode selection based on project complexity
- **Worker Thread Compatible** - Cache state shared across worker threads

## Quick Start

### Basic Usage

```typescript
import {
  Kernel,
  RealFileSystemAdapter,
  CacheConfig,
} from '@angular-modernizer/core';

const kernel = new Kernel({
  tsConfigPath: './tsconfig.json',
  fileSystem: new RealFileSystemAdapter(),
  plugins: [],
  cacheConfig: {
    enabled: true,
    cacheDir: '.angular-modernizer-cache',
    maxSize: 100 * 1024 * 1024, // 100MB
    maxAge: 24 * 60 * 60 * 1000, // 24 hours
    compression: true,
    version: '1.0.0',
  },
});

await kernel.initialize();

// Get cache manager
const cacheManager = kernel.getCacheManager();
```

### Unified Cache System (Specialized Scan Tools)

Each scan MCP tool keeps its own cache. `scan-all` reads and writes only `all/`; it does not read the caches of the individual scans (an earlier version did, but opened them with its own cache version `scan-all-2`, so their entries never matched and were invalidated on every run).

```typescript
// Cache structure (one CacheManager per directory):
// .angular-modernizer-cache/
// ├── architecture/
// │   ├── ast/{name}.{hash}.ast.json
// │   └── analysis/{name}.{hash}.analysis.json
// ├── solid/        (same layout)
// ├── angular/      (same layout)
// ├── all/          (scan-all, same layout)
// └── health/last-run.json   (compare-health snapshot)

// scan-all opens only its own cache
import {
  MultiCacheFallbackManager,
  createMultiCacheConfig,
} from '@angular-modernizer/core';

const multiCacheManager = new MultiCacheFallbackManager(
  fileSystem,
  ['all'],
  createMultiCacheConfig(rootPath),
);
```

Entries are stored at `<cacheDir>/(ast|analysis)/<name>.<hash>.(ast|analysis).json`. The path does not depend on the date: the former `<cacheDir>/YYYY/MM/DD/...` layout looked entries up under the current date, so entries written before midnight were never found afterwards. Now the `maxAge` (24 hours by default) applies as intended. Old date folders are ignored, not deleted.

**Cache Invalidation:**

```bash
# Clear all caches
rm -rf .angular-modernizer-cache/

# Clear specific tool cache
rm -rf .angular-modernizer-cache/architecture/
rm -rf .angular-modernizer-cache/solid/
rm -rf .angular-modernizer-cache/angular/
rm -rf .angular-modernizer-cache/all/
```

### Orchestration Integration

```typescript
// Cache works seamlessly with parallel and incremental analysis
const kernel = new Kernel({
  tsConfigPath: './tsconfig.json',
  fileSystem: new RealFileSystemAdapter(),
  plugins: [new SolidPlugin()],
  cacheConfig: { enabled: true },
});

await kernel.initialize();

// Parallel analysis with caching (intelligent mode selection enabled by default)
const results = await kernel.analyzeParallel(files, rules, {
  // autoSelectMode: true,  // DEFAULT: Enabled automatically
  enableProgress: true,
  // Cache is automatically used with intelligent mode selection
});

// Incremental analysis leverages cache for >90% hit rate
const incrementalResults = await kernel.analyzeIncremental({
  rules,
  baseBranch: 'main',
  enableProgress: true,
});
```

### Worker Thread Compatibility & Intelligent Mode Selection

```typescript
// Cache state is shared across worker threads
const kernel = new Kernel({
  cacheConfig: { enabled: true },
  // No orchestration config needed - intelligent mode selection active
});

// Intelligent mode selection automatically chooses:
// - Promise mode for small projects (<50 files)
// - Promise with monitoring for medium projects (50-500 files)
// - Worker threads for large projects (500+ files)
// Cache works seamlessly with all modes
const results = await kernel.analyzeParallel(files, rules, {
  // autoSelectMode: true,  // DEFAULT: Already enabled
  enableProgress: true,
});

// Manual worker thread mode (opt-out of auto-selection)
const manualResults = await kernel.analyzeParallel(files, rules, {
  autoSelectMode: false, // Disable intelligent selection
  useWorkerThreads: true, // Force worker mode
  maxWorkers: 4,
});

// Workers automatically benefit from cached AST and analysis results
// Memory-aware worker count: 2-8 workers based on system RAM
```

### Direct Usage

```typescript
import { CacheManager } from '@angular-modernizer/core';
import { InMemoryFileSystemAdapter } from '@angular-modernizer/core';

const fileSystem = new InMemoryFileSystemAdapter();
const cache = new CacheManager(fileSystem, {
  enabled: true,
  cacheDir: '/cache',
  maxSize: 50 * 1024 * 1024, // 50MB
  compression: true,
});

await cache.initialize();

// Cache AST
await fileSystem.writeFile('/app/component.ts', sourceCode);
await cache.setAst('/app/component.ts', astString);

// Retrieve cached AST
const cached = await cache.getAst('/app/component.ts');
if (cached) {
  console.log('Cache hit!');
  console.log('AST:', cached.ast);
  console.log('Content hash:', cached.metadata.contentHash);
}

// Cache analysis results
await cache.setAnalysisResult('/app/component.ts', results, [
  '/app/shared/types.ts',
  '/app/core/base.ts',
]);

// Get statistics
const stats = cache.getStats();
console.log(`Hit rate: ${stats.hitRate.toFixed(1)}%`);
console.log(`Total entries: ${stats.totalEntries}`);
console.log(`AST entries: ${stats.entriesByType.ast}`);
console.log(`Analysis entries: ${stats.entriesByType.analysis}`);
```

## Configuration

### CacheConfig

```typescript
interface CacheConfig {
  enabled: boolean; // Enable/disable caching
  cacheDir: string; // Cache directory path
  maxSize: number; // Max cache size in bytes (0 = unlimited)
  maxAge: number; // Max age in milliseconds (0 = unlimited)
  version: string; // Cache version (invalidates on mismatch)
  compression: boolean; // Enable gzip compression
}
```

### Default Configuration

```typescript
{
  enabled: true,
  cacheDir: '.angular-modernizer-cache',
  maxSize: 100 * 1024 * 1024,      // 100MB
  maxAge: 24 * 60 * 60 * 1000,     // 24 hours
  version: '1.0.0',
  compression: true
}
```

## Performance Characteristics

### Real-World Results

**Cache Performance on Production Codebase (2,947 files):**

- **4.90x speedup** on file loading (45ms → 9ms cold start)
- **323,502 files/sec** throughput with warm cache
- **>90% cache hit rate** on incremental analysis runs
- **Memory efficient** - 100MB cache handles 3000+ file codebase

**Sequential vs Cached Analysis:**

```
Sequential Analysis: 6.48s
Cached Analysis:     0.62s (10.45x speedup with cache)
Incremental Re-run: 0.62s (>90% cache hit rate)
```

### Micro-Benchmarks

Based on comprehensive benchmarks (see `__tests__/cache-performance.bench.ts`):

| Operation               | Performance       | Notes                    |
| ----------------------- | ----------------- | ------------------------ |
| Cache Hit               | 50,657 ops/sec    | 1.58x faster than writes |
| Cache Miss              | 208,606 ops/sec   | Ultra-fast null return   |
| Write Small AST (1KB)   | 32,015 ops/sec    | No compression           |
| Write Large AST (100KB) | 33,953 ops/sec    | No compression           |
| Write with Compression  | 60,157 ops/sec    | 10KB with gzip           |
| File Change Detection   | 52,052 ops/sec    | Unchanged files          |
| File Change Detection   | 298,571 ops/sec   | Changed files            |
| LRU Eviction            | 11,678 ops/sec    | When maxSize exceeded    |
| Statistics              | 2,882,646 ops/sec | Nearly instant           |
| Cache Invalidation      | 19,991 ops/sec    | Single file              |
| Clear Cache             | 965 ops/sec       | 100 entries              |

### Scaling Characteristics

The cache is highly efficient with size scaling:

- **100KB takes only 0.94x longer than 1KB** (essentially O(1) for practical sizes)
- Compression provides additional speed benefits (60K+ ops/sec with gzip)
- Memory usage remains constant regardless of codebase size

### Orchestration Performance

**Parallel Analysis with Cache:**

- **1.5-2x speedup** on 100+ files with parallel execution
- **Combined speedup** - Cache + Parallel = 6-8x improvement
- **Worker threads** - Cache compatible with true multi-threading

**Incremental Analysis:**

- **>90% cache hit rate** on subsequent runs with few changes
- **Git-based detection** - Only analyzes changed files
- **Dependency tracking** - Invalidates when related files change

## Cache Invalidation

### Automatic Invalidation

Cache entries are automatically invalidated when:

1. **File content changes** - Detected via SHA-256 content hash
2. **Cache version changes** - When `version` in config changes
3. **File is deleted** - File no longer exists
4. **Max age exceeded** - Entry older than `maxAge`
5. **Dependencies change** - For analysis results with dependencies

### Manual Invalidation

```typescript
// Invalidate specific file
await cache.invalidate('/app/component.ts', 'ast');
await cache.invalidate('/app/component.ts', 'analysis');

// Invalidate both AST and analysis
await cache.invalidate('/app/component.ts');

// Clear entire cache
await cache.clear();
```

## File Change Detection

The cache uses a multi-layered approach to detect file changes:

```typescript
interface FileChangeStatus {
  filePath: string;
  changed: boolean;
  reason?: 'version' | 'missing' | 'content' | 'mtime';
  oldHash?: string;
  newHash?: string;
}
```

1. **Version Check** - Fast check if cache version matches
2. **File Existence** - Verify file still exists
3. **Modification Time** - Quick check using `mtime`
4. **Content Hash** - SHA-256 hash comparison if mtime changed
5. **Max Age** - Check if entry has expired

This approach minimizes expensive hash computations while ensuring accuracy.

## Dependency Tracking

For analysis results that depend on other files:

```typescript
// Cache analysis with dependencies
await cache.setAnalysisResult('/app/component.ts', analysisResults, [
  '/app/shared/types.ts',
  '/app/core/base.component.ts',
]);

// Cache automatically invalidates if any dependency is deleted
const cached = await cache.getAnalysisResult('/app/component.ts');
// Returns null if dependencies changed
```

## LRU Eviction

When cache size exceeds `maxSize`, the Least Recently Used (LRU) eviction strategy removes oldest entries:

```typescript
const cache = new CacheManager(fileSystem, {
  maxSize: 50 * 1024 * 1024, // 50MB limit
  // ... other config
});

// Cache automatically evicts oldest entries when limit is reached
await cache.setAst('/file1.ts', largeAst); // Might trigger eviction
```

Eviction is based on file modification time (mtime), ensuring least recently accessed entries are removed first.

Eviction, `clear()` and the size statistics consider only the manager's own entries (`<cacheDir>/(ast|analysis)/*.(ast|analysis).json`). A `CacheManager` on the base `.angular-modernizer-cache` directory therefore no longer deletes other tools' caches in subdirectories or the health snapshot `health/last-run.json`.

## Compression

Enable gzip compression to reduce disk usage:

```typescript
const cache = new CacheManager(fileSystem, {
  compression: true,
  // ... other config
});

// Entries are automatically compressed/decompressed
await cache.setAst('/file.ts', largeAst);
const cached = await cache.getAst('/file.ts'); // Automatically decompressed
```

**Compression Characteristics:**

- Provides significant disk space savings for text-heavy AST data
- Minimal performance impact (60K+ ops/sec with compression)
- Compressed data stored as base64-encoded strings

## Cache Statistics

Track cache performance with built-in statistics:

```typescript
interface CacheStats {
  totalEntries: number;
  totalSize: number;
  hits: number;
  misses: number;
  hitRate: number; // Percentage (0-100)
  entriesByType: {
    ast: number;
    analysis: number;
  };
}

const stats = cache.getStats();
console.log(`
  Cache Statistics:
  - Total Entries: ${stats.totalEntries}
  - Total Size: ${(stats.totalSize / 1024 / 1024).toFixed(2)} MB
  - Hit Rate: ${stats.hitRate.toFixed(1)}%
  - Hits: ${stats.hits}
  - Misses: ${stats.misses}
  - AST Entries: ${stats.entriesByType.ast}
  - Analysis Entries: ${stats.entriesByType.analysis}
`);
```

## Best Practices

### 1. Choose Appropriate maxSize

```typescript
// For small projects (<1000 files)
maxSize: 50 * 1024 * 1024; // 50MB

// For medium projects (1000-5000 files)
maxSize: 100 * 1024 * 1024; // 100MB

// For large projects (>5000 files)
maxSize: 200 * 1024 * 1024; // 200MB

// For CI/CD (unlimited)
maxSize: 0; // No limit
```

### 2. Enable Cache for Orchestration

```typescript
// Always enable cache with orchestration features
const kernel = new Kernel({
  cacheConfig: {
    enabled: true,
    maxSize: 100 * 1024 * 1024,
    compression: true,
  },
  orchestrationConfig: {
    enableParallel: true,
    enableIncremental: true,
    useWorkerThreads: true, // Cache works with worker threads
  },
});
```

### 3. Use Version-Based Invalidation

Increment version when making breaking changes to cache format:

```typescript
cacheConfig: {
  version: '1.1.0',  // Changed from 1.0.0
  // All old cache entries automatically invalidated
}
```

### 4. Enable Compression for Large Codebases

```typescript
cacheConfig: {
  compression: true,  // Saves disk space with minimal performance cost
}
```

### 5. Monitor Cache Performance

```typescript
// Periodically log cache statistics
setInterval(() => {
  const stats = cache.getStats();
  if (stats.hitRate < 50) {
    console.warn('Low cache hit rate detected');
  }
}, 60000); // Every minute
```

### 6. Clear Cache on Major Changes

```typescript
// Clear cache when TypeScript version changes
// or when ts-morph API changes
await cache.clear();
```

### 7. Worker Thread Cache Strategy

```typescript
// Cache is shared across worker threads
// No special configuration needed - workers automatically benefit
const kernel = new Kernel({
  cacheConfig: { enabled: true },
  orchestrationConfig: {
    useWorkerThreads: true,
    maxWorkers: 4,
  },
});

// Cache hit rate improves as workers process more files
const results = await kernel.analyzeParallel(largeFileSet, rules);
```

## Testing

The cache system includes comprehensive benchmarks and integration tests:

```bash
# Run performance benchmarks
pnpm --filter @angular-modernizer/core bench

# Run cache-specific tests
pnpm --filter @angular-modernizer/core test -- --testPathPattern=cache

# Run orchestration tests (cache + parallel + workers)
pnpm --filter @angular-modernizer/core test -- --testPathPattern=orchestration
```

### Benchmark Results

```
================================================================================
Cache Performance Benchmarks
================================================================================
Benchmark                                     Ops/sec     Avg Time
--------------------------------------------------------------------------------
Write small AST (1KB, no compression)        32015.72        0.031ms
Write large AST (100KB, no compression)      33953.90        0.029ms
Write AST (10KB, with compression)           60157.61        0.017ms
Cache hit (read existing AST)                50657.63        0.020ms
Cache miss (non-existent entry)             208606.78        0.005ms
File change detection (unchanged)            52052.00        0.019ms
File change detection (changed)             298571.43        0.003ms
LRU eviction (size exceeded)                 11678.00        0.086ms
Cache statistics                           2882646.00        0.000ms
Cache invalidation (single file)             19991.00        0.050ms
Clear cache (100 entries)                      965.00        1.036ms
```

### Real-World Integration Tests

The cache system is validated through comprehensive real-world testing:

- **Production codebases (2,947+ files)**: 4.90x speedup validation
- **Incremental Analysis**: >90% cache hit rate testing
- **Worker Thread Compatibility**: Cache state sharing across threads
- **Orchestration Integration**: Parallel analysis with caching
- **Memory Management**: Large codebase memory efficiency testing

## Integration with Kernel

The Kernel automatically manages the CacheManager lifecycle:

```typescript
// Kernel initialization
const kernel = new Kernel({
  cacheConfig: { enabled: true },
});

await kernel.initialize(); // CacheManager initialized here

// Access cache manager
const cache = kernel.getCacheManager();

// Returns undefined if caching is disabled
if (!cache) {
  console.log('Caching is disabled');
}
```

### Platform Integration

The cache system integrates seamlessly with all platform features:

#### MCP Tools (17 Tools)

```typescript
// Cache is automatically used by all 17 MCP tools

// Analysis Tools (6) - All use unified cache system
await callTool('scan-all', {
  rootPath: '/workspace/app',
  // Reads from architecture/, solid/, angular/ caches
});

await callTool('scan-architecture', {
  rootPath: '/workspace/app',
  orchestration: {
    incremental: true, // >90% cache hit rate on subsequent runs
  },
});

// Planning & Transform (3) - Cache transformations
await callTool('transform-code', {
  filePath: '/workspace/app/src/app/user.component.ts',
  transformation: 'constructor-to-inject',
  // Cache validates before transformation
});

// Build & Validation (5) - Cache validation results
await callTool('validate-before-transform', {
  filePath: '/workspace/app/src/app/user.component.ts',
  transformation: 'standalone-component',
  // Cache validation checks
});

// Rollback & Learning (3) - Cache pattern recognition
await callTool('learn-patterns', {
  rootPath: '/workspace/app',
  category: 'architectural-patterns',
  // Cache discovered patterns
});
```

#### Orchestration Features

```typescript
// Cache enhances all orchestration capabilities
const kernel = new Kernel({
  cacheConfig: { enabled: true },
  orchestrationConfig: {
    enableParallel: true,
    enableIncremental: true,
    useWorkerThreads: true,
  },
});

// Combined performance: Cache + Parallel + Workers
const results = await kernel.analyzeParallel(files, rules);
```

#### Real-World Performance Stack

```
Base Analysis:        6.48s (sequential)
+ Cache:              0.62s (10.45x speedup)
+ Parallel:           0.45s (14.4x total speedup)
+ Worker Threads:     0.32s (20.25x total speedup)
+ Incremental:        0.08s (81x total speedup)
```

## Advanced Usage

### Custom Cache Key Generation

The cache uses MD5 hashes of normalized file paths for cache keys:

```typescript
// Internal implementation (for reference)
private getCacheKey(filePath: string, type: 'ast' | 'analysis'): string {
  const normalized = path.normalize(filePath);
  const hash = crypto.createHash('md5').update(normalized).digest('hex');
  return `${hash}.${type}.json`;
}
```

### Cache Metadata

Every cached entry includes comprehensive metadata:

```typescript
interface CacheMetadata {
  filePath: string; // Original file path
  contentHash: string; // SHA-256 hash of content
  mtime: number; // File modification time
  cachedAt: number; // Timestamp when cached
  size: number; // File size in bytes
  version: string; // Cache version
}

interface CachedAst {
  metadata: CacheMetadata;
  ast: string; // Serialized AST
}

interface CachedAnalysisResult {
  metadata: CacheMetadata;
  results: unknown[]; // Analysis results
  dependencies: string[]; // File dependencies
}
```

## Troubleshooting

### Low Hit Rate

If you're experiencing low cache hit rates:

1. **Check file modifications** - Frequent file changes invalidate cache
2. **Verify cache directory** - Ensure write permissions
3. **Increase maxSize** - Cache might be evicting too aggressively
4. **Check dependencies** - Analysis result dependencies might be changing

### High Memory Usage

If cache is consuming too much memory:

1. **Reduce maxSize** - Lower the cache size limit
2. **Enable compression** - Reduces storage footprint
3. **Reduce maxAge** - Cache entries expire sooner
4. **Clear cache periodically** - `await cache.clear()`

### Slow Performance

If cache operations are slow:

1. **Disable compression** - Trade space for speed
2. **Check disk I/O** - Slow filesystem might be bottleneck
3. **Use InMemoryFileSystemAdapter for tests** - Much faster
4. **Profile operations** - Use benchmarks to identify bottlenecks

## Related Documentation

- [Kernel API](./src/kernel.ts) - Kernel integration
- [File System Adapter](./src/file-system.ts) - File system abstraction
- [Orchestration System](./src/orchestration/) - Parallel and incremental analysis
- [Worker Thread Implementation](./src/orchestration/worker-thread.ts) - Multi-threaded execution
- [Cache Types](./src/cache/types.ts) - Type definitions
- [Performance Benchmarks](./__tests__/cache-performance.bench.ts) - Benchmark suite
- [MCP_README.md](../../MCP_README.md) - MCP tool integration
- [MCP_USAGE.md](../../MCP_USAGE.md) - MCP tool API reference (17 tools)
- [DEVELOPER_USAGE.md](../../DEVELOPER_USAGE.md) - Developer API usage guide
