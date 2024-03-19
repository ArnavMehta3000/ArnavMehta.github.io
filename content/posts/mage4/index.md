---
title: "MAGE - Part 4"
date: 2024-03-19T19:59:57Z
summary: "Making a game engine - Part 4 - Windowing"
---

{{< lead >}}
Windows made by Windows!
{{< /lead >}}

{{< alert "github" >}}
Check out Nui Engine on [Github](https://github.com/ArnavMehta3000/NuiEngine.git)
{{< /alert >}}

Now that I have a logging and assertion framework up and running. The next thing I want to add is an application window. For this I will be using the Win32 API and no external libraries. Here is what I want my window class to support:
- Simple styling
- Custom title text
- Self contained message pump
- Message routing (with callbacks)

## Window Styling

 I will be supporting 4 different window styles

- Windowed
- Windowed Fullscreem
- Borderless
- Borderless Fullscreem

These are the high level styles which are exposed to the end user. They map to an internal style enum class which enables certain styles (like window drop shadows).

```cpp
// User facing styles (part of Window class)
enum class Style
{
    Windowed,
    Windowed Fullscreem,
    Borderless,
    Borderless Fullscreem
}
```

In the .cpp  file, hidden from the end user there is an internal style enum:

```cpp
enum StyleInternal : DWORD
{
    Windowed             = WS_OVERLAPPEDWINDOW | WS_THICKFRAME | WS_CAPTION | WS_SYSMENU | WS_MINIMIZEBOX | WS_MAXIMIZEBOX,
    WindowedFullscreen   = Windowed | WS_MAXIMIZE,
    AeroBorderless       = WS_POPUP | /*WS_THICKFRAME |*/ WS_CAPTION | WS_SYSMENU | WS_MAXIMIZEBOX | WS_MINIMIZEBOX,
    BasicBorderless      = WS_POPUP | /*WS_THICKFRAME |*/ WS_SYSMENU | WS_MAXIMIZEBOX | WS_MINIMIZEBOX,
    BorderlessFullscreen = WS_MAXIMIZE
};
// Borderless styles have no WS_THICKFRAME to disable window resizing (we will handle that manually later)
```

WHen making my window I use a **style proxy** which strips fullscreen inforamtion from the style. Allowing me to then convert the new style to an internal window style and manually fullscreen the window if needed.

```cpp
// Returns a stripped window style and tells if the style contained fullscreen flag
Window::Style GetStyleProxy(Window::Style style, bool& outIsFullscreen)
{
    Window::Style styleProxy = style;
    outIsFullscreen = false;

    if (styleProxy == Window::Style::WindowedFullscreen)
    {
        outIsFullscreen = true;
        styleProxy = Window::Style::Windowed;
    }
    else if (styleProxy == Window::Style::BorderlessFullscreen)
    {
        outIsFullscreen = true;
        styleProxy = Window::Style::Borderless;
    }
    return style;
}
```

Now that I have our proxy style, I need to convert this to the internal style and this is done using these three functions:

```cpp
bool IsCompositionEnabled()
{
    BOOL compositionEnabled = FALSE;
    bool success = ::DwmIsCompositionEnabled(&compositionEnabled) == S_OK;
    return compositionEnabled && success;
}

StyleInternal GetBorderlessStyle()
{
    return IsCompositionEnabled() ? StyleInternal::AeroBorderless : StyleInternal::BasicBorderless;
}

StyleInternal ConvertStyle(Window::Style style)
{
    switch (style)
    {
    case Window::Style::Windowed:
        return StyleInternal::Windowed;

    case Window::Style::WindowedFullscreen:
        return StyleInternal::WindowedFullscreen;

    case Window::Style::Borderless:
    case Window::Style::BorderlessFullscreen:
        return GetBorderlessStyle();
    }
    return StyleInternal::Windowed;
}
```

`AeroBorderless` are the new types of windows which have *Desktop Window Manager Composition* toggled (available after Windows 8). Making all Borderless windows `AeroBordless` by default. Becuase of this some extra work will have to be done to handle window resizing (assuming WS_THICKFRAME is enabled).

> One could say that I can simply use enums (instead of enum classes) and bit mask flags instead of doing all these unecessary conversions. And you are probably right. But I wrote this code at 4 AM, heavily sleep deprived. And sleepy me thought this was a great idea and I stuck with it. Now its too late to go back and to be honest I am kinda proud of this mess XD.

## The Window Class

Now that I have the window styling logic out of the way. I can start creating the window itself. Starting with the constructor and the destructor

```cpp
static const StringW s_windowClassName = L"NuiApp";

Window::Window(Window::Style style, StringViewW title, Window::Size size)
: m_style(style)                      // Window::Style
, m_title(title)                      // Wide string for window title
, m_size(size)                        // Structure containing sizeX and sizeY
, m_hWnd(nullptr)                     // Window handle
, m_isFocused(false)                  // Bool to check if window has focus
, m_hInstance(GetModuleHandle(NULL))  // Application instance handle
{
    MakeWindow();
}

Window::~Window()
{
    ::DestroyWindow(m_hWnd);
    ::UnregisterClass(s_windowClassName.c_str(), m_hInstance);
}
```

The constructor calls the `MakeWindow` function which handles window creation which contains the following steps.

### Registering Window Class

The first step to creating a Win32 window is to register a window class. You can have multiple (actual) windows part of a window class.

Here I register a window class, where:

- The window redraw's if the width or height changes (see [window class styles](https://learn.microsoft.com/en-us/windows/win32/winmsg/window-class-styles))
- The window uses a static Windows Procedure (explained later)
- The background is white
- The cursor is the default cursor

The class is then registered using the `RegisterClassExW` function. This may fail is the class is already registered (which is not a problem), so I simply log that as an error and try to continue. If this was a breaking failure, then the next function where I create a window will fail whose error code will correspond to the window class registeration,

```cpp
// Register window class
WNDCLASSEXW wcx{};
wcx.cbSize        = sizeof(wcx);
wcx.style         = CS_HREDRAW | CS_VREDRAW;
wcx.hInstance     = m_hInstance;
wcx.lpfnWndProc   = WndProc;
wcx.lpszClassName = s_windowClassName.c_str();
wcx.hbrBackground = reinterpret_cast<HBRUSH>(COLOR_WINDOW + 1);
wcx.hCursor       = ::LoadCursorW(nullptr, IDC_ARROW);

if (!::RegisterClassExW(&wcx))
{
    // Log error but try to continue regardless
    NUI_LOG(Error, Window, "Failed to register window class. ", GetWin32ErrorString(GetLastError()));
}
```

### Centring The Window

This is a completely optional step. I like all muy windows to be created at the centre of the screen.

Using the Win32 API to get the desktop window rect. Then using that RECT to calculate the `x` and `y` coordinates to place my window (based on the provided window size).

```cpp
RECT desktopRect;
::GetWindowRect(::GetDesktopWindow(), &desktopRect);
I32 posX = (desktopRect.right / 2) - (m_size.X / 2);
I32 posY = (desktopRect.bottom / 2) - (m_size.Y / 2);
```

### Creating The Window

This is the part where the window is actually created. Again using the Win32 API.

```cpp
StyleInternal style = ConvertStyle(m_style);
m_hWnd = ::CreateWindowExW(
    0,                          // No extended styles
    s_windowClassName.c_str(),  // Registered WNDCLASSEXW name
    m_title.c_str(),            // Title of the window
    style,                      // Converted internal style
    posX, posY,                 // Position of the window
    m_size.X, m_size.Y,         // Size of the window
    nullptr,                    // No parent
    nullptr,                    // No menu
    m_hInstance,                // Application instance
    this                        // LPARAM
);

NUI_ASSERT(m_hWnd, "Failed to create window, handle is nullptr");
```

Here I pass `this` as the **LPARAM**, since I will later be using that to route the window messages to the class memer functions

### Applying Styling

Now that I have the window finally created, it is time to apply the styling I have ranting about from the start.

```cpp
// Get style proxy (we will be using this to apply styles)
bool fullscreen = false;
Style styleProxy = GetStyleProxy(m_style, fullscreen);

// We are modifying the style only if the window style is borderless
if (HasStyleFlag((DWORD)styleProxy, (DWORD)Style::Borderless))
{
    StyleInternal style = ConvertStyle(styleProxy);
    StyleInternal oldStyle = static_cast<StyleInternal>(::GetWindowLongW(m_hWnd, GWL_STYLE));

    if (style != oldStyle)
    {
        // Set the new style
        ::SetWindowLongW(m_hWnd, GWL_STYLE, static_cast<LONG>(style));

        // Extend frame for resizing (AeroBorderless style)
        if (IsCompositionEnabled())
        {
            static const MARGINS shadowState[2]{ { 0, 0, 0, 0 }, { 1, 1, 1, 1 } };
            ::DwmExtendFrameIntoClientArea(m_hWnd, &shadowState[style != StyleInternal::Windowed]);
        }

        // Redraw frame
        ::SetWindowPos(m_hWnd, nullptr, 0, 0, 0, 0, SWP_FRAMECHANGED | SWP_NOMOVE | SWP_NOSIZE);
    }
}

// Show window normal or fullscreen
::ShowWindow(m_hWnd, fullscreen ? SW_MAXIMIZE : SW_SHOWNORMAL);

// Update window size
RECT rect{};
AdjustWindowRect(&rect, (DWORD)m_style, FALSE);
m_size.X = rect.right - rect.left;
m_size.Y = rect.bottom - rect.top;
```

### Window Message Pump

Windows based applications are event-driven. These events are passed into the windows procedure call after they have been processed by the [windows message pump](https://learn.microsoft.com/en-us/windows/win32/winmsg/using-messages-and-message-queues).

I don't want the engine/external code to deal with the windows message loop. So I made a function called `WantsToClose`

```cpp
bool Window::WantsToClose() const
{
    // Windows message pump
    MSG msg;
    while (::PeekMessage(&msg, NULL, 0, 0, PM_REMOVE))
    {
        ::TranslateMessage(&msg);
        ::DispatchMessageW(&msg);

        if (msg.message == WM_QUIT)
        {
            return true;
        }
    }

    return false;
}
```

This can be used like this. Here I check if the window wants to close (this runs the message pump and dispatches all the events which can be handled manually).

```cpp
while (!window->WantsToClose())
{
    now = timer.GetElapsedSeconds();
    // Perform update code here
    dt = now - elapsed;
    elapsed = now;
}
```
> This is very similar to the actual code being used in the engine to update the application and the window.

#### Window Procedure

When creating the WNDCLASSEXW I passed in the `WndProc` as my window procedure function. This is a static member method that like an interface for calling the private `MessageRouter` function.

When the event `NCCREATE` (sent prior WM_CREATE when a window is first created) is sent. I extract the `this` pointer we passed at the LPARAM when creating the window. Then reinterpret the pointer as this window class and set that as the **Window Long Pointer**. Later, I try to retrieve the long pointer. If succeeded, that means we have our window, on which I call the `MessageRouter` else call the `DefWndProc`

```cpp
LRESULT WINAPI Window::WndProc(HWND hWnd, UINT uMsg, WPARAM wParam, LPARAM lParam)
{
    switch (uMsg)
    {
    case WM_NCCREATE:
    {
        LPCREATESTRUCT pCreateStruct = reinterpret_cast<LPCREATESTRUCT>(lParam);
        Window* pObj                 = reinterpret_cast<Window*>(pCreateStruct->lpCreateParams);
        ::SetWindowLongPtr(hWnd, GWLP_USERDATA, reinterpret_cast<LONG_PTR>(pCreateStruct->lpCreateParams));
        return ::DefWindowProc(hWnd, uMsg, wParam, lParam);
    }
    }

    Window* pObj = reinterpret_cast<Window*>(::GetWindowLongPtr(hWnd, GWLP_USERDATA));
    if (pObj)
        return pObj->MessageRouter(hWnd, uMsg, wParam, lParam);
    else
        return ::DefWindowProc(hWnd, uMsg, wParam, lParam);
}
```

#### Message Router

This the first layer of message indirection where I expose a bit of the low level API to the end user. In my Window class header file have the following defines.

```cpp
#define NUI_WNDPROC_ARGS Nui::Window*, UINT, WPARAM, LPARAM
#define NUI_WNDPROC_NAMED_ARGS Nui::Window* window, UINT uMsg, WPARAM wParam, LPARAM lParam
using WndCallback = std::function<LRESULT(NUI_WNDPROC_ARGS)>;
```

Using that type alias, I have a member map where I can call any message the user wants to interject.

```cpp
/**
* @brief Window message callbacks
* @note The key is the message
* @note The value is the callback
* @note The callback will be called when the message is received in the MessageRouter
*/
std::map<U32, WndCallback> m_callbacks;
```

This is how the router function works. It checks if the window message is present in the map and calls the user bound `std::function<WndCallback>` else calls the actual private member method, the `MessageHandler`.

```cpp
LRESULT Window::MessageRouter(HWND hWnd, UINT uMsg, WPARAM wParam, LPARAM lParam)
{
    if (m_callbacks.contains(uMsg))
    {
        return m_callbacks[uMsg](this, uMsg, wParam, lParam);
    }
    else
    {
        return MessageHandler(hWnd, uMsg, wParam, lParam);
    }
}
```

#### Message Handler

Finally this is the final function where a Win32 event will arrive to before either being processed internally, dispatched to another system (such as input messages) or be sent to the the default windows procedure call (because we don't care about it).

```cpp
LRESULT Window::MessageHandler(HWND hWnd, UINT uMsg, WPARAM wParam, LPARAM lParam)
{
    switch (uMsg)
    {
    case WM_NCCALCSIZE: 
    {
        if (wParam == TRUE && HasStyleFlag((DWORD)m_style, (DWORD)Style::Borderless))
        {
            auto& params = *reinterpret_cast<NCCALCSIZE_PARAMS*>(lParam);
            AdjustMaximizedClientRect(params.rgrc[0]);
            return 0;
        }
        break;
    }

    case WM_NCHITTEST: 
    {
        // When we have no border or title bar, we need to perform our
        // own hit testing to allow resizing and moving.
        if (HasStyleFlag((DWORD)m_style, (DWORD)Style::Borderless)) 
        {
            return HitTest(POINT{ GET_X_LPARAM(lParam), GET_Y_LPARAM(lParam) });
        }
        break;
    }

    case WM_DESTROY:
    {
        ::PostQuitMessage(0);
        return 0;
    }

    case WM_CLOSE:
    {
        DestroyWindow(hWnd);
        return 0;
    }

    case WM_SETFOCUS:
    {
        m_isFocused = true;
        return 0;
    }

    case WM_KILLFOCUS:
    {
        m_isFocused = false;
        return 0;
    }
    return ::DefWindowProc(hWnd, uMsg, wParam, lParam);
}
```

In the above function I handle the following messages
- **WM_NCCALCSIZE**
  - Sent when the size and position of a window's client area must be calculated.
  - Allows me to control the size and area of the the client
- **WM_NCHITTEST**
  - Sent in order to determine what part of the window corresponds to a particular screen coordinate
  - Allows me to manually control check for hit test when resizing a bordeless window (when *WS_THICKFRAME* is enabled)
- **WM_DESTROY**
  - Sent when the window is destroyed
  - I use this to trigger the close of the application
- **WM_CLOSE**
  - Sent when the close button or Alt+F3 is pressed
  - I use this to destroy the window, causing WM_DESTROY to be sent
- **WM_SETFOCUS/WM_KILLFOCUS**
  - Sent when the window gains or loses input focus
  - Use this to check if the window is currently focused, will be usefull when designing the input system

#### Manual Hit Testing

Here is the function that I call whenever the WM_NCHITTEST is sent

```cpp
LRESULT Window::HitTest(POINT cursor)
{
    // Identify borders and corners to allow resizing the window.
    // Note: On Windows 10, windows behave differently and
    // allow resizing outside the visible window frame.
    // This implementation does not replicate that behavior.
    const POINT border
    {
        ::GetSystemMetrics(SM_CXFRAME) + ::GetSystemMetrics(SM_CXPADDEDBORDER),
        ::GetSystemMetrics(SM_CYFRAME) + ::GetSystemMetrics(SM_CXPADDEDBORDER)
    };

    RECT window;
    if (!::GetWindowRect(m_hWnd, &window)) {
        return HTNOWHERE;
    }

    enum region_mask 
    {
        client = 0b0000,
        left   = 0b0001,
        right  = 0b0010,
        top    = 0b0100,
        bottom = 0b1000,
    };

    const auto result =
        left   * (cursor.x < (window.left + border.x)) |
        right  * (cursor.x >= (window.right - border.x)) |
        top    * (cursor.y < (window.top + border.y)) |
        bottom * (cursor.y >= (window.bottom - border.y));

    // Return where the mouse hit test
    switch (result) 
    {
    case left          : return HTLEFT;
    case right         : return HTRIGHT;
    case top           : return HTTOP;
    case bottom        : return HTBOTTOM;
    case top | left    : return HTTOPLEFT;
    case top | right   : return HTTOPRIGHT;
    case bottom | left : return HTBOTTOMLEFT;
    case bottom | right: return HTBOTTOMRIGHT;
    case client        : return HTCLIENT;
    default            : return HTNOWHERE;
    }
}
```

## Conclusion

And finally with all that work, we finally have a window. Although one can simply get away with registering a window class and setting the window style to `WS_OVERLAPPEDWINDOW` along with performing proper message routing (with the message pump). I decided to go ahead and implement a alightly fancier windowing system. Which may or may not be too complex / over the top for basic needs.

But I am glad you stuck around till the end of the article. This was a long one. In the next one we will look at using the message routing system to create an **Input System**. Which will allow read keyboard and mouse inputs. See you in the next one! 👋

<div style="display: flex; flex-wrap: wrap; gap: 10px;">
  {{< badge >}}Programming{{< /badge >}}
  {{< badge >}}Game Engine{{< /badge >}}
  {{< badge >}}C++{{< /badge >}}
</div>