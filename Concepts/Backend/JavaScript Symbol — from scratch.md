# JavaScript Symbol — from scratch

## The problem it solves

Before Symbol existed, all object property keys in JavaScript were strings. This caused a nasty problem: **name collisions**.

Imagine you write a library that adds a helper property to a user's object:

```js
function addMeta(obj) {
  obj.id = generateId();  // your library adds "id"
}

const user = { id: 42, name: "Alice" };  // user already has "id"
addMeta(user);

console.log(user.id);  // your library just overwrote their id!
```

The user's `id` is gone. Silently. No error, no warning.

This was a real problem for JavaScript libraries and frameworks — there was no safe way to attach internal metadata to an object without risking a clash with the developer's own properties.

**Symbol was introduced in ES6 (2015) to solve exactly this.**

---

## What is a Symbol?

A Symbol is a **primitive value** that is guaranteed to be unique — every time you create one, it is different from every other Symbol ever created, even if they have the same label.

```js
const a = Symbol("id");
const b = Symbol("id");

console.log(a === b);  // false — always!
console.log(a === a);  // true  — only equal to itself
```

The string you pass in (`"id"`) is just a human-readable **description** for debugging. It has no effect on the value itself.

---

## How to create a Symbol

```js
const mySymbol = Symbol();           // no description
const withDesc = Symbol("userId");   // with description (recommended)

console.log(typeof mySymbol);        // "symbol"
console.log(withDesc.description);   // "userId"
console.log(withDesc.toString());    // "Symbol(userId)"
```

> **Never use `new Symbol()`** — it throws a TypeError. Call it as a plain function.

---

## How Symbol solves the collision problem

Because every Symbol is unique, you can use one as an object property key and it will **never clash** with any string key or another Symbol:

```js
// Your library
const INTERNAL_ID = Symbol("id");

function addMeta(obj) {
  obj[INTERNAL_ID] = generateId();  // safe — uses a Symbol key
}

// User's code
const user = { id: 42, name: "Alice" };
addMeta(user);

console.log(user.id);            // 42 — untouched!
console.log(user[INTERNAL_ID]);  // the library's internal value
```

The user's `id` string key and the library's `Symbol("id")` key are completely separate. They cannot collide.

---

## Symbols are hidden from normal enumeration

Symbol keys are **invisible** to the most common ways of inspecting an object:

```js
const SECRET = Symbol("secret");
const obj = { name: "Alice", [SECRET]: "hidden value" };

console.log(Object.keys(obj));       // ["name"]   — no SECRET
console.log(JSON.stringify(obj));    // '{"name":"Alice"}'  — no SECRET

// To access Symbol keys explicitly:
console.log(Object.getOwnPropertySymbols(obj));  // [Symbol(secret)]
console.log(obj[SECRET]);                        // "hidden value"
```

This makes Symbols perfect for **internal metadata** — data that belongs to a library or framework, not the end user.

---

## The 7 primitive types in JavaScript

Symbol is one of seven primitive types. It is the only one that guarantees uniqueness:

|Type|Example|Unique?|
|---|---|---|
|`string`|`"hello"`|No|
|`number`|`42`|No|
|`boolean`|`true`|No|
|`undefined`|`undefined`|No|
|`null`|`null`|No|
|`bigint`|`9007199254740991n`|No|
|`symbol`|`Symbol("x")`|**Yes**|

---

## Symbol.for() — sharing a Symbol globally

Sometimes you _want_ two different modules to share the exact same Symbol (for example, across different files or iframes). Use `Symbol.for()`, which stores and retrieves Symbols from a global registry:

```js
const s1 = Symbol.for("app.userId");
const s2 = Symbol.for("app.userId");

console.log(s1 === s2);           // true — same Symbol from registry
console.log(Symbol.keyFor(s1));   // "app.userId"
```

Compare this to regular `Symbol()`:

```js
const a = Symbol("x");
const b = Symbol("x");
a === b;  // false — always new, never shared
```

Use `Symbol()` when you want a private, one-off key. Use `Symbol.for()` when you need to share across modules.

---

## Well-known Symbols — hooking into JavaScript itself

JavaScript uses built-in Symbols to let you customise how your objects behave with native language features. These are called **well-known Symbols**.

### Example: `Symbol.iterator`

This Symbol lets you make any object work with `for...of` and the spread operator `[...x]`:

```js
class Range {
  constructor(start, end) {
    this.start = start;
    this.end = end;
  }

  [Symbol.iterator]() {
    let current = this.start;
    const end = this.end;
    return {
      next() {
        return current <= end
          ? { value: current++, done: false }
          : { done: true };
      }
    };
  }
}

const range = new Range(1, 5);
console.log([...range]);  // [1, 2, 3, 4, 5]

for (const n of range) {
  console.log(n);  // 1, 2, 3, 4, 5
}
```

### Other well-known Symbols

|Symbol|What it controls|
|---|---|
|`Symbol.iterator`|Makes objects iterable with `for...of` and `[...x]`|
|`Symbol.toPrimitive`|Controls how an object converts to a number/string|
|`Symbol.hasInstance`|Customises `instanceof` behaviour|
|`Symbol.asyncIterator`|Makes objects work with `for await...of`|

---

## Common gotchas

### 1. Cannot concatenate with strings

```js
const s = Symbol("hello");

"Value: " + s;     // TypeError!
`Value: ${s}`;     // TypeError!

// Do this instead:
"Value: " + s.toString();    // "Value: Symbol(hello)"
"Value: " + s.description;  // "Value: hello"
```

### 2. JSON.stringify silently drops Symbol keys

```js
const TOKEN = Symbol("token");
const obj = { [TOKEN]: "abc123", name: "Alice" };

JSON.stringify(obj);  // '{"name":"Alice"}' — TOKEN is gone!
```

This is intentional. If you need to serialise Symbol-keyed data, you must handle it manually.

### 3. Never use `new Symbol()`

```js
new Symbol();   // TypeError: Symbol is not a constructor
Symbol();       // correct
```

---

## Quick summary

| Question                                  | Answer                                               |
| ----------------------------------------- | ---------------------------------------------------- |
| What problem does Symbol solve?           | Preventing property name collisions on objects       |
| Is every Symbol unique?                   | Yes — even with the same description                 |
| Can Symbol keys clash with strings?       | No — they are a separate key type                    |
| Are Symbol keys visible in `for...in`?    | No — they are non-enumerable by default              |
| How do you share a Symbol across modules? | Use `Symbol.for("key")`                              |
| What are well-known Symbols?              | Built-in Symbols that hook into JS language features |