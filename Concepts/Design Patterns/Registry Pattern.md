---
tags:
  - design-patterns
  - oop
---

Concept: The Registry pattern is a design pattern that provides a **centralized storage mechanism** (a "registry") **for objects or data that need to be accessed globally across an application.** It acts like a **key-value** store or a **container** where objects are registered (stored) and retrieved by a unique key or identifier. Unlike the Singleton, which enforces a single instance of a class, the Registry pattern allows multiple instances of different classes to be stored and accessed.

**Key Characteristics**:

- **Centralized Storage**: Objects are stored in a single registry (e.g., a map or dictionary) and accessed via keys.
- **Flexible Access**: Objects can be registered, retrieved, or removed dynamically during runtime.
- **Global or Scoped Access**: The registry can be globally accessible or scoped to a specific part of the application.
- **Dynamic Management**: Objects can be added or replaced at runtime, unlike the rigid structure of a Singleton.

**Use Cases**:

1. **Service Locator**: Storing and retrieving services (e.g., database service, email service) by name or key, allowing different parts of the application to access them.
2. **Dependency Injection Container**: Managing dependencies by registering objects (e.g., controllers, services) and retrieving them when needed.
3. **Plugin System**: Registering plugin objects dynamically so the application can load and use plugins at runtime.
4. **Event Handlers**: Storing event listeners or handlers in a registry to manage and trigger them based on events.
5. **Configuration Registry**: Storing various configuration objects (e.g., API keys, environment settings) that different modules can access by key.

**Pros**:

- Provides a centralized way to manage and access multiple objects.
- Flexible, as objects can be registered or replaced dynamically.
- Reduces tight coupling by allowing components to retrieve dependencies from the registry.

**Cons**:

- Can introduce hidden dependencies, making it harder to track object usage.
- Global access can lead to issues similar to global variables, like difficulty in testing.
- May become a "god object" if too many objects are stored, leading to maintenance challenges.