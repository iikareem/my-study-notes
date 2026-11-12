# Git Flow vs GitHub Flow

## Overview

||Git Flow|GitHub Flow|
|---|---|---|
|**Complexity**|High|Low|
|**Best for**|Scheduled releases|Continuous deployment|
|**Branch count**|Many (5+ types)|Minimal (2 types)|
|**Learning curve**|Steep|Gentle|

---

## Git Flow

Introduced by Vincent Driessen in 2010, Git Flow is a strict branching model designed around scheduled releases.

### Branch Structure

```
main         ──────────────────────────────────────────►
              \                                  /
hotfix         \──────────────────────────────/
                \
develop          \──────────────────────────────────────►
                  \          /        \         /
feature            \────────/          \───────/
                             \
release                       \────────/
```

### Branch Types

|Branch|Purpose|Branches from|Merges into|
|---|---|---|---|
|`main`|Production-ready code|—|—|
|`develop`|Integration branch|`main`|—|
|`feature/*`|New features|`develop`|`develop`|
|`release/*`|Release preparation|`develop`|`main` + `develop`|
|`hotfix/*`|Production bug fixes|`main`|`main` + `develop`|

### Workflow Steps

1. **Start a feature**
    
    ```bash
    git checkout develop
    git checkout -b feature/my-feature
    ```
    
2. **Finish a feature**
    
    ```bash
    git checkout develop
    git merge --no-ff feature/my-feature
    git branch -d feature/my-feature
    ```
    
3. **Create a release**
    
    ```bash
    git checkout -b release/1.2.0 develop
    # bump version, fix minor bugs
    git checkout main && git merge --no-ff release/1.2.0
    git tag -a v1.2.0
    git checkout develop && git merge --no-ff release/1.2.0
    ```
    
4. **Hotfix**
    
    ```bash
    git checkout -b hotfix/critical-bug main
    # fix the bug
    git checkout main && git merge --no-ff hotfix/critical-bug
    git checkout develop && git merge --no-ff hotfix/critical-bug
    ```
    

### Pros & Cons

**✅ Pros**

- Clear structure for managing multiple versions in parallel
- Isolated release preparation without blocking ongoing work
- Dedicated hotfix path for production emergencies
- Well-suited for versioned software (libraries, apps with release cycles)

**❌ Cons**

- Heavy and complex — many branches to track
- Long-lived branches increase merge conflict risk
- Overhead is often excessive for small teams or simple projects
- Slows down delivery in fast-moving environments

---

## GitHub Flow

A simpler model popularized by GitHub. It assumes you can deploy `main` at any time.

### Branch Structure

```
main     ──────────────────────────────────────────────►
          \        /    \       /    \               /
feature    \──────/      \─────/      \─────────────/
```

### Branch Types

|Branch|Purpose|
|---|---|
|`main`|Always deployable; source of truth|
|`feature/*` (or any name)|All work — features, fixes, experiments|

### Workflow Steps

1. **Create a branch from main**
    
    ```bash
    git checkout main
    git checkout -b feature/add-login
    ```
    
2. **Commit changes**
    
    ```bash
    git add .
    git commit -m "Add login form with validation"
    git push origin feature/add-login
    ```
    
3. **Open a Pull Request**
    
    - Request review from teammates
    - Discuss, iterate, and push additional commits
4. **Deploy for review** _(optional)_
    
    - Deploy the branch to a staging environment to verify
5. **Merge into main**
    
    ```bash
    # After approval:
    git checkout main
    git merge --no-ff feature/add-login
    git push origin main
    ```
    
6. **Deploy immediately**
    
    - `main` is deployed to production right after merge (often automated via CI/CD)

### Pros & Cons

**✅ Pros**

- Simple and easy to understand
- Encourages frequent, small merges (reduces drift)
- Aligns naturally with CI/CD pipelines
- Less merge conflict overhead
- Pull Requests are central — great for code review culture

**❌ Cons**

- Requires robust CI/CD and automated testing to maintain `main` stability
- Doesn't natively support multiple concurrent release versions
- Not ideal for projects with formal, scheduled release cycles
- Hotfixes and features use the same process (no distinction)

---

## Side-by-Side Comparison

|Aspect|Git Flow|GitHub Flow|
|---|---|---|
|**Branch model**|Multiple long-lived branches|Single long-lived branch (`main`)|
|**Release style**|Scheduled / versioned|Continuous deployment|
|**Hotfixes**|Dedicated `hotfix/*` branch|Just another feature branch from `main`|
|**PR / Code review**|Optional|Central to the process|
|**CI/CD dependency**|Low|High|
|**Team size**|Any (best for larger teams)|Any (best for smaller/agile teams)|
|**Parallel versions**|Supported|Not supported|
|**Complexity**|High|Low|

