---
title: "Proximity Game Engine"
date: 2023-06-04T15:20:05+01:00
summary: "2D C++ and DirectX 11 game engine I made for my final year project"
---

For my final year project for my university I made a simple 2D game engine. For this project I research what goes into making a game engine and try to implement my learnings. Written in C++ 20, DirectX 11 and HLSL and based on and **ECS** and **event-driven architecture**. The editor is also written in C++ using [ImGui](https://github.com/ocornut/imgui). It also supports entity scripting using **Lua**.

{{< alert "github" >}}
Check out the project repository on [Github](https://github.com/ArnavMehta3000/Proximity2D).
{{< /alert >}}

Watch the demo video [here](#engine-features)

## Architecture

The project consists of two sub projects - **The Engine** and the **The Editor**. The engine compiles to a *static library* (.lib) and the editor compiles to an *executable* (.exe)

{{< figure
    src="architecture.png"
    alt="Proximity Engine Architecture"
    caption="Proximity Engine Architecture"
    >}}

Proximity's architecture is inspired from [Hazel](https://hazelengine.com/) and the initialization chain is inspired from [Unreal Engine](https://www.unrealengine.com/en-US).

{{< figure
    src="initchain.png"
    alt="Proximity Engine Initialization Chain"
    caption="Proximity Engine Initialization Chain"
    >}}

## Engine Features

{{< button href="https://www.youtube.com/watch?v=1LSTma-BbsM" target="_self" >}}
Watch the demo video
{{< /button >}}

Here are the key features of the engine, some of which are explained in detail after:

- Window and Input Handling
- Audio Engine using [XAudio2](https://learn.microsoft.com/en-us/windows/win32/xaudio2/xaudio2-introduction)
- Custom math library
- Event driven architecture using [Actions](#actions)
- Graphics engine and rendering using DirectX 11
- Filesystem and directory management
- Engine logging system
- Engine utilities
- Asset libraries (Audio, Shaders, Material, Textures, Scripts)
- Memory managers (Stack and Pool Allocators)
- Detailed [HLSL shader reflection](#hlsl-shader-reflection)
- Scripting using [Lua](#lua-scripting)
- Physics using [Box2D](https://box2d.org/)
- Game/Project Serialization using [Yaml-Cpp](https://github.com/jbeder/yaml-cpp)
- Multiple scenes and project management
- Entity Component System using [ENTT](https://github.com/skypjack/entt)

{{< figure
    src="editor.png"
    alt="Proximity Editor"
    caption="Proximity Editor"
    >}}

### HLSL Shader Reflection

One of the most impressive features of the game engine is the shader reflection system. The user can write any HLSL code (vertex or pixel shader). The shader is then compiled and reflected within in the engine. The user can then create a *Material* which is a pair of vertex and pixel shaders. This material is then reflected in the editor; constant buffers and input resource slots can be manually edited from within the editor

{{< figure
    src="shaderreflection.png"
    alt="Proximity Editor"
    caption="Shader reflection system"
    >}}

### Actions

I recreated the *System.Action<>* class from C# in C++ 20. And this is what drives the communicatation between the engine and the editor and between the different editor panels.

The user can create custom action events (which may or may not take arguments) and bind their own functions/lamdas to these actions. Whenever an action is called, all the bound functions to this action are also called.

### Lua Scripting

The engine supports entity scripting using **Lua** and the [**Sol2**](https://github.com/ThePhD/sol2/tree/main) library. The engine creates a custom runtime environment for each script in the game. The user can call functions to and from C++ and Lua and all of this is made easy using the Sol2 library. The scripts are written in the built in script editor which also has support for **syntax hightlighting** for engine specific functions.

## Conclusion

During this project, I learnt so many new things about C++, HLSL, DirectX 11 as a whole. I believe I have built this project to be the showcase of my skills a C++ and engine programmer and is definitely will be the highlight project of my portfolio along with being the **most complex project** I've worked on till date. With some changes to the project and scope, I will definitely make Proximity 2.0, which will be faster with even more supported features.

I did not talk about the intricate details about all the features of this project or this would turn into a 30 minute long article. But you can always check out the project repository on github or contact me :)

<div style="display: flex; flex-wrap: wrap; gap: 10px;">
  {{< badge >}}C++{{< /badge >}}
  {{< badge >}}DirectX 11{{< /badge >}}
  {{< badge >}}HLSL{{< /badge >}}
  {{< badge >}}Lua{{< /badge >}}
  {{< badge >}}ImGui{{< /badge >}}
  {{< badge >}}Programming{{< /badge >}}
  {{< badge >}}Game Engine{{< /badge >}}
  {{< badge >}}Backend Development{{< /badge >}}
</div>
