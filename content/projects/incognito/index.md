---
title: "Incognito"
date: 2023-02-04T19:25:39+01:00
description: "Project page for Incognito Game"
---

Incognito is a rogue-lite procedural stealth game made in **Unreal Engine 5** using **C++** and **Blueprints**. The player playes through a series of *rooms* (level). After finishing a level, they are brought back to the *hub* where they can choose weapon and stealth upgrades for their next run. Due to procedural level generation, each level/run is different.

{{< button href="https://youtu.be/9ibWrHwfzL4" target="_self" >}}
Game Trailer
{{< /button >}}

## My Role

This game was made  during a university **studio simulation** module where second and third year students from various fields in game development (design, art, sound, programming) get together to make a game in **6 weeks**. The seniors act like deparment leads and the juniors work along with and learn under their guidance.

The goal of this module was to make a game in 6 weeks. This game did not have be a complete game, but a vertical slice of a possible game with as many features added that time would allow.

I was the **tech lead** and **project manager** for a team of **22 students**. I made the core gameplay systems and the procedural algorithm that generates the map/levels of the game.

{{< figure
    src="thumb.jpg"
    alt="Incognito Promo Art"
    caption="Incogniito Promotional Art"
    >}}

## My Work

### As the Project Manager

As the project manager, my role was to make sure all the other seniors (deparment leads) were on track and make sure the juniors were completing the tasks assigned to them. This was done using **Jira** to make sure all work was done in time. I also was heavily invovled in deciding the scope of the project.

### As the Tech Lead

As the tech lead, my role was to design and implement the core gameplay systems and also build a framework easy enough so that the jkuniors can take over along with being designer friendly. Along with feature implementation, I also had to perform version control on the project and handle various department branches and merge conflicts. Most of the version control management was done using **command line Git** (since handling merge conflicts for UE5 blueprints is a pain).

{{< figure
    src="repo.png"
    alt="Managing the project via Git"
    caption="Managing the project via Git"
    >}}

## Procedural Level Generation

Incognito is a stealth game where the levels the user plays (internally referred to as *rooms*) are generated based on a stripped down version of the [Wave Function Collapse (WFC) Algorithm](https://github.com/mxgmn/WaveFunctionCollapse). All the code for the generation was written in C++ and was highly optimized (100 rooms generated in about 2 seconds).

{{< figure
    src="rooms.png"
    alt="Generating 100 rooms using WFC"
    caption="Generating 100 rooms using WFC (for fun)"
    >}}

The code was written in such a way that no designer ever had to touch any C++. All required functionality was exposed to the Unreal Editor.

I provided the designers with a list of templates for each type of room they had to make. All these rooms were then input into a custom data table (written in C++).

{{< figure
    src="layout.png"
    alt="Room layout template provided to designers"
    caption="Room layout template provided to designers"
    >}}

The rooms were divided into three types based on the number of openings they had:

- **C-Type Rooms:** Rooms with 1 opening
- **L-Type Rooms:** Rooms with 2 openings adjacent to each other
- **I-Type Rooms:** Rooms with 2 openings opposite to each other

Each type of room had 4 variants (for every type of rotation). All that was needed to be done was to fill in th neighbor and connections info in the data table. Provide the data table to the generator and generate the level when needed.

{{< figure
    src="deslayout.png"
    alt="Designer friendly room layout infographic"
    caption="Designer friendly room layout infographic"
    >}}

## Additional Work

### Enemy Ai

I also worked on other features such as developing the core (3 layer) communication system for all the Ai that was spawned in each room:

- **Ai-Ai communication:** One Ai enemy communicating with other nearby Ai enemies
- **Ai-Room communication:** Ai enemies communicating with the room to ambush the player in [lockdown mode](#lockdown-mode)
- **Room-Level communication:** Each room communicated with a game manager gathering telemetry data such as number of enemies killed and how stealthily the player completed a room. This was also used to check if a certain level was finished

I also worked on building a base for all Ai enemies, which was later extended by other senior programmers.

### Lockdown Mode

One of the most fun features of this game is the **Lockdown Mode** I helped build with the other senior programmer on the team. It was like a panic mode in the game. If during a player's 'escape' from a room was caught by the Ai. All the enemies in the room would be made aware of the players location and they would collectively attack the player. During this phase, the player no longer has to complete the level stealthily. They have to kill as many enemies, find the key to the exit and leave the room. This intermediate game mode totally changed the way the game is played.

If the player managed to find the key and exit the room without killing all the enemies in the room. The next entered room would begin on lockdown mode (keeping in mind player has limited ammo and health)

### Dynamic Enemy Wander

One of features I worked on a dynamic enemy wander system. Since all the rooms were pre-built by designers and the generator only placed the room in the game world. If the enemy roaming spots were already pre-defined, the player could simply remember them and easily defeat a room.

To counter this, I made use of th Unreal's **Environmnet Query System (EQS)** to get a random list of wander points based on multiple criterias. These wander points would be randomized and shuffled for each enemy in each room.

{{< alert "github" >}}
Check out the project repository on [Github](https://github.com/ArnavMehta3000/Collab-2023)
{{< /alert >}}

</br>

<div style="display: flex; flex-wrap: wrap; gap: 10px;">
  {{< badge >}}Game{{< /badge >}}
  {{< badge >}}Unreal Engine{{< /badge >}}
  {{< badge >}}C++{{< /badge >}}
  {{< badge >}}Ai{{< /badge >}}
  {{< badge >}}Blueprints{{< /badge >}}
  {{< badge >}}Team Work{{< /badge >}}
  {{< badge >}}Programming{{< /badge >}}
  {{< badge >}}Management{{< /badge >}}
  {{< badge >}}Design{{< /badge >}}
  {{< badge >}}Game Dev{{< /badge >}}
</div>
