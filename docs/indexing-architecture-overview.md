# IntelliJ IDEA Indexing Architecture Overview

## Introduction

IntelliJ IDEA uses a sophisticated indexing system to provide fast code navigation, search, and refactoring capabilities. This document explains the architecture, core concepts, and implementation details of the indexing system.

## Core Concepts

### What is Indexing?

Indexing is the process of analyzing source code files and building data structures (indexes) that enable fast lookup and search operations. IntelliJ's indexing system is based on the **inverted index** concept, which maps keys (identifiers, symbols, etc.) to the files containing them.

### Why Indexing?

Without indexes, every code search or navigation operation would require scanning all files in the project, which would be prohibitively slow for large codebases. Indexes enable:
- Fast "Find Usages" operations
- Quick navigation (Go to Declaration, Go to Implementation)
- Code completion suggestions
- Fast text search
- Refactoring operations

## Architecture Components

The indexing system consists of several layers:

### 1. Index Extension Layer (`IndexExtension`)

**Location**: `platform/util/src/com/intellij/util/indexing/IndexExtension.java`

This is the base abstraction that defines the contract for all indexes. An index extension specifies:

```java
public abstract class IndexExtension<Key, Value, Input> {
  // Unique identifier for this index
  public abstract @NotNull IndexId<Key, Value> getName();
  
  // The indexer that transforms input to key-value pairs
  public abstract @NotNull DataIndexer<Key, Value, Input> getIndexer();
  
  // Serialization for keys
  public abstract @NotNull KeyDescriptor<Key> getKeyDescriptor();
  
  // Serialization for values
  public abstract @NotNull DataExternalizer<Value> getValueExternalizer();
  
  // Version number - when changed, index is rebuilt
  public abstract int getVersion();
}
```

**Key Components**:
- **IndexId**: Unique name/identifier for the index
- **DataIndexer**: Transforms input data into key-value pairs for indexing
- **KeyDescriptor**: Defines how keys are serialized/deserialized
- **DataExternalizer**: Defines how values are serialized/deserialized
- **Version**: When incremented, triggers a complete rebuild of the index

### 2. File-Based Index Extension (`FileBasedIndexExtension`)

**Location**: `platform/indexing-api/src/com/intellij/util/indexing/FileBasedIndexExtension.java`

Extends `IndexExtension` to specialize for file-based indexing:

```java
public abstract class FileBasedIndexExtension<K, V> extends IndexExtension<K, V, FileContent> {
  // Filter to determine which files should be indexed
  public abstract @NotNull FileBasedIndex.InputFilter getInputFilter();
  
  // Whether this index depends on file content (vs just metadata)
  public abstract boolean dependsOnFileContent();
  
  // Whether to index directories (usually false)
  public boolean indexDirectories() { return false; }
}
```

**Key Features**:
- **InputFilter**: Determines which files are processed by this index (e.g., only `.java` files)
- **Content Dependency**: Some indexes only need file metadata (name, size), while others need full content
- **Extension Point**: Plugins can register new indexes via `com.intellij.fileBasedIndex` extension point

### 3. Data Indexer Interface (`DataIndexer`)

**Location**: `platform/util/src/com/intellij/util/indexing/DataIndexer.java`

The core interface responsible for transforming input into indexed data:

```java
public interface DataIndexer<Key, Value, Data> {
  /**
   * Map input to its associated data.
   * Returns a map where:
   * - Key: the index key (e.g., identifier name)
   * - Value: associated data (e.g., usage context)
   */
  @NotNull Map<Key, Value> map(@NotNull Data inputData);
}
```

**Purpose**: This is the "map" part of the map-reduce paradigm. Each file is processed independently, producing key-value pairs that will be merged into the index.

### 4. Inverted Index (`InvertedIndex`)

**Location**: `platform/util/src/com/intellij/util/indexing/InvertedIndex.java`

The core data structure interface:

```java
public interface InvertedIndex<Key, Value, Input> {
  // Get all values associated with a key
  @NotNull ValueContainer<Value> getData(@NotNull Key key) throws StorageException;
  
  // Process values for a key
  <E extends Exception> boolean withData(@NotNull Key key,
                                         @NotNull ValueContainerProcessor<Value, E> processor);
  
  // Map input and prepare index update
  @NotNull StorageUpdate mapInputAndPrepareUpdate(int inputId, @Nullable Input content);
  
  // Persist changes
  void flush() throws StorageException;
  
  // Clear all data
  void clear() throws StorageException;
}
```

