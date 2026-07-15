---
tags:
  - design-patterns
  - oop
---

### 🔧 **Definition (Simplified):**

The **Factory Pattern** is a **creational design pattern** that provides a way to **create objects without specifying the exact class of the object that will be created**.

## 💡 Why Use It?

Imagine you're building something where:

- The exact type of object to create **depends on dynamic input**
    
- You want to avoid calling `new ClassName()` everywhere
    
- You want to **centralize object creation** and **encapsulate logic**
    

The Factory Pattern lets you do this.

## ✅ Key Characteristics:

| Feature                    | Description                                                   |
| -------------------------- | ------------------------------------------------------------- |
| **Central creation logic** | All object creation is handled in one place (the factory)     |
| **Abstraction**            | Client code does not need to know which class is instantiated |
| **Polymorphism-friendly**  | Useful when many related classes implement a common interface |
| **Decoupling**             | Object creation is separated from object usage                |

The **Factory Pattern** and **Registry Pattern** are design patterns used in software engineering, particularly in object-oriented programming, to address different concerns around object creation and management. Below is a breakdown of each, their purposes, and key differences:

### Factory Pattern

- **Purpose**: A creational pattern that provides a way to create objects without specifying the exact class of the object that will be created. It delegates the responsibility of object instantiation to a factory class or method.
- **How It Works**:
    - A factory class or method encapsulates the logic for creating objects.
    - Clients call the factory, passing parameters (if needed) to determine the type of object to create.
    - The factory returns an instance, often of an abstract type or interface, hiding the concrete class from the client.
- **Use Case**:
    - When the exact type of object needed depends on runtime conditions (e.g., configuration, user input).
    - To centralize object creation logic and improve flexibility.
- **Example**:
    
    java
    
    CollapseWrap
    
    Copy
    
    `interface Product { void use(); } class ConcreteProductA implements Product { public void use() { System.out.println("Using Product A"); } } class ConcreteProductB implements Product { public void use() { System.out.println("Using Product B"); } } class ProductFactory { public Product createProduct(String type) { if ("A".equals(type)) { return new ConcreteProductA(); } else if ("B".equals(type)) { return new ConcreteProductB(); } throw new IllegalArgumentException("Unknown product type"); } } // Usage public class Main { public static void main(String[] args) { ProductFactory factory = new ProductFactory(); Product product = factory.createProduct("A"); product.use(); // Output: Using Product A } }`
    
- **Pros**:
    - Encapsulates object creation, promoting loose coupling.
    - Easy to extend for new types (add new classes and update factory).
    - Hides complex instantiation logic from the client.
- **Cons**:
    - Can become complex if many types are added (factory class grows).
    - Still requires the client to know some form of type identifier (e.g., "A" or "B").

### Registry Pattern

- **Purpose**: A structural pattern (sometimes considered a variation of a creational pattern) that maintains a registry (e.g., a map or dictionary) of objects, typically pre-instantiated or lazily loaded, and allows access to them by a key or identifier.
- **How It Works**:
    - A central registry (e.g., a map) stores objects or factories, mapped to keys (strings, enums, etc.).
    - Clients query the registry by key to retrieve or create the desired object.
    - Often used for managing singletons, services, or plugins.
- **Use Case**:
    - When you need to manage a collection of objects or services, accessible globally or within a context.
    - Common in plugin systems, dependency injection, or service locators.
- **Example**:
    
    java
    
    CollapseWrap
    
    Copy
    
    `import java.util.HashMap; import java.util.Map; interface Service { void execute(); } class ServiceA implements Service { public void execute() { System.out.println("Executing Service A"); } } class ServiceB implements Service { public void execute() { System.out.println("Executing Service B"); } } class ServiceRegistry { private Map<String, Service> registry = new HashMap<>(); public void registerService(String key, Service service) { registry.put(key, service); } public Service getService(String key) { Service service = registry.get(key); if (service == null) { throw new IllegalArgumentException("Service not found: " + key); } return service; } } // Usage public class Main { public static void main(String[] args) { ServiceRegistry registry = new ServiceRegistry(); registry.registerService("serviceA", new ServiceA()); registry.registerService("serviceB", new ServiceB()); Service service = registry.getService("serviceA"); service.execute(); // Output: Executing Service A } }`
    
- **Pros**:
    - Centralizes management of objects, making them easily accessible.
    - Flexible for dynamic registration (e.g., plugins at runtime).
    - Useful for managing pre-existing objects or singletons.
- **Cons**:
    - Can become a god object if overused, leading to tight coupling.
    - Global access (if implemented as a singleton) can make testing and maintenance harder.

### Key Differences

|Aspect|Factory Pattern|Registry Pattern|
|---|---|---|
|**Focus**|Object creation|Object storage and retrieval|
|**Purpose**|Encapsulates instantiation logic|Manages and provides access to objects|
|**When Used**|When object type depends on runtime data|When objects are pre-created or registered|
|**Structure**|Factory class/method creates new instances|Registry (e.g., map) stores/retrieves objects|
|**Instantiation**|Creates new objects on demand|Often uses pre-instantiated objects|
|**Flexibility**|Good for dynamic creation|Good for lookup and plugin systems|
|**Coupling**|Loosely coupled to concrete classes|Can lead to coupling if globally accessible|

### When to Use

- **Factory Pattern**: Use when you need to create objects dynamically based on conditions, and you want to hide the creation logic. Example: Creating database connectors (MySQL, PostgreSQL) based on a config file.
- **Registry Pattern**: Use when you need to manage a set of objects (e.g., services, plugins) and access them by a key, especially in systems where objects are registered at startup or runtime. Example: A plugin system where plugins register themselves by name.

### Combined Use

Sometimes, these patterns are used together. A factory can create objects and register them in a registry for later retrieval, combining dynamic creation with centralized access.

Which pattern fits your needs depends on your scenario—creation-focused (Factory) or management-focused (Registry)? Let me know if you’d like a deeper dive or an example tailored to a specific context!