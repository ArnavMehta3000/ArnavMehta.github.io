---
title: "MAGE - Part 8"
date: 2024-06-22T15:35:46+01:00
summary: "Making a Game Engine - Part 8 - New Build System"
---

{{< lead >}}
From Premake to Xmake!
{{< /lead >}}

{{< alert "github" >}}
Check out Nui Engine on [Github](https://github.com/ArnavMehta3000/NuiEngine.git)
{{< /alert >}}

In [Part 2](https://arnavmehta3000.github.io/posts/mage2/), I decided to make use of [Premake5](https://premake.github.io/) to generate my visual studio solution files. Premake not not a complete build system (like CMake), instead it is only useful to generate IDE solution/project files. Now that Nui Engine has grown quite a bit, I have decided to migrate to a new build system called [Xmake](https://xmake.io/#/).

## What is Xmake?

From Xmake's website:

- XMake is a cross platform and lightweight Lua based build utility.
- Xmake can be used to directly build source code (like Make or Ninja), or it can also be used to project generate source files like XMake or Meeson.
- Xmake comes with its own package manager called [XRepo](https://xrepo.xmake.io/#/), but also supports compiling external CMake based projects.

`Xmake = Build backend + Project Generator + Package Manager + [Remote|Distributed] Build + Cache`

In other words:

`Xmake ≈ Make/Ninja + CMake/Meson + Vcpkg/Conan + distcc + ccache/sccache`

## Migrating From Premake to Xmake

### Removing Premake Files

First step in migrating from Premake to Xmake, involves deleting all Premake Lua build scripts and the Premake5 binary.

Remove:

- BuildCore.lua
- BuildGraphics.lua
- BuildTestbench.lua
- Template script files
- CreateProject.py (Python script that created the empty project for us)
- Premake5.exe

### Understanding Xmake

After we have removed Premake file, we have to create the `xmake.lua` file in the project root. But before we can do that we need to understand how xmake handles solutions/projects.

- In Xmake you create a new `project`, this is equivalent to your solution.
- Premake's equivalent to solution projects in Xmake is referred to as a `target`
- A single project can have multiple targets (similar to how a visual studio solution can have multiple projects)
- Targets and projects have rules applied to them. These can be custom rules or in-built.

### Creating Root xmake.lua

We will start by creating a new file in the project root called `xmake.lua`, this file will handle including all of our other projects and actions (explained later).

```lua
local solution_name = "Nui"

-- XMake version
set_xmakever("2.9.2")

-- Build configs
add_rules("mode.debug", "mode.release", "mode.releasedbg")
set_allowedmodes("debug", "release", "releasedbg")

-- Platform and architecture - only support x64 Windows
set_allowedplats("windows")
set_allowedarchs("windows|x64")

-- Set C/C++ language version
set_languages("c17", "cxx23")

-- Project name and version
set_project(solution_name)
set_version("0.0.1")

-- Set default build mode
set_defaultmode("debug")

-- Generate clang compile commands on build
-- add_rules("plugin.compile_commands.autoupdate")

-- Update generated visual studio project files on build
-- add_rules("plugin.vsxmake.autoupdate")

-- Add defines
add_defines("UNICODE")

if is_mode("debug", "releasedbg") then
	add_defines("NUI_DEBUG")
end

if is_mode("release") then
    add_defines("NUI_RELEASE")
end

-- Include all xmake projects
includes("**/xmake.lua")

```

A big chunk of this file is self explanatory, but here is a detailed explanation of what is happening in this file.

- Since it is a Lua script, I have a local variable at the start that will allow the end user to choose a solution name (only important if generating IDE files)
- Next I set the minimum Xmake version, which is the latest at the time of writing
- I then add rules and build configurations for all projects. I want to support Debug, ReleaseDbg and Release which is similar to my Premake configuration of Debug, Release and Shipping. These modes are built into Xmake so that means that I do not have to spend a lot of time configuring each compiler setting
- Next I set the project name and version
- After that, based on the build configuration/mode, I set the Nui macro defines

### Creating Engine xmake.lua

The engine build files in this case are the `xmake.lua` scripts for the Engine and Graphics projects. They both are very similar:

For Nui Graphics project:

```cpp
target("NuiGraphics")
    set_default(false)
    set_kind("static")
    add_files("**.cpp")
    add_headerfiles("**.h")
    add_includedirs("$(scriptdir)/Engine/")
    add_links("User32.lib")
    set_group("Nui")
target_end()
```

For Nui Core project:

```cpp
target("NuiCore")
    set_default(false)
    set_kind("static")
    add_files("**.cpp")
    add_headerfiles("**.h")
    add_extrafiles("$(projectdir)/Engine/External/DirectXTK.inl")
    add_includedirs("$(scriptdir)/Engine/")
    add_links("NuiGraphics", "User32.lib")
    add_deps("NuiGraphics")
    set_group("Nui")
target_end()
```

These script files are quite straight forward

- Here I am creating two targets called NuiCore and NuiGraphics
- Both have `set_default` to false, since we do not want to run them on startup
- Both are static libraries
- Both have thier include paths, external files and library links and dependencies set up
- They both are part of the `Nui` group

## Generating User Project Files

When using premake, I had a Python script that generated the premake lua script and set up the directories. When using xmake, I do have have to make use of that. Xmake allows running of custom lua script and also supports custom commands. So that is what we will use.

## Nui Create Action

Xmake allows creation of actions and plugins which are basically custom commands. I want to create a command called `nui-create` which would create a new game project. So it would look something like this.

```bash
xmake nui-create -p Testbench
```

In the Scripts folder, let's create a new `xmake.lua` file. This file will contain all the custom commands that we will use. Starting with create a new project

```lua
-- Task to create a new Nui game project
task("nui-create")
	set_category("action")
	on_run("CreateProject")

    -- Set the command line options for the plugin. There are no parameter options here, just the plugin description.
    set_menu
    {
        -- Settings menu usage
        usage = "xmake nui-project [project-name]",
        -- Setup menu description
        description = "Create a new Nui project",

        -- Set menu options, if there are no options, you can set it to {}
        options =
        {
            -- Set kv as the key-value parameter and set the default value: black
            {'p', "project", "kv", "NuiGame", "Set the project name." }
        }
    }
task_end()
```

Since the file is also called `xmake.lua`, it is automatically picked by our root `xmake.lua` file. Thanks to pattern matching!

In this script

- We create a new action (category) called `nui-create`
- Whenever this action is triggered (`on_run`), we will run another Lua file called `CreateProject.lua`
- We can set the menu setting as well, which will be displayed when we add the `-h` or `--help` flag

Now if we run the command: `xmake -h`, you will see the `nui-create` action

If you run the following command:

```
xmake nui-create -h
```

We get the following output:

```bash
PS E:\Dev\NuiEngine> xmake nui-create -h
xmake v2.9.2+master.33f3078f4, A cross-platform build utility based on Lua
Copyright (C) 2015-present Ruki Wang, tboox.org, xmake.io
                         _
    __  ___ __  __  __ _| | ______
    \ \/ / |  \/  |/ _  | |/ / __ \
     >  <  | \__/ | /_| |   <  ___/
    /_/\_\_|_|  |_|\__ \|_|\_\____|
                         by ruki, xmake.io

    👉  Manual: https://xmake.io/#/getting_started
    🙏  Donate: https://xmake.io/#/sponsor


Usage: $xmake nui-project [project-name]

Create a new Nui project

Common options:
    -q, --quiet                      Quiet operation.
    -y, --yes                        Input yes by default if need user confirm.
        --confirm=CONFIRM            Input the given result if need user confirm.
                                         - yes
                                         - no
                                         - def
    -v, --verbose                    Print lots of verbose information for users.
        --root                       Allow to run xmake as root.
    -D, --diagnosis                  Print lots of diagnosis information (backtrace, check info ..) only for developers.
                                     And we can append -v to get more whole information.
                                         e.g. $ xmake -vD
    -h, --help                       Print this help message and exit.

    -F FILE, --file=FILE             Read a given xmake.lua file.
    -P PROJECT, --project=PROJECT    Change to the given project directory.
                                     Search priority:
                                         1. The Given Command Argument
                                         2. The Envirnoment Variable: XMAKE_PROJECT_DIR
                                         3. The Current Directory

Command options (nui-create):
    -p PROJECT, --project=PROJECT    Set the project name. (default: NuiGame)
```

At the bottom of this output, you can see the usage help message we defined in `set_menu`.

## Create Project Lua Script

Now if we create a new Lua script called `CreateProject.lua` in the same directory as the above `xmake.lua` script - `./Scripts/CreateProject.lua`, when we run this action with a project name, it will run this Lua file.

```lua
import("core.base.option")

projectName = nil
templatesPath = nil
scriptDir = nil
rootDir = nil
projectDir = nil


function CreateXmakeFile()
	-- Get file paths
    local templateFilePath = path.join(templatesPath, "XmakeProjectTemplate")
    local outputFilePath = path.join(projectDir, "xmake.lua")

    -- Get template file contents
    local templateFile = io.open(templateFilePath, "r")
    if not templateFile then
       	raise("Failed to open xmake template file. Expected path: " .. templateFilePath)
    end
    local templateContent = templateFile:read("*all")
    templateFile:close()

    -- Replace '%PROJECT_NAME%' with the actual project name
    local modifiedContent = templateContent:gsub("%%PROJECT_NAME%%", projectName)

    -- Create and write to the output file
    local outputFile = io.open(outputFilePath, "w")
    if not outputFile then
        error("Failed to write xmake project file. Expected path: " .. outputFilePath)
    end
    outputFile:write(modifiedContent)
    outputFile:close()

    cprint("Created project file: ${underline}%s", outputFilePath)
end

function CreateMainCppFile()
	-- Get file paths
    local templateFilePath = path.join(templatesPath, "MainCppProjectTemplate")
    local outputFilePath = path.join(projectDir, "Main.cpp")

    -- Get template file contents
    local templateFile = io.open(templateFilePath, "r")
    if not templateFile then
       	raise("Failed to open xmake template file. Expected path: " .. templateFilePath)
    end
    local templateContent = templateFile:read("*all")
    templateFile:close()

    -- Replace '%PROJECT_NAME%' with the actual project name
    local modifiedContent = templateContent:gsub("%%PROJECT_NAME%%", projectName)

    -- Create and write to the output file
    local outputFile = io.open(outputFilePath, "w")
    if not outputFile then
        error("Failed to write xmake project file. Expected path: " .. outputFilePath)
    end
    outputFile:write(modifiedContent)
    outputFile:close()

    cprint("Created main file: ${underline}%s", outputFilePath)
end

function CreateApplicationHeaderFile()
	-- Get file paths
    local templateFilePath = path.join(templatesPath, "ApplicationHeaderProjectTemplate")
    local outputFilePath = path.join(projectDir, path.join(projectName, projectName .. "App.h"))

    -- Get template file contents
    local templateFile = io.open(templateFilePath, "r")
    if not templateFile then
       	raise("Failed to open xmake template file. Expected path: " .. templateFilePath)
    end
    local templateContent = templateFile:read("*all")
    templateFile:close()

    -- Replace '%PROJECT_NAME%' with the actual project name
    local modifiedContent = templateContent:gsub("%%PROJECT_NAME%%", projectName)

    -- Create and write to the output file
    local outputFile = io.open(outputFilePath, "w")
    if not outputFile then
        error("Failed to write xmake project file. Expected path: " .. outputFilePath)
    end
    outputFile:write(modifiedContent)
    outputFile:close()

    cprint("Created application header file: ${underline}%s", outputFilePath)
end

function CreateApplicationCppFile()
	-- Get file paths
    local templateFilePath = path.join(templatesPath, "ApplicationCppProjectTemplate")
    local outputFilePath = path.join(projectDir, path.join(projectName, projectName .. "App.cpp"))

    -- Get template file contents
    local templateFile = io.open(templateFilePath, "r")
    if not templateFile then
       	raise("Failed to open xmake template file. Expected path: " .. templateFilePath)
    end
    local templateContent = templateFile:read("*all")
    templateFile:close()

    -- Replace '%PROJECT_NAME%' with the actual project name
    local modifiedContent = templateContent:gsub("%%PROJECT_NAME%%", projectName)

    -- Create and write to the output file
    local outputFile = io.open(outputFilePath, "w")
    if not outputFile then
        error("Failed to write xmake project file. Expected path: " .. outputFilePath)
    end
    outputFile:write(modifiedContent)
    outputFile:close()

    cprint("Created application cpp file: ${underline}%s", outputFilePath)
end

function CreateAssetsFolder()
	-- Get asset directory
	local assetDir = path.join(projectDir, "Assets")

	-- Create assets folder
	os.mkdir(assetDir)

	cprint("Created assets directory: ${underline}%s", assetDir)
end

function main()
	-- Get parameter content and display information
    projectName = option.get("project")

    -- Scripts directory
    scriptDir = os.scriptdir()
    -- Root Nui directory
    rootDir = os.projectdir()
    -- User project directory
    projectDir = path.join(rootDir, projectName)
    -- Template files directory
    templatesPath = path.join(scriptDir, "Templates")

    -- Create project xmake file
    CreateXmakeFile()

    -- Create Main.cpp file
    CreateMainCppFile()

    -- Create <Project>App.cpp file
    CreateApplicationCppFile()

    -- Create <Project>App.h file
    CreateApplicationHeaderFile()

    -- Create assets folder
    CreateAssetsFolder()

    cprint("${green}Done")
end
```

Here in this file, I have a bunch of template files, which have `%PROJECT_NAME%` in them. This script takes those templates, replaces that string with the passed in project name and creates new project files in the directories in the root directory.

> Here I do not discuss the contents of the template files, since the the contents are similar to what we have discussed in the past. If you want to take a look at the contents of the template files then you can check them out on the repository.

## Conclusion

I know this article was meant to expand on the previous ECS framework that we built. But I ended up changing the build system and decided to talk about this first. In the next one I will go back to the ECS framework and continue building the engine.

<div style="display: flex; flex-wrap: wrap; gap: 10px;">
  {{< badge >}}Programming{{< /badge >}}
  {{< badge >}}Game Engine{{< /badge >}}
  {{< badge >}}C++{{< /badge >}}
  {{< badge >}}Build Systems{{< /badge >}}
</div>