# IntelliJ IDEA Indexing - Implementation Details

This document provides deep technical details about how indexing is implemented in IntelliJ IDEA, with code-level explanations and examples.

## Table of Contents
1. [Core Implementation Classes](#core-implementation-classes)
2. [Index Lifecycle](#index-lifecycle)
3. [File Processing Pipeline](#file-processing-pipeline)
4. [Storage Implementation](#storage-implementation)
5. [Update Algorithm](#update-algorithm)
6. [Stub Indexing Deep Dive](#stub-indexing-deep-dive)
7. [Composite Indexing](#composite-indexing)
8. [Performance Considerations](#performance-considerations)

## Core Implementation Classes

### 1. FileBasedIndexImpl

**Location**: `platform/lang-impl/src/com/intellij/util/indexing/FileBasedIndexImpl.java`

This is the central implementation class managing all file-based indexes.

#### Key Fields

```java
@Internal
public final class FileBasedIndexImpl extends FileBasedIndexEx {
  // Registry of all indexes
  private volatile RegisteredIndexes myRegisteredIndexes;
  
  // Collects changed files from VFS events
  private final ChangedFilesCollector myChangedFilesCollector;
  
  // Queue of files to be indexed
  private final FilesToUpdateCollector myFilesToUpdateCollector;
  
  // Track indexed status of documents
  private final PerIndexDocumentVersionMap myLastIndexedDocStamps;
  
  // Read-write lock for thread safety
  final Lock myReadLock;
  public final Lock writeLock;
  
  // VFS creation timestamp for version tracking
  private volatile long vfsCreationStamp;
}
```

#### Core Indexing Method

```java
private @NotNull FileIndexingResult doIndexFileContent(
    @Nullable Project project,
    @NotNull CachedFileContent content,
    @Nullable FileType cachedFileType,
    @NotNull ApplicationMode applicationMode,
    FileIndexingStamp indexingStamp) {
  
  ProgressManager.checkCanceled();
  
  final VirtualFile file = content.getVirtualFile();
  int inputId = getFileId(file);
  
  // Create indexed file wrapper
  IndexedFileImpl indexedFile = new IndexedFileImpl(file, project);
  
  // Lists to collect index updates
  List<SingleIndexValueApplier<?>> appliers = new ArrayList<>();
  List<SingleIndexValueRemover> removers = new ArrayList<>();
  
  // Process each registered index
  for (ID<?, ?> indexId : getRequiredIndexes(indexedFile)) {
    try {
      // Get index instance
      UpdatableIndex<?, ?, FileContent, ?> index = getIndex(indexId);
      
      // Check if file should be indexed by this index
      if (!shouldIndexFile(indexedFile, indexId)) {
        // Remove from index if previously indexed
        if (isIndexed(inputId, indexId)) {
          removers.add(new SingleIndexValueRemover(index, inputId));
        }
        continue;
      }
      
      // Prepare index update
      if (isIndexed(inputId, indexId)) {
        // File was indexed before - incremental update
        appliers.add(new SingleIndexValueApplier<>(
          index, inputId, content, indexingStamp));
      } else {
        // First time indexing this file
        appliers.add(new SingleIndexValueApplier<>(
          index, inputId, content, indexingStamp));
      }
    } catch (RuntimeException e) {
      // Handle indexing errors
      requestIndexRebuildOnException(e, indexId);
    }
  }
  
  // Return result with all updates to apply
  return new FileIndexingResult(
    this, inputId, file, indexingStamp, 
    appliers, removers, false, true, 
    applicationMode, fileType, doTraceSharedIndexUpdates()
  );
}
```

**Key Points**:
- Processes one file at a time
- Iterates through all registered indexes
- Creates update operations (appliers/removers) for each index
- Returns batch of updates to be applied atomically
- Error handling prevents one broken index from affecting others

### 2. MapReduceIndex

**Location**: `platform/util/src/com/intellij/util/indexing/impl/MapReduceIndex.java`

The core implementation of the inverted index data structure.

#### Construction

```java
protected MapReduceIndex(
    @NotNull IndexExtension<Key, Value, Input> extension,
    @NotNull IndexStorage<Key, Value> storage,
    @Nullable ForwardIndex forwardIndex,
    @Nullable ForwardIndexAccessor<Key, Value> forwardIndexAccessor) {
  
  myIndexId = extension.getName();
  myExtension = extension;
  myIndexer = myExtension.getIndexer();
  myStorage = storage;
  myForwardIndex = forwardIndex;
  myForwardIndexAccessor = forwardIndexAccessor;
  
  // Setup value serialization checker in debug mode
  myValueSerializationChecker = 
    new ValueSerializationChecker<>(extension, getSerializationProblemReporter());
  
  // Register low memory watcher
  myLowMemoryFlusher = LowMemoryWatcher.register(() -> clearCaches());
}
```

#### Update Preparation

The index update is split into two phases for efficiency:

**Phase 1: Map Input**

```java
@Override
public @NotNull StorageUpdate mapInputAndPrepareUpdate(int inputId, @Nullable Input content) {
  if (content == null) {
    // Content deleted - remove from index
    return prepareUpdate(inputId, InputData.empty());
  }
  
  try {
    // Call the indexer to extract key-value pairs
    Map<Key, Value> data = myIndexer.map(content);
    
    // Validate serialization in debug mode
    if (myValueSerializationChecker != null) {
      myValueSerializationChecker.checkValueSerialization(data, content);
    }
    
    // Wrap in InputData
    InputData<Key, Value> inputData = new InputData<>(data);
    
    // Prepare the update
    return prepareUpdate(inputId, inputData);
    
  } catch (Exception e) {
    throw new MapReduceIndexMappingException(e, myIndexId);
  }
}
```

**Phase 2: Prepare Update**

```java
@Override
public @NotNull StorageUpdate prepareUpdate(int inputId, @NotNull InputData<Key, Value> data) {
  // Get old data from forward index
  InputData<Key, Value> oldData = readDataFromForwardIndex(inputId);
  
  // Calculate diff between old and new data
  UpdatedKeys<Key, Value> updatedKeys = calculateUpdateKeys(oldData, data);
  
  // Return update operation
  return new StorageUpdate() {
    @Override
    public void update() throws StorageException {
      updateIndex(inputId, updatedKeys, data);
    }
  };
}

private void updateIndex(int inputId, 
                        UpdatedKeys<Key, Value> updatedKeys,
                        InputData<Key, Value> newData) throws StorageException {
  try {
    // 1. Remove from old keys
    for (Key key : updatedKeys.getRemovedKeys()) {
      myStorage.removeValue(key, inputId);
    }
    
    // 2. Add to new keys
    for (Map.Entry<Key, Value> entry : updatedKeys.getAddedKeys().entrySet()) {
      myStorage.addValue(entry.getKey(), inputId, entry.getValue());
    }
    
    // 3. Update changed keys
    for (Map.Entry<Key, Value> entry : updatedKeys.getUpdatedKeys().entrySet()) {
      myStorage.updateValue(entry.getKey(), inputId, entry.getValue());
    }
    
    // 4. Update forward index
    if (myForwardIndex != null) {
      myForwardIndexAccessor.put(inputId, newData);
    }
    
    // Increment modification stamp
    myModificationStamp.incrementAndGet();
    
  } catch (IOException e) {
    throw new StorageException(e);
  }
}
```

**Key Algorithm Points**:
1. **Diff Calculation**: Compare old and new data to minimize storage operations
2. **Atomic Updates**: All changes for a file applied together
3. **Forward Index**: Enables efficient incremental updates
4. **Error Handling**: StorageException thrown if update fails

### 3. RegisteredIndexes

**Location**: `platform/lang-impl/src/com/intellij/util/indexing/RegisteredIndexes.java`

Manages the registry of all available indexes.

```java
@ApiStatus.Internal
public final class RegisteredIndexes {
  private final IndexConfiguration configuration;
  private final Map<ID<?, ?>, UpdatableIndex<?, ?, FileContent, ?>> indexes;
  
  // Get index by ID
  public <K, V> @NotNull UpdatableIndex<K, V, FileContent, ?> getIndex(ID<K, V> indexId) {
    UpdatableIndex<K, V, FileContent, ?> index = 
      (UpdatableIndex<K, V, FileContent, ?>)indexes.get(indexId);
    
    if (index == null) {
      throw new IllegalStateException("Index not registered: " + indexId);
    }
    
    return index;
  }
  
  // Initialize all indexes on startup
  public static @NotNull RegisteredIndexes initialize() {
    // Collect all FileBasedIndexExtension implementations
    List<FileBasedIndexExtension<?, ?>> extensions = 
      FileBasedIndexExtension.EXTENSION_POINT_NAME.getExtensionList();
    
    Map<ID<?, ?>, UpdatableIndex<?, ?, FileContent, ?>> indexes = new HashMap<>();
    
    for (FileBasedIndexExtension<?, ?> extension : extensions) {
      try {
        // Create storage layout
        VfsAwareIndexStorageLayout<?, ?> layout = 
          createStorageLayout(extension);
        
        // Create index instance
        UpdatableIndex<?, ?, FileContent, ?> index = 
          createIndex(extension, layout);
        
        indexes.put(extension.getName(), index);
      } catch (IOException e) {
        LOG.error("Failed to initialize index: " + extension.getName(), e);
      }
    }
    
    return new RegisteredIndexes(indexes);
  }
}
```

## Index Lifecycle

### Initialization Sequence

1. **Application Startup**:
   ```java
   // FileBasedIndexImpl constructor
   public FileBasedIndexImpl(@NotNull CoroutineScope coroutineScope) {
     // Initialize lock
     ReadWriteLock lock = new ReentrantReadWriteLock();
     myReadLock = lock.readLock();
     writeLock = lock.writeLock();
     
     // Load VFS creation stamp
     vfsCreationStamp = loadVfsCreationStamp();
     
     // Initialize registered indexes
     myRegisteredIndexes = RegisteredIndexes.initialize();
     
     // Setup listeners
     setupListeners();
   }
   ```

2. **Version Check**:
   ```java
   // IndexVersion.java
   private static @NotNull IndexVersion getIndexVersion(@NotNull ID<?, ?> indexName) {
     try {
       Path versionFile = IndexInfrastructure.getVersionFile(indexName);
       try (DataInputStream in = new DataInputStream(
           new BufferedInputStream(Files.newInputStream(versionFile)))) {
         
         IndexVersion version = new IndexVersion(in);
         
         // Check if version matches current
         int currentVersion = getIndexExtension(indexName).getVersion();
         if (version.myIndexVersion != currentVersion) {
           // Version mismatch - needs rebuild
           return NON_EXISTING_INDEX_VERSION;
         }
         
         return version;
       }
     } catch (IOException e) {
       // Version file doesn't exist or corrupted
       return NON_EXISTING_INDEX_VERSION;
     }
   }
   ```

3. **Project Opening**:
   ```java
   // ProjectFileBasedIndexStartupActivity.kt
   override suspend fun execute(project: Project) {
     // Start scanning for unindexed files
     val scanner = UnindexedFilesScanner(project)
     scanner.queue()
   }
   ```

### Shutdown Sequence

```java
// FileBasedIndexImpl.ShutDownIndexesTask
static final class ShutDownIndexesTask implements Runnable {
  @Override
  public void run() {
    // 1. Stop accepting new indexing requests
    markShutdownStarted();
    
    // 2. Wait for ongoing indexing to complete
    waitForIndexingCompletion();
    
    // 3. Flush all indexes
    for (UpdatableIndex<?, ?, ?, ?> index : getAllIndexes()) {
      try {
        index.flush();
      } catch (StorageException e) {
        LOG.error("Error flushing index: " + index.getIndexId(), e);
      }
    }
    
    // 4. Dispose indexes
    for (UpdatableIndex<?, ?, ?, ?> index : getAllIndexes()) {
      try {
        index.dispose();
      } catch (Exception e) {
        LOG.error("Error disposing index: " + index.getIndexId(), e);
      }
    }
  }
}
```

## File Processing Pipeline

### 1. File Scanning

**UnindexedFilesScanner** identifies files that need indexing:

```java
// UnindexedFilesScanner.kt
class UnindexedFilesScanner(private val project: Project) {
  
  fun queue() {
    // Collect files that need indexing
    val filesToIndex = collectFilesToIndex()
    
    // Submit to background executor
    UnindexedFilesScannerExecutor.getInstance(project)
      .submitTask(UnindexedFilesIndexer(project, filesToIndex))
  }
  
  private fun collectFilesToIndex(): Collection<FileIndexingRequest> {
    val result = mutableListOf<FileIndexingRequest>()
    
    // Iterate project content roots
    for (contentRoot in project.contentRoots) {
      VfsUtilCore.iterateChildrenRecursively(contentRoot, 
        filter = { file -> shouldVisitFile(file) },
        iterator = { file ->
          if (needsIndexing(file)) {
            result.add(FileIndexingRequest.updateRequest(file))
          }
          true // continue iteration
        }
      )
    }
    
    return result
  }
  
  private fun needsIndexing(file: VirtualFile): Boolean {
    if (!file.isValid) return false
    
    val fileId = FileBasedIndex.getFileId(file)
    
    // Check each index
    for (indexId in getAllIndexIds()) {
      val indexingState = getIndexingState(fileId, indexId)
      
      if (indexingState == FileIndexingState.OUT_DATED) {
        return true
      }
    }
    
    return false
  }
}
```

### 2. File Content Loading

**CachedFileContent** provides efficient access to file content:

```java
public interface CachedFileContent {
  // Get file reference
  @NotNull VirtualFile getVirtualFile();
  
  // Get file content (lazy loaded)
  byte @NotNull [] getContent();
  
  // Get charset
  @NotNull Charset getCharset();
  
  // Get PSI file (if applicable, lazy created)
  @Nullable PsiFile getPsiFile();
  
  // Get file type
  @NotNull FileType getFileType();
}
```

### 3. Parallel Processing

**UnindexedFilesIndexer** processes files in parallel:

```java
class UnindexedFilesIndexer(
  private val project: Project,
  private val files: Collection<FileIndexingRequest>
) : Task.Backgroundable(project, "Indexing", true) {
  
  override fun run(indicator: ProgressIndicator) {
    val threadCount = UnindexedFilesUpdater.getNumberOfIndexingThreads()
    
    // Create thread pool
    val executor = AppExecutorUtil.createBoundedApplicationPoolExecutor(
      "Indexing", threadCount)
    
    try {
      // Process files in parallel
      val futures = files.map { fileRequest ->
        executor.submit<FileIndexingResult> {
          processFile(fileRequest, indicator)
        }
      }
      
      // Wait for all to complete
      futures.forEach { future ->
        try {
          val result = future.get()
          applyIndexingResult(result)
        } catch (e: Exception) {
          LOG.error("Error indexing file", e)
        }
      }
      
    } finally {
      executor.shutdown()
    }
  }
  
  private fun processFile(
    request: FileIndexingRequest,
    indicator: ProgressIndicator
  ): FileIndexingResult {
    
    indicator.checkCanceled()
    
    // Load file content
    val content = loadContent(request.file)
    
    // Index file
    return FileBasedIndexImpl.getInstance()
      .indexFileContent(project, content)
  }
}
```

## Storage Implementation

### IndexStorage Interface

```java
public interface IndexStorage<Key, Value> {
  
  // Add value for a key
  void addValue(Key key, int inputId, Value value) throws StorageException;
  
  // Remove value for a key
  void removeValue(Key key, int inputId) throws StorageException;
  
  // Get all values for a key
  @NotNull ValueContainer<Value> read(Key key) throws StorageException;
  
  // Clear all data
  void clear() throws StorageException;
  
  // Flush changes to disk
  void flush() throws StorageException;
  
  // Close storage
  void close() throws StorageException;
}
```

### PersistentHashMap-based Storage

Most indexes use **PersistentHashMap** for storage:

```java
public class VfsAwareMapReduceIndex<Key, Value> 
    extends MapReduceIndexBase<Key, Value, FileContent> {
  
  private final PersistentHashMap<Key, ValueContainer<Value>> myStorage;
  
  public VfsAwareMapReduceIndex(
      @NotNull IndexExtension<Key, Value, FileContent> extension,
      @NotNull File storageFile) throws IOException {
    
    // Create persistent hash map
    myStorage = new PersistentHashMap<>(
      storageFile,
      extension.getKeyDescriptor(),
      new ValueContainerExternalizer<>(extension.getValueExternalizer())
    );
  }
  
  @Override
  public void addValue(Key key, int inputId, Value value) throws StorageException {
    try {
      myStorage.appendData(key, appender -> {
        ValueContainerImpl<Value> container = read(key);
        container.addValue(inputId, value);
        
        // Serialize updated container
        DataOutputStream out = new DataOutputStream(appender);
        container.saveTo(out);
      });
    } catch (IOException e) {
      throw new StorageException(e);
    }
  }
  
  @Override
  public ValueContainer<Value> read(Key key) throws StorageException {
    try {
      byte[] data = myStorage.get(key);
      if (data == null) {
        return ValueContainerImpl.createNewValueContainer();
      }
      
      DataInputStream in = new DataInputStream(
        new ByteArrayInputStream(data));
      
      ValueContainerImpl<Value> container = 
        ValueContainerImpl.createNewValueContainer();
      container.readFrom(in);
      
      return container;
      
    } catch (IOException e) {
      throw new StorageException(e);
    }
  }
}
```

### ValueContainer Structure

A **ValueContainer** stores all file IDs and associated values for a specific key:

```java
public interface ValueContainer<Value> {
  
  // Iterator over all (fileId, value) pairs
  interface ValueIterator<Value> {
    boolean hasNext();
    void next();
    int getCurrentFileId();
    Value getCurrentValue();
  }
  
  // Iterate over all values
  @NotNull ValueIterator<Value> getValueIterator();
  
  // Process each value
  boolean forEach(@NotNull BiPredicate<? super Integer, ? super Value> processor);
  
  // Get size
  int size();
}

// Implementation
class ValueContainerImpl<Value> implements ValueContainer<Value> {
  // Map from fileId to value
  private Int2ObjectMap<Value> myData;
  
  @Override
  public boolean forEach(BiPredicate<? super Integer, ? super Value> processor) {
    for (Int2ObjectMap.Entry<Value> entry : myData.int2ObjectEntrySet()) {
      if (!processor.test(entry.getIntKey(), entry.getValue())) {
        return false; // Stop iteration
      }
    }
    return true; // Processed all
  }
  
  void addValue(int fileId, Value value) {
    myData.put(fileId, value);
  }
  
  void removeValue(int fileId) {
    myData.remove(fileId);
  }
  
  // Serialization
  void saveTo(DataOutput out) throws IOException {
    out.writeInt(myData.size());
    for (Int2ObjectMap.Entry<Value> entry : myData.int2ObjectEntrySet()) {
      out.writeInt(entry.getIntKey());
      valueExternalizer.save(out, entry.getValue());
    }
  }
  
  void readFrom(DataInput in) throws IOException {
    int size = in.readInt();
    myData = new Int2ObjectOpenHashMap<>(size);
    
    for (int i = 0; i < size; i++) {
      int fileId = in.readInt();
      Value value = valueExternalizer.read(in);
      myData.put(fileId, value);
    }
  }
}
```

## Update Algorithm

### Incremental Update with Forward Index

The forward index enables efficient incremental updates:

```java
// Without forward index - must reindex entire file
public void updateWithoutForwardIndex(int inputId, Input newContent) {
  // 1. Get all keys currently in index for this file
  Set<Key> oldKeys = getAllKeysForFile(inputId); // EXPENSIVE!
  
  // 2. Index new content
  Map<Key, Value> newData = indexer.map(newContent);
  
  // 3. Calculate diff
  Set<Key> keysToRemove = new HashSet<>(oldKeys);
  keysToRemove.removeAll(newData.keySet());
  
  // 4. Update index
  for (Key key : keysToRemove) {
    storage.removeValue(key, inputId);
  }
  for (Map.Entry<Key, Value> entry : newData.entrySet()) {
    storage.addValue(entry.getKey(), inputId, entry.getValue());
  }
}

// With forward index - efficient update
public void updateWithForwardIndex(int inputId, Input newContent) {
  // 1. Read old data from forward index - O(1)
  Map<Key, Value> oldData = forwardIndex.get(inputId);
  
  // 2. Index new content
  Map<Key, Value> newData = indexer.map(newContent);
  
  // 3. Calculate diff - fast comparison
  UpdatedKeys<Key, Value> diff = calculateDiff(oldData, newData);
  
  // 4. Update inverted index - only changed keys
  for (Key key : diff.getRemovedKeys()) {
    storage.removeValue(key, inputId);
  }
  for (Map.Entry<Key, Value> entry : diff.getAddedKeys().entrySet()) {
    storage.addValue(entry.getKey(), inputId, entry.getValue());
  }
  for (Map.Entry<Key, Value> entry : diff.getUpdatedKeys().entrySet()) {
    storage.updateValue(entry.getKey(), inputId, entry.getValue());
  }
  
  // 5. Update forward index
  forwardIndex.put(inputId, newData);
}

private UpdatedKeys<Key, Value> calculateDiff(
    Map<Key, Value> oldData,
    Map<Key, Value> newData) {
  
  Set<Key> removedKeys = new HashSet<>(oldData.keySet());
  removedKeys.removeAll(newData.keySet());
  
  Map<Key, Value> addedKeys = new HashMap<>();
  Map<Key, Value> updatedKeys = new HashMap<>();
  
  for (Map.Entry<Key, Value> newEntry : newData.entrySet()) {
    Key key = newEntry.getKey();
    Value newValue = newEntry.getValue();
    
    if (!oldData.containsKey(key)) {
      // New key
      addedKeys.put(key, newValue);
    } else {
      Value oldValue = oldData.get(key);
      if (!Objects.equals(oldValue, newValue)) {
        // Value changed
        updatedKeys.put(key, newValue);
      }
      // else: no change, skip update
    }
  }
  
  return new UpdatedKeys<>(removedKeys, addedKeys, updatedKeys);
}
```

**Performance Impact**:
- **Without Forward Index**: O(K) where K = total keys in index (very expensive)
- **With Forward Index**: O(k) where k = keys in single file (very fast)

## Stub Indexing Deep Dive

### Stub Tree Structure

**Stubs** are lightweight AST nodes that store essential information:

```java
// Base stub class
public abstract class StubElement<T extends PsiElement> {
  private final StubElement<?> myParent;
  private final IStubElementType myElementType;
  
  // Child stubs
  private List<StubElement<?>> myChildren;
  
  protected StubElement(StubElement<?> parent, IStubElementType elementType) {
    myParent = parent;
    myElementType = elementType;
  }
  
  public StubElement<?> getParentStub() {
    return myParent;
  }
  
  public List<StubElement<?>> getChildrenStubs() {
    return myChildren != null ? myChildren : Collections.emptyList();
  }
}

// Example: Java class stub
public class PsiClassStub extends StubBase<PsiClass> {
  private final String myQualifiedName;
  private final String myName;
  private final byte myFlags; // is interface, is abstract, etc.
  
  public PsiClassStub(
      StubElement parent,
      String qualifiedName,
      String name,
      byte flags) {
    super(parent, JavaStubElementTypes.CLASS);
    myQualifiedName = qualifiedName;
    myName = name;
    myFlags = flags;
  }
  
  public String getQualifiedName() {
    return myQualifiedName;
  }
  
  public boolean isInterface() {
    return (myFlags & IS_INTERFACE) != 0;
  }
}
```

### Stub Building Process

```java
// StubTreeBuilder
public class StubTreeBuilder {
  
  public static Stub buildStubTree(PsiFile file) {
    // Get parser definition for the language
    ParserDefinition parserDef = 
      LanguageParserDefinitions.INSTANCE.forLanguage(file.getLanguage());
    
    // Create stub building visitor
    StubBuildingVisitor visitor = new StubBuildingVisitor();
    
    // Walk PSI tree and build stubs
    file.accept(visitor);
    
    return visitor.getRoot();
  }
}

class StubBuildingVisitor extends PsiRecursiveElementVisitor {
  private Deque<StubElement> myStack = new ArrayDeque<>();
  
  @Override
  public void visitElement(@NotNull PsiElement element) {
    IStubElementType stubType = getStubType(element);
    
    if (stubType != null) {
      // Create stub for this element
      StubElement stub = stubType.createStub(element, myStack.peek());
      myStack.push(stub);
      
      // Visit children
      super.visitElement(element);
      
      myStack.pop();
    } else {
      // No stub for this element, but visit children
      super.visitElement(element);
    }
  }
  
  public StubElement getRoot() {
    return myStack.isEmpty() ? null : myStack.getLast();
  }
}
```

### Stub Serialization

Stubs are serialized to a compact binary format:

```java
public class StubSerializationHelper {
  
  public static byte[] serializeStub(Stub stub) throws IOException {
    ByteArrayOutputStream baos = new ByteArrayOutputStream();
    DataOutputStream out = new DataOutputStream(baos);
    
    // Write stub tree
    writeStub(out, stub);
    
    return baos.toByteArray();
  }
  
  private static void writeStub(DataOutputStream out, Stub stub) throws IOException {
    // Write stub type ID
    IStubElementType type = stub.getStubType();
    int typeId = StubSerializerEnumerator.getInstance().getId(type);
    out.writeInt(typeId);
    
    // Write stub data
    type.serialize(stub, out);
    
    // Write children recursively
    List<? extends Stub> children = stub.getChildrenStubs();
    out.writeInt(children.size());
    for (Stub child : children) {
      writeStub(out, child);
    }
  }
  
  public static Stub deserializeStub(byte[] data) throws IOException {
    DataInputStream in = new DataInputStream(
      new ByteArrayInputStream(data));
    
    return readStub(in, null);
  }
  
  private static Stub readStub(DataInputStream in, Stub parent) 
      throws IOException {
    // Read stub type
    int typeId = in.readInt();
    IStubElementType type = 
      StubSerializerEnumerator.getInstance().getType(typeId);
    
    // Deserialize stub
    Stub stub = type.deserialize(in, parent);
    
    // Read children
    int childrenCount = in.readInt();
    for (int i = 0; i < childrenCount; i++) {
      readStub(in, stub);
    }
    
    return stub;
  }
}
```

### StubUpdatingIndex Implementation

```java
public final class StubUpdatingIndex extends SingleEntryFileBasedIndexExtension<SerializedStubTree> {
  
  @Override
  public @NotNull SingleEntryIndexer<SerializedStubTree> getIndexer() {
    return new SingleEntryIndexer<SerializedStubTree>(false) {
      @Override
      protected SerializedStubTree computeValue(@NotNull FileContent inputData) {
        try {
          // Get or create PSI file
          PsiFile psiFile = inputData.getPsiFile();
          
          if (psiFile == null) {
            // Binary file - use binary stub builder
            return buildBinaryStub(inputData);
          }
          
          // Build stub tree from PSI
          Stub stubTree = StubTreeBuilder.buildStubTree(psiFile);
          
          if (stubTree == null) {
            return null;
          }
          
          // Serialize stub tree
          byte[] serialized = StubSerializationHelper.serializeStub(stubTree);
          
          // Calculate hash for all indexed data
          byte[] indexedHash = calculateIndexedHash(stubTree);
          
          return new SerializedStubTree(serialized, indexedHash);
          
        } catch (Exception e) {
          LOG.error("Error building stubs for: " + inputData.getFileName(), e);
          return null;
        }
      }
    };
  }
  
  private byte[] calculateIndexedHash(Stub stub) {
    // Collect all data that goes into stub indexes
    MessageDigest digest = MessageDigest.getInstance("SHA-256");
    
    collectIndexedData(stub, digest);
    
    return digest.digest();
  }
  
  private void collectIndexedData(Stub stub, MessageDigest digest) {
    // Add data from this stub
    for (StubIndexExtension<?, ?> extension : StubIndexExtension.EP_NAME.getExtensions()) {
      Map<?, ?> indexed = extension.getIndexer().map(stub);
      for (Object key : indexed.keySet()) {
        digest.update(key.toString().getBytes());
      }
    }
    
    // Process children
    for (Stub child : stub.getChildrenStubs()) {
      collectIndexedData(child, digest);
    }
  }
}
```

## Composite Indexing

### Per-File-Type Indexers

Some indexes use different indexers for different file types:

```java
// IdIndex uses composite indexing
public class IdIndex extends FileBasedIndexExtension<IdIndexEntry, Integer> {
  
  @Override
  public @NotNull DataIndexer<IdIndexEntry, Integer, FileContent> getIndexer() {
    return new CompositeDataIndexer<
        IdIndexEntry, Integer, 
        FileTypeSpecificSubIndexer<IdIndexer>, 
        String>() {
      
      @Override
      public @Nullable FileTypeSpecificSubIndexer<IdIndexer> 
          calculateSubIndexer(@NotNull IndexedFile file) {
        
        FileType type = file.getFileType();
        
        // Get file-type-specific indexer
        IdIndexer indexer = IdTableBuilding.getFileTypeIndexer(type);
        
        if (indexer == null) {
          return null; // No indexer for this file type
        }
        
        return new FileTypeSpecificSubIndexer<>(indexer, type);
      }
      
      @Override
      public @NotNull String getSubIndexerVersion(
          @NotNull FileTypeSpecificSubIndexer<IdIndexer> indexer) {
        // Version includes indexer class and version
        return indexer.getSubIndexerType().getClass().getName() + ":" +
               indexer.getSubIndexerType().getVersion() + ":" +
               indexer.getFileType().getName();
      }
      
      @Override
      public @NotNull KeyDescriptor<String> getSubIndexerVersionDescriptor() {
        return EnumeratorStringDescriptor.INSTANCE;
      }
      
      @Override
      public @NotNull Map<IdIndexEntry, Integer> map(
          @NotNull FileContent inputData,
          @NotNull FileTypeSpecificSubIndexer<IdIndexer> indexer) {
        
        // Delegate to file-type-specific indexer
        return indexer.getSubIndexerType().map(inputData);
      }
    };
  }
}
```

### Sub-Indexer Versioning

Each sub-indexer has its own version:

```java
// PersistentSubIndexerRetriever
public final class PersistentSubIndexerRetriever<SubIndexerType, SubIndexerVersion> {
  
  private final CompositeDataIndexer<?, ?, SubIndexerType, SubIndexerVersion> myIndexer;
  private final PersistentEnumerator<SubIndexerVersion> myVersionEnumerator;
  private final IntFileAttribute myFileAttribute;
  
  public FileIndexingState getSubIndexerState(int fileId, @NotNull IndexedFile file) 
      throws IOException {
    
    // Get sub-indexer ID stored for this file
    int storedSubIndexerId = myFileAttribute.readInt(fileId);
    
    if (storedSubIndexerId == 0) {
      return FileIndexingState.OUT_DATED;
    }
    
    if (storedSubIndexerId == UNINDEXED_STATE) {
      return FileIndexingState.NOT_INDEXED;
    }
    
    // Calculate current sub-indexer version
    SubIndexerVersion currentVersion = getVersion(file);
    
    if (currentVersion == null) {
      // File not accepted by any sub-indexer
      return FileIndexingState.UP_TO_DATE;
    }
    
    int currentSubIndexerId = myVersionEnumerator.enumerate(currentVersion);
    
    // Compare versions
    if (currentSubIndexerId == storedSubIndexerId) {
      return FileIndexingState.UP_TO_DATE;
    } else {
      return FileIndexingState.OUT_DATED;
    }
  }
  
  public @Nullable SubIndexerVersion getVersion(@NotNull IndexedFile file) {
    // Determine which sub-indexer handles this file
    SubIndexerType type = myIndexer.calculateSubIndexer(file);
    
    if (type == null) {
      return null; // Not indexed by this index
    }
    
    // Get version for this sub-indexer
    return myIndexer.getSubIndexerVersion(type);
  }
  
  public void setIndexedState(int fileId, @NotNull IndexedFile file) throws IOException {
    SubIndexerVersion version = getVersion(file);
    
    if (version == null) {
      myFileAttribute.writeInt(fileId, UNINDEXED_STATE);
    } else {
      int subIndexerId = myVersionEnumerator.enumerate(version);
      myFileAttribute.writeInt(fileId, subIndexerId);
    }
  }
}
```

## Performance Considerations

### 1. Lazy Index Updates

```java
public <K> boolean ensureUpToDate(
    @NotNull ID<K, ?> indexId,
    @Nullable Project project,
    @Nullable GlobalSearchScope filter) {
  
  // Check if index is already up-to-date
  if (isUpToDate(indexId, project)) {
    return true;
  }
  
  // Get files that need indexing
  Collection<VirtualFile> filesToUpdate = 
    getFilesToUpdate(project, indexId, filter);
  
  if (filesToUpdate.isEmpty()) {
    return true;
  }
  
  // Small number of files - index synchronously
  if (filesToUpdate.size() < 10) {
    for (VirtualFile file : filesToUpdate) {
      indexFile(file, project);
    }
    return true;
  }
  
  // Large number of files - schedule background task
  scheduleIndexingTask(project, filesToUpdate);
  return false; // Not yet up-to-date
}
```

### 2. Memory-Efficient Streaming

For large files, content is processed in streams:

```java
public Map<Key, Value> indexLargeFile(FileContent content) {
  // Don't load entire content into memory
  try (InputStream in = content.getFile().getInputStream()) {
    return indexStream(in);
  }
}
```

### 3. Caching

Multiple caching layers reduce disk I/O:

```java
// L1: In-memory cache of ValueContainers
private final Map<Key, SoftReference<ValueContainer<Value>>> myCache = 
  new ConcurrentHashMap<>();

// L2: Memory-mapped file cache (OS-level)
// Handled by PersistentHashMap

public ValueContainer<Value> read(Key key) throws StorageException {
  // Check L1 cache
  SoftReference<ValueContainer<Value>> ref = myCache.get(key);
  if (ref != null) {
    ValueContainer<Value> cached = ref.get();
    if (cached != null) {
      return cached;
    }
  }
  
  // Read from storage (may hit L2 cache)
  ValueContainer<Value> container = myStorage.get(key);
  
  // Update L1 cache
  myCache.put(key, new SoftReference<>(container));
  
  return container;
}
```

### 4. Write-Ahead Logging (WAL)

Some indexes use WAL for crash recovery:

```java
// Enable WAL for an index
@Override
public boolean enableWal() {
  return true; // Enable write-ahead logging
}

// WAL ensures atomicity:
// 1. Write to WAL: "About to update key X for file Y"
// 2. Update index
// 3. Mark WAL entry as committed
// 
// On crash recovery:
// - Uncommitted WAL entries are replayed
// - Or rolled back if not completed
```

## Summary

This document covered the deep technical implementation details of IntelliJ's indexing system:

1. **Core Classes**: FileBasedIndexImpl, MapReduceIndex, RegisteredIndexes
2. **Lifecycle**: Initialization, version checking, shutdown
3. **Pipeline**: Scanning, loading, parallel processing
4. **Storage**: PersistentHashMap, ValueContainer, serialization
5. **Update Algorithm**: Forward index, diff calculation, atomic updates
6. **Stub Indexing**: Stub trees, serialization, StubUpdatingIndex
7. **Composite Indexing**: Sub-indexers, per-file-type versioning
8. **Performance**: Lazy updates, caching, streaming, WAL

The implementation is sophisticated, handling:
- Millions of files efficiently
- Crash recovery and corruption handling
- Thread safety and concurrency
- Memory efficiency
- Incremental updates
- Plugin extensibility

Key design patterns:
- **Map-Reduce**: Parallel file processing
- **Inverted Index**: Fast key-based lookups
- **Forward Index**: Efficient incremental updates
- **Composite Pattern**: Extensible sub-indexers
- **Lazy Evaluation**: Update only when needed
- **Layered Caching**: Multiple cache levels
