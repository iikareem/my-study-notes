---
tags:
  - abstraction
  - code
  - code-craft
  - readability
  - slap
---

# Enforce the Single Level of Abstraction Principle (SLAP)

Here is a structural and mental tip that fundamentally changes how you organize logic, write clean code, and prevent functions from ballooning into unmaintainable monsters.

## The Tip: Enforce the "Single Level of Abstraction" Principle (SLAP)

Most developers split their code into functions based on _length_ or _reusability_. If a function gets longer than 50 lines, or if they need to copy-paste it, they break it apart.

**The advanced approach:** Don't split functions based on how long they are; split them based on their **level of detail**. Every single line of code inside a given function should be at the exact same level of abstraction. Mixing high-level business logic with low-level implementation details in the same function is an anti-pattern.

### Why This is Unpopular

It forces you to write small, tightly focused functions that often get called only _once_. To a mid-level developer, creating a function that is only 3 lines long and used in exactly one place feels like an unnecessary abstraction that forces them to jump around the codebase to see what is happening.

### Why It’s Actually Crucial (The Experience Factor)

When you mix levels of abstraction, you force the reader's brain to constantly shift gears between **the big picture (the "What")** and **the tiny details (the "How")**.

Imagine reading a cooking recipe that says:

1. Preheat the oven to 180°C.
    
2. Chop the vegetables.
    
3. Take a knife with an ergonomic polymer handle, grip it at a 45-degree angle, apply 15 Newtons of downward pressure to slice through the cell walls of the onion...
    
4. Mix everything in a bowl.
    

Step 3 completely breaks your flow because it dives into extreme implementation mechanics while the rest of the steps are high-level concepts.

When your code does this, it is incredibly draining to read. You can't skim a high-level function to understand the business logic because your eyes are constantly tripping over complex regex formatting, SQL string manipulation, or raw array indexing buried in the middle of a business workflow.

### Code Comparison: The Mental Disruption

Let's look at a function that processes an online application.

#### The Mid-Level Way (The Abstraction Salad)

JavaScript

```
async function processApplication(applicationId) {
    // 1. High-level domain step
    const app = await database.fetchApplication(applicationId);
    
    // 2. LOW-LEVEL DETAIL: Messy string parsing and regex validation mixed right in
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    if (!emailRegex.test(app.email)) {
        throw new Error("Invalid email format");
    }

    // 3. High-level domain step
    app.status = "processed";
    await database.save(app);

    // 4. LOW-LEVEL DETAIL: Raw HTTP headers and byte streams
    const response = await fetch("https://api.thirdparty.com/notify", {
        method: "POST",
        headers: { "Content-Type": "application/json", "Authorization": "Bearer X" },
        body: JSON.stringify({ id: app.id, type: "alert_processed" })
    });
}
```

Notice how your brain has to shift from thinking about the high-level business concept of "processing an application" down into the weeds of regex syntax and HTTP headers, and then back up to the database logic.

#### The Advanced Way (Perfect SLAP Alignment)

We break the low-level details out into their own dedicated, single-purpose functions. The main function should read like a high-level executive summary.

JavaScript

```
// High-Level Function (Only coordinates steps)
async function processApplication(applicationId) {
    const app = await database.fetchApplication(applicationId);
    
    validateApplicationEmail(app.email); // Clean abstraction
    
    app.status = "processed";
    await database.save(app);
    
    await sendThirdPartyNotification(app.id); // Clean abstraction
}

// Low-Level Function (Handles the mechanics of validation)
function validateApplicationEmail(email) {
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    if (!emailRegex.test(email)) throw new Error("Invalid email format");
}

// Low-Level Function (Handles the mechanics of the network call)
async function sendThirdPartyNotification(appId) {
    await fetch("https://api.thirdparty.com/notify", {
        method: "POST",
        headers: { "Content-Type": "application/json", "Authorization": "Bearer X" },
        body: JSON.stringify({ id: appId, type: "alert_processed" })
    });
}
```

### Why the Advanced Way Wins

1. **Unmatched Skimmability:** A code reviewer or a team lead can look at `processApplication` and completely understand the business workflow in exactly three seconds. They don't have to decipher regex or understand HTTP settings to see _what_ the system is doing.
    
2. **Isolate Complexity:** If the regex needs to change or the third-party API changes its headers, you go directly to those low-level functions. You minimize the blast radius of your changes.
    
