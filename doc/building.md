# Building

The CKAN and NetKAN clients are built using standard .NET/Mono tools with [Cake](http://cakebuild.net/) (C# Make)
as the build system which glues multiple steps together. This document is designed to be a comprehensive overview of
how the CKAN build system currently works and how changes to it should be made.

## tl;dr
```
> git clone https://github.com/KSP-CKAN/CKAN
> cd CKAN
> ./build
> _build/repack/ckan.exe version
v1.23.45-dev+abcdef
> _build/repack/netkan.exe --version
v1.23.45-dev+abcdef
```

## Requirements

The following are the minimum requirements to build CKAN and NetKAN from source:

### Unix-like Systems
- A POSIX compliant shell interpreter at `/bin/sh`
- [Mono](http://www.mono-project.com/) >= 4.0.0
- [cURL](https://curl.haxx.se/) accessible in PATH 

### Windows Systems
- PowerShell
- MSBuild (usually by installing [Visual Studio](https://www.visualstudio.com/vs/community/))
- .NET Framework >= 4.5.0

## Build Directory

A core philosophy of the build system is that builds should be "clean", that is it should be easy to restore the source
directory to its original state after a build. As such **ALL** build artifacts should be output to a top-level 
directory named `_build`. This directory is listed in `.gitignore` so that is not committed to source control. Removing
this directory (`rm -r _build`) should return the source directory to its original state. Therefore any build artifacts
that are output outside this directory should be considered a bug.

### Subdirectories

The `_build` directory contains several subdirectories for different purposes. Nothing should be output to the `_build`
directory itself, instead use one of the existing or create a new subdirectory.

- `_build/cake`: Used to store packages used by Cake. Use of this directory by Cake is controlled by the `cake.config`
  file in the top-level directory.
    - `_build/cake/addins`: Used to store [Cake addins](http://cakebuild.net/addins).
    - `_build/cake/modules`: Used to store Cake modules.
- `_build/lib`: Used to store packages that are *automatically* resolved by a package manager.
    - `_build/lib/nuget`: Used to store packages that are automatically resolved by NuGet.
- `_build/meta`: Used to store source code that is dynamically generated by the build system. This is primarily used to
  generate version information that is embeded in the built assemblies.
- `_build/out`: Used to store the output of project builds. As CKAN/NetKAN is currently only a .NET project it has no
  subdirectories. If, however, multiple project types are introduced in the future output should be moved to
  subdirectories for each project type, e.g. `_build/out/dotnet`. Each .NET project is built to its own subdirectory
  based on its assembly name. Below that directory is one or more directories for each build configuration (currently
  either Debug or Release).
  - `_build/out/$ASSEMBLY_NAME/$CONFIGURATION/bin`: The final output directory for .NET builds. Usable assemblies are
    found here.
  - `_build/out/$ASSEMBLY_NAME/$CONFIGURATION/obj`: Directory used by the build system for intermediate object
    generation. Developers should not care about the contents in these directories.
- `_build/repack`: Used to store the output of [ILRepack](https://github.com/gluck/il-repack).
- `_build/test`: Used to store the output of test runners.
    - `_build/test/nunit`: Used to store the output of the NUnit test runner.
- `_build/tools`: Used to store tools downloaded by the bootstraper or Cake.

## Bootstrapper Scripts

The primary entry point to the CKAN build system are the two bootstraper scripts in the top level directory `build` and
`build.ps1`. Developers use these scripts for all build related operations.

`build` is POSIX shell script for use on Unix-like systems (BSD, Linux, macOS, etc.). `build` is exected by `/bin/sh`
which expected to be a POSIX compatible shell. It is quite intentionally **not** a bash shell script and as such
contains no [bashisms](https://en.wiktionary.org/wiki/bashism). It can therefore be used on systems where `/bin/sh` is
not bash but instead an alternative shell interpreter like dash. Any changes to this script should be careful not to
introduce such bashisms that would break this compatibility.

`build.ps1` is a PowerShell script for use on Windows systems. Users on Windows *must* use PowerShell, the old
`cmd.exe` interpreter is not supported.

Both of these scripts are used to "bootstrap" the build environment. They function identically and as such we will
treat them as a singular script (`./build`) for all platforms. The basic operation of the bootstrap scripts are to:

1. Download [NuGet](https://www.nuget.org/)
2. Use NuGet to install packages defined in `packages.config` (currently, only Cake)
3. Parse the command line arguments
4. Execute Cake with the appropriate command line arguments
5. Quit with the exit code generated by Cake

The specifics of each operations are given in the following sections.

## NuGet

NuGet is the standard .NET package manager. Since nearly every useful .NET tool and library is provided through NuGet
it can be used to bootstrap the rest of the build system. Specifically the bootstrapper scripts are used to download
NuGet which is then used to install Cake. The version of Cake to use is stored in `packages.config`, a standard NuGet
file for storing package dependencies. The bootstrapper scripts install all packages defined in `packages.config` so
it can be used to install any other high level packages necessary for the build, although this should not be necessary
that often (if at all) since most other packages can be installed as Cake addins or tools.

## Cake

After Cake is installed by NuGet, the bootstrapper scripts then parse the command line and execute Cake.

The bootstrapper scripts use the following command line:

```
./build [<target>] [<cake-args>...]
```

Which is then used to execute Cake with the following command line:

```
cake.exe build.cake [--target=<target>] [<cake-args>...]
```

The first argument is the Cake script to execute. This is always `build.cake` and does not ever need to be changed. The
second argument is the `--target` argument which tells Cake which task in `build.cake` to execute. If not provided it
defaults to the (appropriately named) `Default` task. Finally, the bootstrapper scripts will pass any additional
arguments to Cake as is.

### `build.cake`

`build.cake` is the Cake script which does most of the heavy lifting for the build. Cake scripts are C#-like files
which are dynamically compiled by either the Roslyn compiler (on Windows systems) or the Mono script compiler (on
Unix-like systems). Pretty much all the same rules as standard C# apply with the exception of Cake-specific
[preprocessor directives](http://cakebuild.net/docs/fundamentals/preprocessor-directives).

The CKAN build script is fairly simply and self-explanatory. Some conventions of note:

- With the exception of high-level tasks which are designed to be used by developers (`Ckan`, `Netkan`, etc.). Tasks
  should be named as `Verb-Noun`. This roughly follows the
  [convention](https://msdn.microsoft.com/en-us/library/ms714428%28v=vs.85%29.aspx?f=255&MSPPError=-2147217396) of
  naming PowerShell cmdlets without the restrict verb requirements.
- Tasks that perform a specific task without invoking any dependent tasks should have a `+Only` suffix applied. For
  example the `Test+Only` task will execute tests without first building the solution.

#### Tasks

The purpose of each defined task is as follows:

- `Default`: This is the task executed if none is explicitly specified. Will build CKAN and NetKAN executables.
- `Ckan`: Builds *only* the CKAN executable.
- `Netkan`: Builds *only* the NetKAN executable.
- `Restore-Nuget`: Restores NuGet dependencies for the entire solution.
- `Build-DotNet`: Builds the solution file using `msbuild` (on Windows systems) or `xbuild` (on Unix-like systems).
- `Generate-GlobalAssemblyVersionInfo`: Generates a `GlobalVersionAssemblyInfo.cs` file in `_build/meta` which is
  linked to by all projects containing dynamically generated version information.
- `Repack-Ckan`: Use ILRepack to generate a self-contained `ckan.exe`.
- `Repack-Netkan`: Use ILRepack to generate a self-contained `netkan.exe`.
- `Test`: Execute all tests (and build the solution).
- `Test+Only`: Execute all tests (without building the solution).
- `Test-UnitTests+Only`: Execute unit (NUnit) tests.
- `Test-Executables+Only`: Execute smoke tests for repacked CKAN and NetKAN executables.
- `Test-CkanExecutable+Only`: Execute smoke tests for repacked CKAN executable.
- `Test-NetkanExecutable+Only`: Execute smoke tests for repacked NetKAN executable.
- `Version`: Print the dynamically generated version information.

#### Helper Methods

`build.cake` also contains a couple of helper methods.

- `GetVersion()`: Read the most recent version number from `CHANGELOG.md` and and attach a short version (12-character)
  of the current git commit hash if available. The output is a
  [Sematic Versioning 2.0.0](http://semver.org/spec/v2.0.0.html) string.
- `GetGitCommitHash()`: Executes `git` (assumed to be in PATH) to get the current commit hash.
- `RunExecutable(FilePath, string)`: Wrapper around Cake's `StartProcess()` method that returns string output of process and
  automatically throws an exception if the process does not exit with an exit code of 0.

## Build process

The basic operation of the actual build process is as follows:

- Restore NuGet packages
    - NuGet is executed on the solution file (`CKAN.sln`) which restores the packages for each project. Each project
      maintains its own `packages.config` file which defines its dependencies. All packages are stored in
      `_build/lib/nuget` across all projects. This is controlled by the `nuget.config` file in the top-level directory.
      Under no circumstances should NuGet packages be committed to source control.
- The solution is built
    - `msbuild` (on Windows systems) and `xbuild` (on Unix-like systems) is executed on the solution file (`CKAN.sln`).
      This causes each project file (`.csproj`) to be built producing an assembly in `_build/out`. Each project file
      has the following lines which control its output directory. Any new projects should include these lines:
      ```xml
      <OutputPath>..\_build\out\$(AssemblyName)\$(Configuration)\bin\</OutputPath>
      <IntermediateOutputPath>..\_build\out\$(AssemblyName)\$(Configuration)\obj\</IntermediateOutputPath>
      ```
- Exeutables are repacked
    - ILRepack is executed to merge the CKAN executable and all its dependencies and the NetKAN executable and all its
      dependencies into singular executable files. This allows the CKAN and NetKAN clients to be distributed as single
      files without complex install procedures. These are the "final" output of the build process and are stored in:
      `_build/repack/$CONFIGURATION`.
- Unit tests are executed
    - The NUnit unit tests in `CKAN.Tests.dll` are executed
- Smoke tets are executed
    - These are simple tests designed to make sure there isn't anything grossly wrong with the build. All they do is
      execute the repacked `ckan.exe` and `netkan.exe` and make sure their version output matches the expected output.

## Travis

Travis is the continuous integration (CI) environment that is used to automatically build CKAN and NetKAN. Travis's
configuration is controlled by `.travis.yml` in the top-level directory. Upon every push Travis will execute the cross
product of mono versions and build configurations. As there are currently three specified mono versions (`4.6.2`,
`4.2.4`, and `4.0.5`) and two specified build configurations (`Debug`, `Release`), Travis will execute *six* builds per
push. Travis's operation is pretty simple:

- A bunch of packages are installed to make the build work.
- Some commands are executed to simulate a graphical environment for testing.
- The solution is built using `./build --configuration=$BUILD_CONFIGURATION`
- The solution is tested using `./build test+only --configuration=$BUILD_CONFIGURATION --exclude=FlakyNetwork`
    - The `--exclude=FlakyNetwork` argument is used to not execute tests that rely upon a network connection and could
      thus behave nondeterministically (frustrating behavior for a CI system).
- The built executables (`ckan.exe` and `netkan.exe`) are uploaded to Amazon S3 if the following conditions are met:
    - All previous steps were successful
    - Its on the `master` branch
    - The build configuraton is `Release`
    - The mono version used is the latest (currently `4.6.2`)
- A GitHub release is generated if the following conditions are met:
    - All previous steps were successful
    - The commit is tagged
    - The build configuration is `Release`
    - The mono version used is the latest (currently `4.6.2`)