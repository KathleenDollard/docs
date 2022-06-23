# Developing incremental source generators

The process of writing source generators is complicated by the fact you are writing code to create code, which increases the number of abstractions to consider and the ways that you need to test. Roslyn generators further complicate the picture because they are deployed as NuGet packages, and dependencies on NuGet packages that you are also actively developing is challenging. And finally, you need to ensure your generator is fast.

Breaking building a generator into discreet steps lets you focus on each step: 

[[make links when the names settle]]
* Structure the solution and project layout
* Create at least one sample project.
* Design a data model and check it for completeness.
* Create the code that builds the data model and test it in isolation.
* Create the code that outputs the generated code and test it.
* Create the generator and test end to end.
* Test performance.
* Create a NuGet package.

You can use these steps to build a solution that has a good development inner loop - meaning you can easily make changes, see the impact of those changes, and check for regressions. The [Roslyn incremental generator tutorial](tutorial.md) creates a sample project following these steps. This article covers what's important in each step and how they provide an appropriate solution layout, example project, technique for testing your generator, strategy for managing NuGet packages, and mechanisms for testing.

Before you get started, check the [limitations of Roslyn incremental source generators](overview.md#limitations-of-generators) in the Overview. It will also be helpful to read the about the [Roslyn incremental generators pipeline](pipeline.md).

## Structure the solution and project layout

The project layout of your solution must manage test and production concerns, generation-time and runtime concerns, and NuGet package references used to deploy generators.

Plan for a solution that is at least 4 or 5 projects:

* The *example project*.
* The project containing the *runtime library* - optional but may include attributes and base classes.
* The *incremental generator*.
* The *unit test project*.
* An *integration test project*.

You may need additional projects to organize code sharing between projects. Dependencies of the generator are deployed as NuGet packages. Projects shared to support the example project that are *not* generator dependencies can be either project or NuGet package references. Projects to support testing can also be either project or NuGet package references. The dependencies of the base set of projects are:

| Project               | Dependencies         |
|-----------------------|----------------------|
| Example               | Runtime              |
| Runtime library       |                      |
| Incremental generator | Runtime (project ref in development, package ref in release)|
| Unit test             | Generator            |
| Integration test      | Unit test, generator |

All of these dependencies are normal project references, except the dependency of the incremental generator on the runtime library.

The runtime library is separate from the generator because it includes code that is needed at both compile time and runtime. The generator depends on the runtime library and this will be a package reference when deployed. However, this package reference is challenging to manage during inner-loop development. You can treat these kinds of dependencies as project references during development and package references for deployment using a conditional MSBuild property:

```xml
<ItemGroup Condition="'$(CreatePackage)' == 'true'">
    <PackageReference Include="IncrementalGeneratorSamples.Runtime" Version="0.0.1-alpha" />
</ItemGroup>

<ItemGroup Condition="'$(CreatePackage)' != 'true'">
    <ProjectReference Include="../IncrementalGeneratorSamples.Runtime/IncrementalGeneratorSamples.Runtime.csproj" />
</ItemGroup>
```

## Create at least one example project

The important first step of writing a source generator is to create a working example that shows exactly what you want to output. This example also shows how your generated code will behave in the context of a working project.

Developers often get to a level of understanding of a problem and begin writing code. If you begin with that level of understanding to write the code of your generator, rather than your example project, you will work out the details in a relatively slow inner loop. If you create an example project, you can copy parts of it as the starting point for outputting code and use it as the basis for defining input. The details will already be worked out.

Your example project is just a normal .NET project. You might create it via the .NET templates in the .NET CLI or an editor like Visual Studio, or you might use a sample project that you've already created. If it is an executable, you can manually test it or write unit tests. If it is a library, you'll need unit tests. If you write unit tests for the example, you can use them as the basis for integration tests.

As you create this example project, you may need to use [partial classes and methods](https://docs.microsoft.com/dotnet/csharp/programming-guide/classes-and-structs/partial-classes-and-methods) to isolate the code you want to generate and isolate it from code that is logically part of the same class. Roslyn source generators add new syntax trees, effectively new files, to the compilation. Partial classes allow you to merge these knew files with existing source.

Within this example project isolate the code you plan to generate into separate files, and isolate these files in a subdirectory. This will:

* Let you design the interaction between generated classes and their base and partial classes.
* Provide the basis for designing how developers communicate with your generator to provide input (attributes, interfaces, well known names, etc.)
* Be available to copy as a starting point for outputting code.
* Provide the context for end to end testing - especially being able to compile your generated code in context.
* (Optional) Provide examples for unit tests to ensure your generated code runs correctly in the context of a project.
* Provide an example of how your generator works to use as part of your documentation.

Later this article suggests copying the directory that contains the code you will generate and overwriting the original. Because of this, you might name this subdirectory something like "OverwrittenInTests".

Ensuring this code runs and does what is expected before you begin will save time as you create your generator.

## Design a data model and check it for completeness.

Your generator will gather input data from sources like the user's code or external files. Regardless of the source, define an explicit data model even if it is only one value.

Incremental generators rely on caching, and caching relies on data models. You might only use one model, and it might contain only one value, or you may have several models that you transform during development. The key is that you extract the data you need from the underlying source in a discreet operation as early in the pipeline processing as possible. This discrete operation will be created either by calling `SyntaxValueProvider.CreateSyntaxProvider` or `Select` on any of the other providers of `IncrementalGeneratorInitializationContext`. The [provider section or the incremental source generator pipeline article](pipeline.md#providers) and the [incremental generator specification](https://github.com/dotnet/roslyn/blob/main/docs/features/incremental-generators.md) have more information on providers.

While a very simple model might be just a value or a tuple, most generators will use one or more data models that each contains several values. Each model must have value equality, including using `SequenceEquals` for any collections. This is most easily done using records or record structs and customizing the equality to accommodate collections or any other data that doesn't naturally have value equality.

Your data model should be sensible in the domain you are creating. If the domain is a command line definition, like System.CommandLine, the domain would include commands, arguments and options. Most generator use input values in multiple ways, so modeling your expected output does not work well.

Once you have an example project and a data model, map the generated code to ensure the data model contains all of the variable information. For example, a class name in the generated code may be variable and therefore depend on data in the model. If source code will provide the input for the data model, also map the data model to the non generated code in the example project to understand where each value originates. You may need to update the example project to better communicate the generation input. If the input data comes from another source, such as an additional file or configuration, map the data model to that source. These mappings can be simple bulleted lists.

## Create the code that builds the data model and test it

Some generators have a single source for the data model used for generation, and no further transformations are needed. Other generators have a transformation pipeline that combines data from multiple sources and generates numerous files from different data models. For complex generators, extra effort at planning transformations may be needed.

Regardless of the number of sources you need, first extract each one independently into an initial model that supports value equality.

### Extracting and testing a data model based on an attribute in C# or VB code

[[ Notes on Cyrus's tool]]

### Extracting and testing an initial data model from C# or VB code

If the input comes from source code, use the `IncrementalGeneratorInitializationContext.SyntaxValueProvider.CreateSyntaxProvider` method to create an `IncrementalValuesProvider` specific to your needs. To use this you need two delegates: the predicate and the transform. The simplest way to make these is to use methods. When using methods as a delegate, you can just omit the parentheses.

The first of these delegates is a predicate that takes a `SyntaxNode` and returns a Boolean. This method should be very fast. For example, pattern matching the `SyntaxNode` against a syntax type like `ClassDeclarationSyntax` is very fast. Once you make that first cut at filtering syntax nodes, you can further refine checking for specific attributes or well known names. The goal of the predicate is to *exclude* as many syntax nodes as possible from further consideration. If you need the semantic model to complete filtering, you will need to return true and filter further in the transform.

The transform is called for each syntax that passes the predicate. It takes a `GeneratorSyntaxContext` which provides the syntax node and the `SemanticModel`. You can use the semantic model's `GetSymbol` method to retrieve the corresponding `ISymbol` or the `GetOperation ` method to retrieve an `IOperation`. These give you more information that you can use to build your data model. 

When the extra information leads you to discard the node from further consideration, return `null` and use `Where` to filter out these entries. This may happen at unexpected times because the code is complete. For example, if you pattern match against a symbol that is not currently resolved, you will get a symbol representing an error instead of the expected symbol. Your generator will be called more often with invalid code than code that successfully compiles, so graciously manage this, generally by returning null and skipping generation.

Both of these methods are passed a `CancellationToken`. Ideally, all work in the predicate is very fast and in that case, the cancellation token can be discarded. The `CancellationToken` should always be passed to the methods of the semantic model that have a `CancellationToken` parameter, and should be checked before and after any slow operations and within any unbounded loops.

An example of a syntax provider is:

```csharp
var commandModelValues = initContext.SyntaxProvider
    .CreateSyntaxProvider(
        predicate: ModelBuilder.IsSyntaxInteresting,
        transform: ModelBuilder.GetModel)
    .Where(static m => m is not null)!;
```

### Extracting and testing an initial data model from other source

It is straightforward to use the providers on `IncrementalGeneratorInitializationContext`. You can find out more about the [available providers](pipeline.md#pipeline-operations). 

The all work similarly to the `AdditionalTextsProvider`. The provider supplies an `IncrementalValuesProvider<AdditionalText>` object with a member for each file in the project that is not part of the compilation. `AdditionalText` is not itself cacheable. You can use the LINQ like methods on `IncrementalValuesProvider` to filter and transform the provided data into a cacheable form:  

```c#
// get the contents of files that end with .txt
IncrementalValuesProvider<(string fileName, string content)> textFiles = 
    context.AdditionalTextsProvider
        .Where(static f => f.Path.EndsWith(".txt"))
        .Select((additionalText, cancellationToken) => 
            (fileName: Path.GetFileNameWithoutExtension(additionalText.Path), 
             content: additionalText.GetText(cancellationToken)!.ToString()));
```

Note that the `Select` method of `IncrementalValuesProvider` supplies a `CancellationToken`. Where a cancellation token is available, be sure to pass it on to any you methods that have a `CancellationToken` parameter.

This is a case where [considering the order of operations can improve performance](performance-guidelines.md#consider-operation-order). the `AdditionalText` objects returned from the provider are not cacheable. The result of the `Where` clause is also `AdditionalText`and not cacheable. The result of the `Select` is cacheable, and thus it would be possible to provide a cacheable result a little earlier in the pipeline. However, opening each file is expensive, so performance is better if the available files filtered prior to `GetText`.

### Combining, collection and transforming initial data models

You may have more than one initial data model, you may need to coalesce the members of an `IncrementalValuesProvider` into a single member, or you may need to transform models in some other way. [The available `IncrementalValuesProvider` methods are listed in the pipelines article](pipeline.md#pipeline-operations) and the [Roslyn incremental source generator specification](https://github.com/dotnet/roslyn/blob/main/docs/features/incremental-generators.md).

`Select`, `SelectMany` and `Where` work very similar to the way they work in LINQ, with the addition that `Select` and `SelectMany` take a cancellation token. By implication, if you need to include a cancellable method in filtering, first use `Select` to gather the data, and then filter.

### Collect

`Collect` allows you to create a single item that is a collection of the members. The signature of this method clarifies that it takes a `..Values..` and returns a `..Value..` provider that contains an `ImmutableArray`:

```csharp
IncrementalValueProvider<ImmutableArray<TSource>> Collect<TSource>(this IncrementalValuesProvider<TSource> source);
```

An example of when you may need to do this is calling `RegisterSourceOutput`. Calling this with a `..Values..` provider results in a new syntax tree per member. You might want this if you were creating a new class per item.Calling it with the collection in a `..Value..` provider results in one new syntax tree for the collection. You would need this if you were creating a new property per item.

### Combine

`Combine` is the most powerful and complicated of the transforming methods. There are three overloads that differ by when `..Value..` and `..Values..` is used:

```csharp
IncrementalValueProvider<(TLeft Left, TRight Right)> Combine<TLeft, TRight>(
    this IncrementalValueProvider<TLeft> provider1, IncrementalValueProvider<TRight> provider2);
IncrementalValueProvider<(TLeft Left, TRight Right)> Combine<TLeft, TRight>(
    this IncrementalValueProvider<TLeft> provider1, IncrementalValueProvider<TRight> provider2);
IncrementalValuesProvider<(TLeft Left, TRight Right)> Combine<TLeft, TRight>(
    this IncrementalValuesProvider<TLeft> provider1, IncrementalValueProvider<TRight> provider2);
```

Note that the return value matches the first parameter type - whether it is `..Value..` or `..Values..`. The generic type of the resulting provider is a tuple.

Combining two `..Value..` providers results in a `..Value..` provider with a tuple of the left and right values. You would use this if you had a `..Value..` provider that contained collection of data models extracted from syntax, and a second `..Value..` provider that contained a data model with details extracted from the compilation.

Combining a `..Value..` provider and a `..Values..` provider results in a `..Value..` provider with a tuple of the left value and an `ImmutableArray` of the right values. You would use this if you had a `..Value..` provider  that contained a data model with details extracted from the compilation, and a `..Values ..` provider that contained data models extracted from individual syntax and you wanted a single result. For example, you would use this if the left model held information needed to create a class and the right model contained information for creating properties.

Combining a `..Values..` provider and a `..Value..` provider results in a `..Values..` provider where each member is a tuple of one of the left values and the right value. You would use this if you had a `..Values ..` provider that contained data models extracted from individual syntax and a `..Value..` provider that contained a data model with details extracted from the compilation, and you wanted a multiple individual results. For example, you would use this if the left model held information like a namespace that was needed to generate a set of classes and the right model contained information for each, and you wanted each class to be in a separate syntax tree or file.

`Combine` does not provide a `..Values` to  `Values` overload by design because it would result in a cross product. If you need to join two collections, you can use `Collect` to create two `..Value..` providers, use `Combine` to create a single `..Value..` provider, and then use `Select` or `SelectMany` to manipulate the collections.

The [Roslyn incremental source generator specification](https://github.com/dotnet/roslyn/blob/main/docs/features/incremental-generators.md) has diagrams illustrating the different `Combine` overloads.

### Testing the data model










The first step of your generator will be to extract data into your model. If this step does not result in the correct model your generated code will be incorrect, so it is important to test the model in isolation for smoother development and to catch regressions later. Th first step uses extract it from 

### Providers on `IncrementalGeneratorInitializationContext`

If you are building your model using one of the providers on `IncrementalGeneratorInitializationContext`. You can find out more about the [available providers](pipeline.md#pipeline-operations). You will simply extract the data using `Select`:

```c#
// get the contents of files that end with .txt
IncrementalValuesProvider<(string fileName, string content)> textFiles = 
    context.AdditionalTextsProvider
        .Where(static f => f.Path.EndsWith(".txt"))
        .Select((additionalText, cancellationToken) => 
            (fileName: Path.GetFileNameWithoutExtension(additionalText.Path), 
             content: additionalText.GetText(cancellationToken)!.ToString()));

```

The cancellation token is important because you cannot control the size of the files, even if you anticipate them being small. Note that in the code above `Where` takes an `IncrementalValuesProvider<AdditionalText>` and returns `IncrementalValuesProvider<AdditionalText>` that contains only the files you are interested in. `Select` takes an `IncrementalValuesProvider<AdditionalText>` and uses methods and properties to create a data model - in this case a tuple of the file name and the file content as `IncrementalValuesProvider<(string fileName, string content)>`.

### `SyntaxValueProvider`

If you're retrieving data from the user's source code, you'll take a slightly different that uses the `IncrementalGeneratorInitializationContext.SyntaxValueProvider`. Its `CreateSyntaxProvider` method creates an `IncrementalValuesProvider<T>` specific to the syntax you are interested in using two delegates. If this delegate is a separate method, you can call it by omitting the parentheses. This will allow you to test that you are retrieving only the syntax you expect. Retrieving excess syntax nodes can hurt the performance of your generator:

```csharp
var commandModelValues = initContext.SyntaxProvider
    .CreateSyntaxProvider(
        predicate: ModelBuilder.IsSyntaxInteresting,
        transform: ModelBuilder.GetModel)
    .Where(static m => m is not null)!;
```

The first of these delegates filtered syntax. This delegate is called a massive number of times so should be extremely fast:

```csharp
public static bool IsSyntaxInteresting(SyntaxNode syntaxNode, CancellationToken cancellationToken)
{
    // filter syntax here
}
```

You can test the method used as the predicate in isolation by passing random syntax nodes - some of which should pass and some fail.

The second delegate is used to build the data model. This method is passed a `GeneratorSyntaxContext` and a `CancellationToken`. The `GeneratorSyntaxContext` provides the `SyntaxNode` and the `SemanticModel` of the compilation. A second overload lets you test without a `GeneratorSyntaxContext`:

```csharp
public static CommandModel? GetModel(GeneratorSyntaxContext generatorContext,
                                    CancellationToken cancellationToken)
    => GetModel(generatorContext.Node,
                generatorContext.SemanticModel, 
                cancellationToken);

public static CommandModel? GetModel(SyntaxNode syntaxNode,
                                     SemanticModel semanticModel,
                                     CancellationToken cancellationToken)
{
    // build model here syntax here
}
```

You can test `GetModel` by creating a set of syntax nodes to test by parsing test source code into a `SyntaxTree` and selecting the `DescendantNodes` that of type `SyntaxNode`. You can check that the number of matches is what you expect, and that they are the correct syntax nodes.

Find out more about [unit testing supporting incremental generator methods in isolation in the testing article](testing.md#unit-tests).

## Create the code that outputs the generated code and test it

The methods used for [outputting generated code are described in the incremental generator pipeline article](pipeline.md#output) and an example can be found in the [tutorial](tutorial.md#creating-output).

The method that creates the string to output is a static method that takes an instance of your model and a `CancellationToken` and it returns a string. 

To create output pattern, return to your example project. Copy the contents of each file your want to generate into a verbatim interpolated string, or starting in C# 11 a verbatim raw string literal. If you are using a recent version of Visual Studio, copying into the verbatim literal string will double the double quotes and curly brackets. If it does not, you will need to double them:

```csharp
var x = $@"
using System;

#nullable enable

namespace TestExample
{{
    public partial class AddLine
    {{
        Option<System.IO.FileInfo?> fileOption = 
            new Option<System.IO.FileInfo?>(""--file"", 
                ""The file to read and display on the console."");
        Option<string> lineOption = 
            new Option<string>(""--line"", 
                ""Delay between lines."");

// rest of class skipped for readability
";
```

Include the using statements you require, as you cannot depend on global implicit usings being available in the user's project. Also, explicitly include `#nullable enable` if you support nullable reference types.

Checking your notes, replace the portions of the file that you anticipate getting from your data model. Here the items that depend on the model are the name of the class and the fields for the options. Simply replacing these items creates a generation pattern that remains easy to read:

```csharp
var x = $@"
using System;

#nullable enable

namespace TestExample
{{
    public partial class {modelData.CommandName}
    {{
        {OptionFields(modelData.Options)}
// rest of class skipped for readability
";
```

The interpolated string calls the `OptionFields` method to generate fields. This shows how you can handle loops in the generation pattern. It also shows how you can use methods to provide looping without making it difficult to read. The method for :

```csharp
static string OptionFields(IEnumerable<OptionModel> options)
    => string.Join("\n        ", options.Select(
        o => $"Option<{o.Type}> {o.Name.AsField()}Option =" + 
        $"\n            new Option<{o.Type}>({OptionAlias(o)}," + 
        $"\n                {o.Description.InQuotes()});"));
```

You can test your output methods by creating an instance of your model within the test, and passing it an a cancellation token to the method. You can compare the resulting output with the output expected - copied from your example project. 

You can do this comparison with the asserts of your unit test framework, but a verification framework like Verify or ApprovalTests offer a difference comparison with the expected output and let you find and resolve problems much more quickly.

If you have plan to output code via `RegisterPostInitializationOutput`, copy it to a string that is available to the generator. Because this code is an explicit string, testing is not needed.

## Create the generator and test end to end.

## Test performance.

## Create a NuGet package.







