---
title: "Against Asimov [Ai and Games Jam 2021]"
date: 2021-06-05T23:26:37+01:00
description: "Submission for AI and Games Jam 2021"
---

This game was made for the Ai and Games Jame 2021. The main goal of the project was to improve my AI development and design skills.

The theme of this game jam was **Breaking The Rules**. So in this game the robots broke the [Three Laws of Robotics](https://en.wikipedia.org/wiki/Three_Laws_of_Robotics) made by Issac Assimov (hence the name).

In this game you play as a robot, where you break Asimov's 3 rules/laws of robotics by fighting against intelligent AI.

The Ai will randomly patrol, and if caught will shoot on sight. But also go and take cover and call for backup (creates copy of itself) when its own health is low

## How the AI Works

This was the first time I made my own Finite State Machine (FSM) framework in Unity.

- By default the enemy robots will wander around
- If the player comes in the field of vision, they will switch to chase state and follow the player
- Once close enough the enemy will start shooting the player
- If the enemy itself takes enough damage, it will try to take cover by finding the nearest safe cover spot
- If it stays safe for some time without the player noticing, backup will be called (3 enemies are spawned)
- The main goal of the player should be to prevent the enemy robot from going to safety and calling for backup whilst trying to complete the game in the quickest time possible

## Images

Here are some screenshots from the game:

{{< figure
    src="asimov1.jpg"
    alt="Against Asimov Gameplay Image 1"
    caption="Against Asimov Gameplay Image 1"
    >}}

</br>

{{< figure
    src="asimov2.jpg"
    alt="Against Asimov Gameplay Image 2"
    caption="Against Asimov Gameplay Image 2"
    >}}

</br>

{{< figure
    src="asimov3.jpg"
    alt="Against Asimov Gameplay Image 3"
    caption="Against Asimov Gameplay Image 3"
    >}}

</br>

{{< alert "itch.io" >}}
Download and play the game on [Itch.io](https://nexus-of-gaming.itch.io/against-asimov)
{{< /alert >}}

</br>

<div style="display: flex; flex-wrap: wrap; gap: 10px;">
  {{< badge >}}C#{{< /badge >}}
  {{< badge >}}Ai{{< /badge >}}
  {{< badge >}}Unity{{< /badge >}}
  {{< badge >}}Game Jam{{< /badge >}}
  {{< badge >}}AI and Games Jam 2021{{< /badge >}}
  {{< badge >}}Game Development{{< /badge >}}
</div>