---

## When to Use Which

### Choose **Git Flow** when:

- You ship versioned software (e.g., v1.0, v2.0 releases)
- You need to maintain multiple versions simultaneously
- Your release cycle is scheduled (e.g., quarterly releases)
- Your team is large and needs strict process guardrails

### Choose **GitHub Flow** when:

- You practice continuous delivery or deployment
- You have a single production environment
- You want a lightweight process for fast iteration
- Your team is small to medium and values simplicity

---

## Quick Reference

```
Git Flow:    main ← hotfix
             develop ← feature, release → main

GitHub Flow: main ← feature (via PR) → deploy
```

---

## Granular Feature Control in Git Flow

One of Git Flow's most powerful advantages is that you have **full control over what ships in each release**.

Because `develop` acts as a buffer between ongoing work and production, you can have multiple feature branches in progress at the same time and choose _which ones_ make it into the next release. If a feature isn't ready, you simply don't merge it into the `release` branch — the others ship without it.

**Example scenario:**

```
develop  ──────────────────────────────────────────►
          \          /    \        (not merged yet)
feature/A  \────────/      \──────────────────────►  ← not ready, stays out
                    \
feature/B            \──────/  ← ready, merged into release
                              \
release/1.0                    \──────► main (v1.0.0)
```

Feature A keeps living on its branch. Feature B ships. You decide — not the process.

This is fundamentally different from GitHub Flow, where everything merged into `main` is potentially going to production. In Git Flow, **merging to `develop` doesn't mean shipping** — you still have the release gate to control what goes out.

### The control levers in Git Flow:

|Stage|What you control|
|---|---|
|`feature` → `develop`|Which features are integrated for testing|
|`develop` → `release`|Which features make the cut for this release|
|`release` → `main`|What actually hits production|

---

## Releases in GitHub Flow

GitHub Flow is built around continuous deployment — the assumption is that `main` is always deployable and you ship frequently. This means it **doesn't have a formal release mechanism built in**, but teams handle it in a few ways:

### Option 1 — Git Tags _(most common)_

Merge to `main`, then tag that commit with a version number. The tag becomes your release artifact.

```bash
git checkout main
git merge --no-ff feature/my-feature
git tag -a v1.2.0 -m "Release version 1.2.0"
git push origin v1.2.0
```

Simple, but you don't get branch-level protection around the release.

### Option 2 — Release Branches _(borrowing from Git Flow)_

Some GitHub Flow teams cut a `release/1.2.0` branch off `main` when they need to stabilize before shipping. This adds structure at the cost of simplicity.

### Option 3 — Feature Flags _(the GitHub Flow way)_

Code is merged into `main` but features are **toggled off** in production until they're ready. This is how large-scale teams (including GitHub itself) ship safely without needing complex branching.

```
main  ────────────────────────────────────────►
       ↑ feature merged, but flag is OFF in prod
                              ↑ flag turned ON → feature is live
```

### The honest tradeoff:

> Git Flow gives you **structural** control over releases. GitHub Flow gives you **speed**, and expects you to build the control layer yourself (flags, CI/CD, tagging).

If you don't have robust CI/CD and feature flags, controlling _what goes out_ in a GitHub Flow release becomes awkward.

---

## What is a Git Tag?

A **tag** is a permanent label attached to a specific commit — a frozen bookmark that says _"this exact point in history is important."_

Unlike a branch, a tag **never moves**. A branch pointer advances with every new commit. A tag stays on the commit you tagged, forever.

### Visually:

```
commits:   A ── B ── C ── D ── E  ← main (moves forward with every commit)
                     ↑
                   v1.0.0         ← tag (frozen here forever)
                              ↑
                            v1.1.0 ← tag (frozen here forever)
```

### Two types of tags:

|Type|Command|Notes|
|---|---|---|
|**Lightweight**|`git tag v1.0.0`|Just a pointer, no extra info|
|**Annotated**|`git tag -a v1.0.0 -m "..."`|Stores author, date, message — recommended for releases|

### Creating and pushing a tag:

```bash
# Create an annotated tag
git tag -a v1.2.0 -m "Release version 1.2.0"

# Push the tag to remote
git push origin v1.2.0

# Checkout code at that exact point in time
git checkout v1.2.0
```

### Why tags matter for releases:

When you use GitHub Flow and merge everything into `main`, tags are how you answer the question _"what exactly was deployed on a specific date?"_ You look up the `v1.2.0` tag and you know the exact commit — no guessing, no ambiguity.

In Git Flow, the `release` branch + tag together serve that purpose, so it's more structured by default. Tags are essentially how GitHub Flow compensates for the lack of a formal release branch.