---
title: "MAGE - Part 1 "
date: 2024-03-10T14:24:17Z
summary: Making a game engine - Part 1 - Engine architecture
---

{{< lead >}}
A new game engine!
{{< /lead >}}


{{< alert "github" >}}
Check out Nui Engine on [Github](https://github.com/ArnavMehta3000/NuiEngine.git)
{{< /alert >}}

## Introducing Nui Engine

This game engine project is called **Nui Engine** and will be built using C++. Unlike most game engines, this will be an *editor-less* game engine. I want the user to control everything via C++ code. There will be a command console using which the user can communicate with the game engine (such as executing commands like game cheats and getting engine information).

## Engine Architecture

The engine will be split into the following modules:

- **NuiCore**
  - Built as a static library
  - Contains core engine functionality such as windowing, input and main loop management
  - Also contains the main entry point
  - Links with the NuiGraphics and NuiAudio libraries
- **NuiGraphics**
  - Built as a static library
  - Contains the DirectX 11 2D renderer
- **NuiAudio**
  - Built as a static library
  - Contains all audio related functionality
- **GameCode**
  - Built as an executable
  - Contains all user written game code which auto-registers with the engine and is dynamically called
  - Links with NuiCore
- **NuiTest**
  - Uses MSTest to test engine functionality (thereforce built as a DLL)
  - Can be extended by the user to test gameplay code
  - Links with all modules
- **ProjectGenerator**
  - A collection of templates and python scripts used to generate solution and project files

See below a diagram of the planned engine architecture.

{{< figure
    src="architecture.png"
    alt="Engine architecture"
    caption="Nui Engine Architecture"
    >}}

</br>

With the engine architecture planned, I will next work on generating project files. So that we can then start working on the engine itself.

*The name 'Nui' was proposed by my univerity lecturer (who I asked for name suggestions)  since this engine has no editor (No UI). Thanks Tom!*

<div style="display: flex; flex-wrap: wrap; gap: 10px;">
  {{< badge >}}Programming{{< /badge >}}
  {{< badge >}}Game Engine{{< /badge >}}
  {{< badge >}}Software Architecture{{< /badge >}}
</div>
