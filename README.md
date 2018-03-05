# msbuild basics

Demo of basic usage of msbuild and the csproj format.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

**Table of Contents**
* [What is MSBuild?](#what-is-msbuild)
* [Preparation](#preparation)
* [Basics](#basics)
* [Getting Started](#getting-started)
* [Resources](#resources)
* [License](#license)

## What is MSBuild?

MSBuild is the [Microsoft Build Engine](https://docs.microsoft.com/en-us/visualstudio/msbuild/msbuild) and can be used to build applications. It uses project files written in XML to control the build process. MSBuild is used by Visual Studio, but doesn't depend on it.


## Preparation

Before you get started, make sure that `msbuild` is installed and added to your PATH. MSBuild should be at least in version 15.

```console
$ msbuild /version
Microsoft (R)-Buildmodul, Version 15.5.180.51428 für .NET Framework
Copyright (C) Microsoft Corporation. Alle Rechte vorbehalten.

15.5.180.51428
```

To get a feel for the `msbuild` command, you can can consult the `/help` option. You will see the general usage and available options.

```console
$ msbuild /help
Microsoft (R)-Buildmodul, Version 15.5.180.51428 für .NET Framework
Copyright (C) Microsoft Corporation. Alle Rechte vorbehalten.

Syntax:              MSBuild.exe [Optionen] [Projektdatei | Verzeichnis]
```

## Basics

The XML schema for MSBuild is basically a small programming language which is used to define various build steps.

 - **Properties** are similar to variables. The are key/value pairs and can be reused throughout the project file.
 - **Items** are inputs into the build system and typically represent files.
 - **Tasks** are chunks of executable code (written in managed code). So they act somewhat like functions.
 - **Targets** can be compared to function as well. The group multiple tasks together and represent potential entry points.

## Getting Started

### Hello World

Let's start with a simple Hello World. 

```xml
<!-- demo.csproj -->
<Project>
    <Target Name="TargetOne">
        <Message Text="Hello World" />
    </Target>
</Project>
```

We define a simple target `TargetOne` with only one `Message` task. The `Message` tasks simply outputs a log message to the console.

```console
$ MSBuild demo.csproj /t:TargetOne
Microsoft (R)-Buildmodul, Version 15.5.180.51428 für .NET Framework
Copyright (C) Microsoft Corporation. Alle Rechte vorbehalten.

Der Buildvorgang wurde am 01.03.2018 10:19:05 gestartet.
Projekt "C:\src\misc\csproj-demo\demo.csproj" auf Knoten "1", TargetOne Ziel(e).
TargetOne:
  Hello World
Die Erstellung von Projekt "C:\src\misc\csproj-demo\demo.csproj" ist abgeschlossen (TargetOne Ziel(e)).


Der Buildvorgang wurde erfolgreich ausgeführt.
    0 Warnung(en)
    0 Fehler

Verstrichene Zeit 00:00:00.06
```

### Properties / Variables

Next up, we want to store some values and reuse them throughout our project file. This can be achieved with properties.

```xml
<!-- demo.csproj -->
<Project>
    <PropertyGroup>
        <VariableOne>This is a variable</VariableOne>
    </PropertyGroup>

    <Target Name="TargetTwo">
        <Message Text="$(VariableOne)" />
    </Target>
</Project>
```

First we define a new property with the name "VariableOne" and assign it the value "This is a variable". Then we use it for the Message task with `$(VariableOne)` instead of hardcoding the message.

```console
$ MSBuild demo.csproj /t:TargetTwo
TargetTwo:
  This is a variable
```

### Parameters

Next up, we want to modify the behaviour of the project, by defining parameters (think about build configuration Debug/Release for example).

```xml
<!-- demo.csproj -->
<Project>
    <Target Name="TargetThree">
        <Message Text="$(ParameterOne)" />
    </Target>
</Project>
```

Let's simply output the value of the given parameter. The parameter itself can be specified via an option during the command line call.

```console
$ MSBuild demo.csproj /t:TargetThree /p:ParameterOne="Hello from params"
TargetThree:
  Hello from params
```

### Conditions / IF

Now we want to modify the behaviour of the project, depending on the parameter. MSBuild doesn't support `if`-clauses like other languages. Instead, all elements accept a `Condition` attribute. If the condition evaluates to false, the whole element will be skipped.

```xml
<!-- demo.csproj -->
<Project>
    <Target Name="TargetCondition">
        <Message
            Condition="'$(ParameterOne)'!=''"
            Text="Param is defined" />
        <Message
            Condition="'$(ParameterOne)'==''"
            Text="Param is NOT defined" />
    </Target>
</Project>
```

If we build the project file with and without the parameter, we get the correct message as expected.

```console
$ MSBuild demo.csproj /t:TargetCondition
TargetCondition:
  Param is NOT defined

$ MSBuild demo.csproj /t:TargetCondition /p:ParameterOne="12312"
TargetCondition:
  Param is defined
```

### Dependencies

Since a build can consist of multiple dependent targets, it would be great if we can define some kind of dependencies between targets. And in fact this is possible with the attributes `BeforeTargets/AfterTargets` on the `<Target>` element.

```xml
<!-- demo.csproj -->
<Project>
    <Target Name="TargetFour">
        <Message Text="TargetFour" />
    </Target>

    <Target Name="TargetBeforeFour" BeforeTargets="TargetFour">
        <Message Text="Before TargetFour" />
    </Target>
</Project>
```

Now msbuild takes care, that all `Target`s which should be run before/after a specific `Target` are called in the correct order.

``` console
$ MSBuild demo.csproj /t:TargetFour
TargetBeforeFour:
  Before TargetFour
TargetFour:
  TargetFour
```

### Reuse Propertes/Targets

Now csproj files might get quite big and some parts might be reused in other projects. Luckily you can move properties or targets into external files and import them.

External files can be imported with `<Import Project="./external.targets" />`. Everything defined in the external file is then available in the original csproj file.

Common names for external files are
 - Properties: ".props"
 - Targets: ".targets"


```xml
<!-- demo.csproj -->
<Project>
    <Import Project="./external.targets" />
</Project>

<!-- external.targets -->
<Project>
    <Import Project="./external.props" />
    <Target Name="ExternalTarget">
        <Message Text="$(ExternalVariable) Message" />
    </Target>
</Project>

<!-- external.props -->
<Project>
    <PropertyGroup>
        <ExternalVariable>External</ExternalVariable>
    </PropertyGroup>
</Project>
```

Calling the `Target` "ExternalTarget will work, as if it was specified inside `demo.csproj`.

``` console
$ MSBuild demo.csproj /t:ExternalTarget
ExternalTarget:
  External Message
```

### Predefined Tasks

There are a lot of [predefined tasks](https://docs.microsoft.com/en-us/visualstudio/msbuild/msbuild-task-reference) which are included with msbuild and are ready to use. Let's see the [Copy Task](
https://docs.microsoft.com/en-us/visualstudio/msbuild/copy-task) as an example.

The `Copy Task` requires at least the `SourceFiles` parameter and either `DestinationFolder` or `DestinationFiles`.

```xml
<!-- demo.csproj -->
<Project>
    <ItemGroup>
        <MySourceFiles Include="external.props;external.targets"/>
    </ItemGroup>
    <Target Name="CopyFiles">
        <Copy
            SourceFiles="@(MySourceFiles)"
            DestinationFolder="./dist" />
    </Target>
</Project>
```

Now all specified files will be copied to the specified `DestinationFolder`.

``` console
$ msbuild demo.csproj /t:CopyFiles
CopyFiles:
  Die Datei wird von "external.props" in "./dist\external.props" kopiert.
  Die Datei wird von "external.targets" in "./dist\external.targets" kopiert.
```

## Resources

* [MSBuild | Visual Studio Docs](https://docs.microsoft.com/en-us/visualstudio/msbuild/msbuild)
* [MSBuild Task Reference | Visual Studio Docs](https://docs.microsoft.com/en-us/visualstudio/msbuild/msbuild-task-reference)

## License

MIT License (see [LICENSE.md](LICENSE.md))