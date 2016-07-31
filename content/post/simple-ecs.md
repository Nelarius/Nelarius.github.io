+++
date = "2015-07-29T14:18:44+03:00"
draft = true
title = "Entity-component systems for fun and profit"
menu = "main"
description = "An overview of a small and fast entity-component system I wrote in C++."

+++

Separating logic and systems in a computer game can go a long way to simplifying the code base of the game. Take the simple game of minesweeper, for example. At any given time, there are a number of different looking tiles on screen, with many different outcomes depending on which tile you happen to click. The data that the scene contains is the grid containing the tile values, and the visual representation of the tiles themselves. Minesweeper also has logic involved. Do we let the tiles themselves handle what happens when you click on them, or should we let the class containing the grid figure it out?

I wrestled with this problem in my very first GUI program, a minesweeper clone. My conclusion was that there were two types of "data": minesweeper contains entities possessing "behavior" or "drawable" traits, or both. The solution? An entity containing pointers to a Behavior and Drawable object. The game loop updated all entities, which then delegated the updating task to the behavior and drawable objects that it owned. The problem? This wasn't true separation of logic and data, since the drawable class was responsible for drawing itself. Different entities required to be drawn in different ways, and so I had to write many new classes inheriting from Drawable. On top of that, I decided that each unique entity was to have its own class. I amassed tens of classes in my source folder, all with awe-inspiring names like SplashScreenDrawable, SplashScreenBehavior, SplashScreenEntity. I quickly became bogged down in an unmaintainable nightmare that I had created for myself.

More recently, I had another go at separating data from logic. I wrote a small, and strict entity component system (ECS). This means that data, such as the visual representation of a tile in minesweeper, is not to contain any logic whatsoever. The data is stored in components, which are unadorned POD structs. Logic is implemented in systems, which can use any number of components and can communicate with each other. The entity merely ties different components together and provides access to them. My ECS module is heavily based on EntityX, from which most of its API and some implementation details draw inspiration from. In this post, I will provide a small overview of my ECS module, and at the end of this post, I present the results from a small benchmark that I made.

In my ECS, the Entity class is simply a unique handle to data stored in the entity manager. Entities are created and dispensed by an instance of the EntityManager class:

```cpp
auto entity = entityManager.create();
```

This entity handle can be invalidated or destroyed, after which you cannot use the handle anymore. The entity can be copied in the usual way.

```cpp
entity2 = entity;

entity.invalidate();
entity.isValid();   // false
entity2.isValid();  // also false

entity.destroy();   // this will now fail
```

The underlying handle within the entity class can be obtained by calling entity.id(), which returns the type Id. Id actually stores two uint32_t values. The first is the unique id of the entity, and the second is the version of the entity. The EntityManager instance which created the entity also stores a version for that entity. When you destroy the entity, the entity manager increments its version value. Thus the validity of the entity object can be easily checked by comparing the entity's version to the entity manager's version. If they differ, then the entity was destroyed.

## Using components
Data is contained in components, which are completely unadorned structs:

```cpp
struct Position {
    Vector3f position;
};
```

Components are assigned to entities by using `Entity::assign<C>( Args&&... args )`:

```cpp
entity.assign<Position>( 0.0f, 0.0f, 0.0f );
```

Note that the component is created in memory using brace-syntax; the same parameters you would use in the aggregate initialization of the component struct must be passed to `Entity::assign<C>( Args&... )`.

Here are a few other template methods provided by Entity:

```cpp
// has the Position-component been assigned?
entity.has<Position>();
// remove the position component
entity.remove<Position>();
/*
 * The component method returns a handle.
 * The handle acts like a pointer to the desired component.
 * The actual type of 'pos' is ComponentHandle<Position>.
 * */
auto pos = entity.component<Position>();
pos->position = Vector3f( 1.0f, 1.0f, 1.0f );
```

Using `Entity::component<C>()` has O(1) time complexity. The pointer is obtained from a block of memory using the entity's unique id.

EntityManager provides an iterator interface for accessing entities. The iterators can be obtained by using the join, join<C...> methods.

```cpp
// iterate over all valid entities
for ( Entity entity: entityManager.join() ) {
    // do work
}

// iterate over all valid entities containing the Position component
for ( Entity entity: entityManager.join<Position>() ) {
    // do work
}

// iterate over all valid entities containing the Position and Velocity component
for ( Entity entity: entityManager.join<Position, Velocity>() ) {
    // do work
}
```

Components are also managed by the entity manager. The entity manager allocates a block of memory for each component type. The unique entity id is used as the offset for the component within the memory block. Thus, fetching the component will always have O(1) time complexity, with the tradeoff that more memory is allocated than strictly needed.

## Implementing functionality using systems

Program logic is implemented using systems. Systems must inherit from the class `System<S>`, where `S` is the deriving class itself. Two pure virtual methods can be overridden: `configure( EventManager& )` and `update( EventManager&, EntityManager&, float )`.

Logic is implemented in the update-method using the entity manager to gain access to components. For instance, a very simple movement system might be implemented as:

