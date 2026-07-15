---
tags:
  - design-patterns
  - oop
---

The SOLID principles are a set of five design principles in object-oriented programming that aim to make software designs more understandable, flexible, and maintainable. Here's a concise explanation of each principle:

1. **S - Single Responsibility Principle (SRP)**  
   A class should have only one reason to change, **meaning it should have a single responsibility or purpose.**  
   *Example*: A `User` class handles user data but shouldn't also manage database connections. Split that into a separate `DatabaseConnection` class.

2. **O - Open/Closed Principle (OCP)**  
   Software entities (classes, modules, etc.) should be open for extension but closed for modification. You should be able to add new functionality without changing existing code.  
   *Example*: Use interfaces or abstract classes to allow new behavior (e.g., adding a new payment method) without modifying the core payment processing class.
	You make classes extendable via abstraction (like interfaces, abstract classes, or polymorphism) rather than modifying the original code.

4. **L - Liskov Substitution Principle (LSP)**  
   Subtypes must be substitutable for their base types without altering the program's correctness. If a class inherits from another, it should behave as expected when used in place of the parent.  
   *Example*: A `Bird` class with a `fly` method should work correctly for a `Sparrow` subclass but not force a `Penguin` subclass to implement `fly` if it can't.

5. **I - Interface Segregation Principle (ISP)**  
   Clients should not be forced to depend on interfaces they don't use. Keep interfaces specific to the needs of the client.  
   *Example*: Instead of a large `Worker` interface with methods like `work`, `eat`, and `sleep`, split it into smaller interfaces like `Workable` and `Eatable` so classes only implement what's relevant.

6. **D - Dependency Inversion Principle (DIP)**  
   High-level modules should not depend on low-level modules; both should depend on abstractions (e.g., interfaces). Abstractions should not depend on details; details should depend on abstractions.  
   *Example*: A `PaymentProcessor` class should depend on a `PaymentGateway` interface, not a specific `PayPal` or `Stripe` implementation, allowing easy swapping of payment providers.

These principles guide developers to create modular, testable, and scalable code. Applying them reduces tight coupling, improves maintainability, and makes systems easier to extend. If you want a deeper dive into any principle or code examples, let me know!