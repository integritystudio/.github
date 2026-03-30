# Gzip Compression Support

**Version**: 1.7.0
**Status**: Implemented

## Overview

The observability-toolkit supports transparent reading of gzip-compressed JSONL files (`.jsonl.gz`). This reduces storage by 5-10x while maintaining full query functionality.

## Architecture

```
File Discovery
├─ listFiles() with regex pattern
│  └─ Matches both .jsonl and .jsonl.gz
└─ getFilesInRange() filters by date

File Reading (transparent decompression)
├─ isGzipFile() - detects .gz extension
├─ createJsonlStream() - returns appropriate stream
│  ├─ .jsonl → createReadStream()
│  └─ .jsonl.gz → createReadStream().pipe(createGunzip())
└─ streamJsonl() - uses createJsonlStream()

Indexing
├─ getIndexPath() strips .gz before adding .idx
│  └─ traces.jsonl.gz → traces.jsonl.idx
├─ buildIndex() uses createJsonlStream()
└─ readLinesByNumber() uses createJsonlStream()

Cleanup
└─ cleanupOldFiles() handles both extensions
```

## File Patterns

| Type | Pattern |
|------|---------|
| Traces | `traces-YYYY-MM-DD.jsonl(.gz)?` |
| Logs | `logs-YYYY-MM-DD.jsonl(.gz)?` |
| Metrics | `metrics-YYYY-MM-DD.jsonl(.gz)?` |
| LLM Events | `llm-events-YYYY-MM-DD.jsonl(.gz)?` |

## Usage

### Reading (automatic)
Compressed files are read transparently - no code changes needed:
```typescript
// Works with both .jsonl and .jsonl.gz
for await (const span of streamJsonl<TraceSpan>(filePath)) {
  // ...
}
```

### Creating compressed files
```bash
# Compress existing file (keeps original)
gzip -k traces-2026-01-29.jsonl

# Compress and remove original
gzip traces-2026-01-29.jsonl
```

### Storage comparison

| Records/Day | Uncompressed | Compressed | Savings |
|-------------|--------------|------------|---------|
| 10,000 | ~5-10 MB | ~0.5-1 MB | ~90% |
| 100,000 | ~50-100 MB | ~5-10 MB | ~90% |
| 1,000,000 | ~500 MB | ~50 MB | ~90% |

## Index Files

Index files (`.idx`) reference line numbers in the source file:
- `traces-2026-01-29.jsonl` → `traces-2026-01-29.jsonl.idx`
- `traces-2026-01-29.jsonl.gz` → `traces-2026-01-29.jsonl.idx`

Both compressed and uncompressed files share the same index path, enabling seamless compression of existing files without rebuilding indexes (though indexes will rebuild on first query if mtime changed).

## Implementation Details

### Key Functions

| Function | Location | Purpose |
|----------|----------|---------|
| `isGzipFile()` | file-utils.ts | Check if file has .gz extension |
| `createJsonlStream()` | file-utils.ts | Create read stream with optional gunzip |
| `streamJsonl()` | file-utils.ts | Async generator for JSONL records |
| `getIndexPath()` | indexer.ts | Get index path (strips .gz) |

### Dependencies
- Node.js built-in `zlib` module (no external dependencies)

## Performance

- **Memory**: Streaming decompression, O(1) memory regardless of file size
- **CPU**: ~10-20% overhead for decompression
- **I/O**: Reduced disk reads due to smaller file size (net positive)

## See Also

- [observability-audit.md](observability-audit.md) - OTel compliance
- [code-review.md](code-review.md) - Architecture overview
- [ROADMAP.md](ROADMAP.md) - Feature roadmap
