# Plugin Linting Performance Troubleshooting Guide

This guide helps plugin developers diagnose and fix performance issues related to code analysis, linting, and error checking in IntelliJ IDEA plugins.

## Common Performance Issues

### 1. Slow Code Analysis / Highlighting

#### Symptoms
- Long delays before errors/warnings appear
- Typing becomes sluggish
- Traffic light icon shows spinning indicator for extended periods
- IDE becomes unresponsive during analysis

#### Root Causes
- Heavy computation in highlighting passes
- Blocking I/O operations on EDT
- Excessive PSI traversal
- Memory leaks causing GC pressure
- Threading violations

#### Diagnostic Steps

**Step 1: Enable Performance Monitoring**
```
# Add to IDE VM options (Help → Edit Custom VM Options)
-XX:+UnlockDiagnosticVMOptions
-XX:+TraceClassLoading
-XX:+LogVMOutput
-XX:+PrintGCDetails

# Enable internal mode for more diagnostics
-Didea.is.internal=true
```

**Step 2: Use Built-in Profiling Tools**
1. Go to **Help → Diagnostic Tools → CPU Usage**
2. Enable **View → Internal Mode** in main menu
3. Use **Tools → Internal Actions → Show Performance Warnings**

**Step 3: Monitor Daemon Status**
```kotlin
// Add this debugging code to monitor daemon status
val daemonCodeAnalyzer = DaemonCodeAnalyzer.getInstance(project) as DaemonCodeAnalyzerImpl
val status = daemonCodeAnalyzer.getFileStatusMap()
LOG.info("Analysis status: ${status.allDirtyScopes}")
```

#### Solutions

**Optimize Highlighting Passes:**
```java
public class OptimizedHighlightingPass extends ProgressableTextEditorHighlightingPass {
    @Override
    protected void collectInformationWithProgress(@NotNull ProgressIndicator progress) {
        // ✅ Good: Check for cancellation frequently
        ProgressManager.checkCanceled();
        
        // ✅ Good: Process in batches
        List<PsiElement> elements = getElementsToProcess();
        for (int i = 0; i < elements.size(); i += BATCH_SIZE) {
            ProgressManager.checkCanceled();
            processBatch(elements.subList(i, Math.min(i + BATCH_SIZE, elements.size())));
        }
    }
    
    private static final int BATCH_SIZE = 100;
}
```

**Avoid EDT Blocking:**
```java
// ❌ Bad: Blocking operation on EDT
public void badExample() {
    ApplicationManager.getApplication().invokeLater(() -> {
        String result = expensiveOperation(); // Blocks EDT!
        updateUI(result);
    });
}

// ✅ Good: Background computation
public void goodExample() {
    ApplicationManager.getApplication().executeOnPooledThread(() -> {
        String result = expensiveOperation();
        ApplicationManager.getApplication().invokeLater(() -> {
            updateUI(result);
        });
    });
}
```

**Use Modern Async APIs:**
```kotlin
// ✅ Modern approach using coroutines
suspend fun analyzeFileAsync(file: PsiFile): AnalysisResult {
    return readAction {
        // PSI access in read action
        performAnalysis(file)
    }
}

// Usage in highlighting pass
override suspend fun collectInformationWithProgress(progress: ProgressIndicator) {
    val result = analyzeFileAsync(myFile)
    withContext(Dispatchers.EDT) {
        updateHighlighting(result)
    }
}
```

### 2. Memory Leaks in Analysis Code

#### Symptoms
- Growing memory usage over time
- OutOfMemoryError during analysis
- IDE becomes slower after prolonged use

#### Common Leak Sources

**PSI Element References:**
```java
// ❌ Bad: Holding strong references to PSI
public class LeakyInspection extends LocalInspectionTool {
    private final Set<PsiElement> cachedElements = new HashSet<>(); // Memory leak!
    
    @Override
    public ProblemDescriptor[] checkFile(@NotNull PsiFile file, ...) {
        cachedElements.add(file); // Don't do this!
        return super.checkFile(file, manager, isOnTheFly);
    }
}

// ✅ Good: Use weak references or avoid caching PSI
public class OptimizedInspection extends LocalInspectionTool {
    private final Map<VirtualFile, AnalysisResult> cache = 
        new ConcurrentHashMap<>(); // Cache by VirtualFile, not PSI
    
    @Override
    public ProblemDescriptor[] checkFile(@NotNull PsiFile file, ...) {
        VirtualFile vFile = file.getVirtualFile();
        if (vFile != null) {
            AnalysisResult cached = cache.get(vFile);
            // Use cached result if valid
        }
        return super.checkFile(file, manager, isOnTheFly);
    }
}
```

