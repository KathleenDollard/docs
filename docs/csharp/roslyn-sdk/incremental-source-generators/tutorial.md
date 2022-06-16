---
title: How to build a Roslyn incremental source generator
description: Explore the steps to creating a Roslyn incremental source generator.
author: KathleenDollard
ms.author: kdollard
ms.date: 6/11/2022 
ms.topic: tutorial
---
# Tutorial: Building an incremental generator

There are 3 execution steps in incremental generators;

* Data extraction into a model.
* Transforming models (when needed).
* Outputting source code.

This tutorial uses the `SyntaxValueProvider` to create an `IncrementalValueProvider<GenerationModel>` which is used to output source code without needing further transformations. 

The signature of the `CreateSyntaxProvider` method is:

```csharp
public IncrementalValuesProvider<T> CreateSyntaxProvider<T>(
    Func<SyntaxNode, CancellationToken, bool> predicate, 
    Func<GeneratorSyntaxContext, CancellationToken, T> transform);
```

You'll see how to use the predicate to limit the syntax nodes under consideration, and to use the transform to further filter the syntax nodes and transform available data into a model that can be used to output source code.

## The steps of a generator
 
The steps to build an incremental generator are;

[[make links when the names settle]]
* Create at least one sample project.
* Design a data model.
* Check the data model for completeness.
* Create and test selecting syntax nodes in isolation.
* Create and test transforming into the data model for a single syntax node.
* Create and test code to extract the full data model via a test generator.
* Create and test code to transform the data model if needed.
* Create and test code to create the output via your generator.
* Create an integration test.

If it seems like a lot, its more without.

Check the limitations in the Overview

Pairs with tutorial.

## The structure of the solution

[If you want to follow along, here is the link](). The tutorial solution layout is described in the [setting up an effective inner loop for Roslyn incremental source generators]()

The structure of the solution used in this tutorial has 5 projects:

* The example project
* The project containing the runtime library - specifically the attribute and base class.
* The incremental generator.
* The unit test project for the incremental generator.
* An integration test project.

All of the projects except the runtime library depend on the runtime library. The unit test project and the integration test project depend on the generator. The integration test project depends on the unit test project for some helper methods. All of these dependencies are project dependencies during development to facilitate the inner loop.

When deployed, the incremental generator project needs a package reference to the runtime project. This very awkward during development. You can avoid this by basing package creation conditionally on an MSBuild property;

```dotnetcli

```

The same technique and property can be used to avoid creating a package on every full build during development.

## Create at least one example project

The important first step of writing a source generator is to create a working example project that shows exactly what you want to create. This is just a normal project that accomplishes your goals from the .NET templates via the .NET CLI or an editor like Visual Studio.