**Key Features**:
- Maps keys to file IDs and associated values
- Supports efficient lookup by key
- Thread-safe read operations
- Atomic update operations

### 5. Map-Reduce Index Implementation (`MapReduceIndex`)

**Location**: `platform/util/src/com/intellij/util/indexing/impl/MapReduceIndex.java`

The primary implementation of `InvertedIndex`:

```java
public abstract class MapReduceIndex<Key, Value, Input> 
    implements InvertedIndex<Key, Value, Input> {
  
  private final DataIndexer<Key, Value, Input> myIndexer;
  private final IndexStorage<Key, Value> myStorage;
  private final ForwardIndex myForwardIndex;  // Optional
  private final ForwardIndexAccessor<Key, Value> myForwardIndexAccessor;
}
```

**Components**:
- **IndexStorage**: Persistent storage for the inverted index (key → file IDs + values)
- **ForwardIndex**: Optional forward index (file ID → data), used for efficient updates
- **DataIndexer**: The mapper function that extracts key-value pairs from input
- **ForwardIndexAccessor**: Provides access to forward index data

**Update Algorithm**:
1. Use forward index to get old data for the file
2. Call indexer to get new data for the file
3. Calculate diff between old and new data
4. Update inverted index: remove old keys, add new keys
5. Update forward index with new data

### 6. File-Based Index Service (`FileBasedIndex`)

**Location**: `platform/indexing-api/src/com/intellij/util/indexing/FileBasedIndex.java`

The main service interface for accessing indexes:

```java
public abstract class FileBasedIndex {
  public static FileBasedIndex getInstance() { ... }
  
  // Query methods
  public abstract @NotNull <K, V> List<V> getValues(
    @NotNull ID<K, V> indexId, 
    @NotNull K dataKey, 
    @NotNull GlobalSearchScope filter);
  
  public abstract @NotNull <K, V> Collection<VirtualFile> getContainingFiles(
    @NotNull ID<K, V> indexId,
    @NotNull K dataKey,
    @NotNull GlobalSearchScope filter);
  
  // Process all keys in an index
  public abstract <K> boolean processAllKeys(
    @NotNull ID<K, ?> indexId,
    @NotNull Processor<? super K> processor,
    @Nullable GlobalSearchScope scope,
    @Nullable IdFilter idFilter);
}
```

**Key Operations**:
- **getValues()**: Get all values for a given key in the index
- **getContainingFiles()**: Get all files that contain a given key
- **processAllKeys()**: Iterate over all keys in the index

### 7. File-Based Index Implementation (`FileBasedIndexImpl`)

**Location**: `platform/lang-impl/src/com/intellij/util/indexing/FileBasedIndexImpl.java`

The concrete implementation managing all indexes:

```java
@Internal
public final class FileBasedIndexImpl extends FileBasedIndexEx {
  private volatile RegisteredIndexes myRegisteredIndexes;
  private final ChangedFilesCollector myChangedFilesCollector;
  private final FilesToUpdateCollector myFilesToUpdateCollector;
  
  // Core indexing method
  private @NotNull FileIndexingResult doIndexFileContent(
    @Nullable Project project,
    @NotNull CachedFileContent content,
    @Nullable FileType cachedFileType,
    @NotNull ApplicationMode applicationMode,
    FileIndexingStamp indexingStamp) {
    // ... implementation
  }
}
```

**Responsibilities**:
- Manages all registered indexes
- Coordinates file scanning and indexing
- Handles incremental updates when files change
- Manages dumb mode (when indexes are being built)
- Provides query interface to all indexes

## Index Types

IntelliJ includes several built-in indexes:

### 1. Stub Index

**Location**: `platform/indexing-impl/src/com/intellij/psi/stubs/StubUpdatingIndex.java`

The most important index - stores abstract syntax tree (AST) stubs for fast PSI (Program Structure Interface) access.

**Key Features**:
- Stores serialized stub trees (lightweight AST representation)
- Enables fast PSI element lookup without parsing
- Language-specific stub implementations
- Supports both text and binary files

**Example**: Java class stub includes class name, method names, field names - but not method bodies.

### 2. ID Index

