# Incremental source generator pipeline

// @BillWagner It is this first section that I wonder if we want a general article on. Also, any feedback on this section.

Thee Roslyn incremental source generator uses a pipeline. This style of programming is common in functional programming and has been described in the F# community as Railway Oriented Programming or as a conveyer belt in a factory. The infrastructure is designed to pass a container and numerous methods can receive the container and return a new container with different contents.

LINQ supports this style of programming where the container is an `IEnumerable<T1>` collection. As you call methods of the LINQ pipeline like `Select` or `OfType` a new `IEnumerable<T2>` is returned. `T1` and `T2` may be of the same or different types. And like LINQ, the pipeline is a series of delegates that are defined and executed at a later point, possibly executed multiple times.

There is a great deal of theoretical work around these types of containers, and you can explore this by searching for terms like "monads" or functional programming books. Key to this approach is that the container is for the function, which is a delegate in .NET. You can see this in LINQ because an operation like `Select` or `Where` do nothing until an operation that uses the result, like a `foreach` loop or a call to `.ToList()` occurs.

All you need to know to understand incremental generator pipelines is a few basics. In .NET, containers are generic, such as `ContainerA<T>`. There are operations to create the first instance of `ContainerA<T1>` and containers have methods like `Select` that return a new instance `ContainerA<T2>`. Like LINQ, the generic type of the new container is often a different than the generic type. Eventually, the pipeline is complete and a method returns a value based on the contents of the last container.

@chsienki The spec refers to IValueProvider<T>, but I cannot find this interface. Is this now just a logical grouping of the ..Value.. and ..Values.. providers? (That is how I wrote this article)

The incremental generator pipeline supports two container types: `IncrementalValueProvider<T>` and `IncrementalValuesProvider<T>`. Notice that the first is singular and the second plural of  `Value`. The operations to create these containers are [providers](#providers). The available [pipeline operations](#pipeline-operations) transform the containers, and the pipeline ends by creating source code using the `RegisterSourceOutput` method.

## Providers

The incremental generator pipeline is created in the generators `Initialize` method. This is the only member of the `IIncrementalGenerator` interface. This method is passed a single parameter which is an `IncrementalGeneratorInitializationContext`.

 There are two ways to get these containers. The `IncrementalGeneratorInitializationContext` can provide several providers and the `SyntaxValueProvider`. The ones provided by the context are:

| Provider                      | Type                                                      |
|-------------------------------|-----------------------------------------------------------|
| CompilationProvider           | `IncrementalValueProvider<Compilation>`                   |
| AdditionalTextsProvider       | `IncrementalValuesProvider<AdditionalText>`               |
| AnalyzerConfigOptionsProvider | `IncrementalValueProvider<AnalyzerConfigOptionsProvider>` |
| MetadataReferencesProvider    | `IncrementalValueProvider<MetadataReference>`             |
| ParseOptionsProvider          | `IncrementalValueProvider<ParseOptions>`                  |

The `SyntaxValueProvider` works a little differently because you create an instance of it specific to your needs. The signature is;

```csharp
public readonly struct SyntaxValueProvider
    {
        public IncrementalValuesProvider<T> CreateSyntaxProvider<T>(
            Func<SyntaxNode, CancellationToken, bool> predicate, 
            Func<GeneratorSyntaxContext, CancellationToken, T> transform);
    }
```

This creates a provider that is specific to your request.

The transform step will frequently use the `SemanticModel` of the `GeneratorSyntaxContext` to do further filtering. An easy way to do this is to return `null` from the transform and then filter the result. [Check out this tip to avoid a null-reference warning](tips.md#-wherenotnull--method).

## Pipeline operations

The pipeline methods for incremental generators are extension methods on one or both of the provider types. This table abbreviates `IncrementalValueProvider` and `IncrementalValuesProvider` to make it easier to read:


| Method           | Input type                                 | Return type  | Comments                                       |
|------------------|--------------------------------------------|--------------|------------------------------------------------|
| Select           | `..Value..`                                | `..Value..`  |                                                |
| Select           | `..Values..`                               | `..Values..` |                                                |
| SelectMany       | `..Value..`                                | `..Values..` | IEnumerable and  immutable array overload      |
| SelectMany       | `..Values..`                               | `..Values..` | IEnumerable immutable array overload           |
| Where            | `..Values..`                               | `..Values..` |                                                |
| Collect          | `..Values..`                               | `..Value..`  |                                                |
| Combine          | `..Value..`, `..Value..`                   | `..Value..`  |                                                |
| Combine          | `..Values..`, `..Value..                   | `..Values..` |                                                |
| WithComparer     | `..Value..`, `IEqualityComparer<TSource>`  | `..Value..`  |                                                |
| WithComparer     | `..Values..`, `IEqualityComparer<TSource>` | `..Values..` | Allows per transformation equality for caching |
| WithTrackingName | `..Values..`, `string`                     | `..Values..` |                                                |
| WithTrackingName | `..Value..`, `string`                      | `..Value..`  |                                                |


You can find out more about how these methods work in the [Incremental Generators spec](https://github.com/dotnet/roslyn/blob/main/docs/features/incremental-generators.md).

## Output

The pipeline ends when you generate code output using the `RegisterImplementationSourceOutput` method. This method takes two parameters; an `IncrementalValuesProvider` and a delegate that creates the output. The generator iterates over the `IncrementalValuesProvider` and calls the delegate for each member passing a `SourceProductionContext` and the current member of the collection. Another overload of `RegisterSourceOutput` takes an `IncrementalValueProvider` and supports generating code from for a single value.

`RegisterSourceOutput` is passed an instance of the `SourceProductionContext` which can be used to add source files or report diagnostics.

`RegisterImplementationSourceOutput` is similar to `RegisterImplementationSourceOutput`, and also indicates that the source has no semantic impact. You can use this if the code you generate does not impact the IDE, such as providing IntelliSense entries or diagnostics like errors or warnings.

`RegisterPostInitializationOutput` allows you to output code that needs no inputs. This is useful for adding code that should always be in the user's compilation. This is especially useful for attributes and base classes. Although, also see the tip on [whether to user `RegisterPostInitializationOutput` or a separate runtime library](tips.md#attributes-in-registerpostinitializationoutput-or-a-separate-package)