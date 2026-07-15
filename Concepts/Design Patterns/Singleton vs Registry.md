---
tags:
  - design-patterns
  - oop
---

### Key Differences Between Singleton and Registry

- **Purpose**: Singleton ensures a single instance of a class; Registry manages multiple objects of different types.
- **Instance Control**: Singleton enforces one instance; Registry allows multiple instances and dynamic management.
- **Access**: Singleton provides direct access to one instance; Registry provides access to objects via keys.
- **Flexibility**: Registry is more flexible, supporting multiple objects and types, while Singleton is rigid.

### When to Use

- **Use Singleton** when you need exactly one instance of a class to coordinate actions or manage a resource (e.g., a single database connection or logger).
- **Use Registry** when you need to manage and access multiple objects dynamically, such as services, plugins, or configurations, across the application.

Both patterns should be used cautiously, as they can introduce global state or hidden dependencies, which may complicate testing and maintenance.