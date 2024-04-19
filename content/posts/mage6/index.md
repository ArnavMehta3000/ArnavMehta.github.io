---
title: "MAGE - Part 6"
date: 2024-04-19T21:20:06+01:00
summary: "Making a game engine - Part 6 - Application Entry Point"
---

{{< lead >}}
Finally! Something on the screen!
{{< /lead >}}

{{< alert "github" >}}
Check out Nui Engine on [Github](https://github.com/ArnavMehta3000/NuiEngine.git)
{{< /alert >}}

Up to this point, when we run the program nothing happens. In fact it technically should not compile since we have not yet defined the main function of the program. So that's what we will get to in this post.

> This article shows usage of custom utility classes/functions such as `Singleton` and `Timer` that I will not be going into detail in these posts. If you want to see their implementation, head over to the github repository.

## Entry Point

In Nui Engine, for the end user, I do not want the user to describe the entry point (main/WinMain function). That should be present in the NuiCore itself.

First I will create a new file called `EntryPoint.h` in the NuiCore project with the WinMain function:

```cpp
int WINAPI wWinMain(
	_In_     HINSTANCE hInstance,
	_In_opt_ HINSTANCE hPrevInstance,
	_In_     LPWSTR lpCmdLine,
	_In_     int nShowCmd
)
{
	return 0;
}
```

We will come back to this file later. For now, Let's make the base application class

## Application Base

Every user application class will derive from this base class which will handle internal initialization. So here is the back-bone of the AppBase class:

```cpp
// In Core/Application/AppBase.h
namespace Nui
{
	class AppBase : public Window
	{
		friend class Engine;
	public:
		explicit AppBase(WStringView appName, Window::Style style, Window::Size size);
		virtual ~AppBase() = default;

		virtual void OnInit() {}
		virtual void OnShutdown() {}

	private:
		AppBase(const AppBase&) = delete;
		AppBase(AppBase&&) = delete;

		void Tick(F64 dt);
	};

    namespace Internal
	{
		extern std::unique_ptr<Nui::AppBase> MakeApp();
	}
}

#define NUI_DECLARE_APP(app, ...)                 \
namespace Nui::Internal                           \
{                                                 \
	std::unique_ptr<Nui::AppBase> MakeApp()       \
	{                                             \
		return std::make_unique<app>(__VA_ARGS__);\
	}                                             \
}
```

As you can see the `AppBase` class inherits from the `Window` class. Allowing the user, when they inherit their application from this class, they can configure the type of window they want their application to have.

### Decalaring An Application

Of course when you create an application, you need some way to create it. That's where the extern (internal) function `MakeApp` comes in. Along with the macro `NUI_DECLARE_APP`. By declaring a function extern you tell the compiler that this function will exist. Just not now. But somewhere in the codebase, later on. If this does not make sense. Just keep reading and all these things will start connecting :)

## The Engine Class

Next up is the engine class. The `Engine` is running for the entire lifetime of the application. In fact it is what initializes and shuts down the application. For now it will have a simple header with a unique pointer to the application class.

```cpp
 // In Core/Engine.h
namespace Nui
{
	class Engine
	{
	public:
		Engine();
		~Engine();

		void Run();
		void Quit();

		inline F64 GetEngineUpTime() const noexcept { return m_engineTimer.GetElapsedSeconds(); }
		AppBase* GetApp() const noexcept;

	private:
		std::unique_ptr<AppBase> m_app;
		bool m_isRunning;
		Timer m_engineTimer;
	};
}
```

In the constructor of the engine class we will open the log file and make the application (for now). For fun I also like to have timers for my initialization and shutdown functions. Here you can see the use of the `MakeApp` extern (internal) function I defined when making the `AppBase` class above.

```cpp
Engine::Engine()
    : m_isRunning(true)
{
    // Start engine up timer
    m_engineTimer.Start();

    Timer timer;
    timer.Start();

    // Open engine log file
    Log::Internal::OpenLogFile(Filesystem::GetCurrentWorkingDirectory() / "Saved" / "NuiEngine.log");

    NUI_LOG(Debug, Engine, "Initializing Nui Engine...");

    // Make application
    m_app = Internal::MakeApp();
    NUI_ASSERT(m_app.get(), "Failed to create application");

    timer.Stop();
    NUI_LOG(Debug, Engine, "Nui Engine initialized successfully in ", timer.GetElapsedSeconds().ToString(), " seconds");
}
```

<div style="display: flex; flex-wrap: wrap; gap: 10px;">
  {{< badge >}}Programming{{< /badge >}}
  {{< badge >}}Game Engine{{< /badge >}}
  {{< badge >}}C++{{< /badge >}}
</div>
