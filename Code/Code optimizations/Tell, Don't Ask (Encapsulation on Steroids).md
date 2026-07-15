---
tags:
  - code
  - code-craft
  - encapsulation
  - oop
  - tell-dont-ask
---

# Tell, Don't Ask

Here is a deep, architectural tip regarding how components, modules, or services should communicate with one another. It fixes the nightmare of "spaghetti code" where changing one minor feature unexpectedly breaks three completely unrelated parts of the system.

## The Tip: Tell, Don't Ask (Encapsulation on Steroids)

Most developers write code that constantly "asks" an object or data structure about its internal state, makes a decision based on that state, and then modifies the object from the outside.

**The advanced approach:** Never ask an object for its state just to make a decision for it. Instead, **tell** the object exactly what you want it to do, and let _it_ internalize the logic and handle its own state internally.

### Why This is Unpopular

It forces you to move business logic out of your main, procedural control flows (like your API controllers or orchestrator services) and bury it inside your domain models or classes. To a developer used to writing simple data transfer objects (DTOs) alongside massive service layers, this feels like scattering logic all over the codebase.

### Why It’s Actually Crucial (The Experience Factor)

When you write code that "asks," you are violently violating encapsulation. You are forcing your outer service layer to know exactly how your inner data structures work. This creates a hidden architectural flaw called **Feature Envy**—where a service layer is more interested in the data of an object than the object itself.

If the internal structure of that data ever changes (e.g., a variable is renamed, or an array becomes an object), your application breaks in every single place where a service was "asking" about that data.

By shifting to a **Tell, Don't Ask** mindset, you treat objects as independent black boxes. The outer layer doesn't care _how_the object does its job; it just gives an order.

### Code Comparison: The E-Commerce Membership

Imagine an app where users can stream content. If a user’s subscription lapses, or if they hit a specific streaming limit, their account needs to be flagged as restricted.

#### The Mid-Level Way (Asking and Micromanaging)

JavaScript

```
// Inside StreamingService.js
async function startStream(userId, videoId) {
    const user = await database.getUser(userId);

    // ASKING: The service is micromanaging the user object's internal rules
    if (user.subscriptionStatus === 'expired' && user.freeTierCalculatedMinutes > 180) {
        user.isRestricted = true;
        user.restrictionReason = "Exceeded free tier limits";
        await database.saveUser(user);
        throw new Error("Playback blocked.");
    }

    return mediaServer.stream(videoId);
}
```

- **The Problem:** The `StreamingService` now knows the exact internal flags of a user (`subscriptionStatus`, `freeTierCalculatedMinutes`, etc.). If the marketing team introduces a new "Silver Tier" tomorrow, you have to find every service file that checks these variables and manually update the complex `if` statements.
    

#### The Advanced Way (Telling and Trusting)

Instead of asking, we delegate the decision entirely to the `User` class or domain model.

JavaScript

```
// Inside the User Domain Model (User.js)
class User {
    // The business rule lives safely inside the object it concerns
    evaluateStreamingEligibility() {
        if (this.subscriptionStatus === 'expired' && this.freeTierCalculatedMinutes > 180) {
            this.isRestricted = true;
            this.restrictionReason = "Exceeded free tier limits";
        }
    }
    
    get isBlockedFromStreaming() {
        return this.isRestricted;
    }
}
```

Now, the service layer becomes beautiful, clean, and highly insulated:

JavaScript

```
// Inside StreamingService.js
async function startStream(userId, videoId) {
    const user = await database.getUser(userId);

    // TELLING: Command the object to update itself, then query a clean abstraction
    user.evaluateStreamingEligibility();
    await database.saveUser(user);

    if (user.isBlockedFromStreaming) {
        throw new Error(`Playback blocked: ${user.restrictionReason}`);
    }

    return mediaServer.stream(videoId);
}
```

### Why the Advanced Way Wins

1. **Isolation of Change:** If the business rules for what makes a user "restricted" change next week, you only modify a single file: `User.js`. The `StreamingService` doesn't care, doesn't change, and doesn't break.
    
2. **Elimination of Duplication:** You don't risk another developer writing a slightly different version of that expiration logic inside a completely separate file (like `DownloadService.js` or `BillingService.js`).
    
3. **Meticulous Unit Testing:** You can test the intricate business logic of user restrictions by passing mock data directly to the `User` class, without having to spin up complex service layers or mock entire network infrastructures.
    

> 💡 **Experienced Insight:** Don't treat your classes and modules like dumb filing cabinets where you constantly pull data out, sort it on your desk, and put it back. Treat them like capable, autonomous co-workers. Tell them what the goal is, and let them handle the details.

## See also

- [[Treat Data as Immutable by Default]]
- [[The State Pattern]]
- [[Code MOC]]
