# IntelliJ IDEA Indexing - Quick Reference Guide

This is a quick reference guide for developers working with IntelliJ's indexing system.

## Quick Start

### Creating a Simple Index

```java
public class MyIndex extends ScalarIndexExtension<String> {
  public static final ID<String, Void> ID = ID.create("my.index.id");
  
  @Override public @NotNull ID<String, Void> getName() { return ID; }
  @Override public int getVersion() { return 1; }
  @Override public boolean dependsOnFileContent() { return true; }
  
  @Override
  public @NotNull DataIndexer<String, Void, FileContent> getIndexer() {
    return inputData -> {
      // Extract data from inputData and return Map<Key, Void>
      return Map.of("key", null);
    };
  }
  
  @Override
  public @NotNull KeyDescriptor<String> getKeyDescriptor() {
    return EnumeratorStringDescriptor.INSTANCE;
  }
  
  @Override
  public @NotNull FileBasedIndex.InputFilter getInputFilter() {
    return new DefaultFileTypeSpecificInputFilter(StdFileTypes.JAVA);
  }
}
```

### Querying an Index

```java
// Find all files containing a key
Collection<VirtualFile> files = FileBasedIndex.getInstance()
  .getContainingFiles(MyIndex.ID, "key", 
    GlobalSearchScope.projectScope(project));

// Get all values for a key
List<Value> values = FileBasedIndex.getInstance()
  .getValues(MyIndex.ID, "key", 
    GlobalSearchScope.projectScope(project));

// Process all keys
FileBasedIndex.getInstance().processAllKeys(
  MyIndex.ID,
  key -> {
    System.out.println(key);
    return true; // continue
  },
  GlobalSearchScope.projectScope(project),
  null
);
```

## Core Classes Reference

### FileBasedIndexExtension<K, V>
**Location**: `platform/indexing-api/src/com/intellij/util/indexing/FileBasedIndexExtension.java`

The main extension point for creating indexes.

**Key Methods**:
- `getName()` - Unique index ID
- `getIndexer()` - Returns the data indexer
- `getKeyDescriptor()` - Key serialization
- `getValueExternalizer()` - Value serialization
- `getInputFilter()` - Which files to index
- `dependsOnFileContent()` - Need file content?
- `getVersion()` - Index version (increment to rebuild)

### DataIndexer<K, V, Input>
**Location**: `platform/util/src/com/intellij/util/indexing/DataIndexer.java`

Transforms input to key-value pairs.

```java
public interface DataIndexer<Key, Value, Data> {
  @NotNull Map<Key, Value> map(@NotNull Data inputData);
}
```

### FileBasedIndex
**Location**: `platform/indexing-api/src/com/intellij/util/indexing/FileBasedIndex.java`

Main service for querying indexes.

**Key Methods**:
```java
// Get service instance
FileBasedIndex.getInstance()

// Query methods
getValues(ID<K,V>, K key, GlobalSearchScope)
getContainingFiles(ID<K,V>, K key, GlobalSearchScope)
processAllKeys(ID<K,?>, Processor<K>, GlobalSearchScope, IdFilter)

// Update methods
requestRebuild(ID<?,?>)
ensureUpToDate(ID<K,?>, Project, GlobalSearchScope)
```

## Index Types

| Type | Base Class | Use Case | Example |
|------|-----------|----------|---------|
| **Scalar Index** | `ScalarIndexExtension<K>` | Simple key presence | File name index |
| **Map Index** | `FileBasedIndexExtension<K,V>` | Key with values | Method signatures |
| **Single Entry** | `SingleEntryFileBasedIndexExtension<V>` | One value per file | Word count cache |
| **Stub Index** | `StubIndexExtension<K,V>` | PSI element lookup | Java class index |

## Common Patterns

### Pattern: Content-Independent Index

```java
@Override
public boolean dependsOnFileContent() {
  return false; // Only need file metadata
}

@Override
public @NotNull DataIndexer<String, Void, FileContent> getIndexer() {
  return inputData -> {
    // Can only safely use: getFileName(), getFile()
    // DO NOT call: getContentAsText(), getPsiFile()
    String name = inputData.getFileName();
    return Map.of(name, null);
  };
}
```

### Pattern: File Type Filtering

```java
@Override
public @NotNull FileBasedIndex.InputFilter getInputFilter() {
  // Only index specific file types
  return new DefaultFileTypeSpecificInputFilter(
    StdFileTypes.JAVA, 
    StdFileTypes.XML
  );
}
```

