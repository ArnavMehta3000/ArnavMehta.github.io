---
title: "MAGE - Part 7"
date: 2024-06-04T20:52:28+01:00
summary: "Making a Game Engine - Part 7 - Entity Component System"
---

{{< lead >}}
Entities, Components and Systems!
{{< /lead >}}

{{< alert "github" >}}
Check out Nui Engine on [Github](https://github.com/ArnavMehta3000/NuiEngine.git)
{{< /alert >}}

Finally we are at a point where we can run the application, see the initialization, update and shutdown sequence, along with frame timing. The next thing we will be working on is the Entity Component System, or ECS for short.

## ECS Architecture

Since I am writing a custom ECS framework, I wanted to design it with **Systems** being the main focus. I will say now that this is by no means the best ECS implementation or the fastest. I've always wanted to learn and figure out how to write my own ECS framework. 

> In the repository take a look at the `Docs/Resources.txt` file for the resources I used when making this ECS framework

Here is the planned file structure

```
NuiEngine/
└── Engine/
    └── Core/
        └── Engine/
            └── ECS/
                ├── Common.h
                ├── Component.h
                ├── Context.h
                ├── ECS.h
                ├── Entity.cpp
                ├── Entity.h
                ├── Event.h
                └── System.h
```

Since ECS (in C++) are usually heavily templated systems, we are going to define most of the functions in the headers files and create almost all implementations in the `ECS.h` header file.

{{< alert "circle-info">}}
**NOTE!** This article doesn't go into full ECS implementation detail and just covers the ECS classes outline. To see the implementation details, check out `ECS.h` or the ECS folder (mentioned above) in the repository
{{< /alert >}}

## The Common ECS File

All ECS headers will include the `ECS/Common.h` header file. This file will include the Engine common header along with a couple forward declarations and a helper function.

```cpp
#pragma once
#include "Core/Common/CommonHeaders.h"
#include <type_traits>
#include <typeinfo>
#include <typeindex>

// Contains ECS forward declarations

namespace Nui::ECS
{
	using TypeIndex = std::type_index;

	class Context;
	class Entity;
	class SystemBase;

	namespace Internal
	{
		class EntityView;
		class EventSubscriberBase;
		template<typename... Types> class EntityComponentView;
	}

	template <typename T> class ComponentHandle;
	template <typename T> class EventSubscriber;

	template <typename T>
	constexpr TypeIndex GetTypeIndex()
	{
		return TypeIndex(typeid(T));
	}
}
```

> Take notice of the `Nui::ECS::Internal` namespace, we will be using that quite a bit to hide implementation details of certain classes

In this file:

- We make an alias for `std::type_index` called `Nui::ECS::TypeIndex`
- Forward declare a bunch of classes (including Internal)
- Create a function called `GetTypeIndex`

The function `GetTypeIndex<T>()` will return a unique type index for every type.

## Component Header File

The next file is called `Component.h`, which contains the class declarations for component classes. 

```cpp
#pragma once
#include "Core/Engine/ECS/Common.h"

namespace Nui::ECS
{
	namespace Internal
	{
		struct ComponentContainerBase
		{
			virtual ~ComponentContainerBase() = default;
			virtual void OnRemove(Entity* entity) = 0;
		};

		template <typename T>
		struct ComponentContainer : public ComponentContainerBase
		{
			T m_data;

			ComponentContainer() = default;
			ComponentContainer(const T& data) : m_data(data) {}

		protected:
			virtual void OnRemove(Entity* entity) override;
		};
	}

	template <typename T>
	class ComponentHandle
	{
	public:
		ComponentHandle() : m_component(nullptr) {}
		ComponentHandle(T* component) : m_component(component) {}

		bool IsValid() const noexcept { return m_component != nullptr; }
		T& Get() const noexcept { return *m_component; }
		T* operator->() const { return m_component; }
		operator bool() const { return IsValid(); }

	private:
		T* m_component;
	};
}
```

### Component Container Base

`ComponentContainerBase` is an _internal_ abstract base struct for component containers, containing a virtual destructor and an abstract `OnRemove` function that is called on the container when an entity is removed

### Component Container

`ComponentContainer` is an _internal_ struct that derives from the [Component Container Base](#component-container-base). This struct contains a `m_data` member, which is the actual content of the component.

### Component Handle

`ComponentHandle` is an accessor/wrapper class around a the component data (from the `ComponentContainer`)

<br></br>

## The Entity Header File

The `Entity.h` file contains a couple classes, we will start by looking at the the `Entity` class first.

### The Entity Class

The `Entity` class is how we will represent each entity in the ECS framework. To create and Entity, we meed to provide it (the constructor) the ECS context and the entity id.

```cpp
#pragma once
#include "Core/Engine/ECS/Component.h"

namespace Nui::ECS
{
	class Entity
	{
		friend class Context;
	public:
		constexpr static U64 InvalidId = 0;
		using ComponentMap = std::unordered_map<TypeIndex, std::unique_ptr<Internal::ComponentContainerBase>>;

	public:
		Entity(Context* context, U64 id) : m_context(context), m_id(id) {}
		~Entity() { RemoveAll(); }
		inline Context* GetContext() const noexcept { return m_context; }
		inline U64 GetId() const noexcept { return m_id; }
		inline bool IsPendingDestroy() const noexcept { return m_pendingDestroy; }

		template<typename T>
		bool Has() const

		template<typename T, typename... Args>
		ComponentHandle<T> Add(Args&&... args);

		template<typename T>
		ComponentHandle<T> Get();
		
		template<typename T>
		bool Remove();

		void RemoveAll();

		template<typename... Types>
		bool With(typename std::common_type<std::function<void(ComponentHandle<Types>...)>>::type func)

	private:
		ComponentMap m_components;
		Context*     m_context;
		U64          m_id;
		bool         m_pendingDestroy{ false };
	};
}
```

The `ComponentMap` maps component type indices to unique pointers to component containers.

The Entity class has:

- Getter functions
- Function to check if the entity is marked to be destroyed at the end of frame
- Functions to operate on/with components
- **Members**:
  - A map of component containers
  - The ECS context
  - The Entity id
  - A boolean to check if entity is pending destruction

### Iterators

The `Entity.h` file contains additional iterator classes that assist in iteration over components of an Entity.

#### Entity Component Iterator

`EntityComponentIterator` is an iterator class to allow easy iteration over entities with components.

```cpp
namespace ECS::Internal
{
	template <typename... Types>
	class EntityComponentIterator
	{
	public:
		EntityComponentIterator(Context* context, U64 index, bool isEnd, bool includePendingDestroy);
		inline U64 GetIndex() const noexcept { return m_index; }
		inline bool IncludePendingDestroy() const noexcept { return m_includePendingDestroy; }
		inline Context* GetContext() const noexcept { return m_context; }
		bool IsEnd() const noexcept;
		Entity* GetEntity() const noexcept;

		Entity* operator*() const noexcept
		bool operator==(const EntityComponentIterator<Types...>& other) const
		bool operator!=(const EntityComponentIterator<Types...>& other) const
		EntityComponentIterator<Types...>& operator++();

	private:
		bool     m_isEnd;
		U64      m_index;
		Context* m_context;
		bool     m_includePendingDestroy;
	};
}
```

#### Entity Component View

`EntityComponentView` is a class to represent a (non-owning) view over a range of entities in an ECS Context.

```cpp
namespace ECS::Internal
{
	template<typename... Types>
	class EntityComponentView
	{
	public:
		EntityComponentView(const EntityComponentIterator<Types...>& first, const EntityComponentIterator<Types...>& last);

		const EntityComponentIterator<Types...>& begin() const
		{
			return m_first;
		}

		const EntityComponentIterator<Types...>& end() const
		{
			return m_last;
		}

	private:
		EntityComponentIterator<Types...> m_first;
		EntityComponentIterator<Types...> m_last;
	};
}
```

#### Entity Iterator

`Entity Iterator` class supports easy iteration over entities

```cpp
namespace ECS::Internal
{
	class EntityIterator
	{
	public:
		EntityIterator(Context* context, U64 index, bool isEnd, bool includePendingDestroy);
		bool IsEnd() const noexcept;
		inline U64 GetIndex() const noexcept { return m_index; }
		inline bool IncludePendingDestroy() const noexcept { return m_includePendingDestroy; }
		inline Context* GetContext() const noexcept { return m_context; }
		Entity* GetEntity() const noexcept;

		Entity* operator*() const noexcept { return GetEntity(); }
		bool operator==(const EntityIterator& other) const
		bool operator!=(const EntityIterator& other) const
		EntityIterator& operator++();

	private:
		bool     m_isEnd;
		U64      m_index;
		Context* m_context;
		bool     m_includePendingDestroy;
	};
}
```

#### Entity View

`EntityView` class is used to represent a view over a range of entities in an ECS Context.

```cpp
namespace ECS::Internal
{
	class EntityView
	{
	public:
		EntityView(const EntityIterator& first, const EntityIterator& last)
			: m_first(first), m_last(last)
		{
			if (m_first.GetEntity() == nullptr || m_first.GetEntity()->IsPendingDestroy() && !m_first.IncludePendingDestroy())
			{
				++m_first;
			}
		}

		const EntityIterator& begin() const
		{
			return m_first;
		}

		const EntityIterator& end() const
		{
			return m_last;
		}

	private:
		EntityIterator m_first;
		EntityIterator m_last;
	};
}
```

> You can notice a patterm here. Every type of iterator class has a view class along with it. The iterator class allowing the ECS Context/Systems to iterate over the entities and thier components. The views are used to access the underlying iterator.

## The Event System

One of the features of this ECS framework is how it includes an event system built into it - Systems created for the ECS can make use of this event system. This section describes how the event system is designed.

### Event Subscriber

Systems that want to participate in the ECS event system, need to inherit from the `EventSubscriber` class, which inherits from the `EventSubscriberBase` class. 

```cpp
namespace Nui::ECS
{
	namespace Internal
	{
		class EventSubscriberBase
		{
		public:
			virtual ~EventSubscriberBase() = default;
		};
	}

	template <typename T>
	class EventSubscriber : public Internal::EventSubscriberBase
	{
	public:
		virtual ~EventSubscriber() = default;
		virtual void OnEvent(Context* context, const T& event) = 0;
	};
}
```

The `EventSubscriber` class is templated and contains a pure virtual function called `OnEvent`, which allows the user to pass in custom event types.

Each event is declared as a struct and is passed as the `event` argument for the `OnEvent` function along with the ECS Context.

### ECS Events

The ECS framework contains some built-in functions that the Systems can use, and are automatically dispatched by the ECS Context. 

```cpp
namespace Events
{
	struct OnEntityCreate
	{
		Entity* Entity;
	};

	struct OnEntityDestroy
	{
		Entity* Entity;
	};

	template <typename T>
	struct OnComponentAdd
	{
		Entity* Entity;
		ComponentHandle<T> Component;
	};

	template <typename T>
	struct OnComponentRemove
	{
		Entity* Entity;
		ComponentHandle<T> Component;
	};
}
```

The user can create custom events as structs and make use of the `EventSubscriber<T>` class to dispatch the event

## Systems

Systems is what I want the user to be working with when writing their own code. Nui Engine will have some built in systems (for example systems to update the transform, dispatch draw commands, etc.) that will be registered on initialization.

```cpp
#pragma once
#include "Core/Engine/ECS/Common.h"

namespace Nui::ECS
{
	class SystemBase
	{
	public:
		virtual ~SystemBase() = default;
		virtual void OnInit(Context* ctx) {}
		virtual void OnUpdate(Context* ctx, const F64 dt) {}
		virtual void OnShutdown(Context* ctx) {}
		bool IsEnabled() const noexcept { return m_enabled; }
		void SetIsEnabled(bool enabled) noexcept { m_enabled = enabled; }

	private:
		bool m_enabled{ true };
	};
}
```

## ECS Context

Throughtout this article, I've been talking about the ECS Context, what is it?

The `Context` is a class that handles and manages all entities, components and systems. Consider it like an ECS manager.

Here is the `Context` class:

```cpp
#pragma once
#include "Core/Engine/ECS/Common.h"
#include <concepts>

namespace Nui::ECS
{
	template <typename T>
	concept IsSystem = std::is_base_of<SystemBase, T>::value;

	class Context
	{
	public:
		using SubscriberMap = std::unordered_map<TypeIndex, std::vector<Internal::EventSubscriberBase*>, std::hash<TypeIndex>, std::equal_to<TypeIndex>>;

	public:
		Context();
		virtual ~Context();
		inline U64 GetEntityCount() const noexcept { return m_entities.size(); }
		Entity* GetEntityById(U64 id);
		Entity* GetEntityByIndex(U64 index);
		Entity* CreateEntity();
		void DestroyEntity(Entity* e, bool immediate = false);
		bool ClearPending();
		void Reset();
		
		template <typename T, typename... Args>
		T* RegisterSystem(Args&&... args) requires IsSystem<T>;

		template <typename T>
		void UnregisterSystem() requires IsSystem<T>;

		void UnregisterAllSystems();

		template <typename T>
		T* GetSystem() requires IsSystem<T>;

		template <typename T>
		void EnableSystem() requires IsSystem<T>;

		template <typename T>
		void DisableSystem() requires IsSystem<T>;

		template <typename T>
		bool IsSystmEnabled() requires IsSystem<T>;

		template <typename T>
		void SubscribeEvent(EventSubscriber<T>* subscriber);

		template<typename T>
		void UnsubscribeEvent(EventSubscriber<T>* subscriber);

		void UnsubscribeAll(void* subscriber);

		template<typename T>
		void EmitEvent(const T& event);

		template<typename... Types>
		Internal::EntityComponentView<Types...> Each(bool includePendingDestroy = false);

		template<typename... Types>
		void Each(typename std::common_type<std::function<void(Entity*, ComponentHandle<Types>...)>>::type viewFunc, bool includePendingDestroy = false);

		Internal::EntityView All(bool includePendingDestroy = false);
		
		void All(std::function<void(Entity*)> viewFunc, bool includePendingDestroy = false);
		
		void Tick(const F64 dt);

	private:
		std::vector<std::unique_ptr<Entity>>     m_entities;
		std::vector<std::unique_ptr<SystemBase>> m_systems;
		SubscriberMap                            m_subscribers;
		U64                                      m_lastEntityId{ 0 };
	};
}
```

The `Context` class has the following functionality:

- Get the total entities in the context
- Get Entity (class) by id
- Get Entity (class) by index (from the entites collection)
- Create an Entity
- Destroy and Entity
- Clear all entities that are marked to be destroyed
- Reset deletes all entities and their components but keeps the systems
- Functions to operate on Systems (makes use of the `IsSystem` concept)
- Functions to make use of the ECS event system
- Functions to iterate over the entities with components
- Function to update the ECS Context every frame (`Tick()`)

## Using The Nui ECS Framework

### Creating A Custom System

Here is a quick demo on how to make use of the ECS framework described above. Starting with creating a System.


Create a custom component

```cpp
struct TestComponent
{
	int x = -1;
	int y = -1;
};
```

Create a custom event

```cpp
struct TestEvent
{
	int value;
};
```

Create a custom system

```cpp
/* Inheritance Details
* - Inherit from SystemBase to mark this class as a ECS System
* - Inherit from EventSubscriber<T> to be able to consume the following events:
*   - OnEntityCreate (built-in event)
*   - OnEntityDestroy (built-in event)
*   - OnComponentAdd<TestComponent> (built-in templated event)
*   - OnComponentRemove<TestComponent> (built-in templated event)
*   - TestEvent (custom user component)
*/
class TestSystem : public Nui::ECS::SystemBase,
				   public Nui::ECS::EventSubscriber<Nui::ECS::Events::OnEntityCreate>,
				   public Nui::ECS::EventSubscriber<Nui::ECS::Events::OnEntityDestroy>,
				   public Nui::ECS::EventSubscriber<Nui::ECS::Events::OnComponentAdd<TestComponent>>,
				   public Nui::ECS::EventSubscriber<Nui::ECS::Events::OnComponentRemove<TestComponent>>,
				   public Nui::ECS::EventSubscriber<TestEvent>
{
public:
	virtual ~TestSystem() {}

	// Called when system is initialized
	virtual void OnInit(Nui::ECS::Context* ctx)
	{
		// Register all events on initialization
		ctx->SubscribeEvent<Nui::ECS::Events::OnEntityCreate>(this);
		ctx->SubscribeEvent<Nui::ECS::Events::OnEntityDestroy>(this);
		ctx->SubscribeEvent<Nui::ECS::Events::OnComponentAdd<TestComponent>>(this);
		ctx->SubscribeEvent<Nui::ECS::Events::OnComponentRemove<TestComponent>>(this);
		ctx->SubscribeEvent<TestEvent>(this);
	}

	// Called every frame
	virtual void OnUpdate(Nui::ECS::Context* ctx, const Nui::F64 dt)
	{

		// Change all entities with test component values to 1
		ctx->Each<TestComponent>(
			[&](Nui::ECS::Entity* e, Nui::ECS::ComponentHandle<TestComponent> comp)
			{
				comp->X = 1;
				comp->Y = 1;
			});
	}

	// We need to unsubscribe this from the event system
	virtual void OnShutdown(Nui::ECS::Context* ctx)
	{
		ctx->UnsubscribeAll(this);
	}

	virtual void OnEvent(class Nui::ECS::Context* ctx, const Nui::ECS::Events::OnEntityCreate& event) override
	{
		// Called whenever a new entity is created
	}

	virtual void OnEvent(class Nui::ECS::Context* ctx, const Nui::ECS::Events::OnEntityDestroy& event) override
	{
		// Called whenever a new entity is destroyed
	}

	virtual void OnEvent(class Nui::ECS::Context* ctx, const Nui::ECS::Events::OnComponentAdd<TestComponent>& event) override
	{
		// Called whenever a new TestComponent is added to an entity
	}

	virtual void OnEvent(class Nui::ECS::Context* ctx, const Nui::ECS::Events::OnComponentRemove<TestComponent>& event) override
	{
		// Called whenever a new TestComponent is removed from an entity
	}

	virtual void OnEvent(class Nui::ECS::Context* ctx, const TestEvent& event) override
	{
		// Called whenever a new the TestEvent is dispatched
	}
};
```

### Using The ECS Context

Now that we have an ECS System, we can make use of it using the ECS Context

Create a new ECS Context pointer

```cpp
std::unique_ptr<ECS::Context> ctx = std::make_unique<ECS::Context>();
```

Register the system

```cpp
TestSystem* system = ctx->RegisterSystem<TestSystem>();
```

Add 10 entities with the `TestComponent`

```cpp
for (I32 i = 0; i < 10; i++)
{
	ECS::Entity* e = ctx->CreateEntity();
	e->Add<TestComponent>(TestComponent{ 0, 0 });
}
```

Update the Context every frame

```cpp
ctx->Tick(deltaTime);
```

You can loop over the entities in two different ways

```cpp
// --- METHOD 1: Using the foreach loop---
for (ECS::Entity* e : ctx->Each<TestComponent>())
{
	ECS::ComponentHandle<TestComponent> comp = e->Get<TestComponent>();
	
	// Operate on component
	comp->X = 2;
	comp->Y = 2;
}

// --- METHOD 2: Using the Context::Each<> function---
ctx->Each<TestComponent>(
	[&](Nui::ECS::Entity* e, Nui::ECS::ComponentHandle<TestComponent> comp)
	{
		// Operate on component
		comp->X = 2;
		comp->Y = 2;
	});
```

You can dispatch events using the `Context::EmitEvent` function. Also allowing you to construc the event struct in place

```cpp
// Set the TestEvent::value to 5
ctx->EmitEvent<TestEvent>(TestEvent{ 5 });
```

## Conclusion

With the ECS system now done, we will next work on the _World_ class, which will handle and manage our ECS Context. Basically a manager for a manager (weird IKR!)

> I know this article isn't as detailed as the other posts. This article was already too long and if I had to explain every function in every ECS class, we would be on ECS for the next three articles. 

<div style="display: flex; flex-wrap: wrap; gap: 10px;">
  {{< badge >}}Programming{{< /badge >}}
  {{< badge >}}Game Engine{{< /badge >}}
  {{< badge >}}C++{{< /badge >}}
  {{< badge >}}ECS{{< /badge >}}
</div>