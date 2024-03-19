---
title: "MAGE - Part 3"
date: 2024-03-16T18:19:56Z
summary: "Making a game engine - Part 3 - Logging and Assertions"
---

{{< lead >}}
Throw it - Catch it - Log it
{{< /lead >}}


{{< alert "github" >}}
Check out Nui Engine on [Github](https://github.com/ArnavMehta3000/NuiEngine.git)
{{< /alert >}}

With the  project files now generated, we can start writing engine code. Every game engine needs a logging and error catching framework. Here is how I designed one for Nui Engine (with pseudocode).

## Logging

### Log Levels

Every log message will have a verbosity level. Which is defined using an enumeration.

```cpp
enum class LogLevel
{
    Debug,     // For low level engine information
    Info,      // For general information
    Warn,      // For warnings
    Error,     // For errors
    Fatal,     // Similar to 'Error', also gives a stacktrace
    Exception  // Similar to 'Error', also gives a stacktrace and crashes the program
}
```

### LogEntry

Each log message will be wrapped in a `LogEntry` structure which contains additional information regarding the log message.

```cpp
struct LogEntry
{
    LogLevel Level;         // Log level of the message
    String Category;        // Log category of the message
    String Message;         // The log message to display
    TimePoint Time;         // Time point of log message
    Stacktrace Stacktrace;  // The stacktrace to the location of log message
}
```

Since I am using C++23 for this project I have access to `std::stacktrace` which is what I am using to generate my stacktrace. There is also a header only stacktrace library called [backward-cpp](https://github.com/bombela/backward-cpp) which contains a bunch of additional features. Although I did not use it here since I want the engine to have minimal external dependencies.

### Log File

Since I plan to use this engine with Visual Studio, I am going to use the VS Output Window as my log message output window. I will also be using a log file in case someone is not using Visual Studio (presently external debggers already hook into the `OutputDebugString` function using which the log message can also be viewed in the debuggers output window).

I also created an internal structure called a `LogFile` which can be opened and closed using helper functions. To which the same formatted log message can be appended.

The log file resides in the `<WorkingDir>/Saved/NuiEngine.log`

### Printing the Log

To print the log I have a function called `Log` which takes a `LogEntry` and formats it and prints it out.

```cpp
void Nui::Log(const LogEntry& entry)
{
    // Make formatted log message from entry
    // Append time, category and verbosity with the message
    String formattedMsg = ... ;

    // Redirects string to output window and file (if opened)
    LogOut(formattedMsg);

    if (entry.Level == LogLevel::Fatal)
    {
        // Format and print the stacktrace to the output window/file
        PrintStacktrace(entry.Stacktrace);
    }
}
```

Although I do not want the user to directly use this function or create a `LogEntry` manually everytime they want to log something. So I made a wrapper macro.

```cpp
// The LogEntry constructor default initializes the stacktrace from this point
#define NUI_LOG(Level, Category, Message) Nui::Log::Log(Nui::Log::LogEntry(Nui::Log::LogLevel::Level, #Category, Message))

```

Here is how the log message looks like in debug build (I have an extension called VSColorOutput enabled which highlights my output messages).

{{< figure
    src="logtest.png"
    alt="Log test output"
    caption="Log test output in VS output window"
    >}}

## Assertions

Instead of using the default `assert` provided by C++. I wanted to make my own assert that makes use of `LogLevel::Exception`.

I made a the following Assert function

```cpp
void Assert(bool condition, StringView conditionString, StringView message, StringView file, I32 line, Stacktrace trace)
{
    if (!condition)
    {
        // Build a log message using the condition string and message
        String logMsg = ... ;

        // Create a log entry with level set to exception
        Log(LogEntry(LogLevel::Exception, ...));

        // Manually print the trace
        PrintStackTrace(trace);

        // Crash the program
        throw std::runtime_error("Assertion failed!");
    }
}
```

Assertions in Nui are meant to be used using this macro
```cpp
#if NUI_DEBUG
#define NUI_ASSERT(Condition, Message) Nui::Log::Assert(Condition, #Condition, Message, __FILE__, __LINE__)
#else
#define NUI_ASSERT(Condition, Message)
#endif
```

Here you can see the output window with the formatted message when an assertion fails, along with the stacktrace and file/line.

{{< figure
    src="asserttest.png"
    alt="Test failed assertion"
    caption="Output window when an assertion fails"
    >}}

## Conclusion

I would like to add the ability to set the log verbosity level. But at the time of development, I'd rather have all the log message printed so this is something we can come back to in the future. I'd also want to make the logging system multithreaded.

Once I have some Win32 specific functionality added in, I'd also like to show a message box when the program fails an assert in release mode such that the end user doesn't have to go through the created log file to find out why the program crashed.

As usual the complete source code for this project is available on my GitHub repository *(see top of page)*. If you want to see the documented implementation details.

<div style="display: flex; flex-wrap: wrap; gap: 10px;">
  {{< badge >}}Programming{{< /badge >}}
  {{< badge >}}Game Engine{{< /badge >}}
  {{< badge >}}C++{{< /badge >}}
</div>
