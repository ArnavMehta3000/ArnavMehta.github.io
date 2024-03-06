---
title: "Goal Oriented Action Planning in Unity"
date: 2023-05-07T21:40:05+01:00
summary: "GOAP in Unity"
---

I have always wanted to try making a complex Ai system before (better than a FSM). And in that list, there were two main ones. GOAP and Neural Networks, specifically NEAT and Genetic Algorithms. Although my original plan for this module was to make a platformer (similar) to Super Mario Bros. and train a NEAT algorithm on it. Due to the scope and time constraints, I decide to drop that idea and move on to make GOAP in a survival game.

## Understanding GOAP

GOAP stands for **Goal Oriented Action Planning**.

The way GOAP works is takes in a starting world state and a goal (target world state). These two, along with a list of all possible actions are sent into the planner. The planner generates a list of actions to take by performing A* search in the actions list based on the preconditions and effect of the provided actions.

## How I Deal With GOAP

In my implementation of GOAP in Unity. I do not have any classes that define a goal. Everything is defined as a world state.

{{< figure
    src="files.png"
    alt="C# files for implementing GOAP in Unity"
    caption="C# files for implementing GOAP in Unity"
    >}}

Which means the planner takes in the current world state and the target world state (and the actions list).

The actions are built by setting preconditions and effects on the world state and by providing a delegate that is executed when an action is executed.

Here is a snippet of code from my implementation
~~~c# 
// Create a new action called move to drinking point and give it an execution cost
GOAP.Action moveToDrinkingPoint = new("Move to drinking point", MOVEMENT_COST);

// Precondition world state for this action to be executed is that 'Player Is Thristy = true'
moveToDrinkingPoint.SetPrecondition((int)WorldKey.PlayerIsThristy, true);

// The effect of the action on the world state on execution is 'Play Is Going To Drink = true'
moveToDrinkingPoint.SetEffect((int)WorldKey.PlayerIsGoingToDrink, true);

// Set up the function that is called when the action is executed and the function that checks if this action can execute on the current world state (runtime check)
moveToDrinkingPoint.SetupExecution(
() =>  // Execution
{
    m_player.MoveAStar(m_drinkingPoint.transform.position);
},
() => // Checker
{
    // Returns true id the player has reached the drinking point
    return m_player.HasReachedDestination;
})
~~~

Like that multiple actions can be set up. The planner then chains them up based on matching preconditions and effects (using A*).

## Action/Goal Chart

Here is a chart showing all the actions and their desired preconditions and effects

{{< figure
    src="table.png"
    alt="Action/Goal relationship chart"
    caption="Action/Goal relationship chart"
    >}}

Along with GOAP, this project has a comprehensive 2D tilemap terrain analysis system (visualization shown in the cover image)

{{< alert "youtube" >}}
Check out the demo and walkthrough on [YouTube](https://www.youtube.com/watch?v=5OXMFjuQ3_c)
{{< /alert >}}

</br>

{{< alert "github" >}}
Check out the project repository on [Github](https://github.com/ArnavMehta3000/2D-Survival-Ai)
{{< /alert >}}

</br>

<div style="display: flex; flex-wrap: wrap; gap: 10px;">
  {{< badge >}}C#{{< /badge >}}
  {{< badge >}}Ai{{< /badge >}}
  {{< badge >}}Unity{{< /badge >}}
  {{< badge >}}Game Development{{< /badge >}}
</div>
