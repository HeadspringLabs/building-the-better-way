---
layout: default
title: Phases
parent: Build Scripts
grand_parent: Development Guidelines
nav_order: 2
---

# Phases of a Build Script
{:.no_toc}
A build script typically has the same set of phases when building software.

- TOC
{:toc}

## Run Every Time

Some tasks in a build script run every time the build script is run. These are primarily used to provide information about the environment for later troubleshooting.

### **DO** output important tooling and environmental information each time the script runs
{: .text-green-100 }
Build scripts are typically run in logged environments and it is important to output the details of that environment to provide context and aid in troubleshooting. For .NET Core projects, at a minimum provide the output of:

```powershell
dotnet --info
```

This will show which SDK and runtime versions of .NET Core are available and which one is being used at the moment considering any `global.json` files or limitations on what is installed in the build agent or developer workstation.

If you are using Docker, output the Docker version as well:

```powershell
docker --version
```

If you are using .NET Core tools, output those versions as well:

```powershell
dotnet tool list --global
dotnet tool list
```

If there are issues with the build, these commands will help you better reproduce the issue and identify potential problems.

## Versioning of Assemblies

### **DO** apply a unique version number to each build produced by a CI system
{: .text-green-100 }
All assemblies produced by a build server must be labeled with a unique version to allow you to track those binaries as they are deployed in different environments back to their original source code. When there is a problem in a particular version, you can pull the exact source code revision and, hopefully, reproduce the issue.

