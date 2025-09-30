# IntelliJ IDEA Code Indexing Documentation

This directory contains comprehensive documentation about IntelliJ IDEA's code indexing system - how it works, how it's implemented, and how to use it.

## Documentation Overview

### 📚 [Indexing Architecture Overview](./indexing-architecture-overview.md)
**Recommended for**: All developers wanting to understand the indexing system

**Contents**:
- Introduction to indexing concepts
- Architecture components and layers
- Index types (Stub, ID, File Name, etc.)
- Indexing workflow (initial and incremental)
- Composite indexing and versioning
- Performance optimizations
- Extension points

**Key Topics**:
- What is indexing and why is it needed?
- Core abstractions: `IndexExtension`, `FileBasedIndexExtension`, `DataIndexer`, `InvertedIndex`
- Map-Reduce paradigm in indexing
- Built-in indexes: Stub, ID, File Name, Todo
- Initial vs. incremental indexing
- Forward index for efficient updates

---

### 🔧 [Implementation Details](./indexing-implementation-details.md)
**Recommended for**: Developers working on indexing internals or creating complex custom indexes

**Contents**:
- Core implementation classes
- Index lifecycle (initialization, shutdown)
- File processing pipeline
- Storage implementation
- Update algorithms with code examples
- Stub indexing deep dive
- Composite indexing mechanics
- Performance considerations

**Key Topics**:
- `FileBasedIndexImpl` - main coordinator
- `MapReduceIndex` - core data structure
- `RegisteredIndexes` - index registry
- Storage layer: `PersistentHashMap`, `ValueContainer`
- Incremental update algorithm with forward index
- Stub tree building and serialization
- Sub-indexer versioning
- Caching strategies and WAL

---

### 💡 [Practical Examples](./indexing-practical-examples.md)
**Recommended for**: Plugin developers creating custom indexes

**Contents**:
- Complete working examples of custom indexes
- Workflow diagrams
- Query examples
- Debugging and troubleshooting guide
- Common patterns and best practices

**Key Topics**:
- Simple string literal index
- Map index with values
- Forward index for caching
- Complete workflow diagrams
- Finding classes, identifiers, TODO items
- Debug logging and index rebuild
- Performance profiling
- Common pitfalls and solutions

---

### ⚡ [Quick Reference Guide](./indexing-quick-reference.md)
**Recommended for**: Quick lookups and reminders

**Contents**:
- Quick start code snippets
- Core classes reference
- Index types comparison
- Common patterns
- Serialization examples
- Key file locations
- Debugging commands
- Common pitfalls
- Performance tips

**Key Topics**:
- Creating and querying indexes in 5 minutes
- One-page reference for core APIs
- Copy-paste ready code snippets
- Do's and Don'ts
- Troubleshooting checklist

---

## Reading Guide

### For First-Time Learners
1. Start with [Architecture Overview](./indexing-architecture-overview.md) to understand concepts
2. Read [Quick Reference](./indexing-quick-reference.md) for practical API overview
3. Try examples from [Practical Examples](./indexing-practical-examples.md)
4. Dive into [Implementation Details](./indexing-implementation-details.md) when needed

### For Plugin Developers
1. Check [Quick Reference](./indexing-quick-reference.md) for API quick start
2. Review [Practical Examples](./indexing-practical-examples.md) for patterns
3. Reference [Architecture Overview](./indexing-architecture-overview.md) for design decisions
4. Consult [Implementation Details](./indexing-implementation-details.md) for advanced topics

### For Core Contributors
1. Study [Implementation Details](./indexing-implementation-details.md) thoroughly
2. Understand [Architecture Overview](./indexing-architecture-overview.md) for context
3. Reference [Practical Examples](./indexing-practical-examples.md) for test cases
4. Use [Quick Reference](./indexing-quick-reference.md) as a cheat sheet

## Key Concepts Summary

### What is Indexing?
Indexing is the process of analyzing source code files and building data structures (indexes) that enable fast lookup and search operations. IntelliJ's indexing system uses **inverted indexes** to map keys (identifiers, symbols, etc.) to files containing them.

### Core Architecture
```
┌─────────────────────────────────────────────────────┐
│ FileBasedIndex (Service)                            │
│  - Query interface                                   │
│  - Ensures indexes are up-to-date                   │
└────────────────┬────────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────────┐
│ FileBasedIndexImpl (Implementation)                 │
│  - Manages all registered indexes                   │
│  - Coordinates scanning and indexing                │
│  - Handles incremental updates                      │
└────────────────┬────────────────────────────────────┘
                 │
         ┌───────┴───────┐
         │               │
┌────────▼─────┐  ┌─────▼──────────┐
│ Index 1      │  │ Index 2        │  ...
│ (Stub Index) │  │ (ID Index)     │
└──────────────┘  └────────────────┘
         │               │
         └───────┬───────┘
                 │
┌────────────────▼────────────────────────────────────┐
│ Storage Layer                                        │
│  - PersistentHashMap (inverted index)               │
│  - ForwardIndex (per-file data)                     │
│  - IndexingStamp (versioning)                       │
└─────────────────────────────────────────────────────┘
```