3. **Self-Documenting Code:** The names of your smaller functions (`validateApplicationEmail`) completely replace the need for comments. The code explains itself through its organization.
    

> 💡 **Experienced Insight:** Keep your functions flat in their thinking. A function should either be a **Manager**(orchestrating high-level steps but doing no manual labor) or a **Worker** (handling low-level implementation details but making no high-level architectural decisions). Never let a manager pick up a shovel.

____
It is incredibly common to confuse the **Single Level of Abstraction Principle (SLAP)** with the **Single Responsibility Principle (SRP)** because they both have the word "Single" in them and both advocate for smaller, cleaner functions.

However, they are attacking two completely different axes of software complexity: **SRP is about _why_ code changes, while SLAP is about _how_ code is read.**

Here is the breakdown of why they are distinct and why SLAP earned its unique name.

## 1. The Core Difference: Dimension vs. Hierarchy

- **SRP (Single Responsibility Principle) is horizontal.** It is about **cohesion and business domains**. It states that a piece of code (usually a class or a module) should have only _one reason to change_. It prevents you from mixing database logic, payment processing, and PDF generation in the same file.
    
- **SLAP (Single Level of Abstraction) is vertical.** It is about **readability and mental hierarchy**. It states that inside a single function, you shouldn't mix high-level conceptual steps with low-level implementation mechanics.
    

## 2. Why it is called "Level of Abstraction"

In computer science, we build systems by layering abstractions.

- **High Abstraction:** Far away from the machine, close to human business logic (e.g., `order.ship()`).
    
- **Low Abstraction:** Close to the machine, dealing with data mechanics (e.g., `regex.test()`, parsing bytes, setting HTTP headers).
    

SLAP gets its name because it demands that a single function pick **one specific altitude (level)** on that ladder of abstraction and stay there for its entire duration.

### Look at this counter-example:

A function can **perfectly follow SRP** while **completely violating SLAP**.

JavaScript

```
// This function follows SRP: It has only ONE responsibility (Processing a User Registration)
async function registerUser(userData) {
    // Step 1: High Abstraction (Domain Logic)
    const user = Domain.createNewUser(userData);

    // Step 2: LOW ABSTRACTION (Violates SLAP!)
    // Deep in the weeds of cryptography mechanics
    const salt = crypto.randomBytes(16).toString('hex');
    const hash = crypto.pbkdf2Sync(userData.password, salt, 1000, 64, 'sha512').toString('hex');
    user.passwordHash = `${salt}:${hash}`;

    // Step 3: High Abstraction (Domain Logic)
    await Database.save(user);
}
```

- **Does it follow SRP?** Yes. If the user registration business rules change, this function changes. It doesn't handle billing, it doesn't handle UI routing. It has one job.
    
- **Does it follow SLAP?** Absolutely not. It forces your brain to zoom from 30,000 feet (Domain Logic) down to 2 feet (cryptographic hashing buffers), and then jump back up to 30,000 feet (Database saving).
    

## 3. How SLAP Fixes the Function

To fix SLAP, we create a **lower-level worker function** to hide the mechanics.

JavaScript

```
// Function 1: High Level of Abstraction (The Orchestrator)
async function registerUser(userData) {
    const user = Domain.createNewUser(userData);
    
    // We abstracted the low-level crypto details away!
    user.passwordHash = hashPassword(userData.password); 

    await Database.save(user);
}

// Function 2: Low Level of Abstraction (The Mechanic)
function hashPassword(password) {
    const salt = crypto.randomBytes(16).toString('hex');
    const hash = crypto.pbkdf2Sync(password, salt, 1000, 64, 'sha512').toString('hex');
    return `${salt}:${hash}`;
}
```

Now, `registerUser` runs entirely at a **single, high level of abstraction**. It reads like a checklist of business goals.

## Summary

Think of it this way to keep them distinct:

- **SRP** tells you **which box** a piece of code belongs in (so business changes don't cause a domino effect across your app).
    
- **SLAP** tells you **how to step down the stairs** inside that box (so an engineer reading it doesn't get a mental whiplash jumping between business goals and raw syntax).

## See also

- [[Flatten Your Code Using Guard Clauses]]
- [[Tell, Don't Ask (Encapsulation on Steroids)]]
- [[Code MOC]]
