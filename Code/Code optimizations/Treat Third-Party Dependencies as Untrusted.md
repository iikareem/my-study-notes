---
tags:
  - adapter-pattern
  - code
  - code-craft
  - dependencies
  - design-patterns
---

# Treat Third-Party Dependencies as Untrusted (Adapter Pattern)

Here is an architectural and organizational tip that experienced developers use to keep large codebases from turning into an unmaintainable nightmare over time.

## The Tip: Treat Your Third-Party Dependencies Like Untrusted Aliens (The Adapter Pattern)

When most developers find a library that solves their problem (like a date-formatting library, an HTTP client, or a charting toolkit), they install it via npm, pip, or nuget, and immediately start importing and using it everywhere across their entire codebase.

**The advanced approach:** Never let a third-party library directly touch your core business logic. Build a thin wrapper (an **Adapter** or an **Interface**) around it first. Your application should only ever talk to _your_ wrapper, and your wrapper talks to the library.

### Why This is Unpopular

It feels like over-engineering and boilerplate. If a library like Axios or Moment.js works perfectly fine out of the box, wrapping it in your own custom class or function feels like an unnecessary extra step that takes time away from shipping the feature.

### Why It’s Actually Crucial (The Experience Factor)

Libraries change, deprecate, break, or go out of style.

If you import a specific logging or payment library directly into 150 different files across a massive application, you are now **hard-coupled** to that vendor.

- What happens if that library introduces a massive breaking change in version 2.0? You have to manually update 150 files.
    
- What happens if the library is abandoned due to a critical security vulnerability? You are stuck.
    
- What happens if management decides to switch from Stripe to Braintree for payments? You have to rewrite half the application.
    

By wrapping the third-party tool, you isolate the dependency to a single file. If the library breaks or you want to swap it out entirely, you only have to modify that **one wrapper file**, and the rest of your application remains completely untouched.

### Code Comparison: The Architecture Shift

Imagine your app needs to send transactional SMS alerts to users using a popular vendor called `SmsTitan`.

#### The Mid-Level Way (Coupled and Vulnerable)

JavaScript

```
// Inside NotificationService.js, UserService.js, OrderService.js, etc.
import { SmsTitanClient } from 'sms-titan-library'; 

function notifyUserOfDispatch(userId, message) {
    const client = new SmsTitanClient({ apiKey: process.env.TITAN_KEY });
    // If SmsTitan changes this method name in an update, your app breaks everywhere
    return client.sendImmediateSmsMessage({ to: userId, text: message });
}
```

#### The Advanced Way (The Isolated Wrapper)

First, you create your own abstract tool—let's call it `SmsAdapter.js`:

JavaScript

```
// SmsAdapter.js - The ONLY file allowed to know SmsTitan exists
import { SmsTitanClient } from 'sms-titan-library';

const client = new SmsTitanClient({ apiKey: process.env.TITAN_KEY });

// This is YOUR clean API interface
export async function sendSMS(phoneNumber, message) {
    // Map their messy library method to your clean function
    return client.sendImmediateSmsMessage({ to: phoneNumber, text: message });
}
```

Now, the rest of your application imports _your_ adapter, not the library:

JavaScript

```
// Inside NotificationService.js, UserService.js, etc.
import { sendSMS } from './infrastructure/SmsAdapter';

function notifyUserOfDispatch(userId, message) {
    // Clean, predictable, and fully insulated
    return sendSMS(userId, message);
}
```

### The Ultimate Payoff

Three years later, `SmsTitan` goes out of business, and your company switches to `Twilio`.

- **The Mid-Level Developer:** Spends an entire week tracking down every instance of `SmsTitanClient`, changing config parameters, adjusting method arguments, fixing broken edge cases, and sweating through regression testing.
    
- **The Advanced Developer:** Spends 20 minutes updating `SmsAdapter.js` to use the new Twilio SDK, ensures it returns the exact same data structure as before, runs the test suite, and goes to lunch.
    

> 💡 **Experienced Insight:** Third-party code is a liability, not an asset. Control your dependencies; don't let your dependencies control you.

## See also

- [[Enforce the "Single Level of Abstraction" Principle (SLAP)]]
- [[Go Project Folder Structure]]
- [[Code MOC]]
