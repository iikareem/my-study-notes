---
tags:
  - backend
  - nestjs
---

### **Request Life Cycle in NestJS**

1. **Request Received**: The client sends an HTTP request to the server (e.g., GET, POST, etc.).
2. **Middleware**: The request first encounters middleware, which can process it before it reaches the route handler.
3. **Guards**: If the request passes middleware, guards determine whether the request is allowed to proceed based on conditions like authentication or roles.
4. **Interceptors (Pre-Controller)**: Interceptors can modify or inspect the request before it reaches the controller’s route handler.
5. **Pipes**: Pipes validate or transform the incoming request data (e.g., query params, body, etc.) before the route handler processes it.
6. **Route Handler**: The controller’s route handler (a method in your controller) processes the request and generates a response.
7. **Interceptors (Post-Controller)**: Interceptors can modify or handle the response after the route handler executes.
8. **Response Sent**: The response is sent back to the client, completing the cycle.

#### **1. Middleware**

- **Role**: Middleware functions are the first to process incoming requests, sitting between the raw request and the route handler. They’re similar to Express middleware and handle tasks that need to happen early in the cycle.
- **Capabilities**:
    - Access and modify the raw request and response objects.
    - Perform tasks like logging, request modification, or early termination (e.g., sending a response directly).
    - Pass control to the next middleware or the route handler.
- **Use Cases**:
    - Logging requests (e.g., timestamp, URL, method).
    - Adding headers or parsing request bodies (e.g., JSON, URL-encoded data).
    - Blocking requests early (e.g., for rate limiting or CORS).
- **Execution**: Runs before guards, interceptors, and pipes. Applied globally, to specific routes, or to controllers.
- **Example**: A logger middleware might log the request method and URL, then call next() to proceed.
#### **2. Guards**

- **Role**: Guards determine whether a request should be allowed to proceed to the route handler based on custom logic (e.g., authentication, authorization).
- **Capabilities**:
    - Access the request object via the ExecutionContext to inspect headers, user data, etc.
    - Return a boolean (or Promise/Observable of a boolean) to allow or deny the request.
    - Throw exceptions (e.g., ForbiddenException) to block with a custom response.
- **Use Cases**:
    - Check if a user is authenticated (e.g., validate a JWT token).
    - Verify user roles or permissions (e.g., only admins can access a route).
    - Protect endpoints from unauthorized access.
- **Execution**: Runs after middleware but before interceptors and pipes. Applied globally, to controllers, or to specific routes.
- **Example**: An AuthGuard checks for a valid token; if absent, it throws an UnauthorizedException.

#### **3. Interceptors**

- **Role**: Interceptors hook into the request/response cycle, allowing you to transform or monitor both the incoming request and outgoing response.
- **Capabilities**:
    - Access the request and response via the ExecutionContext and RxJS observables.
    - Transform the result returned by the route handler (e.g., modify the response data).
    - Handle errors or add side effects (e.g., logging, caching).
    - Run logic before or after the route handler.
- **Use Cases**:
    - Log response time for performance monitoring.
    - Transform response data (e.g., wrap it in a { data: ... } structure).
    - Cache responses or retry failed requests.
- **Execution**: Runs after guards but before pipes (for request) and after the route handler (for response). Applied globally, to controllers, or to routes.
- **Example**: A LoggingInterceptor measures and logs the time taken to process a request.

#### **4. Pipes**

- **Role**: Pipes process and transform input data (e.g., query parameters, route params, request body) before it reaches the route handler.
- **Capabilities**:
    - Validate data (e.g., ensure a field is Robinhoodis a number).
    - Transform data (e.g., convert a string to a number or parse JSON).
    - Throw exceptions if validation fails (e.g., BadRequestException).
    - Access the specific argument value and metadata about the parameter (e.g., type, name).

**Interceptors are the best tool in NestJS to enforce request timeouts** because they are:

- Designed to wrap and observe controller/service execution
    
- Built on top of RxJS (great for handling async/time-based operations)
    
- Flexible, composable, and easily reusable
    
- Centralized and clean — avoiding code duplicati

NestJS doesn't implement timeout checking itself. It uses **RxJS**, a reactive programming library. The `timeout()` operator is part of **RxJS**, and it's very powerful.

|   |
|---|
|The timer runs independently; no blocking of main thread|

|   |
|---|
||