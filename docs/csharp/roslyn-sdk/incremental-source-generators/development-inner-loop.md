# Developing incremental source generators

The process of writing source generators is complicated by the fact you are writing code to create code, which increases the number of abstractions to consider and the ways that you need to test. Roslyn generators further complicate the picture because they are deployed as NuGet packages, and dependencies on NuGet packages that you are also actively developing is challenging. And finally, you need to ensure your generator is fast.

Breaking building a generator into discreet steps lets you focus on each step: 

[[make links when the names settle]]
* Create at least one sample project.
* Design a data model and check it for completeness.
* Create the code that builds the data model and test it in isolation.
* Create the code that outputs the generated code and test it.
* Create the generator and test end to end.
* Test performance.
* Create a NuGet package.

You can use these steps to build a solution that has a good development inner loop - meaning you can easily make changes, see the impact of those changes, and check for regressions. The [Roslyn incremental generator tutorial](tutorial.md) walks through each of these steps. This article covers the things important to a good development inner loop: an appropriate solution layout, an example project, a technique for testing your generator, a strategy for managing NuGet packages, and mechanisms for testing.

Before you get started, check the [limitations of Roslyn incremental source generators](overview.md#limitations-of-generators) in the Overview.

## Structure of the solution and packages

The recommended structure of the solution is at least 4 or 5 projects:

* The *example project*.
* The project containing the *runtime library* - optional but may include attributes and base classes.
* The *incremental generator*.
* The *unit test project*.
* An *integration test project*.

The dependencies between these projects are:

| Project               | Dependencies         |
|-----------------------|----------------------|
| Example               | Runtime              |
| Runtime library       |                      |
| Incremental generator | Runtime (project ref in development, package ref in release)|
| Unit test             | Generator            |
| Integration test      | Unit test, generator |

All of these dependencies are normal project references, except the dependency of the incremental generator on the runtime library.

The runtime library is separate from the generator because it includes code that is needed at both compile time and runtime. The generator depends on the runtime library and this will be a package reference when deployed. However, this package reference is challenging to manage during development. You can treat it as a project reference during development and a package reference for deployment with a conditional MSBuild property:

```xml
<ItemGroup Condition="'$(CreatePackage)' == 'true'">
    <PackageReference Include="IncrementalGeneratorSamples.Runtime" Version="0.0.1-alpha" />
</ItemGroup>

<ItemGroup Condition="'$(CreatePackage)' != 'true'">
    <ProjectReference Include="../IncrementalGeneratorSamples.Runtime/IncrementalGeneratorSamples.Runtime.csproj" />
</ItemGroup>
```

## Create at least one example project

The important first step of writing a source generator is to create a working example project that shows exactly what you want to generate in the context of a working project. This is just a normal .NET project. You might create it via the .NET templates in the .NET CLI or an editor like Visual Studio, or you might use a sample project that you've already created.

Within this example project isolate the code you plan to generate into separate files in a subdirectory. You will use this sample to plan your generator and as a starting point for actual generation. The integration tests recommended here will write generated code into this subdirectory, so you might want a name like "OverwrittenInTests". This example project can also be used in the documentation of how to use your generator.

As you create this example project, you may need to use [partial classes and methods](https://docs.microsoft.com/dotnet/csharp/programming-guide/classes-and-structs/partial-classes-and-methods) to isolate the code you want to generate from code that is logically part of the same class. Roslyn source generators add new syntax trees, effectively new files, to the compilation. Partial classes allow you to merge these knew files with existing source.

Ensuring this code runs and does what is expected before you begin will save time as you create your generator.

## Design a data model and check it for completeness.

Your generator will gather input data from sources like the user's code or external files. Regardless of the source, define an explicit data model even if it is only one value.

Incremental generators rely on caching, and caching relies on data models. You might only use one model, and it might contain only one value, or you may have several models that you transform during development. The key is that you extract the data you need from the underlying source in a discreet operation as early in the pipeline process as possible. This discrete operation will be created either by calling `SyntaxValueProvider.CreateSyntaxProvider` or `Select` on any of the other providers of `IncrementalGeneratorInitializationContext`. The [provider section or the incremental source generator pipeline article](pipeline.md#providers) and the [Incremental generator specification](https://github.com/dotnet/roslyn/blob/main/docs/features/incremental-generators.md) have more information on providers.

While a very simple model might be just a value or a tuple, most generators will use one or more data models that each contains several values. Each model must have value equality, including using `SequenceEquals` for any collections. This is most easily done using records, record structs and records and customizing the equality if need to accommodate collections and any other data that does not naturally have value equality.

Once you build your model, and know the code you plan to generate along with the code the user will write from  the sample project, map the values to check the model for completeness. This will generally be a list of variations in the code your plan to generate, and how each piece of data will be created. Consider a tool to mark variable portions of the code you plan to generate - such as copying the code into Word and highlighting sections.

## Create the code that builds the data model and test it

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