### Pattern: Project-Only Indexing

```java
@Override
public @NotNull FileBasedIndex.InputFilter getInputFilter() {
  return new FileBasedIndex.ProjectSpecificInputFilter() {
    @Override
    public boolean acceptInput(@NotNull IndexedFile file) {
      Project project = file.getProject();
      if (project == null) return false;
      
      ProjectFileIndex fileIndex = ProjectFileIndex.getInstance(project);
      return fileIndex.isInSource(file.getFile());
    }
  };
}
```

### Pattern: Composite Indexing

```java
public class MyCompositeIndex extends FileBasedIndexExtension<K, V> {
  
  @Override
  public @NotNull DataIndexer<K, V, FileContent> getIndexer() {
    return new CompositeDataIndexer<K, V, SubIndexerType, String>() {
      
      @Override
      public SubIndexerType calculateSubIndexer(@NotNull IndexedFile file) {
        // Determine which sub-indexer to use based on file
        FileType type = file.getFileType();
        return getSubIndexerForType(type);
      }
      
      @Override
      public String getSubIndexerVersion(@NotNull SubIndexerType indexer) {
        return indexer.getClass().getName() + ":" + indexer.getVersion();
      }
      
      @Override
      public @NotNull Map<K, V> map(
          @NotNull FileContent inputData, 
          @NotNull SubIndexerType indexer) {
        return indexer.index(inputData);
      }
    };
  }
}
```

## Serialization

### Built-in Key Descriptors

```java
// String keys
EnumeratorStringDescriptor.INSTANCE

// Integer keys (optimized)
new InlineKeyDescriptor<Integer>() {
  public Integer fromInt(int n) { return n; }
  public int toInt(Integer i) { return i; }
}

// Custom keys
new KeyDescriptor<MyKey>() {
  public void save(@NotNull DataOutput out, MyKey value) {
    out.writeUTF(value.name);
    out.writeInt(value.id);
  }
  
  public MyKey read(@NotNull DataInput in) {
    return new MyKey(in.readUTF(), in.readInt());
  }
}
```

### Built-in Value Externalizers

```java
// Integer values
new DataExternalizer<Integer>() {
  public void save(@NotNull DataOutput out, Integer value) {
    out.writeInt(value);
  }
  
  public Integer read(@NotNull DataInput in) {
    return in.readInt();
  }
}

// String values
EnumeratorStringDescriptor.INSTANCE

// Void values (for ScalarIndex)
VoidDataExternalizer.INSTANCE
```

## Key File Locations

### Core Indexing API
```
platform/indexing-api/src/com/intellij/util/indexing/
  ├── FileBasedIndex.java              # Main service interface
  ├── FileBasedIndexExtension.java     # Extension base class
  ├── DataIndexer.java                 # Indexer interface
  ├── CompositeDataIndexer.java        # Composite indexing
  └── ScalarIndexExtension.java        # Scalar index base

platform/util/src/com/intellij/util/indexing/
  ├── IndexExtension.java              # Base extension
  ├── InvertedIndex.java               # Inverted index interface
  └── impl/MapReduceIndex.java         # Core implementation
```

### Main Implementation
```
platform/lang-impl/src/com/intellij/util/indexing/
  ├── FileBasedIndexImpl.java          # Main implementation
  ├── UnindexedFilesUpdater.java       # Indexing coordinator
  ├── RegisteredIndexes.java           # Index registry
  └── IndexVersion.java                # Version management
```

### Built-in Indexes
```
platform/indexing-impl/src/
  ├── com/intellij/psi/stubs/
  │   └── StubUpdatingIndex.java       # Stub index
  └── com/intellij/psi/impl/cache/impl/
      ├── id/IdIndex.java              # Identifier index
      └── todo/TodoIndex.java          # TODO index
```

## Debugging

### Enable Debug Logging

In `idea.properties`:
```properties
idea.log.index.debug=true
idea.log.stub.updates=true
```

### Force Rebuild

```java
FileBasedIndex.getInstance().requestRebuild(IndexId);
```

Via UI: **File → Invalidate Caches / Restart**

### Check Index State

```java
FileBasedIndexImpl impl = (FileBasedIndexImpl) FileBasedIndex.getInstance();
UpdatableIndex<K, V, ?, ?> index = impl.getIndex(IndexId);

System.out.println("Modification stamp: " + index.getModificationStamp());
```

