---
title: "MAGE - Part 2"
date: 2024-03-16T15:52:11Z
summary: "Making a Game Engine - Part 2 - Generating Project Files And Project Structure"
---

{{< lead >}}
First there was project architecture then there was directory structure
{{< /lead >}}


{{< alert "github" >}}
Check out Nui Engine on [Github](https://github.com/ArnavMehta3000/NuiEngine.git)
{{< /alert >}}

## The Process

Since this is an editor-less game engine. We need to plan how the user's game (project) is integrated with the engine/project solution. Here is the proposed usage pipeline:

1. Clone/Fork the Github repository
2. Run a script to generate project build scripts
3. Generate project files
4. Open Visual Studio solution with engine and user game code projects linked!

## The Build System

Since I am only developing this project for the Windows OS, I plan to use Visual Studio as my IDE and therefore MSVC as my C++ compiler. But to generate the solution and add the user project files I will be using [Premake](https://premake.github.io/) as my solution generator/build system.

## Project Folder Structure

When the user clones the repository, they should see the following folder structure

```
NuiEngine (Root)/
├─ Docs/
│  ├─ Doxygen generated docs
├─ Build/
│  ├─ Intermediate/
│  ├─ <BUILD_CONFIG>/
│  │  ├─ NuiCore/
│  │  ├─ NuiTest/
│  │  ├─ NuiConsole/
│  │  ├─ NuiGraphics/
│  │  ├─ NuiAudio/
│  │  ├─ <USER_PROJECT_NAME>/
│  │  │  ├─ Saved/
│  │  │  │  ├─ NuiEngine.log
│  │  │  ├─ <USER_PROJECT_NAME>.exe
├─ Scripts/
│  ├─ Templates/
│  │  ├─ PremakeProjectTemplate.lua
│  │  ├─ PremakeSolutionTemplate.lua
│  ├─ premake5.exe
│  ├─ CreateProject.py
├─ Engine/
│  ├─ Core/
│  │  ├─ BuildCore.lua
│  ├─ Graphics/
│  │  ├─ BuildGraphics.lua
│  ├─ Audio/
│  │  ├─ BuildAudio.lua
│  ├─ Console/
│  │  ├─ BuildConsole.lua
│  ├─ Test/
│  │  ├─ BuildTest.lua
├─ <USER_PROJECT_NAME>/
│  ├─ <USER_PROJECT_NAME>/
│  │  ├─ Game code...
│  ├─ Main.cpp
│  ├─ Build<USER_PROJECT_NAME>.lua
├─ GenerateProjectFiles.bat

```
Initially there will be no *Build* and *User Project* folder. That will be generated using the *Scripts/CreateProject.py* script. All engine modules will be the in the *Engine* folder with their respective Premake Lua build files. These will be defined manually and will not change during project generation. I also include the premake5.exe with the repository.

In the *Scripts* folder there are also premake template files for the solution and user project that the python script will use when creating a project.

## Creating a New Project

To create a new project the user will have to run *Scripts/CreateProject.py* with the project name as the command line argument. The goal is to run this script only once when the project has to be created for the first time.

### Using Python

I decided to use python since it's super easy to easy to write and can be executed on the fly. This is what the Python script does:

- Takes the user project name as an command line argument
  - Usage (cmd): `CreateProject.py -projectname=<USER_PROJECT_NAME>`
- Creates project directory in the root folder
- Creates the `Build<USER_PROJECT_NAME>.lua` using the *PremakeProjectTemplate.lua* file in the project directory
- Creates a sub folder in the project directory with the same name as the project. This is where the user code will be written.
- Calls premake5.exe to generate the Visual Studio solution and project files

### Generating Project Files

There is also a batch script in the root folder called `GenerateProjectFiles.bat`. Which invokes premake5.exe and can be used to update the project and solution when new files are added.

With this system I can also create GitHub actions to run the tests automatically.

With all that done, we now have a project generated with the right links. You can check the implementation details on the GitHub repository.

<div style="display: flex; flex-wrap: wrap; gap: 10px;">
  {{< badge >}}Programming{{< /badge >}}
  {{< badge >}}Game Engine{{< /badge >}}
</div>