Within this example project isolate the generated code into separate files, generally in a subdirectory called something like `ViaGeneration`. You may need to use [partial classes or methods](https://docs.microsoft.com/dotnet/csharp/programming-guide/classes-and-structs/partial-classes-and-methods) to isolate the code you want to generate. Isolation is important because Roslyn source generators add new syntax trees, effectively new files, to the compilation.

Ideally, create unit tests for the important aspects of this example project. [[You can reuse those unit tests during integration testing.]]

Skipping the step of creating a example project, even if it is a simple generator, will make it harder to create a good generator. This example use an approach to wrapping the [System.CommandLine API](https://docs.microsoft.com/dotnet/standard/commandline/). While a more complete approach to System.CommandLine would include subcommands, this simple approach just allows the user to specify the options on they root command using code like the following:

```csharp
using IncrementalGeneratorSamples.Runtime;

namespace TestExample;
[Command]
public partial class Command
{
    /// <summary>
    /// The file to read and display on the console.
    /// </summary>
    public FileInfo? File { get;  }

    /// <summary>
    /// Delay between lines, specified as milliseconds per character in a line.
    /// </summary>
    public int Delay { get;  }

    public int DoWork() 
    {
        // do work, such as displaying the file here
        return 0;
    }
}
```

An adjacent library, IncrementalGeneratorSamples.Runtime,includes the CommandAttribute and base classes. The user does not even need a using statement for System.CommandLine, although the reference is needed and will be included as a dependency of the generator package. 

The generator will create code to manage System.CommandLine:

```csharp
using System.CommandLine;
using System.CommandLine.Invocation;

#nullable enable

namespace TestExample;
public partial class Command
{
    public Command(FileInfo? file, int delay)
    {
        File = file;
        Delay = delay;
    }

    public static void Invoke(string[] args)
        => CommandHandler.Invoke(args);

    internal class CommandHandler : IncrementalGeneratorSamples.Runtime.CommandHandler<CommandHandler>
    {
        private Option<FileInfo?> fileOption;
        private Option<int> delayOption;

        public CommandHandler()
        {
            fileOption = new Option<FileInfo?>("--file", "The file to read and display on the console.");
            RootCommand.AddOption(fileOption);
            delayOption = new Option<int>("--delay", "Delay between lines, specified as milliseconds per character in a line.");
            RootCommand.AddOption(delayOption);
        }

        /// <summary>
        /// The handler invoked by System.CommandLine. This will not be public when generated is more sophisticated.
        /// </summary>
        /// <param name="invocationContext">The System.CommandLine Invocation context used to retrieve values.</param>
        public override int Invoke(InvocationContext invocationContext)
        {
            var commandResult = invocationContext.ParseResult.CommandResult;
            var command = new Command(GetValueForSymbol(fileOption, commandResult), GetValueForSymbol(delayOption, commandResult));
            return command.DoWork();
        }

        /// <summary>
        /// The handler invoked by System.CommandLine. This will not be public when generated is more sophisticated.
        /// </summary>
        /// <param name="invocationContext">The System.CommandLine Invocation context used to retrieve values.</param>
        public override Task<int> InvokeAsync(InvocationContext invocationContext)
        {
            // Since this method is not implemented in the user source, we do not implement it here.
            throw new NotImplementedException();
        }
    }

}
```

in the example project, this code is in the subdirectory *GeneratedViaTest*.

Finally, in order to run the example project and ensure it works correctly, there is a entry point which could be a `main` method or top level statements:

```csharp
using TestExample;

Command.Invoke(args)
```

## Design a data model

Your generator will gather input data from sources like the user's code or external files. Regardless of the source, define an explicit data model even if it contains only one value.

To support caching, incremental generator data models must have value equality.

The data model should contain the minimal information required for generation and will generally be scoped to internal.

In general, your data model should reflect what you are generating and use naming consistent with the logical entity you are modeling. As an example for generating a command line CLI with commands and options that will be created from :

[[ From tutorial. Not in proper form until initial review.]]

```csharp
namespace IncrementalGeneratorSamples.Generator.Models;

internal record GenerationModel
{

    internal GenerationModel(string commandName, IEnumerable<OptionModel> options)
    {
        CommandName = commandName;
        Options = options;
    }

    internal string CommandName { get; }


    internal IEnumerable<OptionModel> Options { get; }

    public virtual bool Equals(GenerationModel model)
        => model is not null && 
            model.CommandName == CommandName &&
            model.Options.SequenceEqual(this.Options);

    public override int GetHashCode()
    {
        var hash = CommandName.GetHashCode();
        foreach(var prop in Options)
        {
            hash ^= prop.GetHashCode();
        }
        return hash;
    }
}

internal record OptionModel
{
    internal OptionModel(string name, string type, string description)
    {
        Name = name;
        Type = type;
        Description = description;
    }

    internal string Name { get; }
    internal string Type { get; }
    internal string Description { get; }
}  
```

## Check the data model for completeness

Review the data model in relation to the example project. Map each variable element in the generated code to the input to check that the data in the model can be extracted:

* The class for the root element will be marked with the `IncrementalGeneratorSamples.Runtime.CommandAttribute`.
* The name and type of each property will be the name and type of the option.
* Casing is adjusted on output as CLI options are almost always lower case.
* The description will be retrieved from the XML comments.

## Filter syntax nodes

The first step in an incremental generator that uses source code as input is to gather the syntax nodes. This is done with via the predicate passed to the `CreateSyntaxProvider` method. This predicate is passed every node in the `SyntaxTree`and returning true if the node should be further considered by the generation pipeline. Doing this in a separate method makes it easy to test in isolation.

When this method is called from the generator, a syntax node and a cancellation token are passed. If possible, the work done in this method is very fast, and cancellation can be handled by the generator. The implementation here is very fast because it relies only on pattern matching against the type, and for classes checking for the `Command` attribute:

```csharp
        public static bool IsSyntaxInteresting(SyntaxNode syntaxNode, CancellationToken _)
            // REVIEW: What's the best way to check the qualified name? 
            // REVIEW: This should be very fast. Is it ok to ignore the cancelation token in that case?
            // REVIEW: Will this catch all the ways people can use attributes
            => syntaxNode is ClassDeclarationSyntax cls &&
                cls.AttributeLists.Any(x => x.Attributes.Any(a =>
                     a.Name.ToString() == "Command" || 
                     a.Name.ToString() == "CommandAttribute"));

```

## Test filtering syntax nodes

Testing this code requires creating a number of `SyntaxNode` and a `CancellationToken`. Create syntax nodes from source code mimics what will happen when your data runs. You can create data for your tests as strings. This sample source code is similar to the [tutorial for System.CommandLine](https://docs.microsoft.com/dotnet/standard/commandline/get-started-tutorial) :

```csharp
private const string SimpleClass = @"
using IncrementalGeneratorSamples.Runtime;

[Command]
public partial class Command
{
    public int Delay { get;  }
}
";
```

If we assume other tests will catch if the wrong `SyntaxNode` is returned, you can test the count of matching syntax nodes, which can take advantage of XUnit's theory feature:

```csharp
[Theory]
[InlineData(1, SimpleClass)]
[InlineData(1, CompleteClass)]
public void Should_select_attributed_syntax_nodes(int expectedCount, string sourceCode)
{
    var cancellationToken = new CancellationTokenSource().Token;
    var tree = CSharpSyntaxTree.ParseText(sourceCode);
    var matches = tree.GetRoot()
        .DescendantNodes()
        .Where(node => ModelBuilder.IsSyntaxInteresting(node, cancellationToken));
    Assert.Equal(expectedCount, matches.Count());
}
```

These passing tests provide confidence that the initial filtering of syntax nodes in the generator will be successful. 

## Create the data model

The next step in the incremental generator is to create an initial data model for each `SyntaxNode` selected by the predicate. This is done in the transform delegate that is passed to the `CreateSyntaxProvider` method. The data models are returned as an `IncrementalValuesProvider<GenerationModel>` (note the *s* on values indicating a collection).

Sometimes additional transformations, such as combining the initial data model with others, or summarizing the data in these models. Additional transformations will take and also return an `IncrementalValueSourceProvider` or an `IncrementalValuesProvider`. Additional transformations are not needed for this tutorials, and you can find out more about them in the [Incremental Generators specification](https://github.com/dotnet/roslyn/blob/main/docs/features/incremental-generators.md#syntaxvalueprovider).

The signature for the transform delegate includes a `GeneratorSyntaxContext` object that cannot be created in a test. Instead, a trivial method can deconstruct the `GeneratorSyntaxContext` into a testable method:

```csharp
public static GenerationModel? GetModel(GeneratorSyntaxContext generatorContext, 
                                        CancellationToken cancellationToken)
    => GetModel(generatorContext.Node, generatorContext.SemanticModel, cancellationToken);

public static GenerationModel? GetModel(SyntaxNode syntaxNode, 
                                        SemanticModel semanticModel, 
                                        CancellationToken cancellationToken)
{
    // actual work here
}
```

The first part of creating the model is find the associated symbol from the semantic model. Note that a cancellation token is passed. The symbol is then cast to `ITypeModel`. This cast should always succeed because the predicate in the previous step of the pipeline filtered to `ClassDeclarationSyntax`. But if it does not, return null:

```csharp
var symbol = semanticModel.GetDeclaredSymbol(syntaxNode, cancellationToken);
if (symbol is not ITypeSymbol typeSymbol)
{ return null; }
```

The `typeModel` allows access to the members of the type, which can be filtered to just the properties for this transform. Because this collection is not bounded, cancellation of the iteration is supported. This loop creates an `OptionModel` for each property (the `GetPropertyDescription` is covered later in this section):

```csharp
var properties = typeSymbol.GetMembers().OfType<IPropertySymbol>();
var options = new List<OptionModel>();
foreach (var property in properties)
{
    // since we do not know how big this list is, so we will check cancellation token
    // REVIEW: Should this return null or throw? 
    if (cancellationToken.IsCancellationRequested)
    { return null; }
    var description = GetPropertyDescription(property);
    options.Add(new OptionModel(property.Name, property.Type.ToString(), description));
}
```

The `typeSymbol.Name` and the `OptionModel` collection are used to create the new `GenerationModel`:

```csharp
return new GenerationModel(typeSymbol.Name, options);
```

One of the many gems in the `SemanticModel` is access to XML documentation. In particular, the description of types, parameters, properties and other members which this sample later uses as the description of the options. Here, this is done in a local static method:

```csharp
static string GetPropertyDescription(IPropertySymbol prop)
{
    var doc = prop.GetDocumentationCommentXml();
    if (string.IsNullOrEmpty(doc))
    { return ""; }
    var xDoc = XDocument.Parse(doc);
    var desc = xDoc.DescendantNodes()
        .OfType<XElement>()
        .FirstOrDefault(x => x.Name == "summary")
        ?.Value;
    return desc is null
        ? ""
        : desc.Replace("\n","").Replace("\r", "").Trim();
}
```

The full `GetModel` method:

```csharp
{
    var symbol = semanticModel.GetDeclaredSymbol(syntaxNode, cancellationToken);
    if (symbol is not ITypeSymbol typeSymbol)
    { return null; }

    var properties = typeSymbol.GetMembers().OfType<IPropertySymbol>();
    var options = new List<OptionModel>();
    foreach (var property in properties)
    {
        // since we do not know how big this list is, so we will check cancellation token
        // REVIEW: Should this return null or throw?
        if (cancellationToken.IsCancellationRequested)
        { return null; }
        var description = GetPropertyDescription(property);
        options.Add(new OptionModel(property.Name, property.Type.ToString(), description));
    }
    return new GenerationModel(typeSymbol.Name, options);

    static string GetPropertyDescription(IPropertySymbol prop)
    {
        // REVIEW: Not crazy about the repeated Parsing of small things.
        var doc = prop.GetDocumentationCommentXml();
        if (string.IsNullOrEmpty(doc))
        { return ""; }
        var xDoc = XDocument.Parse(doc);
        var desc = xDoc.DescendantNodes()
            .OfType<XElement>()
            .FirstOrDefault(x => x.Name == "summary")
            ?.Value;
        return desc is null
            ? ""
            : desc.Replace("\n","").Replace("\r", "").Trim();
    }
}
```

## Test data model creation

To allow reuse, the code to create a data model is in a separate method. This method can be called by tests which provide the source code and assert the results. This method is:

```csharp
private GenerationModel? GetModelForTesting(string sourceCode)
{
    var cancellationToken = new CancellationTokenSource().Token;
    var (compilation, diagnostics) = TestHelpers.GetInputCompilation<Generator>(
            OutputKind.DynamicallyLinkedLibrary, sourceCode);
    Assert.Empty(diagnostics);
    var tree = compilation.SyntaxTrees.Single();
    var matches = tree.GetRoot()
        .DescendantNodes()
        .Where(node => ModelBuilder.IsSyntaxInteresting(node, cancellationToken));
    Assert.Single(matches);
    var syntaxNode = matches.Single();
    return ModelBuilder.GetModel(syntaxNode, 
                                 compilation.GetSemanticModel(tree), 
                                 cancellationToken);
}
```

A cancellationToken is created for later use. The source code is used to create a compilation via a helper method available in the full project. It is very easy to make mistakes when writing  source code as a string, so checking the syntax here can prevent wasting time over a typo or forgotten using statement. The source code used here is the same as used in the earlier testing of the syntax node filtering. and the same filtering is done to prepare to create the model.

Once it has the correct syntax node, it passes it to the same `GetModel` method that will be used by the generator.

Because the assertions are unique to each of the tests, individual tests rather than Theory tests are used. One of these tests is:

```csharp
[Fact]
public void Should_build_model_from_SimpleClass()
{
    var model = GetModelForTesting(SimpleClass); 
    Assert.NotNull(model);
    if (model is null) return; // to appease NRT
    Assert.Equal("Command", model.CommandName);
    Assert.Single(model.Options);
    Assert.Equal("Delay", model.Options.First().Name);
    Assert.Equal("int", model.Options.First().Type);
}
```

After retrieving the model, assertions ensure that the `GenerationModel` was built correctly.

## Creating output



## Test code output



## Writing tests for data extraction

[There are two approaches to testing Roslyn generators](https://github.com/dotnet/roslyn/blob/main/docs/features/source-generators.cookbook.md#unit-testing-of-generators), whether they are V1 generators or incremental generators. This tutorial uses the conceptually simpler approach to focus on incremental generators themselves. These tests will use explicitly run generation, and the data extraction tests will output serialized data as comments in C# syntax by using a separate testing generator. This also means your first incremental will produce very simple code, making it easier to follow the process. The generated content is compared to previously approved content using [Verify](https://github.com/VerifyTests/Verify); alternatively you could use a similar library like [ApprovalTests](https://github.com/approvals/ApprovalTests.Net).

The generator for this test will be a new class in the generator project and will consist of extraction and outputting the serialized model. 




