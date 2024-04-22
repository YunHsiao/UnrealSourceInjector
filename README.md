<!--
SPDX-FileCopyrightText: 2024 Yun Hsiao Wu <yunhsiaow@gmail.com>
SPDX-License-Identifier: MIT
-->

# Crysknife

When implementing plugins with complex engine-level customizations for Unreal® Engine, due to many design decisions of the engine architecture, it is very hard, if not impossible to keep away from modifying stock engine files to make everything fully works.

In fact oftentimes the changes are completely scattered across engine modules, which could be fine for one in-house engine base, but extremely hard to port to any other.

This project aims to completely automate the injection process, with powerful customization capabilities. This not only enables quick upgrade to newer engine versions, but also essentially making your custom engine features deployable to any existing engine code base as a proper engine plugin.

Here's how it works under the hood:

Changes in existing engine files are stored as patches:
* Patches are fuzzy matched with customizable tolerances
* Multiple patches can be generated targeting at different engine versions when they become just too diverged to be fuzzy matched
* When applying patches, the closest matched version to destination engine base will be used
* All injections are strictly reversible with a single command
* As the last resort when patching fails, the error message comes with a full file diff HTML to help you manually resolve the conflicts

> Only actions making observable differences are executed, so an empty console output means everything's up-to-date.

New source files can be either copied or symbolically linked into target directory.

> Although tempting, symbolic links doesn't update with source file changes which may cause
> new updates to be ignored by compiler during development, so copy is made the default operation.

