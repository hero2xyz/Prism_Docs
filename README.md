# PRISM - Probability System for Unreal Engine

Prism is a robust probability evaluation and modification framework for Unreal Engine, providing a thread-safe, context-aware system for handling probability-based mechanics. It excels at modifying and evaluating probabilities while maintaining context and performance optimization through caching and pooling.

## Features
- Thread-safe probability evaluation
- Context tracking and analysis 
- Priority-based evaluator system
- Performance optimization through caching and pooling
- Full Blueprint support
- Built-in performance monitoring

## Requirements
- Unreal Engine 5.3+
- C++ or Blueprint project capability
- Windows platform (other platforms untested)

## Installation 
1. Download `PRISM` from the Unreal Marketplace and add it to your project
2. Enable plugin in Unreal Editor
3. For C++ projects, add to your Build.cs:
```cpp
PublicDependencyModuleNames.AddRange(
    new string[] {
        "Core",
        "CoreUObject",
        "Engine",
        "PRISM"
    }
);
```
## Blueprints 

Documentation for blueprints is still in progress, in the meantime please have a look at the [Demo](https://drive.google.com/drive/u/1/folders/10-AQCA8ElHG92lBjHF_vWMdrNqUXzVxY) project with examples. If you have any questions you can ask me directly in [Discord](https://discord.com/invite/fJj5YcwQ). 

## Documentation
See the [Wiki](wiki/home) for detailed documentation, including:
- [Getting Started](wiki/context-system#getting-started)
- [Core Concepts](wiki/context-system#core-concepts)
- [API Reference](wiki/api)
- [Examples](wiki/examples)
- [Best Practices](wiki/guidelines)

## License
Copyright (c) 2024, Hero2. All rights reserved.