**Location**: `platform/indexing-impl/src/com/intellij/psi/impl/cache/impl/id/IdIndex.java`

Indexes all identifiers in source code files.

**Key-Value Structure**:
- **Key**: Identifier hash (e.g., hash of "myVariable")
- **Value**: Usage context mask (where it appears: code, comments, strings, etc.)

**Use Cases**:
- Find usages of identifiers
- Code completion
- Fast text search

### 3. File Name Index

Indexes files by their names for quick "Go to File" functionality.

### 4. File Type Index

Indexes files by their file type for filtering search results.

### 5. Todo Index

**Location**: `platform/todo/src/com/intellij/psi/impl/cache/impl/todo/TodoIndex.java`

Indexes TODO, FIXME comments for the TODO tool window.

### 6. Framework Detection Index

**Location**: `platform/lang-impl/src/com/intellij/framework/detection/impl/FrameworkDetectionIndex.java`

Detects frameworks used in the project (Spring, Hibernate, etc.).

## Indexing Workflow

### Initial Project Indexing

1. **Project Opening**:
   - `ProjectFileBasedIndexStartupActivity` is triggered
   - `UnindexedFilesScanner` scans project files
   - Determines which files need indexing

2. **File Scanning** (`UnindexedFilesScanner`):
   ```
   - Iterate through project roots
   - Apply file filters
   - Check if file needs reindexing (version, content changed)
   - Add files to indexing queue
   ```

3. **File Indexing** (`FileBasedIndexImpl.doIndexFileContent`):
   ```
   For each file:
   - Load file content (if needed)
   - For each registered index:
     - Check if index accepts this file (InputFilter)
     - Call DataIndexer.map() to extract key-value pairs
     - Calculate diff with existing data (using forward index)
     - Update inverted index
     - Update forward index
   ```

4. **Dumb Mode**:
   - While indexing, IDE enters "dumb mode"
   - Features requiring indexes are disabled
   - Progress indicator shows indexing progress
   - After indexing completes, IDE enters "smart mode"

### Incremental Updates

When a file changes:

1. **Change Detection**:
   - `ChangedFilesCollector` listens to VFS events
   - Collects changed files

2. **Lazy Updates**:
   - Small changes: update on next index access
   - Large changes: schedule background indexing task

3. **Update Process**:
   ```
   For each changed file:
   - Get old data from forward index
   - Compute new data by calling indexer
   - Calculate diff (added, removed, unchanged keys)
   - Update inverted index:
     - Remove file ID from old keys
     - Add file ID to new keys
   - Update forward index with new data
   ```

### Versioning and Rebuilds

**Index Version** (`IndexVersion.java`):
- Each index has a version number
- When version changes (code update, algorithm change):
  - Index is marked as "dirty"
  - Complete rebuild is triggered
- Version stored in: `system/index/<index-name>/version.dat`

**Triggers for Rebuild**:
- Index extension version incremented
- Index storage format changed
- VFS (Virtual File System) timestamp changed
- Corrupted index detected

## Composite Indexing

**Location**: `platform/indexing-api/src/com/intellij/util/indexing/CompositeDataIndexer.java`

Some indexes support multiple "sub-indexers" based on file type or other criteria:

```java
public interface CompositeDataIndexer<K, V, SubIndexerType, SubIndexerVersion> 
    extends DataIndexer<K, V, FileContent> {
  
  // Determine which sub-indexer to use for this file
  @Nullable SubIndexerType calculateSubIndexer(@NotNull IndexedFile file);
  
  // Get version for the sub-indexer
  @NotNull SubIndexerVersion getSubIndexerVersion(@NotNull SubIndexerType type);
  
  // Map using specific sub-indexer
  @NotNull Map<K, V> map(@NotNull FileContent inputData, 
                         @NotNull SubIndexerType indexerType);
}
```

**Example**: IdIndex uses different indexers for different languages:
- `JavaIdIndexer` for Java files
- `PlainTextIdIndexer` for plain text
- Language-specific indexers for other languages

**Benefits**:
- Extensible indexing without changing core index
- Per-language/per-file-type versioning
- Efficient storage of sub-indexer metadata

## Performance Optimizations

### 1. Parallel Indexing

**Location**: `platform/lang-impl/src/com/intellij/util/indexing/UnindexedFilesUpdater.java`

