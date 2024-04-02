---
title: "MAGE - Part 5.1"
date: 2024-03-20T11:57:05Z
summary: "Making a game engine - Part 5.1 - Input System Setup"
---

{{< lead >}}
Finding the keys (and clicks) to success!
{{< /lead >}}

{{< alert "github" >}}
Check out Nui Engine on [Github](https://github.com/ArnavMehta3000/NuiEngine.git)
{{< /alert >}}

In the [last post](https://arnavmehta3000.github.io/posts/mage4/) I set up the creation of the Win32 window and the message pump. Building up from that I will now go over the base of making an input system we'll look at processing them in the next post.

## Types of Input Systems

Input systems can be broadly classified into two types:
- Polling based input system
- Event based input system

### Polling Based

In a polling based input system first all the Win32 input messages are processed (and cached as *key states*). These key states can be queried later in the frame during the update loop and can be used by the user.

{{< figure
    src="PollingInput.png"
    alt="Polling based input system"
    caption="Polling based input system"
    >}}

### Event Based

In an event based input system. A callback is dispatched (usually a function overriden or bound to a delagte by the user) as soon as the Win32 input message is received. This may be configured to execute user code as soon as the input is recieved or may be used via an **event queue**

{{< figure
    src="EventInput.png"
    alt="Event based input system"
    caption="Event based input system"
    >}}

#### Input Event Queue

One variation of the event based input system is using the event queue. In this version, we receive the Win32 input message, process it and add it to an event queue. This queue can be queried every frame to check for the latest input event and if it is the expected type of event (let's say window resize) then we can process our/user code as needed

{{< figure
    src="EventQueueInput.png"
    alt="Event queue based input system"
    caption="Event queue based input system"
    >}}

## Input API Design

For this game enigne, I want the input API to be very similar to the way Unity has it set up (in thier old input system). Which would work best designed as a polling based input system.

Here is the psuedo-code for the Input API

```cpp
if (Input::GetKey(Key::Escape).IsPressed)
{
    // Escape key was pressed this frame
}
```

## Keyboard

Windows provides a way to query the state keyboard keys using the [Virtual Keys](https://learn.microsoft.com/en-us/windows/win32/inputdev/virtual-key-codes) and the [GetKeyState function](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-getkeystate). But this only tells us if the key is up or down, it doesn't tell us if the key is being held. Not only that, but we would also have to use Win32 *defines* in our code which is something I do not want.

### Key Codes

To solve the above issue I made a simple enum class called `KeyCode` which maps the the virtual key codes

> This enum is quite massive, so I've shortened the the code for the post, but the entire enum can be viewed [here](https://github.com/ArnavMehta3000/NuiEngine/blob/main/Engine/Core/Input/KeyCode.h)

```cpp
namespace Nui::Input
{
    enum class KeyCode : U64
    {
        None = 0,
        LeftArrow      = VK_LEFT,
		RightArrow     = VK_RIGHT,
		UpArrow        = VK_UP,
		DownArrow      = VK_DOWN,
		PageUp         = VK_PRIOR,
		PageDown       = VK_NEXT,
		Home           = VK_HOME,
		End            = VK_END,
		Insert         = VK_INSERT
        .
        .
        .
        NumPad7        = VK_NUMPAD7,
		NumPad8        = VK_NUMPAD8,
		NumPad9        = VK_NUMPAD9,
		NumPadDecimal  = VK_DECIMAL,
		NumPadDivide   = VK_DIVIDE,
		NumPadMultiply = VK_MULTIPLY,
		NumPadSubtract = VK_SUBTRACT,
		NumPadAdd      = VK_ADD,
    }
}
```

I also have a constexpr function that converts these keycodes to array indices (this will make sense later)

```cpp
inline constexpr U64 ConvertKeyCodeToArrayIndex(KeyCode keyCode)
{
    switch (keyCode)
    {
    case KeyCode::LeftArrow:      return 0;
    case KeyCode::RightArrow:     return 1;
    case KeyCode::UpArrow:        return 2;
    case KeyCode::DownArrow:      return 3;
    case KeyCode::PageUp:         return 4;
    case KeyCode::PageDown:       return 5;
    case KeyCode::Home:           return 6;
    case KeyCode::End:            return 7;
    .
    .
    .
    case KeyCode::NumPad8:        return 106;
    case KeyCode::NumPad9:        return 107;
    case KeyCode::NumPadDecimal:  return 108;
    case KeyCode::NumPadDivide:   return 109;
    case KeyCode::NumPadMultiply: return 110;
    case KeyCode::NumPadSubtract: return 111;
    case KeyCode::NumPadAdd:      return 112;
    case KeyCode::KEYCODE_COUNT:  return 113;
    }

    // Return out of bounds index
    return ConvertKeyCodeToArrayIndex(KeyCode::KEYCODE_COUNT) + 1;
}
```

### Key Modifiers

When pressing any key, the user can also combine modifiers with the key (like control, shift, etc.). For this, I decided to use an enum so it can be merged using bitmasks

```cpp
enum Modifier : U64
{
    MOD_None = 0,

    MOD_LShift   = 1 << 0,
    MOD_RShift   = 1 << 1,
    MOD_LControl = 1 << 2,
    MOD_RControl = 1 << 3,
    MOD_LAlt     = 1 << 4,
    MOD_RAlt     = 1 << 5,
    MOD_LSuper   = 1 << 6,
    MOD_RSuper   = 1 << 7,
};
```

### The Keyboard Structure

I use the following structure to represent the keyboard.

```cpp
struct Keyboard
{
    constexpr Keyboard();

    struct KeyState
    {
        constexpr KeyState(KeyCode key = KeyCode::None)
            : Key(key)
            , Modifier(Modifier::MOD_None)
            , IsPressed(false)
            , IsReleased(false)
            , IsHeld(false)
        {}

        KeyCode Key;
        Modifier Modifier;
        bool IsPressed;
        bool IsReleased;
        bool IsHeld;
    };

    std::array<KeyState, ConvertKeyCodeToArrayIndex(KeyCode::KEYCODE_COUNT)> KeyStates;
};
```

The keyboard is a collection of `KeyStates`, where every `KeyCode` is mapped to an index using the `ConvertKeyCodeToArrayIndex` function. The reason for this is becuase the enum value of `KeyCode::KEYCODE_COUNT` is not equal to the number of number of actual keys in the `KeyCode` enum

- I have excluded some keys from the `KeyCode` enum (like Print Screen, etc.)
- This causes a mismatch in the indexing of the array if you directly try to map the key code
- The valie of `KeyCode::KEYCODE_COUNT` is **108**, but when mapped using the function, the size of the `Keyboard::KeyStates` is **113**.

## Mouse

### The Mouse Structure

Here is the structure that I am using to represent the mouse state.

```cpp
struct Mouse
{
    constexpr Mouse();

    enum class Button : U32
    {
        None = 0,

        Left    = MK_LBUTTON,   // Can be referenced using 0
        Right   = MK_RBUTTON,   // Can be referenced using 1
        Middle  = MK_MBUTTON,   // Can be referenced using 2
        MouseX1 = MK_XBUTTON1,  // Can be referenced using 3
        MouseX2 = MK_XBUTTON2,  // Can be referenced using 4
    };

    static constexpr U32 ConvertMouseButtonToArrayIndex(Mouse::Button btn)
    {
        switch (btn)
        {
        case Mouse::Button::Left:    return 0;
        case Mouse::Button::Right:   return 1;
        case Mouse::Button::Middle:  return 2;
        case Mouse::Button::MouseX1: return 3;
        case Mouse::Button::MouseX2: return 4;
        }

        // Return out of array bounds index
        return 5;
    }

    struct Point
    {
        constexpr Point(I32 x = 0, I32 y = 0) : X(x), Y(y) {}

        I32 X{ 0 };
        I32 Y{ 0 };
    };

    struct WheelInfo
    {
        constexpr WheelInfo()
            : Delta(0)
            , Modifier(Modifier::MOD_None)
            , Position()
        {}

        I32 Delta;
        Modifier Modifier;
        Point Position;
    };

    struct ButtonState
    {
        constexpr ButtonState(Mouse::Button btn = Mouse::Button::None)
            : Btn(btn)
            , Modifier(Modifier::MOD_None)
            , IsHeld(false)
            , IsPressed(false)
            , IsReleased(false)
        {}

        Button Btn;
        Modifier Modifier;
        bool IsHeld;
        bool IsPressed;
        bool IsReleased;
    };

    Point     Position;
    Point     RawDelta;
    WheelInfo WheelH;
    WheelInfo WheelV;
    std::array<ButtonState, 5> ButtonStates;
};
```

This structure contains an enum for the different mouse buttons (and a constexpr function to convert the enum class to array index), a simple structure to contains the mouse coordinates (or the raw delta), a structure to contains the mouse wheel information and a structure for the mouse button states (similar to `Keyboard::KeyState`).

In the structure we have the following member variables:

- **Position** - Contains the pointer position relative to the client
- **RawDelta** - Contains the raw delta movement (using raw input)
- **WheelH** - The horizontal wheel info
- **WheelV** - The vertical wheel info
- **ButtonStates** - Similar to the KeyStates from the keyboard structure, contains information about the mouse button states

## Conclusion

In the next post I will go over processing the Win32 input messages using a custom input window procedure.
