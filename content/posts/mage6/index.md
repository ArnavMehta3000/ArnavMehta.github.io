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

### Engine Initialization

In the constructor of the engine class we will open the log file and make the application (for now). For fun I also like to have timers for my initialization and shutdown functions to show how long it takes. Here you can see the use of the `MakeApp` extern _(internal)_ function I defined when making the `AppBase` class above.

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

### Engine Shudown

Similar to the constructor which does the initialization, I have the destructor which handles the engine shutdown procedure. Here, for now, we will simply reset the application unique pointer, which will call the destructor of the application.

```cpp
Engine::~Engine()
{
	Timer timer;
	timer.Start();

	NUI_LOG(Debug, Engine, "Shutting down Nui Engine...");

	// Shutdown the application
	m_app.reset();

	timer.Stop();
	m_engineTimer.Stop();

	NUI_LOG(Debug, Engine, "Nui Engine shut down successfully in ", timer.GetElapsedSeconds().ToString(), " seconds");
	NUI_LOG(Debug, Engine, "Nui Engine was active for ", m_engineTimer.GetElapsedSeconds().ToString(), " seconds");

	Log::Internal::CloseLogFile();
}
```

### Running The Engine

The engine `Run` function is where the main update loop of the entire application will be. Before the while loop, we will call the application `OnInit` and after the while loop we will call the `OnShutdown` on our application. insode the while loop we will process the Win32 window messages, update input, calculate delta/frame time and then update the application using the `Tick` function.

```cpp
void Engine::Run()
{
	m_app->OnInit();

	Timer updateLoop;
	updateLoop.Start();

	F64 now = 0.0, dt = 0.0, elapsed = 0.0;
	while (m_isRunning && !m_app->WantsToClose())
	{
		now = updateLoop.GetElapsedSeconds();

		Input::Internal::Update();

		[[unlikely]]  // Will only happen on the first frame so we can mark this branch as 'unlikely'
		if (dt == 0.0f)
		{
			continue;  // Skip update on the first frame
		}

		// Update application
		m_app->Tick(dt);

		dt = now - elapsed;
		elapsed = now;
	}

	updateLoop.Stop();

	m_app->OnShutdown();
}
```

## Updating Entry Point

Now that we have all the classes we need, we can go back to `EntryPoint.h` and modify the `WinMain` function to run the Engine class. Here I am making the Engine class a singleton which allows me to access anything I need using the `Singleton<>` template class. But this can be a global or local variable if preferred.

```cpp
int WINAPI wWinMain(
	_In_     HINSTANCE hInstance,
	_In_opt_ HINSTANCE hPrevInstance,
	_In_     LPWSTR lpCmdLine,
	_In_     int nShowCmd
)
{
	try
	{
		Nui::Singleton<Nui::Engine>::Get().Run();
		Nui::Singleton<Nui::Engine>::Destroy();
	}
	catch (const std::exception& e)
	{
		NUI_LOG(Exception, Main, e.what());
	}

	return 0;
}
```

## User Code

For testing the code, I have made a new project called `Testbench`. This is where we will be testing all our code for this engine. In this project I create an application class (this is the user application) called `TestbenchApp` that derives from our previously made `AppBase` class.

```cpp
// In Testbench/TestApp.h

#pragma once
#include <Core/App/AppBase.h>

class TestbenchApp : public Nui::AppBase
{
public:
	TestbenchApp();
	virtual ~TestbenchApp();

	void OnInit() override;
	void OnShutdown() override;
};
```

In the cpp file we will create the implementation of the above functions, for now the `OnInit` and `OnShutdown` functions are empty (apart from the log message that they were called)

```cpp
// In Testbench/TestApp.cpp
#include "TestApp.h"

TestbenchApp::TestbenchApp()
	: Nui::AppBase(L"TestbenchApp", Nui::Window::Style::Windowed, { 1280, 720 })
{
}

TestbenchApp::~TestbenchApp()
{
}

void TestbenchApp::OnInit()
{
	NUI_LOG(Info, TestbenchApp, "Initializing Testbench App");
}

void TestbenchApp::OnShutdown()
{
	NUI_LOG(Info, TestbenchApp, "Shutting down Testbench App");
}
```

And finally we can create a new file called `Main.cpp`, where we will include `EntryPoint.h` from the Core project and declare our application.

```cpp
// In Testbench/Main.cpp
#include <Core/EntryPoint.h>
#include "Testbench/TestApp.h"

NUI_DECLARE_APP(TestbenchApp)
```

## Conclusion

If the above steps were followed, now when you compile and run the program, you should have a window pop up. Which means that we finally have something on the screen! In the next post I will go over custom ECS + event system for our game engine. See ya there 👋.

<div style="display: flex; flex-wrap: wrap; gap: 10px;">
  {{< badge >}}Programming{{< /badge >}}
  {{< badge >}}Game Engine{{< /badge >}}
  {{< badge >}}C++{{< /badge >}}
</div>
