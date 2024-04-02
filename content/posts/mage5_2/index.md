---
title: "MAGE - Part 5.2"
date: 2024-03-31T22:44:33+01:00
summary: "Making a game engine - Part 5.2 - Input Processing"
---

{{< lead >}}
Capturing the keys (and clicks) to success!
{{< /lead >}}

{{< alert "github" >}}
Check out Nui Engine on [Github](https://github.com/ArnavMehta3000/NuiEngine.git)
{{< /alert >}}

In the [previous post](https://arnavmehta3000.github.io/posts/mage5_1/), I went over setting up the base for the input system. In this one we'll see how I implemented processing Win32 input events.

## Implementing the Input API

I made a new file called `Input.h` and made the API functions for the input system

```cpp
namespace Nui
{
	namespace Input
	{
		namespace Internal
		{
			[[nodiscard]] bool ProcessInputWndProc(HWND hWnd, UINT uMsg, WPARAM wParam, LPARAM lParam);
			void Update();
		}

		[[nodiscard]] const Mouse::Point& GetMousePosition();
		[[nodiscard]] const Mouse::Point& GetMouseRawDelta();
		[[nodiscard]] const Mouse::WheelInfo& GetMouseWheelH();
		[[nodiscard]] const Mouse::WheelInfo& GetMouseWheelV();
		[[nodiscard]] const Mouse::ButtonState& GetMouseButton(Mouse::Button btn);
		[[nodiscard]] const Mouse::ButtonState& GetMouseButton(U32 btn);

		[[nodiscard]] const Keyboard::KeyState& GetKeyState(KeyCode key);
	}
}
```

In the cpp file I have the Keyboard and Mouse as static unique pointers along with a bunch of helper functions (that I'll cover later)

```cpp
// In Input.cpp
static bool s_initialized{ false };
static std::unique_ptr<Mouse> s_mouse{ nullptr };
static std::unique_ptr<Keyboard> s_keyboard{ nullptr };
```

### Constructors

From the Keyboard and Mouse structures made in the last post, each had a constexpr constructor, which we can use to initialize thier internal state.

Starting with the simple mouse constructor:

```cpp
constexpr Mouse::Mouse()
    : Position()
    , RawDelta()
    , WheelH()
    , WheelV()
{
    ButtonStates[0] = ButtonState(Mouse::Button::Left);
    ButtonStates[1] = ButtonState(Mouse::Button::Right);
    ButtonStates[2] = ButtonState(Mouse::Button::Middle);
    ButtonStates[3] = ButtonState(Mouse::Button::MouseX1);
    ButtonStates[4] = ButtonState(Mouse::Button::MouseX2);
}
```

Followed by an annoyingly large Keyboard constructor (shortened for this post):

```cpp
constexpr Keyboard::Keyboard()
{
    KeyStates[ConvertKeyCodeToArrayIndex(KeyCode::LeftArrow)]      = KeyState(KeyCode::LeftArrow);
    KeyStates[ConvertKeyCodeToArrayIndex(KeyCode::RightArrow)]     = KeyState(KeyCode::RightArrow);
    KeyStates[ConvertKeyCodeToArrayIndex(KeyCode::UpArrow)]        = KeyState(KeyCode::UpArrow);
    .
    .
    .
    KeyStates[ConvertKeyCodeToArrayIndex(KeyCode::NumPadMultiply)] = KeyState(KeyCode::NumPadMultiply);
    KeyStates[ConvertKeyCodeToArrayIndex(KeyCode::NumPadSubtract)] = KeyState(KeyCode::NumPadSubtract);
    KeyStates[ConvertKeyCodeToArrayIndex(KeyCode::NumPadAdd)]      = KeyState(KeyCode::NumPadAdd);
}
```


### Input Window Procedure

The first thing to do in the input system is to capture the Win32 messages. And since I am using a namespace, it was quite easy to hook into the the window procedure I made in the last post. 

```cpp
LRESULT Window::MessageHandler(HWND hWnd, UINT uMsg, WPARAM wParam, LPARAM lParam)
{
    //  Existing code

    if (Input::Internal::ProcessInputWndProc(hWnd, uMsg, wParam, lParam))
    {
        // Message was processed by input system
        return 0;
    }

    // Return default window procedure
}
```

#### Initializing the Input System

Using the static bool declared above we can initialize the input system on the first function call (like a do-once function). So inside the input window procedure, I create the mouse and keyboard objects and register raw mouse input using [MSDN as reference](https://learn.microsoft.com/en-us/windows/win32/inputdev/using-raw-input).

```cpp
// If input system has not been initialized, register raw input devices but don't process input
[[unlikely]]
if (!s_initialized)
{
    s_mouse    = std::make_unique<Mouse>();
    s_keyboard = std::make_unique<Keyboard>();

    // Register for raw input
    RAWINPUTDEVICE Rid[1];
    Rid[0].usUsagePage = HID_USAGE_PAGE_GENERIC;
    Rid[0].usUsage     = HID_USAGE_GENERIC_MOUSE;
    Rid[0].dwFlags     = RIDEV_INPUTSINK;
    Rid[0].hwndTarget  = hWnd;

    if (!RegisterRawInputDevices(Rid, 1, sizeof(Rid[0])))
    {
        // Log error 
    }
    
    // Consider input initialized even if registration failed (it's not fatal)
    s_initialized = true;
    return false;
}
```

### Mouse Input

#### Capturing Raw Input

After registering the raw mouse device, we can use the `WM_INPUT` to get the raw mouse data and store it in our mouse structure.

```cpp
case WM_INPUT:
{
    U32 dwSize = sizeof(RAWINPUT);
    static BYTE lpb[sizeof(RAWINPUT)];

    ::GetRawInputData((HRAWINPUT)lParam, RID_INPUT, lpb, &dwSize, sizeof(RAWINPUTHEADER));

    RAWINPUT* raw = (RAWINPUT*)lpb;

    [[likely]]
    if (raw && raw->header.dwType == RIM_TYPEMOUSE)
    {
        NUI_ASSERT((bool)s_mouse, "Mouse is not initialized");
        [[likely]]
        if (s_mouse)
        {
            s_mouse->RawDelta.X = raw->data.mouse.lLastX;
            s_mouse->RawDelta.Y = raw->data.mouse.lLastY;
        }
    }
    return true;
}
```
#### Capturing Mouse Position

This one is quite easy where where we can simply capture the `WM_MOUSEMOVE` message to get the mouse position relative to the client window.

```cpp
case WM_MOUSEMOVE:
{
    if (s_mouse)
    {
        POINTS p = MAKEPOINTS(lParam);
        s_mouse->Position.X = p.x;
        s_mouse->Position.Y = p.y;
    }
    return true;
}
```

#### Capturing Mouse Wheel

Using the `WM_MOUSEWHEEL` and `WM_MOUSEHWHEEL` messages along with the `GET_WHEEL_DELTA_WPARAM()` to get the wheel movement direction/delta. I also have a helper function called `GetModifiers()` to get the modifier keys that were pressed when the mouse wheel message is received.

```cpp
case WM_MOUSEWHEEL:
{
    if (s_mouse)
    {
        s_mouse->WheelV.Delta = GET_WHEEL_DELTA_WPARAM(wParam);
        s_mouse->WheelV.Modifier = GetModifiers();

        POINTS p = MAKEPOINTS(lParam);
        s_mouse->WheelV.Position.X = p.x;
        s_mouse->WheelV.Position.Y = p.y;
    }
    return true;
}

case WM_MOUSEHWHEEL:
{
    if (s_mouse)
    {
        s_mouse->WheelH.Delta = GET_WHEEL_DELTA_WPARAM(wParam);
        s_mouse->WheelH.Modifier = GetModifiers();

        POINTS p = MAKEPOINTS(lParam);
        s_mouse->WheelH.Position.X = p.x;
        s_mouse->WheelH.Position.Y = p.y;
    }
    return true;
}
```

#### Capturing Mouse Button States

One of the helper functions I made was to process mouse button states, here is the function

```cpp
bool ProcessMouseButton(Mouse::Button btn, bool pressed, LPARAM lParam)
{
    if (s_mouse)
    {
        POINTS p = MAKEPOINTS(lParam);
        s_mouse->Position.X = p.x;
        s_mouse->Position.Y = p.y;

        Mouse::ButtonState& state = s_mouse->ButtonStates[Mouse::ConvertMouseButtonToArrayIndex(btn)];
        NUI_ASSERT(state.Btn == btn, "Input button and mouse state button mismatch!");

        state.Modifier = GetModifiers();

        if (pressed)
        {
            if (!state.IsHeld)
            {
                state.IsPressed  = true;
                state.IsHeld     = true;
                state.IsReleased = false;
            }
            else
            {
                // Button was not pressed this frame
                state.IsPressed = false;  // Set pressed to false if the button is already held
            }
        }
        else
        {
            state.IsPressed  = false;
            state.IsHeld     = false;
            state.IsReleased = true;
        }

        return true;
    }

    return false;
}
```

Here I first get the mouse position whenever the mouse button is pressed. If the button was pressed this frame then I set the pressed and held value to true and released to false provided the button is not being held (pressed in a previous frame). If the button is not being pressed then I set the pressed and held to false and released to true.

> I will be using the same logic for keyboard input to set key states

After that we can use this function in the input window procedure:

```cpp
case WM_LBUTTONDOWN:
    return ProcessMouseButton(Mouse::Button::Left, true, lParam);

case WM_MBUTTONDOWN:
    return ProcessMouseButton(Mouse::Button::Middle, true, lParam);

case WM_RBUTTONDOWN:
    return ProcessMouseButton(Mouse::Button::Right, true, lParam);

case WM_LBUTTONUP:
    return ProcessMouseButton(Mouse::Button::Left, false, lParam);

case WM_MBUTTONUP:
    return ProcessMouseButton(Mouse::Button::Middle, false, lParam);

case WM_RBUTTONUP:
    return ProcessMouseButton(Mouse::Button::Right, false, lParam);

case WM_XBUTTONDOWN:
{
    Mouse::Button btn = (HIWORD(wParam) == XBUTTON1) ? Mouse::Button::MouseX1 : Mouse::Button::MouseX2;
    return ProcessMouseButton(btn, true, lParam);
}

case WM_XBUTTONUP:
{
    Mouse::Button btn = (HIWORD(wParam) == XBUTTON1) ? Mouse::Button::MouseX1 : Mouse::Button::MouseX2;
    return ProcessMouseButton(btn, false, lParam);
}
```

### Keyboard Input

#### Capturing Key States

Windows only gives two top level types of key state information, whether the key was pressed this frame (key down) or the key was released this frame (key up). Each of these messages are of two types: *keyboard keys* and *system keys*. We can capture them using `WM_SYSKEYDOWN`, `WM_KEYDOWN`, `WM_SYSKEYUP` and `WM_KEYUP` messages.

For processing key down messages:

```cpp
case WM_SYSKEYDOWN: [[fallthrough]];
case WM_KEYDOWN:
{
    NUI_ASSERT((bool)s_keyboard, "Keyboard is not initialized");
    
    [[likely]]
    if ((U64)wParam < 256)
    {
        // Process some modifiers early
        if (ProcessKeyboard(wParam, true))
            return true;

        Keyboard::KeyState& state = s_keyboard->KeyStates[ConvertKeyCodeToArrayIndex((KeyCode)wParam)];
        NUI_ASSERT(state.Key == KeyCode(wParam), "WPARAM is not a KeyCode");

        if (!state.IsHeld)
        {
            state.IsPressed  = true;
            state.IsHeld     = true;
            state.IsReleased = false;
        }
        else
        {
            state.IsPressed = false;  // Set pressed to false if the key is already held
        }
        state.Modifier = GetModifiers();
        return true;
    }
    return false;
}
```

Similarly for key up messages

```cpp
case WM_SYSKEYUP: [[fallthrough]];
case WM_KEYUP:
{
    NUI_ASSERT((bool)s_keyboard, "Keyboard is not initialized");

    [[likely]]
    if ((U64)wParam < 256)
    {
        // Process some modifiers early
        if (ProcessKeyboard(wParam, true))
            return true;

        Keyboard::KeyState& state = s_keyboard->KeyStates[ConvertKeyCodeToArrayIndex(KeyCode(wParam))];
        NUI_ASSERT(state.Key == KeyCode(wParam), "WPARAM is not a KeyCode");

        state.IsPressed  = false;
        state.IsHeld     = false;
        state.IsReleased = true;
        return true;
    }
    return false;
}
```

The above two code snippets are very similar to the mouse button state code. So I will not go in depth about explaining them. At the same time you must have noticed a function called `ProcessKeyboard`, this function is used to process some modifier keys early.

#### Processing Keyboard Modifiers

So this is weird function that does almost the same thing as the above code, but with a slight difference, it calculates which side of the modifier key was pressed (left or right).

Here is complete function body, with explanations below:

```cpp
bool ProcessKeyboard(U64 vk, bool pressed)
{
    bool handled  = false;
    auto doVKDown = [&](U64 vk)
    {
        Keyboard::KeyState& state = s_keyboard->KeyStates[ConvertKeyCodeToArrayIndex((KeyCode)VK_LSHIFT)];

        if (!state.IsHeld)
        {
            state.IsPressed  = true;
            state.IsHeld     = true;
            state.IsReleased = false;
        }
        else
        {
            state.IsPressed = false;  // Set pressed to false if the key is already held
        }
        state.Modifier = GetModifiers();
    };

    auto doVKUp = [&](U64 vk)
    {
        Keyboard::KeyState& state = s_keyboard->KeyStates[ConvertKeyCodeToArrayIndex((KeyCode)VK_LSHIFT)];

        state.IsPressed  = false;
        state.IsHeld     = false;
        state.IsReleased = true;
        state.Modifier   = GetModifiers();
    };

    if (vk == VK_SHIFT)
    {
        if (IsVKPressed(VK_LSHIFT) == pressed) doVKDown(VK_LSHIFT);
        else doVKUp(VK_LSHIFT);
        if (IsVKPressed(VK_RSHIFT) == pressed) doVKDown(VK_RSHIFT);
        else doVKUp(VK_RSHIFT);
        handled = true;
    }
    if (vk == VK_CONTROL)
    {
        if (IsVKPressed(VK_LCONTROL) == pressed) doVKDown(VK_LCONTROL);
        else doVKUp(VK_LCONTROL);
        if (IsVKPressed(VK_RCONTROL) == pressed) doVKDown(VK_RCONTROL);
        else doVKUp(VK_RCONTROL);
        handled = true;
    }
    if (vk == VK_MENU)
    {
        if (IsVKPressed(VK_LMENU) == pressed) doVKDown(VK_LMENU);
        else doVKUp(VK_LMENU);
        if (IsVKPressed(VK_RMENU) == pressed) doVKDown(VK_RMENU);
        else doVKUp(VK_RMENU);
        handled = true;
    }
    return handled;
}
```

Here there are two lambdas called `doVKUp` and `doVKDown`. These lambdas perform the exact same logic as earlier to capture the key/mouse states. But the main job of the function of this function is to decide what key to call those functions for. That is decided by the if-chain below the lambdas.

Within each if statement, I first check for the *'top-level'* key code regarding what modifier key was pressed. Then using a helper function called `IsVkPressed` (explained later), I check if the key is being pressed or not, which decides which lambda is to be called for which side of the modifier key.

### Helper Functions

In this post you would have seen two functions that I refer to as *'Helper Functions'*. Here is what they are and what they do:

#### Is Virtual Key Pressed

The first helper function is `IVKPressed`, this function is simply a wrapper around the Win32 function `GetKeyState` and checks if a virtual key is up or down.

```cpp
static bool IsVKPressed(I32 vk)
{
    return (::GetKeyState(vk) & 0x8000) != 0;
}
```
#### Getting Key Modifiers

The other helper function that I use is the `GetModifiers` function. This function returns a `Modifier` enum. It makes use of the above `IsVKPressed` function to check for modifier keys

```cpp
static Modifier GetModifiers()
{
    U64 m = 0;  // Modifier::None
    if (IsVKPressed(VK_LSHIFT))   
        m |= Modifier::MOD_LShift;
    if (IsVKPressed(VK_RSHIFT))   
        m |= Modifier::MOD_RShift;
    if (IsVKPressed(VK_LCONTROL)) 
        m |= Modifier::MOD_LControl;
    if (IsVKPressed(VK_RCONTROL)) 
        m |= Modifier::MOD_RControl;
    if (IsVKPressed(VK_LMENU))    
        m |= Modifier::MOD_LAlt;
    if (IsVKPressed(VK_RMENU))    
        m |= Modifier::MOD_RAlt;
    if (IsVKPressed(VK_LWIN))     
        m |= Modifier::MOD_LSuper;
    if (IsVKPressed(VK_RWIN))     
        m |= Modifier::MOD_RSuper;

    return static_cast<Modifier>(m);
}
```

### Updating The Input System

Although input messages will be recieved by the window thanks to the window message pump. But we want to manually be able to reset the internal input state of the entire keyboard and mouse (for example when the window loses focus).

#### Reset Input State

Here is the function that resets the internal input state of the keyboard and mouse by setting all members to their default values. It also takes a boolean in as an argument to check if we want to reset the held state of the keys/buttons as well (explained later).

```cpp
void Reset(bool resetHeld = false)
{
    [[likely]]
    if (s_keyboard)  // Reset keyboard
    {
        for (auto& state: s_keyboard->KeyStates)
        {
            state.IsPressed  = false;
            state.IsReleased = false;
            if (resetHeld)
                state.IsHeld = false;
        }
    }

    [[likely]]
    if (s_mouse) // Reset mouse
    {
        for (auto& state: s_mouse->ButtonStates)
        {
            state.IsPressed  = false;
            state.IsReleased = false;
            if (resetHeld)
                state.IsHeld = false;
        }
        s_mouse->WheelH = Mouse::WheelInfo();
        s_mouse->WheelV = Mouse::WheelInfo();
    }
```

#### The Update function

This is a very simple update function that we call every frame that simply calls the above `Reset`. Here we do not want to reset the 'held-state' of the keys/buttons since we yet do not know if they have been released or not.

```cpp
void Internal::Update()
{
    Reset(false);
}
```

#### On Window Focus Change

Whenever our window loses focus, we want to reset the keyboard and mouse state. We can do this using the Win32 function called `GetFocus` which returns a window handle. We can compare this handle with our window handle and check if the window that has focus is our window (else tour window is not being focused). That can be checked for in the input window procedure function after the initialization code and before the the actual input processing takes place. And if we are going to reset here, then we we want to reset the held state of the keys/buttons as well.

```cpp
bool Internal::ProcessInputWndProc(HWND hWnd, UINT uMsg, WPARAM wParam, LPARAM lParam)
{
    // Initialization code...

    // Check for window focus
    [[unlikely]]
    if (::GetFocus() != hWnd)
    {
        Reset(true);
        return false;
    }

    // Switch statement on uMsg to capture input
}
```

## Conclusion

One thing I have not included in these posts is the function bodies of the Input getter functions since they are quite straight forward. For keyboard, you convert the input key to an array index and return the state of the key. Similarly, do the same with the mouse getter functions.

This input system can now be used as follows:

```cpp
Input::Keyboard::KeyState& state = Input::GetKeyState(Input::KeyCode::Escape))
if (state.IsPressed)
{
    // Escape key was pressed this frame do something...
}
```

Making an input system from scratch is quite a tedious task, not in complexity, but in lines of code. First you have to declare all the keys and buttons you want to support and create a mapping for each one of them. Then actually process the them. I'll agree that my implementation may not be the best (especially with mapping keycodes to array indices). But I always wanted to understand how input systems are writen and wanted to write one myself from scratch. So here we are.