```cpp
struct Position {
    Vector3f position;
};

struct Velocity {
    Vector3f velocity;
};

class Body: public System<Body> {
    public:
        void configure( EventManager& events ) override {}
        void update( EventManager& events, EntityManager& entities, float dt ) override {
            for ( Entity entity: entities.join<Position, Velocity>() ) {
                entity.component<Position>()->position += 
                    entity.component<Velocity>()->velocity * dt;
            }
        }
};
```

Different systems can be tied together using SystemManager:

```cpp
systemManager.add<Body>();
systemManager.configure<Body>();
systemManager.update<Body>( dt );
std::shared_ptr<System> body = systemManager.system<Body>();
Communicating using events
Systems communicate between each other using events. In order for a system to receive events, it must inherit Receiver and implement the following method: void receive( const E& event ), where E is the type of event. Subscribe to the event using EventManager::subscribe<E, S>( const S& ), where S is your system. Events are emitted using EventManager::emit<E, Args...>( Args&&... ).
```

As an example, here's how you could implement a simple trigger system using systems, components, and events. We want a trigger event to fire when an entity wanders into a trigger zone (a rectangle). We define the following components and events:

```
struct TriggerComponent {
    Rectangle area;
};

struct PositionComponent {
    Vector2f position;
};

struct VelocityComponent {
    Vector2f velocity;
    float factor;
};

struct TriggerEvent {
    Entity entity;
};
```

We want the trigger system to fire the event. It can do so in the update method.

```cpp
class Trigger: public System<Trigger> {
public:
    void update( EventManager& events, EntityManager& entities, float dt ) {
        // yucky O(N^2) implementation
        for ( auto posEnt: entities.join<PositionComponent>() ) {
            auto pos = entity.component<PositionComponent>();
            for ( auto triggerEnt: entities.join<TriggerComponent>() ) {
                auto rect = entity.component<TriggerComponent>();
                // if the entity's position is in the rectangle, then fire the event!
                if ( InRectangle( rect, pos  ) ) {
                    // pass the entity to the event's default constructor
                    events.emit<TriggerEvent>( posEnt );    
                }
            }
        }
    }
};
```

Now let's assume that when a trigger fires, the entity who triggered it will speed up by a factor of two. We want the body system to receive the event. We need the configure, update and receive methods to be defined. The body system must additionally inherit from Receiver:

```cpp
class Body: public System<Body>, Receiver {
public:
    void configure( EventManager& events ) override {
        // pass a reference to yourself so that the event manager can contact you :)
        events.subscribe<TriggerEvent>( *this );
    }

    void receive( const TriggerEvent& event ) {
        // speed the entity up!
        event.entity.component<VelocityComponent>()->factor *= 2.0f;
    }

    void update( EventManager& events, EntityManager& entities, float dt ) override {
        for ( Entity entity: entities.join<PositionComponent, VelocityComponent>() ) {
            auto pos = entity.component<PositionComponent>();
            auto vel = entity.component<VelocityComponent>();
            pos->position += vel->velocity * vel->factor * dt;
        }
    }
};
```

EntityManager emits the following events, regardless of whether any system is subscribed to them.

```cpp
/**
 * @class EntityCreatedEvent
 * @brief Emitted when an entity is created.
 */
struct EntityCreatedEvent {
    Entity entity;
};

/**
 * @class EntityDestroyedEvent
 * @brief Emitted just before the entity is destroyed.
 */
struct EntityDestroyedEvent {
    Entity entity;
};

/**
 * @class ComponentAssignedEvent
 * @brief Emitted when an entity is assigned to a component.
 */
template<typename C>
struct ComponentAssignedEvent {
    Entity entity;
    ComponentHandle<C> component;
};

/**
 * @class ComponentRemovedEvent
 * @brief Emitted just before the component is removed.
 */
template<typename C>
struct ComponentRemovedEvent {
    Entity entity;
    ComponentHandle<C> component;
};
```

## The three manager classes

Finally, here is how the three manager classes we've seen should be initialized:

```cpp
pg::ecs::EventManager eventManager{};
pg::ecs::EntityManager entityManager { eventManager };
pg::ecs::SystemManager systemManager { eventManager, systemManager };
```

Other than having a heavily EntityX-inspired API, does my ECS module have any actual merit? Well, I implemented a small and silly system using both my module and EntityX. I logged the execution times of both for a very large number of entities so that I could see how fast the modules are at iterating over the entities and their components:

```
> EntityX: iterating over 1000000 components took 0.18861 seconds
> My ECS: iterating over 1000000 components took 0.0144685 seconds
```

My ECS is actually an order of magnitude faster than EntityX at the time of writing! This is something I am very happy about, as this was an area of functionality in which I diverged from EntityX. I hope to continue to improve on this base performance.

If you're interested in seeing what the code looks like, feel free to check out this folder in my game engine github repo. I tried to keep the code tidy and very readable; the SLOC count is actually quite a bit smaller than that for EntityX.
