# IntelliJ IDEA Linting and Code Analysis Architecture

This document explains how IntelliJ IDEA's code linting, error checking, and syntax analysis system works internally. Understanding this architecture is crucial for plugin developers who want to optimize their code analysis contributions or troubleshoot performance issues.

## Overview

IntelliJ IDEA's linting system is built around a multi-pass highlighting architecture that continuously analyzes code and provides real-time feedback to users. The system consists of several key components working together:

1. **DaemonCodeAnalyzer** - The central coordinator
2. **Highlighting Passes** - Individual analysis tasks
3. **TrafficLightRenderer** - Visual feedback system
4. **ErrorStripeUpdateManager** - UI update coordination
5. **EditorMarkupModel** - Visual representation in the editor

## Architecture Components

### 1. DaemonCodeAnalyzer (`DaemonCodeAnalyzerImpl`)

The `DaemonCodeAnalyzer` is the heart of the linting system. It:

- Orchestrates all code analysis activities
- Manages highlighting sessions
- Coordinates between different analysis passes
- Handles file change notifications
- Controls the analysis lifecycle

**Key responsibilities:**
- Starting and stopping analysis sessions
- Managing the pool of highlighting passes
- Handling priority-based analysis (visible range first)
- Coordinating with the PSI (Program Structure Interface) system

**Location:** `platform/lang-impl/src/com/intellij/codeInsight/daemon/impl/DaemonCodeAnalyzerImpl.java`

### 2. Highlighting Passes

The analysis work is divided into multiple specialized passes, each handling different aspects of code analysis:

#### Core Pass Types:

**LocalInspectionsPass** (`LocalInspectionsPass.java`)
- Runs local code inspections
- Handles language-specific analysis
- Applies inspection profiles
- Manages suppression handling

**LineMarkersPass** (`LineMarkersPass.java`)
- Adds gutter icons and line markers
- Handles navigation targets
- Provides contextual actions

**ExternalAnnotatorPass** (`ExternalToolPass.java`)
- Integrates external analysis tools
- Handles asynchronous analysis results
- Manages tool-specific configurations

**ShowIntentionsPass** (`ShowIntentionsPass.java`)
- Collects available quick fixes and intentions
- Manages intention action availability
- Handles context-sensitive suggestions

#### Pass Execution Flow:

```
File Change Event
        ↓
DaemonCodeAnalyzer.restart()
        ↓
MainPassesRunner.runMainPasses()
        ↓
┌─────────────────────────────────┐
│ Priority Range Analysis          │
│ (visible editor area first)     │
└─────────────────────────────────┘
        ↓
┌─────────────────────────────────┐
│ Parallel Pass Execution:        │
│ - LocalInspectionsPass          │
│ - LineMarkersPass              │
│ - ExternalAnnotatorPass        │
│ - IdentifierHighlighterPass    │
│ - ShowIntentionsPass           │
│ - IndentsPass                  │
└─────────────────────────────────┘
        ↓
Results Collection & UI Update
        ↓
TrafficLightRenderer Update
```

### 3. TrafficLightRenderer

The `TrafficLightRenderer` provides visual feedback about the analysis status through the traffic light icon in the editor's top-right corner.

**Location:** `platform/lang-impl/src/com/intellij/codeInsight/daemon/impl/TrafficLightRenderer.kt`

**Key functions:**
- Aggregates error counts from all passes
- Displays analysis progress
- Shows overall file health status
- Provides click actions for navigation

**Status Types:**
- 🟢 Green: No errors or warnings
- 🟡 Yellow: Warnings present
- 🔴 Red: Errors present
- ⚪ Gray: Analysis in progress or disabled

### 4. ErrorStripeUpdateManager

Manages updates to the error stripe (right-hand side panel showing error markers).

**Location:** `platform/lang-impl/src/com/intellij/codeInsight/daemon/impl/ErrorStripeUpdateManager.kt`

**Responsibilities:**
- Coordinates error stripe repainting
- Manages multiple editor synchronization
- Handles asynchronous UI updates
- Optimizes update frequency

### 5. EditorMarkupModel

Handles the visual representation of highlights, errors, and markers within the editor.

