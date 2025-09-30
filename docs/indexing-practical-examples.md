# IntelliJ IDEA Indexing - Practical Examples and Workflows

This document provides practical examples of how indexing works, including creating custom indexes and understanding the complete indexing workflow.

## Table of Contents
1. [Creating a Custom Index](#creating-a-custom-index)
2. [Indexing Workflow Diagrams](#indexing-workflow-diagrams)
3. [Query Examples](#query-examples)
4. [Debugging and Troubleshooting](#debugging-and-troubleshooting)
5. [Common Patterns](#common-patterns)

## Creating a Custom Index

### Example 1: Simple String Index

Let's create an index that stores all string literals in Java files.

**Step 1: Define the Index Extension**

```java
package com.example.index;

import com.intellij.openapi.fileTypes.StdFileTypes;
import com.intellij.psi.*;
import com.intellij.psi.impl.cache.impl.id.IdIndex;
import com.intellij.util.indexing.*;
import com.intellij.util.io.*;
import org.jetbrains.annotations.NotNull;

import java.util.*;

/**
 * Index that stores all string literals found in Java files.
 * Key: String literal value
 * Value: Usage count in the file
 */
public class StringLiteralIndex extends ScalarIndexExtension<String> {
  
  public static final ID<String, Void> INDEX_ID = 
    ID.create("example.string.literal.index");
  
  @Override
  public @NotNull ID<String, Void> getName() {
    return INDEX_ID;
  }
  
  @Override
  public @NotNull DataIndexer<String, Void, FileContent> getIndexer() {
    return new DataIndexer<String, Void, FileContent>() {
      @Override
      public @NotNull Map<String, Void> map(@NotNull FileContent inputData) {
        Set<String> literals = new HashSet<>();
        
        // Get PSI file
        PsiFile psiFile = inputData.getPsiFile();
        if (psiFile == null) {
          return Collections.emptyMap();
        }
        
        // Visit all elements and collect string literals
        psiFile.accept(new JavaRecursiveElementVisitor() {
          @Override
          public void visitLiteralExpression(@NotNull PsiLiteralExpression expression) {
            super.visitLiteralExpression(expression);
            
            Object value = expression.getValue();
            if (value instanceof String) {
              literals.add((String) value);
            }
          }
        });
        
        // Convert to map (ScalarIndex uses Void as value)
        Map<String, Void> result = new HashMap<>();
        for (String literal : literals) {
          result.put(literal, null);
        }
        
        return result;
      }
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
  
  @Override
  public boolean dependsOnFileContent() {
    return true; // We need file content to find string literals
  }
  
  @Override
  public int getVersion() {
    return 1; // Increment when index format changes
  }
}
```

**Step 2: Register the Index**

Create `plugin.xml`:

```xml
<idea-plugin>
  <extensions defaultExtensionNs="com.intellij">
    <fileBasedIndex implementation="com.example.index.StringLiteralIndex"/>
  </extensions>
</idea-plugin>
```

**Step 3: Query the Index**

```java
package com.example.search;

import com.example.index.StringLiteralIndex;
import com.intellij.openapi.project.Project;
import com.intellij.openapi.vfs.VirtualFile;
import com.intellij.psi.search.GlobalSearchScope;
import com.intellij.util.indexing.FileBasedIndex;

import java.util.Collection;

public class StringLiteralSearcher {
  
  /**
   * Find all files containing a specific string literal
   */
  public static Collection<VirtualFile> findFilesWithLiteral(
      String literal, 
      Project project) {
    
    GlobalSearchScope scope = GlobalSearchScope.projectScope(project);
    
    return FileBasedIndex.getInstance().getContainingFiles(
      StringLiteralIndex.INDEX_ID,
      literal,
      scope
    );
  }
  
  /**
   * Get all string literals in the project
   */
  public static Collection<String> getAllLiterals(Project project) {
    List<String> allLiterals = new ArrayList<>();
    
    GlobalSearchScope scope = GlobalSearchScope.projectScope(project);
    
    FileBasedIndex.getInstance().processAllKeys(
      StringLiteralIndex.INDEX_ID,
      literal -> {
        allLiterals.add(literal);
        return true; // Continue processing
      },
      scope,
      null // No ID filter
    );
    
    return allLiterals;
  }
}
```

### Example 2: Map Index with Values

Let's create an index that stores method names with their parameter count.

```java
package com.example.index;

import com.intellij.openapi.fileTypes.StdFileTypes;
import com.intellij.psi.*;
import com.intellij.util.indexing.*;
import com.intellij.util.io.*;
import org.jetbrains.annotations.NotNull;

import java.io.DataInput;
import java.io.DataOutput;
import java.io.IOException;
import java.util.*;

/**
 * Index that maps method names to their parameter counts.
 * Key: Method name
 * Value: Parameter count
 */
public class MethodSignatureIndex extends FileBasedIndexExtension<String, Integer> {
  
  public static final ID<String, Integer> INDEX_ID = 
    ID.create("example.method.signature.index");
  
  @Override
  public @NotNull ID<String, Integer> getName() {
    return INDEX_ID;
  }
  
  @Override
  public @NotNull DataIndexer<String, Integer, FileContent> getIndexer() {
    return inputData -> {
      Map<String, Integer> result = new HashMap<>();
      
      PsiFile psiFile = inputData.getPsiFile();
      if (psiFile == null) {
        return Collections.emptyMap();
      }
      
      psiFile.accept(new JavaRecursiveElementVisitor() {
        @Override
        public void visitMethod(@NotNull PsiMethod method) {
          super.visitMethod(method);
          
          String methodName = method.getName();
          int paramCount = method.getParameterList().getParametersCount();
          
          // Store method name -> param count
          result.put(methodName, paramCount);
        }
      });
      
      return result;
    };
  }
  
  @Override
  public @NotNull KeyDescriptor<String> getKeyDescriptor() {
    return EnumeratorStringDescriptor.INSTANCE;
  }
  
  @Override
  public @NotNull DataExternalizer<Integer> getValueExternalizer() {
    return new DataExternalizer<Integer>() {
      @Override
      public void save(@NotNull DataOutput out, Integer value) throws IOException {
        out.writeInt(value);
      }
      
      @Override
      public Integer read(@NotNull DataInput in) throws IOException {
        return in.readInt();
      }
    };
  }
  
  @Override
  public @NotNull FileBasedIndex.InputFilter getInputFilter() {
    return new DefaultFileTypeSpecificInputFilter(StdFileTypes.JAVA);
  }
  
  @Override
  public boolean dependsOnFileContent() {
    return true;
  }
  
  @Override
  public int getVersion() {
    return 1;
  }
}
```

**Query Example:**

```java
public class MethodSignatureSearcher {
  
  /**
   * Find all methods with a specific name and parameter count
   */
  public static Collection<VirtualFile> findMethods(
      String methodName,
      int paramCount,
      Project project) {
    
    GlobalSearchScope scope = GlobalSearchScope.projectScope(project);
    
    // Get all files containing the method name
    List<VirtualFile> candidates = FileBasedIndex.getInstance()
      .getContainingFiles(MethodSignatureIndex.INDEX_ID, methodName, scope);
    
    // Filter by parameter count
    List<VirtualFile> result = new ArrayList<>();
    
    for (VirtualFile file : candidates) {
      // Get values for this key in this file
      List<Integer> paramCounts = FileBasedIndex.getInstance()
        .getValues(MethodSignatureIndex.INDEX_ID, methodName, 
                  GlobalSearchScope.fileScope(project, file));
      
      if (paramCounts.contains(paramCount)) {
        result.add(file);
      }
    }
    
    return result;
  }
}
```

### Example 3: Forward Index for Caching

Create an index that caches processed data per file:

```java
package com.example.index;

import com.intellij.util.indexing.*;
import com.intellij.util.io.*;
import org.jetbrains.annotations.NotNull;
import org.jetbrains.annotations.Nullable;

import java.io.*;
import java.util.*;

/**
 * Forward index that caches word count per file.
 * This is a single-entry index (one value per file).
 */
public class WordCountIndex extends SingleEntryFileBasedIndexExtension<Integer> {
  
  public static final ID<Integer, Integer> INDEX_ID = 
    ID.create("example.word.count.index");
  
  @Override
  public @NotNull ID<Integer, Integer> getName() {
    return INDEX_ID;
  }
  
  @Override
  public @NotNull SingleEntryIndexer<Integer> getIndexer() {
    return new SingleEntryIndexer<Integer>(false) {
      @Override
      protected @Nullable Integer computeValue(@NotNull FileContent inputData) {
        // Count words in file
        String content = inputData.getContentAsText().toString();
        String[] words = content.split("\\s+");
        return words.length;
      }
    };
  }
  
  @Override
  public @NotNull DataExternalizer<Integer> getValueExternalizer() {
    return new DataExternalizer<Integer>() {
      @Override
      public void save(@NotNull DataOutput out, Integer value) throws IOException {
        out.writeInt(value);
      }
      
      @Override
      public Integer read(@NotNull DataInput in) throws IOException {
        return in.readInt();
      }
    };
  }
  
  @Override
  public @NotNull FileBasedIndex.InputFilter getInputFilter() {
    return file -> true; // Accept all files
  }
  
  @Override
  public boolean dependsOnFileContent() {
    return true;
  }
  
  @Override
  public int getVersion() {
    return 1;
  }
}
```

**Usage:**

```java
public class WordCountService {
  
  public static int getWordCount(VirtualFile file, Project project) {
    // Query the forward index
    List<Integer> counts = FileBasedIndex.getInstance().getValues(
      WordCountIndex.INDEX_ID,
      FileBasedIndex.getFileId(file),
      GlobalSearchScope.fileScope(project, file)
    );
    
    return counts.isEmpty() ? 0 : counts.get(0);
  }
  
  public static int getTotalWordCount(Project project) {
    final int[] total = {0};
    
    // Process all indexed files
    FileBasedIndex.getInstance().processAllKeys(
      WordCountIndex.INDEX_ID,
      fileId -> {
        List<Integer> counts = FileBasedIndex.getInstance().getValues(
          WordCountIndex.INDEX_ID,
          fileId,
          GlobalSearchScope.projectScope(project)
        );
        
        if (!counts.isEmpty()) {
          total[0] += counts.get(0);
        }
        
        return true; // Continue
      },
      GlobalSearchScope.projectScope(project),
      null
    );
    
    return total[0];
  }
}
```

## Indexing Workflow Diagrams

### 1. Initial Project Indexing Flow

```
┌─────────────────────────────────────────────────────────────────┐
│ Project Opening                                                  │
└────────────┬────────────────────────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────────────────────────────┐
│ ProjectFileBasedIndexStartupActivity                            │
│  - Triggered on project open                                     │
│  - Checks if indexing needed                                     │
└────────────┬────────────────────────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────────────────────────────┐
│ UnindexedFilesScanner                                           │
│  - Scan project content roots                                    │
│  - Check IndexingStamp for each file                            │
│  - Compare with index versions                                   │
│  - Collect files needing indexing                               │
└────────────┬────────────────────────────────────────────────────┘
             │
             ▼
     ┌───────────────┐
     │ Files to      │
     │ Index Queue   │
     └───────┬───────┘
             │
             ▼
┌─────────────────────────────────────────────────────────────────┐
│ UnindexedFilesIndexer (Background Task)                         │
│  - Enter "Dumb Mode"                                            │
│  - Create thread pool (N threads)                                │
│  - Process files in parallel                                     │
└────────────┬────────────────────────────────────────────────────┘
             │
             ▼
     ┌───────────────────────────────────────────┐
     │  For each file (in parallel):             │
     │                                           │
     │  1. Load file content                     │
     │  2. Call FileBasedIndexImpl.              │
     │     doIndexFileContent()                  │
     │  3. For each index:                       │
     │     a. Check InputFilter                  │
     │     b. Call DataIndexer.map()             │
     │     c. Prepare index updates              │
     │  4. Apply all updates atomically          │
     │  5. Update IndexingStamp                  │
     └───────────────┬───────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│ All Files Processed                                             │
│  - Exit "Dumb Mode"                                             │
│  - Enable smart features                                         │
│  - Fire indexing completed events                               │
└─────────────────────────────────────────────────────────────────┘
```

### 2. File Change and Incremental Update

```
┌─────────────────────────────────────────────────────────────────┐
│ File Modified (VFS Event)                                        │
└────────────┬────────────────────────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────────────────────────────┐
│ ChangedFilesCollector                                           │
│  - Listen to VFS events                                          │
│  - Collect changed files                                         │
│  - Mark files as "needs reindexing"                             │
└────────────┬────────────────────────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────────────────────────────┐
│ Index Access Triggered                                          │
│  (e.g., Find Usages, Code Completion)                           │
└────────────┬────────────────────────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────────────────────────────┐
│ FileBasedIndexImpl.ensureUpToDate()                            │
│  - Check if changed files < threshold                           │
└────────────┬────────────────────────────────────────────────────┘
             │
        ┌────┴────┐
        │         │
        ▼         ▼
  Few Files    Many Files
  (< 10)       (>= 10)
        │         │
        │         ▼
        │    ┌─────────────────────────────────────┐
        │    │ Schedule Background Task             │
        │    │  - Don't block current operation     │
        │    │  - Update asynchronously             │
        │    └─────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────────┐
│ Synchronous Update (for few files)                             │
└────────────┬────────────────────────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────────────────────────────┐
│ For each changed file:                                          │
│                                                                  │
│  1. Load file content                                            │
│  2. For each affected index:                                     │
│     ┌──────────────────────────────────────────────────────┐   │
│     │ MapReduceIndex.prepareUpdate()                        │   │
│     │                                                        │   │
│     │  a. Read old data from forward index                  │   │
│     │     oldData = forwardIndex.get(fileId)                │   │
│     │                                                        │   │
│     │  b. Compute new data                                  │   │
│     │     newData = indexer.map(fileContent)                │   │
│     │                                                        │   │
│     │  c. Calculate diff                                    │   │
│     │     - removedKeys = oldKeys - newKeys                 │   │
│     │     - addedKeys = newKeys - oldKeys                   │   │
│     │     - updatedKeys = changed values                    │   │
│     │                                                        │   │
│     │  d. Update inverted index                             │   │
│     │     for key in removedKeys:                           │   │
│     │       storage.removeValue(key, fileId)                │   │
│     │     for key in addedKeys:                             │   │
│     │       storage.addValue(key, fileId, value)            │   │
│     │     for key in updatedKeys:                           │   │
│     │       storage.updateValue(key, fileId, value)         │   │
│     │                                                        │   │
│     │  e. Update forward index                              │   │
│     │     forwardIndex.put(fileId, newData)                 │   │
│     └──────────────────────────────────────────────────────┘   │
│  3. Update IndexingStamp                                         │
└─────────────────────────────────────────────────────────────────┘
```

### 3. Index Query Flow

```
┌─────────────────────────────────────────────────────────────────┐
│ Client Code: Find Usages / Navigation / Search                  │
└────────────┬────────────────────────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────────────────────────────┐
│ FileBasedIndex.getContainingFiles(indexId, key, scope)         │
└────────────┬────────────────────────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────────────────────────────┐
│ 1. Ensure index is up-to-date                                   │
│    ensureUpToDate(indexId, project, scope)                      │
└────────────┬────────────────────────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. Acquire read lock                                            │
│    myReadLock.lock()                                            │
└────────────┬────────────────────────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. Get index instance                                           │
│    index = myRegisteredIndexes.getIndex(indexId)                │
└────────────┬────────────────────────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. Query inverted index                                         │
│    index.withData(key, processor)                               │
│                                                                  │
│    This calls:                                                   │
│    ┌──────────────────────────────────────────────────────┐    │
│    │ storage.read(key) returns ValueContainer             │    │
│    │                                                        │    │
│    │ ValueContainer contains:                              │    │
│    │   Map<FileId, Value>                                  │    │
│    │                                                        │    │
│    │ For each (fileId, value):                            │    │
│    │   if scope.contains(fileId):                          │    │
│    │     processor.process(fileId, value)                  │    │
│    └──────────────────────────────────────────────────────┘    │
└────────────┬────────────────────────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────────────────────────────┐
│ 5. Convert file IDs to VirtualFiles                             │
│    for fileId in results:                                        │
│      file = findFileById(fileId)                                │
│      if file != null: add to results                            │
└────────────┬────────────────────────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────────────────────────────┐
│ 6. Release read lock                                            │
│    myReadLock.unlock()                                          │
└────────────┬────────────────────────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────────────────────────────┐
│ 7. Return results to client                                     │
└─────────────────────────────────────────────────────────────────┘
```

## Query Examples

### Example 1: Find All Java Classes

```java
import com.intellij.psi.search.searches.ClassInheritorsSearch;
import com.intellij.psi.stubs.StubIndex;
import com.intellij.psi.stubs.StubIndexKey;

public class ClassSearchExample {
  
  /**
   * Find all classes with a specific name in the project
   */
  public static Collection<PsiClass> findClassesByName(
      String className, 
      Project project) {
    
    GlobalSearchScope scope = GlobalSearchScope.projectScope(project);
    
    // Use stub index for fast lookup
    return JavaPsiFacade.getInstance(project)
      .findClasses(className, scope);
  }
  
  /**
   * Find all classes implementing a specific interface
   */
  public static Collection<PsiClass> findImplementors(
      PsiClass interfaceClass,
      Project project) {
    
    GlobalSearchScope scope = GlobalSearchScope.projectScope(project);
    
    return ClassInheritorsSearch.search(interfaceClass, scope, false)
      .findAll();
  }
}
```

### Example 2: Search for Identifiers

```java
import com.intellij.psi.impl.cache.impl.id.IdIndex;
import com.intellij.psi.impl.cache.impl.id.IdIndexEntry;
import com.intellij.psi.search.UsageSearchContext;

public class IdentifierSearchExample {
  
  /**
   * Find all files containing an identifier
   */
  public static Collection<VirtualFile> findIdentifier(
      String identifier,
      Project project) {
    
    GlobalSearchScope scope = GlobalSearchScope.projectScope(project);
    
    // Create index entry for identifier
    IdIndexEntry entry = IdIndexEntry.create(identifier);
    
    // Query IdIndex
    return FileBasedIndex.getInstance().getContainingFiles(
      IdIndex.NAME,
      entry,
      scope
    );
  }
  
  /**
   * Find identifier in specific context (code, comments, strings)
   */
  public static void searchInContext(
      String identifier,
      Project project,
      short context) {
    
    GlobalSearchScope scope = GlobalSearchScope.projectScope(project);
    IdIndexEntry entry = IdIndexEntry.create(identifier);
    
    FileBasedIndex.getInstance().processValues(
      IdIndex.NAME,
      entry,
      null, // Accept all files
      (file, value) -> {
        // Check if identifier appears in requested context
        if ((value & context) != 0) {
          System.out.println("Found in: " + file.getName() + 
                           " with context: " + value);
        }
        return true; // Continue processing
      },
      scope
    );
  }
  
  public static void main(String[] args) {
    // Search for identifier in code
    searchInContext("myVariable", project, UsageSearchContext.IN_CODE);
    
    // Search in comments
    searchInContext("myVariable", project, UsageSearchContext.IN_COMMENTS);
    
    // Search in string literals
    searchInContext("myVariable", project, UsageSearchContext.IN_STRINGS);
  }
}
```

### Example 3: Todo Item Search

```java
import com.intellij.psi.impl.cache.impl.todo.TodoIndex;

public class TodoSearchExample {
  
  /**
   * Find all files with TODO comments
   */
  public static Collection<VirtualFile> findAllTodos(Project project) {
    List<VirtualFile> result = new ArrayList<>();
    
    GlobalSearchScope scope = GlobalSearchScope.projectScope(project);
    
    // Process all keys in TodoIndex
    FileBasedIndex.getInstance().processAllKeys(
      TodoIndex.NAME,
      pattern -> {
        // For each TODO pattern, get files
        Collection<VirtualFile> files = 
          FileBasedIndex.getInstance().getContainingFiles(
            TodoIndex.NAME,
            pattern,
            scope
          );
        result.addAll(files);
        return true;
      },
      scope,
      null
    );
    
    return result;
  }
}
```

## Debugging and Troubleshooting

### Enable Debug Logging

Add to `idea.properties`:

```properties
# Enable index debug logging
idea.log.index.debug=true

# Log stub updates
idea.log.stub.updates=true

# Trace specific index
idea.log.index.IdIndex=trace
```

Or programmatically:

```java
// FileBasedIndexImpl.java
public static final Logger LOG = Logger.getInstance(FileBasedIndexImpl.class);

// Enable debug level
LOG.setLevel(Level.DEBUG);

// Log index operations
LOG.debug("Indexing file: " + file.getName());
```

### Force Index Rebuild

**Via UI**: `File → Invalidate Caches / Restart → Invalidate and Restart`

**Programmatically**:

```java
import com.intellij.util.indexing.FileBasedIndex;

public class IndexMaintenance {
  
  public static void rebuildIndex(ID<?, ?> indexId) {
    FileBasedIndex.getInstance().requestRebuild(indexId);
  }
  
  public static void rebuildAllIndexes() {
    FileBasedIndex fileBasedIndex = FileBasedIndex.getInstance();
    
    if (fileBasedIndex instanceof FileBasedIndexImpl) {
      FileBasedIndexImpl impl = (FileBasedIndexImpl) fileBasedIndex;
      
      // Mark all indexes for rebuild
      for (ID<?, ?> indexId : impl.getAllIndexIds()) {
        impl.requestRebuild(indexId);
      }
    }
  }
}
```

### Check Index Statistics

```java
import com.intellij.util.indexing.diagnostic.IndexStatisticGroup;

public class IndexStats {
  
  public static void printIndexStats(Project project) {
    FileBasedIndexImpl index = 
      (FileBasedIndexImpl) FileBasedIndex.getInstance();
    
    for (ID<?, ?> indexId : index.getAllIndexIds()) {
      UpdatableIndex<?, ?, ?, ?> updatableIndex = index.getIndex(indexId);
      
      System.out.println("Index: " + indexId);
      System.out.println("  Modification stamp: " + 
        updatableIndex.getModificationStamp());
      
      if (updatableIndex instanceof MeasurableIndexStore) {
        MeasurableIndexStore store = (MeasurableIndexStore) updatableIndex;
        System.out.println("  Keys count: " + store.keysCountApproximately());
      }
    }
  }
}
```

### Diagnose Indexing Performance

```java
public class IndexingProfiler {
  
  public static void profileIndexing(VirtualFile file, Project project) {
    long startTime = System.currentTimeMillis();
    
    FileBasedIndexImpl index = 
      (FileBasedIndexImpl) FileBasedIndex.getInstance();
    
    // Load content
    long loadStart = System.currentTimeMillis();
    byte[] content = file.contentsToByteArray();
    long loadTime = System.currentTimeMillis() - loadStart;
    
    // Index file
    long indexStart = System.currentTimeMillis();
    index.indexFileContent(project, 
      new CachedFileContentImpl(file, content));
    long indexTime = System.currentTimeMillis() - indexStart;
    
    long totalTime = System.currentTimeMillis() - startTime;
    
    System.out.println("Indexing profile for: " + file.getName());
    System.out.println("  Load time: " + loadTime + "ms");
    System.out.println("  Index time: " + indexTime + "ms");
    System.out.println("  Total time: " + totalTime + "ms");
  }
}
```

## Common Patterns

### Pattern 1: File Type Filtering

```java
public class FileTypeFilterExample extends FileBasedIndexExtension<String, Void> {
  
  @Override
  public @NotNull FileBasedIndex.InputFilter getInputFilter() {
    // Only index JSON files
    return new DefaultFileTypeSpecificInputFilter(JsonFileType.INSTANCE);
  }
}
```

### Pattern 2: Content-Independent Indexing

```java
public class FileNameIndex extends FileBasedIndexExtension<String, Void> {
  
  @Override
  public boolean dependsOnFileContent() {
    return false; // Only need file name, not content
  }
  
  @Override
  public @NotNull DataIndexer<String, Void, FileContent> getIndexer() {
    return inputData -> {
      // Can only safely access metadata
      String fileName = inputData.getFileName();
      return Collections.singletonMap(fileName, null);
    };
  }
}
```

### Pattern 3: Large Value Optimization

```java
public class LargeValueIndex extends FileBasedIndexExtension<String, byte[]> {
  
  @Override
  public @NotNull DataExternalizer<byte[]> getValueExternalizer() {
    return new DataExternalizer<byte[]>() {
      @Override
      public void save(@NotNull DataOutput out, byte[] value) throws IOException {
        // Compress large values
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        try (GZIPOutputStream gzip = new GZIPOutputStream(baos)) {
          gzip.write(value);
        }
        byte[] compressed = baos.toByteArray();
        
        out.writeInt(compressed.length);
        out.write(compressed);
      }
      
      @Override
      public byte[] read(@NotNull DataInput in) throws IOException {
        int length = in.readInt();
        byte[] compressed = new byte[length];
        in.readFully(compressed);
        
        // Decompress
        ByteArrayInputStream bais = new ByteArrayInputStream(compressed);
        try (GZIPInputStream gzip = new GZIPInputStream(bais)) {
          return gzip.readAllBytes();
        }
      }
    };
  }
}
```

### Pattern 4: Project-Specific Indexing

```java
public class ProjectSpecificIndex extends FileBasedIndexExtension<String, Void> {
  
  @Override
  public @NotNull FileBasedIndex.InputFilter getInputFilter() {
    return new FileBasedIndex.ProjectSpecificInputFilter() {
      @Override
      public boolean acceptInput(@NotNull IndexedFile file) {
        // Only index files in project sources, not libraries
        Project project = file.getProject();
        if (project == null) return false;
        
        VirtualFile virtualFile = file.getFile();
        ProjectFileIndex fileIndex = ProjectFileIndex.getInstance(project);
        
        return fileIndex.isInSource(virtualFile) && 
               !fileIndex.isInLibrary(virtualFile);
      }
    };
  }
}
```

## Summary

This document provided practical examples including:

1. **Custom Index Creation**: String literal index, method signature index, word count index
2. **Workflow Diagrams**: Visual representation of indexing processes
3. **Query Examples**: Finding classes, identifiers, TODO items
4. **Debugging**: Logging, rebuilding, statistics
5. **Common Patterns**: Filtering, content-independence, optimization

Key takeaways:
- Indexes are declarative - define what to index, system handles when
- Always implement proper serialization for keys and values
- Consider content dependency - avoid loading content if not needed
- Use appropriate filters to limit indexed files
- Test with large codebases to ensure performance
- Handle errors gracefully - index failures shouldn't crash the IDE

These examples provide a solid foundation for understanding and extending IntelliJ's indexing system.
