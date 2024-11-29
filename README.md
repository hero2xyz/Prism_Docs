# Prism - Probability Evaluation System

## Overview
Prism is a robust probability evaluation and modification framework for Unreal Engine, providing a thread-safe, context-aware system for handling probability-based mechanics. It excels at modifying and evaluating probabilities while maintaining context and performance optimization through caching and pooling.

## Table of Contents
- [Core Features](#core-features)
- [System Architecture](#system-architecture)
- [Installation](#installation)
- [Getting Started](#getting-started)
- [Core Components](#core-components)
- [Advanced Topics](#advanced-topics)
- [Performance Considerations](#performance-considerations)
- [API Reference](#api-reference)

## Core Features
- Context-aware probability evaluation
- Thread-safe operations
- Evaluator pooling and result caching
- Synchronous and asynchronous evaluation support
- Batch processing capabilities
- Hierarchical context system
- Extensive debugging and statistics tracking
- Blueprint support

## System Architecture

### Core Components
1. **PrismSubsystem**: Main entry point and manager
2. **PrismContextTracker**: Manages evaluation contexts
3. **PrismEvaluator**: Base class for probability evaluators
4. **PrismEvaluatorPool**: Manages evaluator instances
5. **PrismCache**: Caches evaluation results

## Installation

### Requirements
- Unreal Engine 5.0 or later
- Compatible platforms: Windows (other platforms untested)

### Setup Steps
1. Add the plugin to your project's Plugins folder
2. Enable the plugin in the Unreal Editor
3. Add to your Build.cs:
```csharp
PublicDependencyModuleNames.AddRange(
    new string[] { 
        "Core",
        "CoreUObject",
        "Engine",
        "Prism"
    }
);
```

## Getting Started

### Basic Usage

#### Initialize the System
```cpp
// Get the Prism subsystem
UPrismSubsystem* PrismSystem = GetGameInstance()->GetSubsystem<UPrismSubsystem>();
```

#### Simple Probability Evaluation
```cpp
// Create a context
FPrismContext Context = UPrismBlueprintLibrary::CreateContext("Basic Check");

// Evaluate a probability
float BaseProbability = 0.5f;
FPrismEvaluation Result = PrismSystem->EvaluateWithContext(BaseProbability, Context);

if (Result.bSuccess)
{
    // Handle success
}
```

### Using Global Evaluators

```cpp
// Register a global evaluator
FGameplayTag EvaluatorTag = FGameplayTag::RequestGameplayTag(FName("Prism.MyEvaluator"));
PrismSystem->RegisterGlobalEvaluator(UMyProbabilityEvaluator::StaticClass(), EvaluatorTag);

// Evaluate using the global evaluator
FPrismEvaluation Result = PrismSystem->EvaluateWithGlobalEvaluator(
    BaseProbability, 
    Context, 
    EvaluatorTag
);
```

## Core Components

### PrismContext
The context system provides metadata and state tracking for evaluations.

#### Context Features
- Unique identifiers (GUID)
- Parent-child relationships
- Extensible metadata storage
- Timestamp tracking

#### Managing Context Metadata
```cpp
// Adding metadata
UPrismBlueprintLibrary::SetContextFloat(Context, "Level", 5.0f);
UPrismBlueprintLibrary::SetContextString(Context, "Zone", "Dungeon");

// Reading metadata
float Level;
if (UPrismBlueprintLibrary::GetContextFloat(Context, "Level", Level))
{
    // Use Level value
}
```

### Custom Evaluators

```cpp
UCLASS()
class UCustomEvaluator : public UPrismEvaluator
{
    GENERATED_BODY()

public:
    virtual FPrismOperationResult Evaluate_Implementation(
        float InProbability, 
        float& OutProbability, 
        const FPrismContext& Context) override
    {
        // Validate input
        FString Error;
        if (!ValidateProbability(InProbability, Error))
        {
            return FPrismOperationResult::Error(
                EPrismResult::InvalidProbability, 
                Error
            );
        }

        // Modify probability
        OutProbability = InProbability;
        
        // Example: Check context metadata
        float Level;
        if (UPrismBlueprintLibrary::GetContextFloat(Context, "Level", Level))
        {
            OutProbability *= (1.0f + (Level * 0.1f));
        }

        return FPrismOperationResult::Success();
    }
};
```

### Asynchronous Operations

```cpp
void OnEvaluationComplete(const FPrismEvaluation& Result)
{
    if (Result.Result.IsSuccess())
    {
        // Handle success
    }
}

// Async evaluation
PrismSystem->EvaluateWithContextAsync(
    BaseProbability,
    Context,
    FPrismEvaluationDelegate::CreateUObject(
        this, 
        &ThisClass::OnEvaluationComplete
    )
);
```

### Batch Processing

```cpp
void OnBatchComplete(const TArray<FPrismEvaluation>& Results)
{
    for (const FPrismEvaluation& Result : Results)
    {
        if (Result.bSuccess)
        {
            // Handle individual success
        }
    }
}

// Setup batch probabilities
TArray<float> Probabilities;
Probabilities.Add(0.5f);
Probabilities.Add(0.3f);
Probabilities.Add(0.7f);

// Configure batch
FPrismBatchConfig Config;
Config.MaxBatchSize = 100;

// Process batch
PrismSystem->EvaluateBatch(
    Probabilities,
    Context,
    FPrismBatchDelegate::CreateUObject(
        this, 
        &ThisClass::OnBatchComplete
    ),
    Config
);
```
# Evaluation Pipeline & Global Evaluators

## Evaluation Pipeline

The Prism evaluation system uses a three-phase pipeline that provides fine-grained control over probability modifications:

### Pre-Evaluation Phase
The pre-evaluation phase is used for:
- Validating input data and context
- Setting up necessary resources or state
- Performing preliminary checks
- Initializing any required data for the main evaluation

Example:
```cpp
virtual FPrismOperationResult PreEvaluate_Implementation(const FPrismContext& Context) override
{
    // Validate required context metadata
    float PlayerLevel;
    if (!UPrismBlueprintLibrary::GetContextFloat(Context, "PlayerLevel", PlayerLevel))
    {
        return FPrismOperationResult::Error(
            EPrismResult::InvalidContext, 
            TEXT("Missing PlayerLevel in context")
        );
    }

    // Initialize state for evaluation
    LastCheckedLevel = PlayerLevel;
    return FPrismOperationResult::Success();
}
```

### Evaluation Phase
The main evaluation phase is where the actual probability modification occurs:
- Modifying the input probability based on game rules
- Applying bonuses or penalties
- Implementing core probability logic
- Reading and utilizing context data

Example:
```cpp
virtual FPrismOperationResult Evaluate_Implementation(
    float InProbability, 
    float& OutProbability, 
    const FPrismContext& Context) override
{
    // Start with base probability
    OutProbability = InProbability;

    // Apply level-based modification
    if (LastCheckedLevel > 10.0f)
    {
        OutProbability *= 1.2f; // 20% bonus for high level
    }

    // Ensure probability stays in valid range
    OutProbability = FMath::Clamp(OutProbability, 0.0f, 1.0f);
    
    return FPrismOperationResult::Success();
}
```

### Post-Evaluation Phase
The post-evaluation phase is for:
- Cleanup operations
- Recording evaluation results
- Updating statistics
- Modifying context for future evaluations
- Triggering follow-up actions

Example:
```cpp
virtual FPrismOperationResult PostEvaluate_Implementation(
    const FPrismContext& Context, 
    bool bSuccess) override
{
    // Record result for future reference
    UPrismBlueprintLibrary::SetContextBool(
        Context, 
        "LastEvaluationResult", 
        bSuccess
    );

    // Update evaluation counter
    float EvalCount;
    if (UPrismBlueprintLibrary::GetContextFloat(Context, "EvalCount", EvalCount))
    {
        UPrismBlueprintLibrary::SetContextFloat(
            Context, 
            "EvalCount", 
            EvalCount + 1.0f
        );
    }

    return FPrismOperationResult::Success();
}
```

## Global Evaluators

Global evaluators are system-wide probability modifiers that can be registered with tags for easy reference and reuse.

### Key Features
- Registered with gameplay tags for identification
- Managed by the Prism subsystem
- Can be enabled/disabled globally
- Support priority ordering
- Maintain global state

### Registering Global Evaluators

```cpp
// Register in C++
FGameplayTag Tag = FGameplayTag::RequestGameplayTag("Prism.GlobalMods.PlayerLevel");
PrismSystem->RegisterGlobalEvaluator(ULevelBasedEvaluator::StaticClass(), Tag);

// Configure through project settings
// In DefaultGame.ini:
[/Script/Prism.PrismSettings]
DefaultEvaluators=(Tag="Prism.GlobalMods.PlayerLevel",EvaluatorClass="/Script/YourGame.LevelBasedEvaluator")
```

### Using Global Evaluators

```cpp
// Single evaluation with specific global evaluator
FPrismEvaluation Result = PrismSystem->EvaluateWithGlobalEvaluator(
    BaseProbability,
    Context,
    FGameplayTag::RequestGameplayTag("Prism.GlobalMods.PlayerLevel")
);

// Async evaluation
PrismSystem->EvaluateWithGlobalEvaluatorAsync(
    BaseProbability,
    Context,
    Tag,
    FPrismEvaluationDelegate::CreateUObject(this, &ThisClass::OnComplete)
);
```

### Global Evaluator Priority

When multiple global evaluators are active, they are processed in order based on their priority:
1. Critical
2. High
3. Normal
4. Low

Example priority setting:
```cpp
virtual EPrismPriority GetPriority() const override
{
    return EPrismPriority::High;
}
```

### Managing Global Evaluators

```cpp
// Check if global evaluator exists
UPrismEvaluator* Evaluator = PrismSystem->GetGlobalEvaluator(Tag);

// Get all registered global evaluators
TArray<FEvaluatorInfo> GlobalEvaluators = PrismSystem->GetGlobalEvaluators();

// Unregister a global evaluator
PrismSystem->UnregisterGlobalEvaluator(Tag);
```

### Best Practices for Global Evaluators
1. Use clear, hierarchical tags for organizationfare
2. Keep evaluators focused on single responsibilities
3. Consider evaluation order when setting priorities
4. Use configuration to enable/disable evaluators
5. Implement proper cleanup in evaluator destruction
# Working with Contexts and Metadata

## Introduction
The Prism system implements a robust hierarchical context model for managing evaluation data and metadata. It offers thread safety, clear data inheritance, and comprehensive tracking capabilities through parent-child relationships. A key feature of Prism is its use of FInstancedStruct for metadata storage, enabling type-safe handling of any UStruct-based data. This ensures an efficient and secure way to store and manage state during evaluations.

## Context Hierarchy

### Core Components
- **Unique Identifiers**: Each context has a `OperationId` (GUID)
- **Relationships**:
  - Single parent (`ParentId`)
  - Multiple possible children
- **Metadata**: Inheritable key-value store
- **Tracking**: Creation timestamps and reasons

### Structure Example
```
BaseContext (OperationId: ABC)
├── ChildContext1 (ParentId: ABC)
│   └── GrandchildContext (ParentId: ChildContext1)
└── ChildContext2 (ParentId: ABC)
```

## Core Concepts

### Context Creation
```cpp
// Create base context
FPrismContext BaseContext = UPrismBlueprintLibrary::CreateContext("InitialRequest");

// Add initial metadata
UPrismBlueprintLibrary::SetContextFloat(BaseContext, "BaseValue", 0.75f);
FPrismContextTracker::Get().RegisterContext(BaseContext);
```

### Child Context Management
```cpp
// Create and configure child
FPrismContext ChildContext = CreateChildContext(ParentContext, "EvaluationPhase1");
UPrismBlueprintLibrary::SetContextFloat(ChildContext, "CalculatedValue", newValue);

// Register after configuration
FPrismContextTracker::Get().RegisterContext(ChildContext);
```

## Implementation Guide

### Evaluator Implementation
```cpp
FPrismOperationResult Evaluate_Implementation(float InProbability, 
    float& OutProbability, 
    const FPrismContext& Context)
{
    // Create evaluation context
    FPrismContext EvalContext = CreateChildContext(Context, "ProbabilityEvaluation");
    
    // Perform calculations
    float ModifiedProbability = CalculateModification(InProbability);
    
    // Store results
    UPrismBlueprintLibrary::SetContextFloat(EvalContext, "ModifiedValue", ModifiedProbability);
    FPrismContextTracker::Get().RegisterContext(EvalContext);
    
    OutProbability = ModifiedProbability;
    return FPrismOperationResult::Success();
}
```

### Accessing Results
```cpp
// Perform evaluation
FPrismEvaluation Result = PrismSubsystem->EvaluateWithContext(Probability, BaseContext);

// Retrieve results
float ModifiedValue;
bool bSuccess = UPrismBlueprintLibrary::GetContextFloat(
    Result.Context, 
    "ModifiedValue", 
    ModifiedValue
);
```

## Best Practices

### Context Creation Rules
1. **Complete Configuration Before Registration**
   ```cpp
   FPrismContext Context = CreateChildContext(Parent, "Reason");
   SetupAllMetadata(Context);  // Configure everything first
   RegisterContext(Context);   // Register only when ready
   ```

2. **Descriptive Reason Strings**
   ```cpp
   // Good
   CreateChildContext(Parent, "CalculatingSpawnProbability");
   
   // Bad
   CreateChildContext(Parent, "Update");  // Too vague
   ```

3. **Metadata Management**
   ```cpp
   // Batch updates for efficiency
   TMap<FName, FInstancedStruct> BatchData = PrepareMetadata();
   UPrismBlueprintLibrary::SetOrUpdateMetadata(Context, BatchData, true);
   ```

### Common Pitfalls to Avoid
```cpp
// ❌ WRONG: Premature registration
FPrismContext Context = CreateChildContext(Parent, "Reason");
RegisterContext(Context);  // Too early!
SetupMetadata(Context);   // Data might be missed by other systems

// ✅ CORRECT: Complete setup before registration
FPrismContext Context = CreateChildContext(Parent, "Reason");
SetupMetadata(Context);   // Configure first
RegisterContext(Context); // Register when ready
```

## Advanced Features

### Context Searching
```cpp
FContextSearchCriteria Criteria;
Criteria.MinAge = 0.0f;
Criteria.MaxAge = 30.0f;
Criteria.bMustHaveChildren = true;
Criteria.RequiredMetadata.Add("Category", CategoryValue);

TArray<FPrismContext> Results = FindContexts(Criteria);
```

### Hierarchy Navigation
```cpp
// Get full ancestry
TArray<FPrismContext> ParentChain = GetParentChain(ContextId);

// Find root context
FPrismContext RootContext;
GetRootContext(ContextId, RootContext);

// Check relationships
bool bIsAncestor = IsAncestorOf(PotentialAncestorId, ContextId);
```

## Debugging and Monitoring

### Debug Tools
```cpp
// Log hierarchy
LogContextHierarchy(Context);

// Get debug information
FString DebugInfo = GetContextDebugString(Context);
FString HierarchyInfo = GetHierarchyDebugString(Context.OperationId);
```

### Performance Monitoring
```cpp
FPrismTrackerStats Stats = GetStats();
UE_LOG(LogPrism, Log, TEXT("Active Contexts: %d"), Stats.ActiveContexts);
UE_LOG(LogPrism, Log, TEXT("Average Age: %.2f"), Stats.AverageContextAge);
UE_LOG(LogPrism, Log, TEXT("Memory Usage: %zu"), Stats.EstimatedMemoryUsage);
```

## Performance and Cleanup

### Automatic Cleanup Triggers
- Exceeding `MaxTrackedContexts`
- Contexts older than `TrackedContextLifetime`
- System memory pressure

### Manual Cleanup
```cpp
// Clean expired contexts
FPrismContextTracker::Get().CleanupExpiredContexts(30.0f);

// Emergency cleanup
FPrismContextTracker::Get().EmergencyCleanup(1000);

// Remove orphaned contexts
FPrismContextTracker::Get().CleanupOrphanedContexts();
```

### Performance Optimization
- Use batch metadata updates when possible
- Clean up old contexts regularly
- Monitor context creation rate
- Use appropriate search criteria

## Thread Safety Considerations

### Protected Operations
- Context registration/unregistration is thread-safe through `FPrismContextTracker`'s `ContextLock`
- Context lookup and hierarchy operations are thread-safe
- Context cleanup operations are thread-safe

### Non-Thread-Safe Operations
- **Metadata Operations**: Operations on a context's metadata are NOT thread-safe
  ```cpp
  // These operations are not thread-safe:
  Context.SetMetadata("Key", Value);
  Context.GetMetadata("Key", OutValue);
  ```

### Thread Safety Implementation Guide

#### For Single-Thread Usage
```cpp
// Safe in single-threaded environment
FPrismContext Context = CreateChildContext(Parent, "Reason");
SetContextFloat(Context, "Value", 1.0f);
RegisterContext(Context);
```

#### For Multi-Thread Usage
You need to implement your own synchronization:

```cpp
// Example of user-implemented thread safety
class FThreadSafeContextWrapper
{
    FPrismContext Context;
    FCriticalSection MetadataLock;

public:
    template<typename T>
    void SetMetadataSafe(const FName& Key, const T& Value)
    {
        FScopeLock Lock(&MetadataLock);
        Context.SetMetadata(Key, Value);
    }

    template<typename T>
    bool GetMetadataSafe(const FName& Key, T& OutValue)
    {
        FScopeLock Lock(&MetadataLock);
        return Context.GetMetadata(Key, OutValue);
    }
};
```

### Best Practices for Thread Safety

1. **Complete Configuration Before Sharing**
   ```cpp
   // Good: Configure everything before making context available to other threads
   FPrismContext Context = CreateChildContext(Parent, "Reason");
   SetupAllMetadata(Context);  // Do all metadata operations while single-threaded
   RegisterContext(Context);   // Only then make it available to other threads
   ```

2. **Avoid Concurrent Metadata Modifications**
   ```cpp
   // Bad: Potential race conditions
   void ThreadFunc1() { Context.SetMetadata("Key1", Value1); }
   void ThreadFunc2() { Context.SetMetadata("Key2", Value2); }

   // Good: Use synchronization if needed
   FCriticalSection ContextLock;
   void ThreadFunc1()
   {
       FScopeLock Lock(&ContextLock);
       Context.SetMetadata("Key1", Value1);
   }
   ```

3. **Read-Only After Registration**
   - Treat contexts as immutable after registration when possible
   - If modifications are needed, implement proper synchronization

### System Design Considerations

1. **Context Lifecycle**
   ```cpp
   // Recommended pattern:
   // 1. Create context (single thread)
   FPrismContext Context = CreateChildContext(Parent, "Reason");
   
   // 2. Configure metadata (single thread)
   SetupMetadata(Context);
   
   // 3. Register (thread-safe operation)
   RegisterContext(Context);
   
   // 4. After this point, treat context as read-only
   // Or implement proper synchronization for modifications
   ```

2. **Batch Operations**
   ```cpp
   // Prepare all data before touching the context
   TMap<FName, FInstancedStruct> BatchData = PrepareMetadata();
   
   // Minimize time spent modifying the context
   FScopeLock Lock(&YourContextLock);
   UPrismBlueprintLibrary::SetOrUpdateMetadata(Context, BatchData, true);
   ```

3. **Performance Implications**
   - Adding synchronization can impact performance
   - Consider using lock-free structures for high-performance requirements
   - Batch operations when possible to minimize lock contention

### Working with Metadata
The context system uses `FInstancedStruct` to store metadata values, providing type-safety and flexibility. Common value types are wrapped in Prism-specific structs:

```cpp
// Basic type wrappers
FPrismFloat    // For float values
FPrismInt      // For integer values
FPrismBool     // For boolean values
FPrismString   // For string values
FPrismName     // For FName values
FPrismVector   // For FVector values
FPrismRotator  // For FRotator values
FPrismTransform // For FTransform values
```

### Setting Metadata
```cpp
// Using blueprint library helpers
UPrismBlueprintLibrary::SetContextFloat(Context, "PlayerLevel", 10.0f);
UPrismBlueprintLibrary::SetContextString(Context, "ZoneID", "Dungeon_01");

// Direct metadata manipulation
Context.SetMetadata<FPrismFloat>("Multiplier", FPrismFloat(1.5f));

// Using custom structs
FMyCustomStruct CustomData;
Context.Metadata.Add("CustomData", FInstancedStruct::Make<FMyCustomStruct>(CustomData));
```

### Reading Metadata
```cpp
// Using blueprint library helpers
float Level;
if (UPrismBlueprintLibrary::GetContextFloat(Context, "PlayerLevel", Level))
{
    // Use Level value
}

// Direct access with type checking
FPrismFloat MultiplierStruct;
if (Context.GetMetadata("Multiplier", MultiplierStruct))
{
    float Multiplier = MultiplierStruct.Value;
}

// Working with custom structs
const FMyCustomStruct* CustomData = Context.Metadata["CustomData"].GetPtr<FMyCustomStruct>();
if (CustomData)
{
    // Use CustomData
}
```

### Context Inheritance
Child contexts inherit metadata from their parents but can override or add new values:

```cpp
// Parent context with base data
FPrismContext ParentContext = UPrismBlueprintLibrary::CreateContext("Parent");
UPrismBlueprintLibrary::SetContextFloat(ParentContext, "BaseMultiplier", 1.0f);

// Child context inherits and can override
FPrismContext ChildContext = ParentContext.CreateChildContext("Child");
UPrismBlueprintLibrary::SetContextFloat(ChildContext, "BaseMultiplier", 1.5f); // Override
UPrismBlueprintLibrary::SetContextString(ChildContext, "SubOperation", "Special"); // New value
```

### Custom Struct Example
```cpp
// Define a custom struct
USTRUCT()
struct FMyProbabilityModifier
{
    GENERATED_BODY()

    UPROPERTY()
    float Multiplier = 1.0f;

    UPROPERTY()
    int32 StackCount = 0;

    UPROPERTY()
    FString Source;
};

// Using with context
void StoreModifier(FPrismContext& Context, const FMyProbabilityModifier& Modifier)
{
    Context.Metadata.Add("ProbModifier", FInstancedStruct::Make<FMyProbabilityModifier>(Modifier));
}

bool GetModifier(const FPrismContext& Context, FMyProbabilityModifier& OutModifier)
{
    const FInstancedStruct* StoredStruct = Context.Metadata.Find("ProbModifier");
    if (const FMyProbabilityModifier* Modifier = StoredStruct->GetPtr<FMyProbabilityModifier>())
    {
        OutModifier = *Modifier;
        return true;
    }
    return false;
}
```

### Best Practices
1. Use the provided wrapper types for basic values
2. Create custom structs for complex metadata
3. Use meaningful keys for metadata
4. Check validity when retrieving metadata
5. Keep context hierarchies shallow and logical
6. Clean up contexts when no longer needed

## Advanced Topics

### Optimizing Cache Usage
The caching system automatically stores and retrieves evaluation results based on context and evaluator combinations.

```cpp
// Configure cache in project settings
UPrismSettings::Get()->bEnableCache = true;
UPrismSettings::Get()->MaxCachedResults = 10000;
UPrismSettings::Get()->CacheDuration = 60.0f;
```

### Memory Management
The evaluator pool helps manage memory usage for frequent evaluations.

```cpp
// Configure pool in project settings
UPrismSettings::Get()->bEnablePool = true;
UPrismSettings::Get()->MaxEvaluatorsPerType = 100;
UPrismSettings::Get()->InitialPoolSize = 10;
```

### Debugging and Statistics

```cpp
// Enable detailed logging
UPrismSettings::Get()->bEnableDetailedLogs = true;

// Get system stats
FPrismStats Stats = PrismSystem->GetStats();
// Access various statistics like:
// - Stats.TotalEvaluations
// - Stats.SuccessfulEvaluations
// - Stats.AverageEvaluationTime
// - Stats.CacheHits
// - Stats.CacheMisses
```

## Performance Considerations

### Best Practices
1. Use batch processing for multiple evaluations
2. Enable caching for frequently repeated evaluations
3. Configure appropriate pool sizes
4. Clean up contexts when no longer needed
5. Use async operations for heavy workloads

### Thread Safety
The system is designed to be thread-safe, but consider:
- Avoid modifying shared state in evaluators
- Use async operations for time-consuming evaluations
- Be careful with context modifications during evaluation

## API Reference

### Key Classes
- `UPrismSubsystem`
- `FPrismContext`
- `UPrismEvaluator`
- `FPrismEvaluatorPool`
- `FPrismEvaluatorCache`
- `UPrismSettings`
- `UPrismBlueprintLibrary`

### Important Enums
- `EPrismPriority`
- `EPrismResult`
- `EPrismLogLevel`

### Key Structs
- `FPrismEvaluation`
- `FPrismOperationResult`
- `FPrismStats`
- `FPrismTrackerStats`
- `FContextSearchCriteria`

## Settings Configuration

The system can be configured through the project settings:

### Core Settings
- Enable/disable global evaluators
- Configure multiple evaluation behavior
- Set operation timeouts

### Pool Settings
- Maximum evaluators per type
- Initial pool size
- Cleanup intervals

### Cache Settings
- Maximum cached results
- Cache duration
- Cleanup intervals

### Debug Settings
- Enable detailed logging
- Track performance metrics
- Configure log history size

## Known Limitations
1. Contexts are not persisted between sessions
2. Statistics are reset on application restart
3. Currently tested only on Windows platform

## Support and Contact
- [Discord Server](https://discord.gg/kH6TC9Hc)
