---
tags:
  - business-rules
  - code
  - code-craft
  - design-patterns
  - specification-pattern
---

# The Specification Pattern

## Comprehensive Guide: The Specification Pattern (The Rule Engine)

The **Specification Pattern** is a structural, domain-driven design pattern used to encapsulate business rules into separate, single-responsibility, and reusable objects. It transforms messy conditional logic into composable, plug-and-play components that can be chained together like a fluent sentence.

## 1. The Core Problem It Solves

As an application grows, business rules naturally accumulate. Without this pattern, developers hardcode these rules directly into service layers, web controllers, or database filters using complex boolean algebra (`&&`, `||`, `!`).

This creates three critical architectural problems:

- **Rule Duplication:** The definition of what makes an entity "valid" or "premium" gets rewritten across different services (e.g., registration, billing, marketing). If the rule changes, you have to track down and update every instance manually.
    
- **Loss of Ubiquitous Language:** Cryptic conditional math blocks (`if (user.score > 700 && user.status === 'ACTIVE' && user.age >= 21)`) hide the _intent_ of the business rules from developers and stakeholders reading the code.
    
- **Code Fragility (Regression Risks):** Modifying a multi-clause `if` statement to accommodate a new edge case frequently introduces unintended bugs in unrelated code paths.
    

## 2. Structural Component Blueprint

The design pattern relies on an object-oriented composite tree architecture consisting of two primary parts:

### The Abstract Base Class (`Specification`)

Defines the uniform interface (`isSatisfiedBy(candidate)`) that all business rules must implement. It also holds the structural mapping logic (`and()`, `or()`, `not()`) that allows independent specifications to chain together dynamically.

### Concrete Specifications

Tiny, isolated objects targeting a single business constraint. They contain no state, accept a domain object as a candidate parameter, and return a clean boolean response.

## 3. Complete Source Code Implementation

Here is a fully functional, standard reference implementation in modern JavaScript (ES6+).

JavaScript

```
// ==========================================
// SECTION A: CORE ARCHITECTURE (REUSABLE INFRASTRUCTURE)
// ==========================================

class Specification {
    /**
     * The core evaluation method that every concrete rule MUST implement.
     * @param {any} candidate - The entity being validated.
     * @returns {boolean}
     */
    isSatisfiedBy(candidate) {
        throw new Error("Method 'isSatisfiedBy' must be implemented.");
    }

    /**
     * Chains a new rule requiring BOTH specifications to be true.
     */
    and(other) {
        return new AndSpecification(this, other);
    }

    /**
     * Chains a new rule requiring AT LEAST ONE specification to be true.
     */
    or(other) {
        return new OrSpecification(this, other);
    }

    /**
     * Inverts the result of the current specification rule.
     */
    not() {
        return new NotSpecification(this);
    }
}

// --- Composite Logical Operational Handlers ---

class AndSpecification extends Specification {
    constructor(left, right) {
        super();
        this.left = left;
        this.right = right;
    }

    isSatisfiedBy(candidate) {
        // Leverages short-circuit evaluation automatically
        return this.left.isSatisfiedBy(candidate) && this.right.isSatisfiedBy(candidate);
    }
}

class OrSpecification extends Specification {
    constructor(left, right) {
        super();
        this.left = left;
        this.right = right;
    }

    isSatisfiedBy(candidate) {
        return this.left.isSatisfiedBy(candidate) || this.right.isSatisfiedBy(candidate);
    }
}

class NotSpecification extends Specification {
    constructor(specToInvert) {
        super();
        this.specToInvert = specToInvert;
    }

    isSatisfiedBy(candidate) {
        return !this.specToInvert.isSatisfiedBy(candidate);
    }
}

// ==========================================
// SECTION B: CONCRETE DOMAIN RULES (SINGLE-RESPONSIBILITY OBJECTS)
// ==========================================

class ExcellentCreditSpec extends Specification {
    isSatisfiedBy(user) {
        return user.creditScore >= 750;
    }
}

class HighNetWorthSpec extends Specification {
    isSatisfiedBy(user) {
        return user.balance >= 100000;
    }
}

class LoyalCustomerSpec extends Specification {
    isSatisfiedBy(user) {
        return user.yearsWithBank >= 2;
    }
}

// ==========================================
// SECTION C: RUNTIME EXECUTION WORKFLOW
// ==========================================

// 1. Instantiate the atomic rule blocks
const hasExcellentCredit = new ExcellentCreditSpec();
const isHighNetWorth = new HighNetWorthSpec();
const isLoyal = new LoyalCustomerSpec();

// 2. Compose the complex rules fluently like an organic sentence
// Business Rule: (Excellent Credit AND High Net Worth) OR (Loyal Customer AND NOT High Net Worth)
const complexLoanPolicy = hasExcellentCredit
    .and(isHighNetWorth)
    .or(isLoyal.and(isHighNetWorth.not()));

// 3. Mock Candidates
const applicantA = { creditScore: 780, balance: 120000, yearsWithBank: 1 }; // Pass (via left branch)
const applicantB = { creditScore: 600, balance: 15000,  yearsWithBank: 4 }; // Pass (via right branch)
const applicantC = { creditScore: 620, balance: 250000, yearsWithBank: 1 }; // Fail

// 4. Execution
console.log("Applicant A Approved:", complexLoanPolicy.isSatisfiedBy(applicantA)); // true
console.log("Applicant B Approved:", complexLoanPolicy.isSatisfiedBy(applicantB)); // true
console.log("Applicant C Approved:", complexLoanPolicy.isSatisfiedBy(applicantC)); // false
```

## 4. How the Chaining Mechanics Work (The Object Tree)

When you write a fluent chain like `specA.and(specB).or(specC)`, the engine evaluates the expression from **left to right**, dynamically building a nested, hierarchical tree of structural wrappers in system memory.

### The Runtime Execution Path

When `isSatisfiedBy(candidate)` is executed on the final composed rule object:

1. The root operational node receives the request and breaks it into evaluations on its `.left` and `.right` target references.
    
2. The runtime engine walks recursively deep into the branches until it hits the terminal atomic rules (the concrete specifications).
    
3. The primitive booleans (`true`/`false`) bubble back up through the logical operational nodes (`And`/`Or`/`Not`), culminating in a clean final evaluation.
    

## 5. Main Advantages & Coding Payoffs

- **Perfect Compliance with the Open/Closed Principle (OCP):** If the business introduces a brand-new rule constraint, you simply author a new standalone specification file. You never touch, break, or risk introducing functional regression bugs into your existing, pre-tested rules.
    
- **Isolated Testability:** Testing a rule requires no mock setups for unrelated data fields. To test your `ExcellentCreditSpec`, pass a minimal payload like `{ creditScore: 800 }` directly into the class instance and run your assertions instantly.
    
- **Flawless Portability:** The rules are no longer bound to any single database engine or orchestrator script. The exact same business model can validate an incoming HTTP request body, enforce domain assertions during core data mutations, or filter in-memory collections inside background workers.
    

## 6. Real-World Architectural Advanced Tip

In enterprise codebases, you can pass specifications directly into a data repository database layer to generate query filters dynamically.

Instead of writing custom methods like `repo.findHighNetWorthLoyalUsers()`, you write a generic search framework method: `repo.findAll(specification)`. Your database repository layer inspects the structural types inside the specification tree to auto-generate corresponding SQL `WHERE` clauses or MongoDB filter documents directly. This unifies your application data checking and your persistence filtering layers under a single source of truth.

## See also

- [[The State Pattern]]
- [[Code MOC]]