```java
public static int getNumberOfIndexingThreads() {
  // Use multiple cores for parallel indexing
  int threadCount = Math.max(1, 
    Math.min(DEFAULT_MAX_INDEXER_THREADS, 
             Runtime.getRuntime().availableProcessors() - 1));
  return threadCount;
}
```

- Files indexed in parallel on multiple threads
- Thread count configurable via system property
- Balances between performance and system responsiveness

### 2. Shared Indexes

- Pre-built indexes can be shared across multiple installations
- Especially useful for standard libraries (JDK, Android SDK)
- Downloaded from CDN instead of building locally
- Significant time savings on initial project setup

### 3. Forward Index Optimization

- Forward index stores per-file data
- Enables efficient updates by computing diff
- Without forward index, would need to reindex entire file
- Trade-off: storage space vs. update performance

### 4. Memory Management

- Indexes use memory-mapped files for efficient I/O
- Caching layers reduce disk access
- LowMemoryWatcher flushes caches under memory pressure
- Aggressive flushing during indexing to limit memory usage

### 5. Content Hashing

- File content hashed to detect changes
- Avoids reindexing unchanged files
- Hash stored in `IndexingStamp`
- Fast comparison without reading entire file

## Storage Layer

### Index Storage

**Location**: `platform/lang-impl/src/com/intellij/util/indexing/storage/`

Indexes are persisted to disk using:
- **PersistentHashMap**: Key-value storage with B-tree-like structure
- **Memory-mapped files**: For efficient I/O
- **Write-Ahead Log (WAL)**: For crash recovery (optional)
- **Compression**: Optional compression for values

**Storage Location**:
- System directory: `system/index/<index-name>/`
- Each index has its own directory
- Contains: storage files, forward index, version file

### Value Container

Stores all file IDs and values for a specific key:
```
Key "foo" -> ValueContainer {
  fileId1 -> value1,
  fileId2 -> value2,
  ...
}
```

## Thread Safety

- **Read operations**: Thread-safe, use lock-free structures where possible
- **Write operations**: Synchronized using read-write locks
- **Update transactions**: Atomic updates to maintain consistency
- **Dumb mode**: Prevents queries during index updates

## Debugging and Diagnostics

### Logging

- `FileBasedIndexImpl.LOG`: Main logging for index operations
- `IndexDebugProperties`: Debug flags for detailed logging
- Trace stub updates: `FileBasedIndexEx.doTraceStubUpdates()`

### Diagnostic Actions

- **Force Index Rebuild**: `ForceIndexRebuildAction.java`
- **Rescan Indexes**: `RescanIndexesAction.kt`
- **Index Statistics**: Available via internal actions

### Common Issues

1. **Stale indexes**: Resolved by invalidating caches
2. **Corrupted indexes**: Automatically detected and rebuilt
3. **Memory issues**: Indexes flushed more aggressively
4. **Performance problems**: Check thread count, disk I/O

## Extension Points

Plugins can extend the indexing system:

### 1. Register Custom Index

```xml
<extensions defaultExtensionNs="com.intellij">
  <fileBasedIndex implementation="com.example.MyCustomIndex"/>
</extensions>
```

### 2. Implement Custom Stub Elements

For language plugins:
```java
public class MyStubElement extends StubElement<MyPsiElement> {
  // Store essential information about PSI element
}
```

### 3. Custom File Type Indexers

For specialized file formats:
```java
public class MyFileTypeIndexer implements FileTypeIdIndexer {
  @Override
  public @NotNull Map<IdIndexEntry, Integer> map(@NotNull FileContent content) {
    // Extract identifiers from custom file format
  }
}
```

## Summary

The IntelliJ indexing system is a sophisticated multi-layered architecture that provides:

- **Fast code navigation**: Through comprehensive indexing of code structure
- **Incremental updates**: Efficient handling of file changes
- **Extensibility**: Plugin-friendly architecture
- **Scalability**: Handles large codebases efficiently
- **Robustness**: Automatic recovery from corruption

Key design principles:
- **Map-Reduce paradigm**: Parallel file processing, merged results
- **Inverted index structure**: Fast key-based lookups
- **Forward index**: Efficient incremental updates
- **Lazy evaluation**: Index updates only when needed
- **Version management**: Automatic rebuilds when needed

The indexing system is fundamental to IntelliJ's powerful IDE features, enabling instant code navigation and intelligent code assistance even in million-line codebases.