**Listener Management:**
```java
// ❌ Bad: Not removing listeners
public class LeakyComponent implements ProjectComponent {
    @Override
    public void projectOpened() {
        EditorFactory.getInstance().getEventMulticaster()
            .addDocumentListener(myListener); // Never removed!
    }
}

// ✅ Good: Proper listener lifecycle
public class ProperComponent implements ProjectComponent, Disposable {
    @Override
    public void projectOpened() {
        EditorFactory.getInstance().getEventMulticaster()
            .addDocumentListener(myListener, this); // Auto-removed on dispose
    }
    
    @Override
    public void dispose() {
        // Cleanup handled automatically
    }
}
```

#### Diagnostic Tools

**Memory Profiling:**
1. Use **Help → Diagnostic Tools → Analyze Memory Usage**
2. Enable **View → Show Memory Indicator**
3. Use JProfiler or YourKit for detailed analysis

**Leak Detection:**
```java
// Add to track object lifecycle
public class DiagnosticPass extends TextEditorHighlightingPass {
    private static final AtomicInteger INSTANCE_COUNT = new AtomicInteger(0);
    
    public DiagnosticPass() {
        int count = INSTANCE_COUNT.incrementAndGet();
        LOG.info("Creating pass instance #" + count);
    }
    
    @Override
    protected void finalize() throws Throwable {
        int count = INSTANCE_COUNT.decrementAndGet();
        LOG.info("Finalizing pass instance, remaining: " + count);
        super.finalize();
    }
}
```

### 3. Threading Issues

#### Symptoms
- `AssertionError: Must not modify PSI inside non-commit transaction`
- `AssertionError: Read access is allowed from event dispatch thread or inside read-action only`
- Random crashes or data corruption

#### Common Threading Mistakes

**PSI Access Violations:**
```java
// ❌ Bad: PSI access without read action
public void badPsiAccess() {
    ApplicationManager.getApplication().executeOnPooledThread(() -> {
        String text = psiElement.getText(); // Threading violation!
    });
}

// ✅ Good: Proper read action
public void goodPsiAccess() {
    ApplicationManager.getApplication().executeOnPooledThread(() -> {
        ApplicationManager.getApplication().runReadAction(() -> {
            String text = psiElement.getText(); // Safe PSI access
        });
    });
}
```

**Modern Approach with Suspending Functions:**
```kotlin
// ✅ Best: Modern async approach
suspend fun analyzeInBackground(element: PsiElement): String {
    return readAction {
        element.text // Safe PSI access in read action
    }
}
```

### 4. External Tool Integration Issues

#### Symptoms
- External linters not running
- Results not appearing in IDE
- Performance degradation during external tool execution

#### Best Practices for External Annotators

```java
public class OptimizedExternalAnnotator extends ExternalAnnotator<CollectedInfo, Results> {
    
    @Override
    public CollectedInfo collectInformation(@NotNull PsiFile file) {
        // ✅ Fast, PSI-based information collection
        return readAction {
            // Collect only essential information
            new CollectedInfo(file.getText(), file.getVirtualFile().getPath());
        };
    }
    
    @Override
    public Results doAnnotate(CollectedInfo collectedInfo) {
        // ✅ Heavy computation in background
        // This runs on background thread automatically
        return runExternalTool(collectedInfo);
    }
    
    @Override
    public void apply(@NotNull PsiFile file, Results annotationResult, @NotNull AnnotationHolder holder) {
        // ✅ Fast result application
        if (annotationResult != null) {
            for (Issue issue : annotationResult.getIssues()) {
                holder.createWarningAnnotation(issue.getRange(), issue.getMessage());
            }
        }
    }
    
    private Results runExternalTool(CollectedInfo info) {
        // Implement with proper timeouts and cancellation
        ProcessBuilder pb = new ProcessBuilder("external-tool", info.getFilePath());
        pb.timeout(30, TimeUnit.SECONDS); // Prevent hanging
        
        try {
            Process process = pb.start();
            return parseOutput(process.getInputStream());
        } catch (IOException | InterruptedException e) {
            LOG.warn("External tool failed", e);
            return null;
        }
    }
}
```

## Performance Monitoring and Debugging

### 1. Built-in Diagnostics

**Enable Performance Tracking:**
```
# Add to IDE VM options
-Dideaplatform.profiling=true
-Didea.slow.operations.assertion=true
```

**Monitor Daemon Performance:**
```kotlin
// Add to your plugin
class DaemonMonitor : DaemonCodeAnalyzer.DaemonListener {
    override fun daemonStarting(fileEditors: Collection<FileEditor>) {
        LOG.info("Daemon starting for ${fileEditors.size} editors")
        val startTime = System.currentTimeMillis()
        // Store start time for duration tracking
    }
    
    override fun daemonFinished() {
        val duration = System.currentTimeMillis() - startTime
        LOG.info("Daemon finished in ${duration}ms")
        if (duration > 5000) { // Warn if taking too long
            LOG.warn("Daemon analysis took ${duration}ms - investigate performance")
        }
    }
}
```

