---
tags:
  - code
  - code-craft
  - comments
  - readability
---

# Write Why Comments, Not What Comments

Here is a tip that hits right at the intersection of psychology, team dynamics, and long-term codebase health. It changes how you communicate with your future self and your teammates.

## The Tip: Stop Writing "What" Comments—Only Write "Why" Comments

Most junior and mid-level developers use comments to explain what a piece of code is doing. They treats comments as a translation layer for someone who doesn't understand the programming language.

**The advanced approach:** Assume whoever is reading your code already knows the language perfectly. Never use comments to explain **what** the code is doing or **how** it works. Use comments exclusively to explain **why** you wrote it that way—especially if the solution looks unusual or non-obvious.

### Why This is Unpopular

We are taught from day one in school: _"Comment your code so it’s easy to read."_ Many automated code-quality linters even force you to write a comment above every single function or class. This leads to developers mindlessly repeating the code's syntax in plain English just to satisfy a rule.

### Why It’s Actually Crucial (The Experience Factor)

There are two major problems with "What" comments:

1. **They Lie:** Code changes constantly during fast-paced sprints, but developers frequently forget to update the comments. Within a few months, the comment says one thing, but the code does another. Now, a future developer is completely confused about which one is the truth.
    
2. **They Clutter:** If your code is so confusing that you need a comment to explain _what_ it is doing, the solution isn't to add a comment—the solution is to rewrite the code to be self-explanatory (using clear variable names, smaller functions, and guard clauses).
    

Code tells you _how_ it works; the version control history (Git) tells you _who_ changed it; but **only a comment can tell you _why_ a specific business decision or compromise was made.**

### Code Comparison: Redundant vs. Valuable Comments

#### The Mid-Level Way (The Redundant "What")

JavaScript

```
// Check if the user is an administrator
if (user.role === 'admin' && user.isActive === true) {
    // Fetch the system log data from the database
    const logs = await database.getSystemLogs();
    // Render the log view component
    return renderLogView(logs);
}
```

Every single comment here is completely useless. Any developer who knows basic JavaScript can read `user.role === 'admin'` and know it's checking if the user is an admin. These comments add noise, take up visual space, and will inevitably rot when someone edits the logic.

#### The Advanced Way (The Contextual "Why")

JavaScript

```
// CRITICAL: We must check 'isActive' explicitly here because deactivated admins 
// retain their role string in the JWT token until the session expires (up to 24 hours).
if (user.role === 'admin' && user.isActive === true) {
    
    // Using a raw query here instead of the ORM because the 'system_logs' table 
    // has over 40M rows; the ORM abstraction causes an un-indexed sequential scan.
    const logs = await database.executeRawQuery("SELECT * FROM system_logs WHERE...");
    
    return renderLogView(logs);
}
```

### Why the Advanced Way Wins

Look at the difference in the second example. If a future developer comes along and thinks, _"Hey, why are we using a messy raw SQL query instead of our nice clean ORM? Let me refactor this!"_—they will instantly stop.

The comment warns them about the 40-million-row performance trap. It prevents them from accidentally re-introducing a massive production bug that was already solved months ago.

### The "Load Bearing" Code Test

Before you type `//`, ask yourself: _If I delete this comment, does the next developer lose critical context about an underlying business rule, a weird browser bug, a third-party API quirk, or a performance constraint?_

- If the answer is **yes**, write a detailed **"Why"** comment.
    
- If the answer is **no**, leave the comment out and let your clean, expressive code speak for itself.
    

> 💡 **Experienced Insight:** Good code is self-documenting about its execution. Great code includes documentation about its intent. Code for the reviewer who has to maintain your system under pressure six months from now.

## See also

- [[Code MOC]]