**Location:** `platform/platform-impl/src/com/intellij/openapi/editor/impl/EditorMarkupModelImpl.kt`

**Key features:**
- Manages range highlighters
- Handles error stripe markers
- Coordinates with the traffic light icon
- Manages editor overlays and tooltips

## Analysis Lifecycle

### 1. Trigger Events

Analysis can be triggered by:
- File content changes
- File focus changes
- PSI tree modifications
- Configuration changes
- External file system events

### 2. Session Management

Each analysis session:
- Has a unique session ID
- Tracks progress across multiple passes
- Handles cancellation gracefully
- Manages resource cleanup

### 3. Priority System

The system uses a priority-based approach:
1. **High Priority**: Visible editor range
2. **Medium Priority**: Current file outside visible range
3. **Low Priority**: Background files

### 4. Cancellation and Restart

- Analysis can be cancelled if new changes occur
- Smart restart logic avoids redundant work
- Progressive cancellation minimizes waste

## Threading Model

The linting system operates across multiple threads:

### Background Thread (BGT)
- Heavy PSI analysis work
- File I/O operations
- External tool invocations

### Event Dispatch Thread (EDT)
- UI updates
- Highlighting application
- User interaction handling

### Thread Coordination
- Uses coroutines for async coordination
- Employs read/write locks for PSI access
- Implements cancellation tokens for cleanup

## Performance Considerations

### 1. Incremental Analysis
- Only re-analyzes changed regions when possible
- Maintains analysis caches
- Implements smart invalidation

### 2. Resource Management
- Limits concurrent analysis sessions
- Implements timeouts for long-running operations
- Uses memory-aware processing

### 3. UI Responsiveness
- Prioritizes visible content
- Uses progressive disclosure
- Implements non-blocking updates

## Integration Points for Plugins

### 1. Custom Highlighting Passes
Plugins can register custom highlighting pass factories:

```java
public class MyHighlightingPassFactory implements TextEditorHighlightingPassFactory {
    @Override
    public TextEditorHighlightingPass createHighlightingPass(@NotNull PsiFile file, @NotNull Editor editor) {
        return new MyCustomPass(file.getProject(), file, editor.getDocument());
    }
}
```

### 2. Custom Inspections
Register local inspections that integrate with the system:

```java
public class MyInspection extends LocalInspectionTool {
    @Override
    public ProblemDescriptor[] checkFile(@NotNull PsiFile file, @NotNull InspectionManager manager, boolean isOnTheFly) {
        // Analysis logic
        return problems.toArray(ProblemDescriptor.EMPTY_ARRAY);
    }
}
```

### 3. External Annotators
For integrating external tools:

```java
public class MyExternalAnnotator extends ExternalAnnotator<CollectedInfo, AnnotationResult> {
    // Implementation for external tool integration
}
```

## Common Issues and Solutions

### 1. Performance Problems
- **Symptom**: Slow highlighting updates
- **Causes**: Heavy computation in passes, blocking operations
- **Solutions**: Move work to background threads, implement caching

### 2. Memory Leaks
- **Symptom**: Growing memory usage
- **Causes**: Holding references to PSI elements, not cleaning up listeners
- **Solutions**: Use weak references, proper disposable management

### 3. Threading Violations
- **Symptom**: Threading assertion failures
- **Causes**: PSI access from wrong threads, EDT violations
- **Solutions**: Use proper read actions, async APIs

## Related Components

- **PSI (Program Structure Interface)**: Provides the syntax tree
- **VirtualFileSystem**: Manages file change notifications
- **ProjectManager**: Handles project lifecycle
- **EditorFactory**: Manages editor instances
- **ActionSystem**: Provides intention actions and quick fixes

## Further Reading

- [IntelliJ Platform SDK Documentation](https://plugins.jetbrains.com/docs/intellij/)
- [Code Inspections and Intentions](https://plugins.jetbrains.com/docs/intellij/code-inspections-and-intentions.html)
- [Editor Basics](https://plugins.jetbrains.com/docs/intellij/editor-basics.html)

## See Also

- [Plugin Performance Troubleshooting Guide](plugin-linting-troubleshooting.md)
- [IntelliJ Platform Threading Model](https://plugins.jetbrains.com/docs/intellij/threading-model.html)