### 2. Custom Performance Metrics

```java
public class PerformanceTracker {
    private static final Logger LOG = Logger.getInstance(PerformanceTracker.class);
    
    public static <T> T measureTime(String operation, Supplier<T> task) {
        long start = System.nanoTime();
        try {
            return task.get();
        } finally {
            long duration = System.nanoTime() - start;
            long milliseconds = duration / 1_000_000;
            
            if (milliseconds > 100) { // Log slow operations
                LOG.warn(operation + " took " + milliseconds + "ms");
            }
        }
    }
}

// Usage in highlighting pass
@Override
protected void collectInformationWithProgress(@NotNull ProgressIndicator progress) {
    List<HighlightInfo> results = PerformanceTracker.measureTime(
        "Custom analysis",
        () -> performAnalysis()
    );
}
```

### 3. Memory Usage Tracking

```java
public class MemoryTracker {
    public static void logMemoryUsage(String context) {
        Runtime runtime = Runtime.getRuntime();
        long maxMemory = runtime.maxMemory();
        long totalMemory = runtime.totalMemory();
        long freeMemory = runtime.freeMemory();
        long usedMemory = totalMemory - freeMemory;
        
        LOG.info(String.format("%s - Memory: %d/%d MB (%.1f%% used)",
            context,
            usedMemory / 1024 / 1024,
            maxMemory / 1024 / 1024,
            (double) usedMemory / maxMemory * 100));
    }
}
```

## Testing Performance Improvements

### 1. Unit Testing for Performance

```java
public class PerformanceTest extends LightPlatformCodeInsightTestCase {
    
    public void testHighlightingPerformance() {
        // Create test file with known content
        configureByText("test.java", generateLargeFile());
        
        long startTime = System.currentTimeMillis();
        
        // Trigger highlighting
        DaemonCodeAnalyzer.getInstance(getProject()).restart();
        DaemonCodeAnalyzerTestCase.waitForDaemon();
        
        long duration = System.currentTimeMillis() - startTime;
        
        // Assert reasonable performance
        assertTrue("Highlighting took too long: " + duration + "ms", 
                   duration < 5000);
    }
    
    private String generateLargeFile() {
        // Generate test content that stresses your analysis
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < 1000; i++) {
            sb.append("public class Class").append(i).append(" {\n");
            sb.append("    // Some content that triggers analysis\n");
            sb.append("}\n");
        }
        return sb.toString();
    }
}
```

### 2. Integration Testing

```java
public class IntegrationPerformanceTest extends LightPlatformTestCase {
    
    public void testRealWorldPerformance() {
        // Test with real project files
        String projectPath = getTestDataPath() + "/performanceTest";
        PsiFile[] files = loadProjectFiles(projectPath);
        
        for (PsiFile file : files) {
            long start = System.currentTimeMillis();
            
            // Run your analysis
            runAnalysisOnFile(file);
            
            long duration = System.currentTimeMillis() - start;
            System.out.println("File " + file.getName() + " analyzed in " + duration + "ms");
        }
    }
}
```

## Best Practices Summary

### Do:
- ✅ Use background threads for heavy computation
- ✅ Implement proper cancellation checking
- ✅ Cache analysis results appropriately
- ✅ Use weak references for PSI elements
- ✅ Implement proper disposable management
- ✅ Monitor performance metrics
- ✅ Use modern async APIs and coroutines

### Don't:
- ❌ Perform heavy operations on EDT
- ❌ Hold strong references to PSI elements
- ❌ Access PSI outside read actions
- ❌ Ignore cancellation requests
- ❌ Implement blocking operations without timeouts
- ❌ Create memory leaks with listeners
- ❌ Assume external tools are always available

## Tools and Resources

### IDE Tools
- **Help → Diagnostic Tools → CPU Usage**
- **Help → Diagnostic Tools → Analyze Memory Usage**
- **View → Show Memory Indicator**
- **Tools → Internal Actions → Show Performance Warnings**

### External Tools
- [JProfiler](https://www.ej-technologies.com/products/jprofiler/overview.html)
- [YourKit](https://www.yourkit.com/)
- [VisualVM](https://visualvm.github.io/)

### Documentation
- [IntelliJ Platform Performance Guidelines](https://plugins.jetbrains.com/docs/intellij/performance.html)
- [Threading Model](https://plugins.jetbrains.com/docs/intellij/threading-model.html)
- [PSI Performance](https://plugins.jetbrains.com/docs/intellij/psi-performance.html)

## See Also

- [IDE Linting Architecture Guide](ide-linting-architecture.md)
- [IntelliJ Platform SDK Documentation](https://plugins.jetbrains.com/docs/intellij/)