More complex injection behaviors can be specified with the [config system](#Config-System).

# Environment Setup

You should clone this repo as an engine plugin:
```
Engine/Plugins/Crysknife/
```
And code patches will be read from the following directory to `Engine/Source`:
```
Engine/Plugins/${PluginName}/SourcePatch/
```
All relative structures from the `SourcePatch` directory will be preserved.

The injector itself doesn't magically change the way your code is organized. Here are the recommended general principles:
* Only inject sources when there is no way around, you can still go very far with only the builtin plugin system
* Try to move as much code as possible to separate new files instead of embedding them inline into existing engine base
* Empirically 90% of the code can be written inside plugin itself, 90% of the rest injections can be organized into new engine source files

# Patch Syntax

For changes in existing engine files we detect & inject in the following forms (with comment guards):

### Multi-line

```cpp
// ${PluginName}${Comments}: Begin
** YOUR CODE BLOCK HERE **
// ${PluginName}: End
```

### Single-line

```cpp
** YOUR ONE-LINER HERE ** // ${PluginName}${Comments}
```

### Next-line
Note that there can be no code at the same line with the comment guard:

```cpp
// ${PluginName}${Comments}
** YOUR ONE-LINER HERE **
```

> For non performance-critical code, try to find the most representative context to insert your code (beginning of class,
> after long-standing public interfaces, etc.), which can greatly increase the chances of finding matches in different engine bases.

Additionally, for modifying stock engine code, follow these steps:
* Comment out the original code block (only line comments are supported atm.)
* Guard the comment block using any of the above forms, with one special tweak*
* Add a new guarded block normally for your own code

Where the special tweak is:

```cpp
// ${PluginName}-${Comments}
```

**The minus sign** after the plugin name is enough to tell the injector that the surrounding code segment is directly from stock engine source, essentially making it a deletion block.

> The minus sign can be omitted in the ending comment for multi-line guards.

## Command Line Interface

* `-P [PLUGIN]` The input plugin folder name (also as the comment guard tag), always required.
* `-I [DIRECTORY]` Customize the source directory where the patches are located
* `-O [DIRECTORY]` Customize the destination directory containing target sources to be patched
* `-D [VAR=VALUE,]...` Define config variables
* `-B` Skip actions on builtin source patches
* `-S` Generate a set of setup scripts for the specified plugin

### Actions

* `-R [FILES|DIRECTORIES]...` Search specified file or directory, register any patched file and generate
* `-U [FILES|DIRECTORIES]...` Search specified file or directory, unregister any patched file and update
* `-G` Generate/update patches
* `-C` Clear patches from target files
* `-A` Apply existing patches and copy all new sources (default action)

> Actions are combinatorial:  
> e.g. `-G -A` for generate & apply (round trip), `-G -C` for generate & clear (retraction), etc.

### Modifiers

* `-i [FILTER]` or `--inclusive-filter [FILTER]` Inclusive target path filter for all actions
* `-e [FILTER]` or `-exclusive-filter [FILTER]` Exclusive target path filter for all actions
* `-l` or `--link` Make symbolic links instead of copy all the new files
* `-f` or `--force` Force override existing files
* `-d` or `--dry-run` Test run, safely executes the action with all engine output remapped to plugin's `Intermediate/Crysknife/Playground` directory
* `-v` or `--verbose` Log more verbosely about everything
* `-t` or `--treat-patch-as-file` Treat patches as regular files, copy/link them directly

### Parameters

* `--patch-context [LENGTH]` Patch context length when generating patches, default to 50
* `--content-tolerance [TOLERANCE]` Content tolerance in [0, 1] when matching sources, default to 0.5
* `--line-tolerance [TOLERANCE]` Line tolerance when matching sources, default to infinity (line numbers may vary significantly between engine versions)

## CLI Examples

For first-time setup you should use the script file matching your operating system and run:
* `Crysknife.sh -P [PLUGIN] -S`

This will generate a few `Setup` scripts to your plugin directory as the main entry for any operations.

### New Engine Source Files

Say we are adding a new source file under `Engine/Source/Runtime/Engine/Private` named `MyEnginePlugin.cpp`:

* Create `MyEnginePlugin.cpp` under `${PluginRoot}/SourcePatch/Runtime/Engine/Private`
* `Setup.sh` (by default runs the apply action)
* The source file should be present under the same hierarchy inside engine source directory

### Modify Existing Engine Source

Say we want to modifying some existing engine source file:
* Go ahead and modify the engine source directly, remember to add the aforementioned comment guards
* `Setup.sh -R ${FullPathToModifiedEngineSourceFile}`
* `Setup.sh -G` afterwards to update all patches before committing

> Before releasing it is recommended to check the cleared source still works: (may require full recompilation)  
> `Setup.sh -G -C` (The retraction action)  
> This can be used to ensure all relevant changes are properly guarded.

### Remove Existing Patch From Engine Source

Say we want to permanently remove all our previous modification from some existing engine source file:

* `Setup.sh -U ${PathToEngineSourceToBeUnpatched}...`
* The source file should be un-patched and the relevant patch files will be deleted

If we only want to temporarily remove the patches from all files under `Engine/Source/Runtime/Engine`:

* `Setup.sh -C -i Runtime/Engine` (To un-patch source files)
* `Setup.sh -i Runtime/Engine` (To re-apply patches)

### Porting To A Completely Different Engine Base

* `Setup.sh`
* Resolve potential conflicts by either adjust the `--content-tolerance` parameter or inspecting the reference diff HTML & manually patch in (remember the comment guards)
* `Setup.sh -G`
* A new set of patches matching the current engine version will be generated and ready to be committed

## Config System

Every `SourcePatch` directory can have one config file `Crysknife.ini` at the root,
to specify more complex patching behaviors such as conditional remapping, etc. in the following framework:

```ini
; Declare any variables you need
[Variables]
Var1=Value4
Var2=True

; Applies to all files
[Global]
; Multiple conditions are allowed
Rule1=Predicate1:Value1|Value2
; Or add them in separate lines
+Rule1=Predicate1:Value3
; Variable references & reverse dependencies
+Rule1=Predicate3:${Var1},Predicate4:!Value5

[Path/To/Dir1]
; Only apply to specified subdirectory
ScopedRule1=Predicate2

; This scope automatically extends from the above parent scope
[Path/To/Dir1/Folder1]
ScopedRule2=Predicate5

; Multiple directories are allowed
[Path1|Path2]
```

* The global section applies the rule to all subdirectories inside `SourcePatch` folder, while custom sections with relative file/directory paths can be used to limit the effective scope of the rules.
* Multiple subdirectories can be specified within one section title, separated with `|`. Scoped rules will apply to all subdirectories. 
* If multiple sections affect the same file, the inner (path) section will automatically extends from the outer section.
* The variables section declares custom variables that can be referenced with `${VariableName}` in any value.
* Any value can be preceded by `!` to indicate a reverse predicate (satisfies if condition is not met).

### Supported Rules

`SkipIf=[PREDICATE]...`
* Skip action if specified predicates are satisfied

`RemapIf=[PREDICATE]...`
* Remap action to a different destination directory if specified predicates are satisfied

`RemapTarget=[PATH]`
* The remap destination, which would be replacing the section name part of the input file path
* Must be specified if `RemapIf` is present

`FlattenIf=[PREDICATE]...`
* Flatten the folder hierarchy if specified predicates are satisfied

### Supported Predicates

`TargetExists:[FILE|DIRECTORY]...`
* Satisfies if any of the specified file/directory exists

`IsTruthy:[SWITCH]...`
* Satisfies if any of the specified value is true

`NameMatches:[NAME]...`
* Satisfies if the input file name matches

`Conjunctions:All|Predicates|Root|TargetExists|IsTruthy|NameMatches...`
* Changes the logical behavior of specified predicate to conjunction (logical AND)
* `Root` means the logical operations between different predicates
* `Predicates` means all logical operations inside every defined predicate
* `All` means `Predicates|Root`
* (By default all conditions are disjunction, i.e. the results are logically OR-ed together)

`^BaseDomain`
* Indicating current rule line is in base domain, therefore can only be overrule by the same domain directives
* Use this iff you want to override base rules defined in [BaseCrysknife.ini](BaseCrysknife.ini)
* Must be defined at the start of the current rule.

`Always` / `Never`
* Always / never satisfies

## Config Examples

### Conditional Skip

Skip any actions for `astc-encoder` directory if `Engine/Source/ThirdParty/astcenc` already exists:

```ini
[ThirdParty/astc-encoder]
SkipIf=TargetExists:ThirdParty/astcenc
```

### Flat Debug

Flatten all patch outputs and remap to `Engine/Source/Test`:

```ini
[Global]
RemapIf=Always
RemapTarget=Test
Flat=True
```

## Builtin Source Patches

We included some useful utilities in the builtin `SourcePatch` folder, which can provide some interesting trade-offs.

|                                   Include Path                                   | Module |                                Comment                                |
|:--------------------------------------------------------------------------------:|:------:|:---------------------------------------------------------------------:|
| [Misc/PrivateAccessor.h](SourcePatch/Runtime/Core/Public/Misc/PrivateAccessor.h) |  Core  | A tiny library for accessing private members from non-friend contexts |