### Profile Indexing

```java
long start = System.currentTimeMillis();
FileBasedIndex.getInstance().ensureUpToDate(IndexId, project, scope);
long duration = System.currentTimeMillis() - start;
System.out.println("Index update took: " + duration + "ms");
```

## Common Pitfalls

### ❌ Don't: Access file content when `dependsOnFileContent() = false`

```java
@Override public boolean dependsOnFileContent() { return false; }

@Override
public @NotNull DataIndexer<String, Void, FileContent> getIndexer() {
  return inputData -> {
    String content = inputData.getContentAsText().toString(); // ❌ WRONG!
    return Map.of(content, null);
  };
}
```

### ✅ Do: Only access metadata

```java
@Override public boolean dependsOnFileContent() { return false; }

@Override
public @NotNull DataIndexer<String, Void, FileContent> getIndexer() {
  return inputData -> {
    String name = inputData.getFileName(); // ✅ Correct
    return Map.of(name, null);
  };
}
```

### ❌ Don't: Use mutable objects as keys

```java
class MutableKey {
  public String value; // Mutable!
  
  @Override public int hashCode() { return value.hashCode(); }
  @Override public boolean equals(Object o) { ... }
}
```

### ✅ Do: Use immutable keys

```java
class ImmutableKey {
  public final String value; // Immutable
  
  public ImmutableKey(String value) { this.value = value; }
  
  @Override public int hashCode() { return value.hashCode(); }
  @Override public boolean equals(Object o) { ... }
}
```

### ❌ Don't: Forget to increment version after changes

```java
@Override
public int getVersion() {
  return 1; // ❌ Didn't update after changing index logic!
}
```

### ✅ Do: Increment version when index changes

```java
@Override
public int getVersion() {
  return 2; // ✅ Incremented after changing indexer logic
}
```

### ❌ Don't: Use expensive operations in indexer

```java
@Override
public @NotNull Map<String, Void> map(@NotNull FileContent inputData) {
  // ❌ Network call in indexer!
  String data = fetchFromServer(inputData.getFileName());
  return Map.of(data, null);
}
```

### ✅ Do: Keep indexing fast and local

```java
@Override
public @NotNull Map<String, Void> map(@NotNull FileContent inputData) {
  // ✅ Fast, local operation
  String name = inputData.getFileName();
  return Map.of(name.toLowerCase(), null);
}
```

## Performance Tips

1. **Minimize Content Loading**: Set `dependsOnFileContent() = false` if possible
2. **Efficient Filters**: Use specific file type filters to reduce indexed files
3. **Compact Serialization**: Use efficient serialization formats
4. **Avoid Large Values**: Consider storing references instead of full data
5. **Use Forward Index**: For indexes that update frequently
6. **Batch Operations**: Use `processAllKeys()` instead of individual queries
7. **Proper Scopes**: Use narrow search scopes when possible

## Index Lifecycle Summary

```
1. Initialization
   ├── Load registered indexes
   ├── Check versions
   └── Mark outdated indexes

2. Project Opening
   ├── Scan project files
   ├── Queue unindexed files
   └── Start background indexing

3. File Indexing
   ├── Load file content
   ├── Call DataIndexer.map()
   ├── Calculate diff (using forward index)
   ├── Update inverted index
   └── Update forward index

4. File Change
   ├── VFS event received
   ├── Mark file as dirty
   └── Trigger lazy update

5. Index Query
   ├── Ensure up-to-date
   ├── Query inverted index
   ├── Filter by scope
   └── Return results

6. Shutdown
   ├── Flush all indexes
   ├── Save version info
   └── Close storage
```

## Useful Links

- **IntelliJ Platform SDK**: https://plugins.jetbrains.com/docs/intellij/file-based-indexes.html
- **Index Extension Point**: `com.intellij.fileBasedIndex`
- **Stub Index Extension Point**: `com.intellij.stubIndex`
- **Source Code**: `platform/indexing-api/` and `platform/lang-impl/`

## Related Documentation

- [Indexing Architecture Overview](./indexing-architecture-overview.md) - Complete system architecture
- [Implementation Details](./indexing-implementation-details.md) - Code-level deep dive
- [Practical Examples](./indexing-practical-examples.md) - Working examples and patterns

---

**Note**: Always test custom indexes with large projects to ensure they don't impact IDE performance!
