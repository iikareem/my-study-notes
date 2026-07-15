---
tags:
  - code
  - code-craft
  - immutability
  - side-effects
---

# Treat Data as Immutable by Default

Here is a data-handling tip that completely changes how you write functions, handle state, and debug complex bugs across any language.

## The Tip: Treat Data as Immutable by Default (Avoid Side Effects)

In most languages, when you pass an object or an array into a function, you are passing a **reference** to that data. Many developers take advantage of this by modifying (mutating) the object directly inside the function to save space or lines of code.

**The advanced approach:** Treat all incoming data as strictly read-only. If a function needs to change an object or an array, it should copy the data, modify the copy, and return the brand-new version. Never mutate arguments passed into a function.

### Why This is Unpopular

Creating copies of objects or arrays feels inefficient. It uses slightly more memory, requires a tiny bit more processor overhead, and forces you to use syntax like spreads (`...`), `Object.assign()`, or cloning utilities. Developers often argue, _"Why waste memory making a copy when I can just change `user.isActive = true` right here?"_

### Why It’s Actually Crucial (The Experience Factor)

Direct mutation causes **side effects**, and side effects are the number one cause of ghost bugs—bugs that happen randomly, are impossible to replicate in testing, and make you want to rip your hair out.

When you mutate an object inside a function, every other part of the system that holds a reference to that object instantly changes too, without knowing it.

Imagine this scenario:

1. You fetch a `User` object from the database.
    
2. You pass it to a function called `generateInvoice(user)`.
    
3. Inside `generateInvoice`, a developer decided to normalize the user's country code to uppercase: `user.country = user.country.toUpperCase()`.
    
4. Later in the code execution, you pass that same `user` object to `updateDatabase(user)`.
    

Suddenly, your database has uppercase country codes, and nobody knows why. You search your database saving logic, and everything looks clean. The bug was actually caused by a completely unrelated reporting function that "silently" modified the data as a side effect.

### Code Comparison: The Mutation Trap

#### The Mid-Level Way (In-Place Mutation)

JavaScript

```
// This function looks innocent, but it alters state globally
function applyDiscount(cart, discountPercent) {
    for (let item of cart.items) {
        // Warning: You are permanently changing the item prices in the original cart!
        item.price = item.price * (1 - discountPercent / 100); 
    }
    return cart;
}

const userCart = { items: [{ name: "Laptop", price: 1000 }] };
const discountedCart = applyDiscount(userCart, 10);

console.log(userCart.items[0].price); // Output: 900 (The original cart is ruined!)
```

If the user goes back to the checkout page or reloads their screen, their base price is now permanently lowered because you mutated the original reference in memory.

#### The Advanced Way (Pure, Immutable Update)

JavaScript

```
function applyDiscount(cart, discountPercent) {
    // 1. Create a shallow copy of the cart, and map over items to create copies of them
    return {
        ...cart,
        items: cart.items.map(item => ({
            ...item,
            price: item.price * (1 - discountPercent / 100) // 2. Modify the copy only
        }))
    };
}

const userCart = { items: [{ name: "Laptop", price: 1000 }] };
const discountedCart = applyDiscount(userCart, 10);

console.log(userCart.items[0].price);       // Output: 1000 (Original data remains safely pristine)
console.log(discountedCart.items[0].price); // Output: 900  (New data is correct)
```

### The Ultimate Payoff

Functions that do not modify their inputs are called **Pure Functions**.

- **They are perfectly predictable:** Given the exact same inputs, they will _always_ return the exact same output, regardless of what state the rest of the application is in.
    
- **They are incredibly easy to test:** You don't have to mock a massive global state or build complex environments. You just pass data in, and assert what comes out.
    
- **They enable safe concurrency:** If you are working in multithreaded environments (like Go, Java, or Rust), immutability completely eliminates **race conditions** because threads aren't fighting to modify the same piece of memory at the same time.
    

> 💡 **Experienced Insight:** Memory is incredibly cheap; developer debugging hours are incredibly expensive. Treat your incoming data as a historical record—look at it, read from it, but never rewrite history.

## See also

- [[Tell, Don't Ask (Encapsulation on Steroids)]]
- [[Memoization via Closures]]
- [[Code MOC]]