### **DO** apply the version number using command line properties to the build script
{: .text-green-100 }
The build script has a “version” property that will be used to apply a version number to published outputs. Since the way that the version is created can vary from project to project, you need a simple way to provide the desired version to the build script. Use command line property overrides to apply the build version ([SemVer](https://semver.org/)) to the build outputs.

```powershell
psake CI -properties ${'version':'1.2.3-dev.5'}
```

In this example, you can use your CI system variable substitution to apply the correct version number to the command line. Make sure it uses single-quotes on the command line, otherwise it may not work as expected.

### **CONSIDER** using a semantic versioning (SemVer) model of assigning version numbers
{: .text-yellow-300 }
Semantic versioning (shortened to [SemVer](https://semver.org/)) is hugely important when sharing your code with other internal or external users downstream. When there are no downstream consumers of your code as a library, you have considerably more latitude in choosing a versioning scheme.

### **DO** use MinVer for versioning code libraries or projects that require a predictable version number
{: .text-green-100 }
[MinVer](https://github.com/adamralph/minver#readme) is a simple tool that generates a SemVer-compliant version number at build-time based on Git history. When building a particular Git revision with MinVer installed, it will always generate the same version number. Unlike other automated versioning tools, MinVer is based exclusively on Git tags and commit "height" between tags, granting a great deal of control and consistency over the version, regardless of branching strategy. This makes it the ideal tool for situations where the difference in code versions is meaningful to consumers, such as libraries or APIs.

MinVer is a NuGet package. Whether you are starting a new project or introducing it later, you should install the [package](https://www.nuget.org/packages/MinVer) for all projects that need to be versioned:

```powershell
dotnet add package MinVer
```

At compile time, MinVer will examine the latest Git tag and set the MSBuild version properties (_not_ `AssemblyInfo.cs`). You can also use the generated numbers for [other purposes](https://github.com/adamralph/minver#can-i-use-the-version-calculated-by-minver-for-other-purposes) or [customize MinVer](https://github.com/adamralph/minver#options) to fit specific numbering and release schemes, if needed.

### **CONSIDER** using GitVersion for versioning top-level applications or projects where the version number is informational or decorative
{: .text-yellow-300 }
[GitVersion](https://gitversion.readthedocs.io/en/latest/) is a more robust alternative to MinVer that takes the repository's branching strategy into consideration when generating version numbers. This comes with additional configuration complexity, but ultimately allows a full version number to be applied with minimal-to-no developer intervention. This can be helpful in situations where versions are not attached to 'shared' code e.g. good for matching standalone app releases with log properties, but bad for for code libraries or APIs (use MinVer instead).

GitVersion is a separate executable that should be installed as a dotnet local tool as part of your `setup.ps1`. Local tools do not require any changes to PATH, which make them ideal for both local development and CI scripts. However, in most scenarios GitVersion can be excluded from your local / development build process. If you are starting a new project, you may not have a tools manifest yet, you can create one easily:

```powershell
dotnet new tool-manifest
dotnet tool install gitVersion.tool
```

GitVersion will generate an environment variable called `GITVERSION_SEMVER` that you should use for setting the assembly versions. Note this is not the default `GITVERSION_FULLSEMVER` which is not compatible in all contexts, such as Docker container tags. The `GITVERSION_SEMVER` string is usable everywhere.

Read the documentation on how to configure a `GitVersion.yml` file and the different modes. Alternatively, there is a wizard that walks you through a few questions and suggests the mode you should use. Version numbers are incremented whenever merges are done back to master, and you can force a major version increase by adding a tag to your commit message.

```powershell
# start the GitVersion project wizard
dotnet gitversion init
```

### **DO** rely on your CI system to produce an environment variable that will be used as your build version and pass it to your build step command line when not using MinVer
{: .text-green-100 }
Whichever method you end up using the overall goal is to generate a unique version and apply it to all or some of the projects in a solution. The .NET Core tooling uses MSBuild properties to set assembly versions instead of `AssemblyInfo.cs` files and attributes. You can pass these parameter to the dotnet build tool using the `-p` command line switch. Here's an example:

```powershell
dotnet build --configuration $configuration --nologo -p:"Product=$($product)" -p:"Copyright=$(get-copyright)" -p:"Version=$($version)"
```

Pay careful attention to the quoted strings, since these variables could end up with special characters, it is safer to quote the strings to avoid unexpected build failures later. This is how the build script sample implements the compilation process.

### **CONSIDER** including the Git hash in your version number as a unique differentiator during development
{: .text-yellow-300 }

When using manual version numbers or versioning tools based on Git, any concurrent development (or solo interactive rebasing) will risk versioning collisions before merging to mainline or explicitly re-numbering. This may cause conflicts or confusion when the version number is utilized, such as setting the build number, or on display in a PR deployment.

To strictly and clearly differentiate revisions in these contexts, you should include the unique Git hash for a commit (or `sha`) with the general version number. In versioning lingo, this is "build metadata."

You may or may not want to include build metadata in release versions. You should use leverage your CI tool and application code to include or exclude the build metadata with the version number based on the branch or context it is being used in.

If you're using MinVer, you can add a custom [build metadata](https://github.com/adamralph/minver#can-i-include-build-metadata-in-the-version) suffix by setting the `MinVerBuildMetadata` environment variable on your build server. This will explicitly set the `MinVerBuildMetadata` property. Here is an example for making this change in Azure DevOps:

```yaml
variables:
  MinVerBuildMetadata: Build.SourceVersion
```

If you're using GitVersion, there are [several environment variables](https://gitversion.net/docs/reference/variables) that automatically include the short or full sha, as well as some with development-relevant information such as branch name or PR info. You should either append the `Sha` or `ShortSha` variables to your SemVer number, or use the longer `FullBuildMetaData` or `InformationalVersion` in place of the general version number outright.

If not using MinVer or GitVersion, you can retrieve the sha from git directly and save it to a variable, and then append it to the general version. You should follow the [SemVer spec](https://semver.org/spec/v2.0.0.html#spec-item-10) and denote the metadata suffix with a `+`:

```powershell
$sha = git rev-parse --short HEAD # exclude '--short' for full sha
$buildVersion = "$version+$sha"
```

## Compilation

The compilation process converts source code to binaries. Most projects that are a unit of build will have a single solution file.

### **DO** use `dotnet build` to compile an entire solution directly
{: .text-green-100 }
The simplest (and preferred) approach is to let the tooling take care of building the right things. Use a command similar to the following:

```powershell
task Compile {
    exec { dotnet build --configuration $configuration --nologo -p:"Product=$($product)" -p:"Copyright=$(get-copyright)" -p:"Version=$($version)" }
}
```

This will find a solution file in the folder and compile it based on the target configuration. By default, you should be building a `Release` configuration. `Debug` is only useful for local development.

The `--nologo` switch will instruct MSBuild to stay silent each time it is invoked. It tends to like to announce itself every time you use it.

### **DO** configure your build to use `Release` configuration by default
{: .text-green-100 }
When you build in Visual Studio, it tends to default to a `Debug` build while you are actively working, testing, and debugging. When you are ready to deploy the code, you should build in `Release` mode and run unit tests on the `Release` code to ensure that the compiler optimizations do not interfere with the expected operation of the system. Your CI system should _always_ compile and test the `Release` code to avoid unexpected failures at runtime.

## Automated Testing

Automated testing is a key phase of automated build scripts. It helps you understand how your code behaves away from a developer workstation in a reproducible environment.

### **DO** define a folder for test reports and related files
{: .text-green-100 }
In your build script, create a property for the test reports folder that will hold all the test report XML files and any analysis files that will need to be collected by a CI system or reviewed by a pull request reviewer.

```powershell
properties {
    $projectRoot = "$(resolve-path .)"
    $testResults = "$projectRoot/TestReports"
}
```

### **DO** configure a separate `Test` task that depends on compilation and will fail the build on a test failure
{: .text-green-100 }
If you use `dotnet test` or `dotnet fixie`, both will return an error if there is a unit test failure by default. Wrapping this in an `exec` block in Psake will give you the desired behavior. If unit tests fail, the build should not be consumed downstream by anyone to avoid extra bug reports or possible data loss due to defects in the software.

### **DO** output test results in a way that can be consumed by CI systems
{: .text-green-100 }
Most CI systems (Azure DevOps, TeamCity, Jenkins) can consume test report files from various formats. You’ll need to review your CI system documentation and the test system documentation to output the test reports in the proper format.

If you’re using XUnit, it will output the test report in the “VsTest” or "TRX" format that is consumable from Azure DevOps. You’ll have one report file per test assembly, and you should name them for the assembly name. If you need a different output format, you'll need to look into what other test loggers are available.

```powershell
# find any directory that ends in "Tests" and execute a test
get-childitem . *Tests -directory | foreach-object {
    exec { dotnet test --configuration $configuration --no-build -l "trx;LogFileName=$($_.name).trx" -r $testResults } -workingDirectory $_.fullname
}
```

If you have a choice, [Fixie](https://fixie.github.io/) is recommended because it "does the right thing" in most cases.

```powershell
get-childitem . src/*.Tests -directory | foreach-object {
   exec { dotnet fixie --configuration $configuration --no-build } -workingDirectory $_.fullname
}
```

You'll need to follow the [instructions](https://github.com/fixie/fixie/wiki/Test-Runner-Integrations ) to setup Azure DevOps to pick up the test case reporting automagically.

### **CONSIDER** customizing your test runner to fit into your CI system outside the build script
{: .text-yellow-300 }
Some CI systems have other tooling to help them run Visual Studio tests or other tests with code coverage more effectively. For example, Azure DevOps Visual Studio Test task can turn on code coverage analysis, and TeamCity can run JetBrains dotCover while unit testing.

When this happens, you should update your build tasks to avoid executing the unit tests twice.

## Static Analysis (ReSharper, ESLint)

Using static analysis tools during the build process will help the whole team maintain a common quality standard. With ReSharper and ESLint, there are baseline configurations you can import.

[Headspring ESLint defaults](https://github.com/HeadspringLabs/eslint-config-headspring)

### **DO** run ESLint automatically as part of your JavaScript build process, and fail on any warnings
{: .text-green-100 }
Follow the instructions for your Javascript build process to include ESLint automatically in the build process, and ensure that it is set up to fail on any warnings. The output will be picked up as part of your build logs and can be viewed in your CI system. Also, whenever the build is run locally, it will error out properly.

### **CONSIDER** running ReSharper Inspections in your .NET build script
{: .text-yellow-300 }
ReSharper code inspections can be run from an MSBuild target or via a command line tool available on NuGet [JetBrains.ReSharper.CommandLineTools](https://www.nuget.org/packages/JetBrains.ReSharper.CommandLineTools/2020.2.0-eap03).

To run them as part of a build script, you’ll need to install them in a neutral tools folder (to be included in your `setup.ps1` script):

```powershell
nuget install JetBrains.ReSharper.CommandLineTools -OutputDirectory tools -ExcludeVersion
```

Then you can run the inspections against your solution:

```powershell
./tools/JetBrins.ReSharper.CommandLineTools/tools/InspectCode.exe path-to-solution.sln -o=testReports/resharper.xml
```

You can see the other command line arguments here on the Inspect Code Documentation page. On its own, it won’t fail the build on any inspection failures, you’ll need to either manually poke at the XML report file, or use a plugin in your CI system to run the inspection instead.

If your team is already using ReSharper inspections in Visual Studio, then any issues should come up as part of the code review process and running an automated check may be unnecessary.

## Publishing Outputs

The purpose of a build is to produce artifacts that can be configured and deployed in target environments.

### **DO** define a folder for storing published build artifacts
{: .text-green-100 }
Your build artifacts should be stored in a separate folder from your source code to make it easier to configure CI systems to collect build outputs. Each top level project output should be in its own folder, and any NuGet packages that are created should be set aside in their own folders as well. This gives you the most flexibility in picking folders to be collected by CI systems without needing to know filenames specifically.

```powershell
properties {
    $projectRoot = "$(resolve-path .)"
    $publish = "$projectRoot/Publish"
}
```

### **DO** use the `publish` command for .NET Core applications to prepare your build outputs
{: .text-green-100 }
You must use the `dotnet publish` command to prepare your application for output. This will bring all the dependencies into the output folder and create the necessary runtime files to allow you to use the dotnet tool to execute your application.

## Integrations

**.NET Core Global Tools**

As of version 2.1, .NET Core provides the ability to install and run console applications through tooling extensions - .NET Core Global Tools. Psake builds can be extended using .NET Core Global Tools. A list of tools is curated here on Github. Tools that might be useful in assisting in the build experience include dotnet-cleanup, dotnet-roundhouse, FluentMigrator.DotNet.Cli and dotnet-format.

Once installed, these console apps are executed directly from the command line.

Global tools are installed to the user profile (`%USERPROFILE%\.dotnet`). The advantage is a known path for installation of useful tools that don’t require their inclusion within the solution directory and source code repository. You would have to include a call to verify installation through `dotnet tool install <tool>`.

### **DO** install then update global tools in your `setup.ps1` script
{: .text-green-100 }
In the .NET Core 3.1 SDK there is full support for adding global and local tools and easily restore them from the command line.

```powershell
# if there isn't a tool manifest for the solution already
dotnet new tool-manifest
dotnet tool install my-tool-name
```

To ensure you have the latest versions of the tools your project requires:

```powershell
dotnet tool restore
```

Add this to your `setup.ps1` script so that whenever a new developer runs it, you'll have the necessary tools installed.

**dotnet-cleanup**

`cleanup` is an executable that cleans a solution folder of bin and obj folders by default, and allows overrides for custom folders. This is an alternative to using the dotnet clean, which leverages an MSBUILD task and its verbose default output.

Install dotnet-cleanup with:

```powershell
dotnet tool install dotnet-cleanup
```

Execute `cleanup -y` from the solution directory to clean all bin and obj directories (-y is to confirm).

Cleanup can be invoked from a Psake task.

```powershell
task Cleanup -Alias clean {
    exec { dotnet cleanup -y } -workingDirectory $solutionDir
}
```

### **DO** use dotnet-roundhouse or FluentMigrator.DotNet.Cli to migrate your database.
{: .text-green-100 }
**dotnet-roundhouse**

If your project uses [RoundhousE](https://github.com/chucknorris/roundhouse) for migrations, you can install RoundhousE to your user profile to execute from inside a Psake task using the `dotnet rh` command. The advantage of using the dotnet-roundhouse here is not having to include the RoundhousE executables in a path within the solution folder. The same `dotnet rh` command is used for kicking off migrations.

```powershell
dotnet tool install dotnet-roundhouse
```

You can then execute the tool locally:

```powershell
task migrate-database -Alias migrate {
    exec { dotnet rh -cs $connectionString } -workingDirectory $databaseDir
}
```

**FluentMigrator.DotNet.Cli**

Projects that use Fluent Migrator also have a global tool option. Fluent Migrator installs with the following command:

```powershell
dotnet tool install FluentMigrator.DotNet.Cli
```

You run the tool with the `dotnet fm` command. `dotnet fm --help` provides help documentation with available commands. For an assembly containing FluentMigrator migrations at the path in variable `$db-migrations-path` and a connection string on `$connectionString`, Psake can migrate the database using the following:

```powershell
task migrate-database -Alias migrate {
    exec { dotnet fm migrate -c $connectionString -a $db-migrations-path } -workingDirectory $solutionDir
}
```

**PowerShell Modules**

Install your modules from PowerShell Gallery. First, the source needs to be trusted,

The first step is to trust the repository. Install the module to the current user scope. Also, here is where you would install any other needed tools.

```powershell
Set-PSRepository -Name PSGallery -InstallationPolicy Trusted
Install-Module Psake -Scope CurrentUser
```

**NuGet packages**

NuGet packages are now integrated into Visual Studio projects as PackageReferences in ItemGroup. NuGet packages are restored automatically to the `%USERPROFILE%\.nuget\packages\` folder automatically as part of the normal dotnet build process.

### **DO** add tasks for installing Node packages and building SPAs as part of your process.
{: .text-green-100 }
**Npx / npm**

Dotnet CLI has built in integrations with NPM. The command `dotnet new react` will create an NPM bridge target for automatically installing NPM packages and running build after executing the `ComputeFilesToPublish` target in the `csproj` file.

In cases where you need to build your SPA application separate from the rest of your code, you will want to create separate tasks for building your app with NPM.

```powershell
task Build-SPA {
    exec { npm install; npm run build }
}
```

A project might have any number of NPX commands created; all of these commands are available to Psake through the PowerShell shell. As an example, your project may need to launch a webpack Dev server before you can debug the web:

```powershell
task Run-WebDevServer {
    exec { npx run-webpack-dev-server }
}
```

**Aliases / Installing to local profile**

Like any PowerShell module, the `Invoke-Psake` command can be aliased. The following steps allow you to type Psake to execute the `Invoke-Psake` command from any location in your PowerShell console. Open the PowerShell profile (`%USERPROFILE%\Documents\WindowsPowerShell\Microsoft.PowerShell_profile.ps1`) and append the following to set an alias.

```powershell
# Create Psake alias
if (-not (Test-Path alias:Psake)) { Set-Alias -Name Psake -Value Invoke-Psake }
```

### **CONSIDER** adding property objects for Psake specific tasks that your use regularly to your PowerShell profile
{: .text-yellow-300 }
You may find that you have to override specific properties on a Psake task. You could create a hashtable (`@{"SomeProperty" = "SomeValue";"AnotherProperty" = "AnotherValue"}`) to hold the values. Then, you can pass your loaded Property hashtable from your PowerShell session without having to type out each setting, each time.

## Integrating with CI Systems

The build task should build as close to what the build server process. Build systems, like Azure DevOps, will provide a composable, or YAML based build workflow. If your project is a brownfield application that already has a build process built on such a workflow, then use the existing process. A Psake build script will still be useful for local DevOps tasks like kicking off a number of tasks to rebuild a test database with data, or to migrate the database.

If the build process is not yet configured to execute more complicated DevOps workflows that the local build script can already perform, you could add in the Psake task as a step, or a YAML job, into the existing workflow for that unmet build requirement.

In the instance that you have a new project and there is no compelling reason to consume a server-built workflow, create a build script that can cover the local and CI server build needs.

### **DO** create a continuous integration specific build task
{: .text-green-300 }
Your default build process locally will likely include tasks that assist in day-to-day developer operations. Create a task specific to building the solution on an Azure DevOps Pipeline. The task should depend on all of the tasks necessary to build the solution in one command.

Some build servers have plugins for running PowerShell or Psake that must be configured separately.

**Azure DevOps**

Running Azure on the build agent requires that Psake is installed as a module. As mentioned in First-time setup, create a `setup.ps1` file for downloading and installing the Psake module from PS Gallery. The first step is to trust the repository. Install the module to the current user scope.

### **DO** Execute `setup.ps1` in a Powershell Build Agent job to configure your required modules
{: .text-green-100 }
Add an Agent Job for the Inline PowerShell. In the Scripts to run text area, invoke your Psake command, with a TaskList of Psake tasks to execute (use your special CI build task here) and pass any needed Azure DevOps Variables thru the `Properties` object.

```powershell
Invoke-Psake ./psakefile.ps1 -TaskList cibuild -Properties @{"db_connection"=$(DbConnectionString;"important_property"=$(SomeImportantPropertyFromAzureDevOpsVariables))
```

**TeamCity**

TeamCity can run Psake scripts using the PowerShell plugin. The Psake readthedocs site documentation outlines the steps for configuring TeamCity server to execute Psake scripts.

If your TeamCity builds rely on environment, system and build properties, pass the TeamCity parameters to the CI tasks with the `TEAMCITY_BUILD_PROPERTIES_FILE` environment variable.

```powershell
$TCParams = ConvertFrom-StringData (Get-Content $env:TEAMCITY_BUILD_PROPERTIES_FILE -Raw);
```

With `$TCParams` loaded, you can now access `build.version` and other properties

```powershell
$TCParams['buld.number']
```

## Integrating with Databases

Ensuring that your local database is migrated to the latest version is easy with a Psake task. [RoundhousE](https://github.com/chucknorris/roundhouse) and [FluentMigrator](https://github.com/fluentmigrator/fluentmigrator) each have executables to invoke. Psake tasks can be created to explicitly migrate your database locally and through all environments.

**Using RoundhousE**

Using the `dotnet-roundhouse` tool, you can easily call the RoundhousE executable without specifying the full path, or exe. If you have a more complicated deployment you can pass in RoundhousE switches for customizing script locations that correspond to your repository.

```powershell
task Update-Database {
    exec { dotnet rh -c $connectionString -f $scriptsPath --dt=$db_type }
}
```

**Using FluentMigrator**

With the `FluentMigrator.DotNet.Cli` global tool installed, you can run the tool in Psake with `dotnet-fm`.

```powershell
task Migrate-Database -Alias migrate {
    exec { dotnet fm migrate up -c $connectionString -a $migrationAssembly } -workingDirectory $solutionDir
}
```