### Main Index Types

| Index | Purpose | Key | Value | Content Dependent |
|-------|---------|-----|-------|-------------------|
| **Stub Index** | PSI structure | Element type + name | Serialized stub tree | Yes |
| **ID Index** | Identifier search | Identifier hash | Usage context | Yes |
| **File Name** | File lookup | File name | None (scalar) | No |
| **File Type** | Type-based filtering | File type | None (scalar) | No |
| **Todo Index** | TODO comments | TODO pattern | TodoItem list | Yes |

### Indexing Workflow

**Initial Indexing**:
```
Project Open → Scan Files → Queue Unindexed → Background Indexing → Index Built
```

**Incremental Update**:
```
File Change → Mark Dirty → Lazy/Eager Update → Diff Calculate → Index Update
```

**Query**:
```
Query Request → Ensure Up-to-Date → Read Lock → Query Index → Return Results
```

## Important Classes

| Class | Location | Purpose |
|-------|----------|---------|
| `FileBasedIndex` | `indexing-api/.../FileBasedIndex.java` | Main service API |
| `FileBasedIndexExtension` | `indexing-api/.../FileBasedIndexExtension.java` | Index extension base |
| `DataIndexer` | `util/.../DataIndexer.java` | Maps input to key-values |
| `FileBasedIndexImpl` | `lang-impl/.../FileBasedIndexImpl.java` | Main implementation |
| `MapReduceIndex` | `util/.../impl/MapReduceIndex.java` | Index data structure |
| `StubUpdatingIndex` | `indexing-impl/.../StubUpdatingIndex.java` | Stub index implementation |
| `IdIndex` | `indexing-impl/.../id/IdIndex.java` | Identifier index |
| `UnindexedFilesUpdater` | `lang-impl/.../UnindexedFilesUpdater.java` | Indexing coordinator |

## Extension Points

### Register Custom Index
```xml
<extensions defaultExtensionNs="com.intellij">
  <fileBasedIndex implementation="com.example.MyIndex"/>
</extensions>
```

### Register Stub Index
```xml
<extensions defaultExtensionNs="com.intellij">
  <stubIndex implementation="com.example.MyStubIndex"/>
</extensions>
```

## Common Use Cases

### Create Index for New Language
1. Implement `FileBasedIndexExtension<K, V>`
2. Define `DataIndexer` to extract language-specific data
3. Implement serialization for keys and values
4. Register via plugin.xml
5. Create query utilities for clients

### Add Custom Search Functionality
1. Determine what to index (identifiers, symbols, patterns)
2. Create appropriate index type (scalar vs map)
3. Implement efficient indexer
4. Provide search API using `FileBasedIndex.getInstance()`

### Optimize Index Performance
1. Check `dependsOnFileContent()` - avoid content loading if possible
2. Use specific file type filters
3. Implement efficient serialization
4. Consider using forward index for frequently updated data
5. Profile with large projects

## Debugging Tips

### Enable Logging
```properties
# In idea.properties
idea.log.index.debug=true
idea.log.stub.updates=true
```

### Force Rebuild
**UI**: File → Invalidate Caches / Restart

**Code**:
```java
FileBasedIndex.getInstance().requestRebuild(indexId);
```

### Check Performance
```java
long start = System.currentTimeMillis();
// ... indexing operation ...
long duration = System.currentTimeMillis() - start;
LOG.info("Indexing took: " + duration + "ms");
```

## Additional Resources

### Official Documentation
- [IntelliJ Platform SDK - File-Based Indexes](https://plugins.jetbrains.com/docs/intellij/file-based-indexes.html)
- [IntelliJ Platform SDK - Stub Indexes](https://plugins.jetbrains.com/docs/intellij/stub-indexes.html)

### Source Code
- Indexing API: `platform/indexing-api/src/com/intellij/util/indexing/`
- Implementation: `platform/lang-impl/src/com/intellij/util/indexing/`
- Built-in indexes: `platform/indexing-impl/src/`

### Related Topics
- PSI (Program Structure Interface)
- VFS (Virtual File System)
- Dumb Mode and Smart Mode
- Project Model

## Contributing

When modifying indexing code:
1. Always increment index version when changing format/logic
2. Test with large projects (1M+ LOC)
3. Profile performance impact
4. Update relevant documentation
5. Add tests for new indexes

## Questions?

For questions about indexing:
1. Check these documentation files first
2. Search existing issues on GitHub
3. Ask on IntelliJ Platform Slack
4. Post on IntelliJ Platform Forum

---

**Last Updated**: 2025
**Maintainers**: IntelliJ Platform Team
