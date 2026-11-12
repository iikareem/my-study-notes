# Design Patterns in Production — What I Built & Why

> A personal reference for interview prep, knowledge review, and blog posts. Every pattern here is grounded in a real problem I faced — not textbook theory. Written so that anyone, even without background in this codebase, can follow the reasoning.

---

## What System Is This?

This is a **NestJS-based advanced filtering engine** (codename _Troodon / Zeustra_). It sits between a frontend and multiple datasources, applying column-level filters, post-fetch row transformations, in-memory caching, and complex SQL generation for nested relational data.

The system grew over time. Requirements changed. New datasources were added. New SQL shapes were needed. The patterns documented here were introduced — one by one — each time the codebase showed a clear pain signal. None of them were added speculatively.

---

## Quick Navigation

|Pattern|Core Pain It Solved|
|---|---|
|[Dependency Inversion](https://claude.ai/chat/94e0d710-e28b-4e41-aef0-3c84f8049335#1-dependency-inversion-principle-dip)|Services were importing vendor SDKs directly — every change was a ripple|
|[Adapter](https://claude.ai/chat/94e0d710-e28b-4e41-aef0-3c84f8049335#2-adapter)|Nest Logger and Sentry have incompatible APIs — one interface had to rule them|
|[Decorator / Composite](https://claude.ai/chat/94e0d710-e28b-4e41-aef0-3c84f8049335#3-decorator--composite)|Needed to log to console, Sentry, or both — without touching callers|
|[Strategy](https://claude.ai/chat/94e0d710-e28b-4e41-aef0-3c84f8049335#4-strategy)|`if/else` trees for hashing, SQL, and validation were growing uncontrollably|
|[Registry](https://claude.ai/chat/94e0d710-e28b-4e41-aef0-3c84f8049335#5-registry)|Named implementations needed to be discovered at runtime, not hard-coded|
|[Factory](https://claude.ai/chat/94e0d710-e28b-4e41-aef0-3c84f8049335#6-factory)|Object construction was too complex to scatter across call sites|
|[Chain of Responsibility](https://claude.ai/chat/94e0d710-e28b-4e41-aef0-3c84f8049335#7-chain-of-responsibility-cor)|Post-fetch transforms were piling up inside one formatter|
|[Builder](https://claude.ai/chat/94e0d710-e28b-4e41-aef0-3c84f8049335#8-builder)|SQL was being assembled by string concatenation — fragile and untestable|
|[Iterator](https://claude.ai/chat/94e0d710-e28b-4e41-aef0-3c84f8049335#9-iterator)|Every feature reimplemented its own tree-walking logic|
|[Proxy](https://claude.ai/chat/94e0d710-e28b-4e41-aef0-3c84f8049335#10-proxy)|Cache, metrics, and timeouts were being bolted onto every query individually|

---

## How to Read This Guide

Each pattern follows this structure:

1. **The concept** — what the pattern actually is, in plain language
2. **The pain without it** — what the code looked like before, and why it hurt
3. **The solution** — what I built and how it works
4. **The code** — real implementation from this codebase
5. **Key insight** — the one sentence that captures why this pattern matters here
6. **Interview talking points** — how to tell the story concisely

---

# Part A — Foundations: The Logging Story

Three patterns work together here and tell the clearest story in the entire codebase. Read them in order: **DIP creates the contract**, **Adapter fills it**, **Decorator composes it**.

```
Feature Services / DBService
         │
         │  depends only on
         ▼
       ILogger  (the contract)
         │
         ├──► LoggerAdapter        (wraps Nest Logger)
         ├──► SentryLoggerAdapter  (wraps Sentry SDK)
         └──► CompositeLogger      (holds both — fans out every call)
```

---

## 1. Dependency Inversion Principle (DIP)

### The Concept

The rule is simple: **high-level business code should not depend on low-level implementation details**. Both should depend on an abstraction in the middle. If your service imports a specific library directly, the service and the library are glued together — change one, touch the other.

Think of it like a power outlet. Your laptop doesn't care about how the electricity is generated. It just needs a standard socket. The "socket" here is an interface.

### The Pain Without It

Imagine `DBService` importing Nest's `Logger` directly:

```ts
// ❌ Without DIP — the bad version
import { Logger } from '@nestjs/common';

export class DBService {
  private logger = new Logger('DBService');

  query(sql: string) {
    this.logger.log(`Running query: ${sql}`);   // tightly coupled
    // ...
  }
}
```

Now ask: what happens when you need to also send errors to Sentry? You modify `DBService`. What if you want `fatal` severity, which Nest doesn't have? You add a workaround inside `DBService`. What if you need to swap the logger in tests? You mock a real Nest module.

Every concern that isn't business logic bleeds into the service. And this repeats across every service that logs.

### The Solution

Define an interface — `ILogger` — that describes _what logging means_ in this domain. Every service depends on that interface, never on a specific library. The interface is stable even when the implementation changes.

```ts
// src/modules/logging/interface/logger.interface.ts
export interface ILogger {
  debug(message: string, meta?: any): void;
  log(message: string, meta?: any): void;
  warn(message: string, meta?: any): void;
  error(message: string, meta?: any): void;
  fatal(message: string, meta?: any): void;  // domain need — not tied to any SDK
}
```

Services inject this interface via a Nest token:

```ts
// ✅ With DIP — DBService never imports Nest Logger or Sentry
export class DBService {
  constructor(
    @Inject(COMPOSITE_LOGGER) private readonly logger: ILogger,
  ) {}

  query(sql: string) {
    this.logger.log(`Running query: ${sql}`);  // calls the interface
  }
}
```

`DBService` doesn't know if it's logging to console, Sentry, or both. That decision lives in the module wiring, not in the service.

### Where Else I Applied This

The same principle governs every cross-cutting concern in this system:

|Interface|Who Uses It|What It Hides|
|---|---|---|
|`ILogger`|DB, indexing, filtration|Nest Logger, Sentry SDK|
|`IMetricsProvider`|`DbMetricsService`, pool, S3 pipeline|Sentry metrics API|
|`OperationPipelineTransformer`|Post-fetch CoR handlers|Concrete transform classes|
|`Buildable<T, K>` / `QueryFactory<T, K>`|Troodon query cores|SQL builder implementations|
|`HashStrategy`|`HashService`|MD5, SHA256 logic|
|`ColumnMetadataStrategy`|`MetadataService`|Column, view, reference metadata|

### Key Insight

> The interface is a firewall. Business logic lives on one side, vendor details live on the other. DIP is what keeps them from leaking into each other.

### Interview Talking Points

- _"I inverted dependencies so `DBService` depends on `ILogger`, not Sentry."_
- _"DIP is what makes the Adapter and Decorator patterns possible here — you need the abstraction first."_
- _"In practice, this meant I could change logging backends without touching a single service."_

---

## 2. Adapter

### The Concept

An Adapter translates one interface into another. It's a wrapper — it takes what a library or class _speaks_ and converts it to what your system _expects_. The caller only ever talks to your interface; the adapter handles the translation underneath.

A real-world analogy: a travel power adapter doesn't change the electricity. It just makes a US plug fit a European socket.

### The Pain Without It

Nest's `Logger` and Sentry's logger are completely different APIs:

```ts
// Nest Logger
logger.log('message');    // "log" means info
logger.error('message');  // error
// no fatal()

// Sentry logger
logger.info('message');   // "info" — not "log"
logger.fatal('message');  // fatal exists here
```

Without adapters, every service that needs to call both would have to know about both APIs. You'd end up with branching inside services:

```ts
// ❌ Without Adapter — painful and fragile
if (useRemote) {
  Sentry.logger.info(message);
} else {
  nestLogger.log(message);
}
```

That's not logging logic — that's routing logic polluting business code. And it duplicates across every service.

### The Solution

Write one adapter class per vendor. Each adapter implements `ILogger` and internally calls the vendor's real API. The service never sees the translation.

**Console Adapter** — wraps Nest's Logger:

```ts
export class LoggerAdapter implements ILogger {
  constructor(private readonly logger: Logger) {}

  log(message: string, meta?: any) {
    this.logger.log(message, meta);       // same name — direct call
  }

  fatal(message: string, meta?: any) {
    this.logger.error(message, meta);     // Nest has no fatal — map to error
  }
}
```

**Sentry Adapter** — wraps the Sentry SDK:

```ts
export class SentryLoggerAdapter implements ILogger {
  log(message: string, meta?: any) {
    Sentry.logger.info(message, meta);    // "log" → "info" translation
  }

  fatal(message: string, meta?: any) {
    Sentry.logger.fatal(message, meta);   // Sentry has fatal — direct call
  }
}
```

From the outside, both classes look identical — they both implement `ILogger`. The differences between Nest and Sentry are fully contained inside the adapters.

**Same idea applied elsewhere:**

- `SentryMetricServiceAdapter` → implements `IMetricsProvider` (hides Sentry's metrics API)
- `ProtobufCodec` → implements `BinaryFormatInterface` (hides protobufjs internals)

### Key Insight

> The Adapter pattern is what lets two things that don't speak the same language work together — without either side changing. Your code speaks `ILogger`; the adapters translate that into whatever the vendor needs.

### Interview Talking Points

- _"Adapter is the bridge: Nest and Sentry both become `ILogger` from the outside."_
- _"If we ever switch from Sentry to Datadog, I write one new adapter — nothing else changes."_
- _"I used the same pattern for metrics and protobuf, not just logging."_

---

## 3. Decorator / Composite

### The Concept

A Decorator wraps an object that implements an interface, adds behavior, and exposes the **same interface**. The caller can't tell the difference between a plain object and a decorated one — they all look the same from the outside.

In this codebase, the `CompositeLogger` is a **Decorator + Composite**: it implements `ILogger`, holds an array of `ILogger`s, and **fans every call out to all of them**. It stacks destinations without the caller knowing.

### The Pain Without It

Three deployment modes were needed:

|Environment|Logging Destination|
|---|---|
|Local dev|Console only|
|Production|Sentry only|
|Staging / debug|Both simultaneously|

Without the Decorator pattern, there are two bad options:

**Option A — Add conditions inside services:**

```ts
// ❌ Bad — routing logic inside business code
this.nestLogger.log(message);
if (isProduction) {
  Sentry.logger.info(message);
}
```

Every service becomes aware of environments. Every service needs updating when modes change.

**Option B — Duplicate services:**

```ts
// ❌ Also bad — combinatorial explosion
class ConsolePlusRemoteDBService { ... }
class ConsoleOnlyDBService { ... }
class RemoteOnlyDBService { ... }
```

Two variants today becomes ten variants tomorrow.

### The Solution

`CompositeLogger` implements `ILogger` and holds an array of `ILogger`s. One call to `log()` on the composite calls `log()` on every logger in the array. Services inject one thing — the composite — and the composite handles fan-out invisibly.

```ts
export class CompositeLogger implements ILogger {
  constructor(private readonly loggers: ILogger[]) {}

  log(message: string, meta?: any) {
    this.loggers.forEach(l => l.log(message, meta));
  }

  error(message: string, meta?: any) {
    this.loggers.forEach(l => l.error(message, meta));
  }

  warn(message: string, meta?: any) {
    this.loggers.forEach(l => l.warn(message, meta));
  }

  fatal(message: string, meta?: any) {
    this.loggers.forEach(l => l.fatal(message, meta));
  }
}
```

**Wired in `logger.module.ts` — the environment decision lives here, not in services:**

```ts
// COMPOSITE_LOGGER = console + Sentry together
{
  provide: COMPOSITE_LOGGER,
  useFactory: (logger: Logger) => {
    return new CompositeLogger([
      new SentryLoggerAdapter(),
      new LoggerAdapter(logger),
    ]);
  },
  inject: [Logger],
},

// CONSOLE_LOGGER = only local
{ provide: CONSOLE_LOGGER, useClass: LoggerAdapter },

// REMOTE_LOGGER = only Sentry
{ provide: REMOTE_LOGGER, useClass: SentryLoggerAdapter },
```

A service that wants both just injects `COMPOSITE_LOGGER`. Nothing else changes.

### Key Insight

> The Decorator pattern lets you compose behavior by wrapping — not by inheriting or branching. Adding a third logging destination means adding one class and changing one line in the module. No service touches it.

### Interview Talking Points

- _"Classic GoF Decorator wraps one object; mine is Decorator + Composite since it fans out to multiple. That distinction matters to senior interviewers."_
- _"Console-only locally, composite in production — the environment decision is in DI wiring, not in business logic."_
- _"The same `ILogger` call sites work whether you inject console, Sentry, or both."_

---

# Part B — Pluggable Behavior

Three patterns often appear together in this codebase:

- **Strategy** = the algorithm itself (how to hash, how to build SQL, how to validate)
- **Registry** = the lookup table (which strategy has which name)
- **Factory** = the creator (builds the right object so callers don't have to)

---

## 4. Strategy

### The Concept

Strategy encapsulates a family of algorithms behind a single interface. The context (the service that needs the algorithm) doesn't know _which_ algorithm it's running — it just calls the interface. At runtime, the right one is selected and injected.

Think of it like a sorting algorithm selector: the list doesn't care whether you're using quicksort or mergesort. It just hands itself to whatever sorter was chosen for this context.

### The Pain Without It

The system handles multiple request types, each needing a different normalization before hashing. Without Strategy, this looks like:

```ts
// ❌ Without Strategy — grows with every new request type
buildHashInput(json: any, requestType: string): string {
  if (requestType === 'NESTED_SELECTION') {
    // 20 lines of nested selection normalization
  } else if (requestType === 'INFO_ENDPOINT') {
    // 15 lines of info endpoint normalization
  } else if (requestType === 'ANALYTICS') {
    // coming soon — add another branch
  }
}
```

Every new request type requires modifying this function. It grows without bound. The function has to know about every algorithm. Testing one algorithm means setting up the whole function. This violates the Open/Closed Principle: the code is not closed for modification.

### The Solution

Define a `HashStrategy` interface. Each normalization algorithm becomes its own class. A resolver picks the right one by key. The service just calls `strategy.buildHashInput(json)` — it never sees the branching.

**A) Hash Normalization:**

```ts
export class HashStrategyResolver {
  private strategies = new Map<string, HashStrategy>([
    ['NESTED_SELECTION', new NestedSelectionHashingNormalizer()],
    ['INFO_ENDPOINT',    new InfoEndpointHashingNormalizer()],
  ]);

  resolve(requestType: string): HashStrategy {
    return this.strategies.get(requestType)
      ?? this.strategies.get('NESTED_SELECTION'); // safe default
  }
}

// Caller:
const strategy = resolver.resolve(requestType);
const hashInput = strategy.buildHashInput(json);
```

Adding a new request type = add one class + one line in the map. Nothing else changes.

**B) Override-info Metadata — using `supports()` instead of a key:**

Sometimes the selection logic is more nuanced than a simple key lookup. The `supports()` pattern lets each strategy declare what it handles:

```ts
// Each strategy says what it can handle
getColumnMetaData<T, R>(dto: T, type: MetaDataEnum): R {
  const strategy = this.strategies.find(s => s.supports(type));
  strategy.validate(dto);
  return strategy.getMetadata(dto);
}
```

Strategies: column metadata, view metadata, reference metadata — each in its own class under `src/modules/override-info/strategies/`.

**C) WHERE Clause SQL:**

`WhereClauseStrategyRegistry` maps `BASIC / IN / NOT_IN` filter types to strategies that build different SQL predicate shapes. The query builder just asks for the right strategy — it doesn't construct SQL itself.

**D) ORDER BY SQL:**

`Distinct / Lateral / GroupBy` are three different SQL shapes for ordering over joins. Each is a strategy. The factory picks which one (see Factory section).

**E) Validation:**

`FieldPickerRegistry` and `OrderByRegistry` hold validation strategies that differ by query core type. Same call shape, different rules per core.

### Key Insight

> Strategy is how you replace a function with a dozen `if/else` branches with a lookup table of focused classes. Each algorithm is independent, testable in isolation, and addable without touching existing code.

### Interview Talking Points

- _"Strategy replaced growing `if/else` trees for hashing, metadata lookups, SQL shapes, and validation."_
- _"Open/Closed in practice: a new hashing algorithm is a new class and one registration line."_
- _"I often pair Strategy with Registry for the lookup and Factory for construction."_

---

## 5. Registry

### The Concept

A Registry is a central map: you register named implementations at startup, and look them up by name at runtime. It's a plugin table. Code that needs behavior doesn't import the behavior class — it asks the registry for it by key.

Think of it like a phone book. You don't hardwire every phone number into your brain — you look up the name and get the number at the moment you need it.

### The Pain Without It

The post-fetch pipeline is the clearest example. Some views need S3 signed URLs. Others need identity validation. The naive approach hardcodes this in `FormatService`:

```ts
// ❌ Without Registry — FormatService knows too much
if (view.postFetchType === 's3Integration') {
  row = await this.s3Service.signUrls(row);
} else if (view.postFetchType === 'validateUserIdentity') {
  row = await this.identityService.validate(row);
} else if (view.postFetchType === 'someNewThing') {
  // adding a new pipeline requires editing FormatService
}
```

`FormatService` now imports every pipeline service. It has to be modified every time a new pipeline type is added. It's testing a formatter — but it carries all the pipeline logic.

### The Solution

A Registry stores pipeline heads by string key. `FormatService` only knows: "get the pipeline for this key and run it." The registry handles the rest.

```ts
export class PipelineRegistry {
  private readonly registry = new Map<string, OperationPipelineTransformer>();

  register(key: string, transformer: OperationPipelineTransformer): void {
    this.registry.set(key, transformer);
  }

  get(key: string): OperationPipelineTransformer | undefined {
    return this.registry.get(key);
  }
}
```

`FormatService` becomes stable:

```ts
// ✅ FormatService — never changes when new pipelines are added
for (const pipelineKey of view.postFetchPipelineV2) {
  const pipeline = this.pipelineRegistry.get(pipelineKey);
  row = await pipeline.transform(row);
}
```

**All the Registries I built:**

|Registry|Key Type|What It Stores|
|---|---|---|
|`PipelineRegistry`|Pipeline name string|CoR chain heads|
|`IndexRegistry`|`IndexEnum` (GIST, BTREE…)|SQL DDL builder functions|
|`WhereClauseStrategyRegistry`|`BASIC / IN / NOT_IN`|WHERE predicate strategies|
|`HashAlgorithmRegistry`|`MD5 / SHA256`|Hashing functions|
|`SchemaRegistryService`|Protobuf schema name|Protobuf roots|
|`FieldPickerRegistry`|Query core type|Validation strategies|
|`OrderByRegistry`|Query core type|Order-by validation strategies|

### Key Insight

> A Registry is what turns a hard-coded list of `if/else` cases into a plugin system. Callers depend on the registry, not on any specific implementation. New implementations slot in without touching existing code.

### Interview Talking Points

- _"Registry is my plugin table: config keys resolve to live objects at runtime."_
- _"Factory often fills the registry at boot; the registry serves lookups during requests."_
- _"The clearest example: `FormatService` runs named pipelines without knowing what they do."_

---

## 6. Factory

### The Concept

A Factory centralizes object creation. Instead of callers doing `new ConcreteClass(dep1, dep2, ...)`, they ask the factory for "the thing I need for this situation." The factory knows which class to instantiate, how to configure it, and how to wire its dependencies.

The key benefit: construction complexity stays in one place. Callers stay ignorant of it.

### The Pain Without It

Building a post-fetch pipeline chain involves: reading a config array, resolving each class from Nest's DI container, linking them with `setNext`, and storing the head in a registry. If this happened at each call site:

```ts
// ❌ Without Factory — construction scattered everywhere
const s3Step = moduleRef.get(S3IntegrationThumbnailPreviewViewService);
const identityStep = moduleRef.get(AccessTokenExtractorPipeline);
s3Step.setNext(identityStep);
const pipeline = s3Step;
registry.register('myPipeline', pipeline);
```

This is repeated wherever a pipeline is needed. It leaks Nest internals, chain-linking logic, and registry knowledge into every consumer.

### The Solution

**A) Pipeline Factory** — builds entire CoR chains at module init from declarative config:

```ts
// The config is just a named list of classes — readable and safe
export const PIPELINES = {
  s3IntegrationThumbnail: [S3IntegrationThumbnailPreviewViewService],
  validateUserIdentity:   [AccessTokenExtractorPipeline],
} as const;
```

```ts
// PipelineFactory.onModuleInit() — runs once at startup
private buildChain(
  transformers: OperationPipelineTransformer[],
): OperationPipelineTransformer {
  for (let i = 0; i < transformers.length - 1; i++) {
    transformers[i].setNext(transformers[i + 1]);  // link the chain
  }
  return transformers[0];  // return the head
}
```

The factory: resolves each class from Nest DI → links them → stores the head in `PipelineRegistry`. By the time any request arrives, all pipelines are ready.

**B) Order-by Relation Factory** — returns the right SQL strategy by type:

```ts
// Callers ask for a type — factory decides the class
OrderByRelationFactory.get(OrderByRelationType.DISTINCT)
// → returns DistinctOrderByStrategy
```

**C) Iterator Factory** — simple, but still centralizes the BFS vs DFS decision:

```ts
get<T>(root: T, type: TraversalType, options?: TraversalOptions<T>): TreeIterator<T> {
  if (type === 'BFS') return new BfsIterator(root, options);
  return new DfsIterator(root, options);
}
```

**D) Query Factory** — `QueryFactory<T, K>` interface shared across `DataCore`, `CountCore`, `NestedSelectionDataCore`, and analytics cores. Nest DI selects which core a feature receives.

### Key Insight

> Factory is what keeps construction from leaking. When creating something involves multiple steps, DI resolution, and linking — that logic belongs in one place. Callers ask for "the right thing" and receive it already assembled.

### Interview Talking Points

- _"`PipelineFactory` is three patterns in one: Factory for creation, CoR for linking, Registry for storage."_
- _"Simple factory (`IteratorService`) hides a one-line decision. DI-backed factory (`PipelineFactory`) hides a whole bootstrap sequence. Both are valid."_
- _"Callers never `new` a pipeline handler. By the time a request arrives, it's already built."_

---

# Part C — Pipelines & Construction

---

## 7. Chain of Responsibility (CoR)

### The Concept

Chain of Responsibility is a pattern where a request passes through a linked sequence of handlers. Each handler decides to do its work and then optionally pass the request to the next handler in the chain. The sender doesn't know how many handlers there are or what they do.

Think of it like a production assembly line. A car chassis passes from station to station. Each station does one job. The chassis doesn't know how many stations there are.

### The Pain Without It

After a SQL query returns rows, some views need their data enriched. S3 storage keys need to be replaced with signed URLs. Certain columns need identity validation. Without CoR, this all lives inside `FormatService`:

```ts
// ❌ Without CoR — FormatService becomes a dumping ground
async formatRow(row: any, view: View): Promise<any> {
  if (view.hasS3Integration) {
    // 30 lines of S3 URL signing logic
    const signedUrl = await this.s3Client.getSignedUrl(...);
    row.thumbnail = signedUrl;
  }

  if (view.requiresIdentityCheck) {
    // 20 lines of token extraction and comparison
    const token = extractToken(row);
    if (token !== currentUser.id) throw new ForbiddenException();
  }

  // more transforms piling up here forever...
  return row;
}
```

The formatter now imports S3 clients, identity services, token utilities. Every new transform makes this function longer. You can't test S3 logic without instantiating a formatter. You can't reorder transforms without reading the whole function.

### The Solution

An abstract base class defines the chain interface. Each handler does one job and delegates to the next. The chain is assembled externally (by the Factory).

```ts
export abstract class PostFetchAbstractClass
  implements OperationPipelineTransformer {

  private nextTransformer: OperationPipelineTransformer | null = null;

  // Link the next handler — used by Factory at boot time
  setNext(transformer: OperationPipelineTransformer) {
    this.nextTransformer = transformer;
    return transformer;
  }

  // Every handler implements this — do your work, then pass it on
  abstract transform(value: any): any;

  // Helper to safely delegate to the next handler
  async passToNext(value: any): Promise<any> {
    if (this.nextTransformer) {
      return await this.nextTransformer.transform(value);
    }
    return value;  // end of chain — return as-is
  }
}
```

**A concrete handler — focused on one job:**

```ts
export class S3IntegrationThumbnailPreviewViewService
  extends PostFetchAbstractClass {

  async transform(row: any): Promise<any> {
    // sign S3 URLs for original, thumbnail, preview
    row.originalUrl  = await this.s3.sign(row.originalKey);
    row.thumbnailUrl = await this.s3.sign(row.thumbnailKey);

    return this.passToNext(row);  // hand off to next handler
  }
}
```

**`FormatService` becomes a thin coordinator:**

```ts
// ✅ FormatService — clean, stable, knows nothing about transforms
for (const pipelineKey of view.postFetchPipelineV2) {
  const pipeline = this.pipelineRegistry.get(pipelineKey);
  row = await pipeline.transform(row);
}
```

**Full runtime flow:**

```
PIPELINES config (declarative, at boot)
         │
         ▼
PipelineFactory links: Handler1 → Handler2 → Handler3
         │
         ▼
PipelineRegistry stores the head of each chain by name
         │
         ▼
FormatService: registry.get('s3IntegrationThumbnail').transform(row)
```

### Key Insight

> Chain of Responsibility turns a growing list of `if` blocks inside a formatter into a sequence of focused, independent, reorderable handlers. Adding a new transform is adding a new class — not editing the formatter.

### Interview Talking Points

- _"Classic CoR with `setNext` / `passToNext`. Each handler is single-responsibility."_
- _"Composed with Factory (builds the chain at boot) and Registry (looks it up by config key)."_
- _"`FormatService` has no idea what S3 or auth logic looks like. It just runs named pipelines."_

---

## 8. Builder

### The Concept

Builder constructs a complex object step by step. Each step adds one piece. The same construction process can produce different final objects depending on which steps are called and what strategies are selected inside them.

The insight is that complex object construction shouldn't be one massive function call or a god object that knows everything. It should be a composable pipeline of steps.

### The Pain Without It

SQL for nested relations with ORDER BY, EXISTS subqueries, and filter groups is not a one-liner. Without a Builder, this ends up as string concatenation scattered across services:

```ts
// ❌ Without Builder — string soup, impossible to maintain
let sql = `SELECT ${columns.join(', ')} FROM ${table}`;

if (joins.length > 0) {
  sql += ` LEFT JOIN ${joins.map(j => `${j.table} ON ${j.condition}`).join(' LEFT JOIN ')}`;
}

if (filters.length > 0) {
  sql += ` WHERE ${filters.map(f => `${f.column} ${f.op} ${f.value}`).join(' AND ')}`;
}

if (orderBy) {
  sql += ` ORDER BY ${orderBy.column} ${orderBy.direction}`;
}
```

This is one function that has to know the full SQL shape. It can't be partially reused. It can't be tested in pieces. When the SQL shape changes (and it always does), this function breaks.

### The Solution — Two Flavors

**A) Fluent Builder — `QueryBuilder` with staged interfaces**

Staged interfaces enforce step order at the type level. You can't call `.addFilter()` before `.from()` — the type system prevents it.

```ts
const query = builder
  .select()
  .from(fromArgs)
  .renderJoins(joinArgs)
  .addFilter(filterArgs)
  .addOuterReference(outerRefsArgs)
  .addOrderBy()
  .groupByPK()
  .limitOffset()
  .build();          // ← only callable at the end
```

Each step is focused on one SQL clause. Each is independently testable. Strategies can be swapped inside steps (e.g. DISTINCT vs LATERAL ORDER BY logic) without changing the builder interface.

**B) Injectable Buildables — `Buildable<T, K>`**

For finer-grained DI-composed SQL pieces:

```ts
export interface Buildable<T extends object, K> {
  build: (args: T) => K;
}
```

Small, injectable classes each build one SQL concern — JOIN compilation, CTE rendering, WHERE predicate grouping. A facade buildable orchestrates them:

```ts
// DataQueryRelationsCompilerBuildable assembles:
// 1. Collect relation refs
// 2. Reconcile against view metadata
// 3. Compile to JOIN SQL
```

`WhereClauseBuilder` uses both styles — it calls strategy-backed predicate builders and assembles their output into the final `WHERE (...)` clause.

**How they compose in the optimized nested-selection path:**

```
RelationLayer pipeline fills RelationContext (shared mutable state)
         │
         ▼
Render layers use fluent QueryBuilder to assemble each SQL section
         │
         ▼
WhereClauseBuilder merges EXISTS subqueries + basic predicates
         │
         ▼
CTE renderers assemble the full query with WITH clauses
```

### Key Insight

> Builder breaks "assemble this complex thing" into a sequence of focused, reusable, testable steps. The SQL shape can change by changing which steps are called or which strategy is selected inside a step — the caller's code barely moves.

### Interview Talking Points

- _"Fluent Builder for correlated subqueries. Injectable Buildables for DI-composed SQL pieces. Both patterns, different granularity."_
- _"Staged interfaces mean you can't call steps out of order — the type system enforces the contract."_
- _"Each step is independently testable. There's no god-object string builder to maintain."_

---

# Part D — Traversal & Interception

---

## 9. Iterator

### The Concept

Iterator provides a standard way to walk through a collection or structure — without the caller needing to know how the structure is organized internally. Is it a queue? A stack? A recursive tree? The caller doesn't care. They just call `hasNext()` and `next()`.

The benefit: the traversal algorithm is encapsulated in one place. BFS and DFS are different algorithms — but callers use the exact same interface.

### The Pain Without It

Views in this system form trees. A view can have `subviews`, each of which can have their own `subviews`. Multiple features need to walk this tree — building relation maps, creating indexes, collecting order-by columns.

Without an Iterator, each feature writes its own recursion:

```ts
// ❌ Feature A — its own recursive walk
function collectRelations(view: View, acc: Relation[] = []): Relation[] {
  acc.push(...view.relations);
  for (const sub of view.subviews ?? []) {
    collectRelations(sub, acc);  // DFS hardcoded
  }
  return acc;
}

// ❌ Feature B — another recursive walk, slightly different
function buildIndexMap(view: View, depth = 0): IndexMap {
  const map = {};
  if (depth > 3) return map;  // max depth hardcoded here too
  for (const sub of view.subviews ?? []) {
    Object.assign(map, buildIndexMap(sub, depth + 1));
  }
  return map;
}
```

Every feature duplicates the traversal. Max depth logic is scattered. Switching from DFS to BFS for performance requires changing every recursive function. Filtering nodes (skip disabled views) is implemented differently in each feature.

### The Solution

A generic `TreeIterator<T>` interface with BFS and DFS implementations. A factory selects the traversal type. Options like `maxDepth` and `filtration` are centralized.

```ts
export interface TreeIterator<T> {
  hasNext(): boolean;
  next(): T | undefined;
  peek(): T | undefined;
  reset(): void;
  readonly currentDepth: number;
  readonly visitedCount: number;
}
```

**BFS uses a queue; DFS uses a stack — but callers see the same interface:**

```ts
// ✅ Any feature — same API regardless of traversal type
const it = iteratorService.get(rootView, 'BFS', {
  filtration: (node) => node.isEnabled,
  maxDepth: 5,
});

while (it.hasNext()) {
  const node = it.next();
  // process node — no idea if this is BFS or DFS
}
```

The factory:

```ts
get<T>(root: T, type: TraversalType, options?: TraversalOptions<T>): TreeIterator<T> {
  if (type === 'BFS') return new BfsIterator(root, options);
  return new DfsIterator(root, options);
}
```

**Honest engineering note:** The module is fully built and exported. Some call sites in the codebase still use manual recursion — an incremental migration that wasn't completed. This is a real tradeoff story worth discussing: the pattern is ready, adoption is gradual.

### Key Insight

> Iterator lets you change _how_ a tree is traversed (BFS, DFS, limited depth, filtered) without changing the code that uses the traversal. Switching from DFS to BFS for performance is a one-word change at the call site.

### Interview Talking Points

- _"Iterator hides BFS vs DFS and options like `maxDepth` from callers."_
- _"Factory selects the concrete iterator — callers just ask for a traversal type."_
- _"Some call sites still use manual recursion — I can walk through why that tradeoff was made and how I'd prioritize migrating them."_

---

## 10. Proxy

### The Concept

A Proxy sits in front of a real object and intercepts calls to it. It can add caching, logging, metrics, timeouts, auth checks — all without the caller knowing the proxy exists, and without modifying the real object.

The key principle: the caller gets a reference and calls it as normal. Behavior is added by wrapping, not by changing.

### The Pain Without It

Consider DB queries that need metrics. Without a Proxy, instrumentation bleeds into every query:

```ts
// ❌ Without Proxy — metrics code in every caller
async function runQuery(sql: string) {
  const start = performance.now();
  try {
    const result = await pool.query(sql);
    counter('db.query.count').increment();
    histogram('db.query.duration').record(performance.now() - start);
    return result;
  } catch (err) {
    counter('db.query.error').increment();
    throw err;
  }
}
```

This block appears everywhere a query is made. Change the metrics library and you touch every query site. Add a timeout and you touch every query site. Test business logic and you deal with metrics setup.

### The Solution

**A) Metrics Proxy — `DbMetricsService.observeQuery`**

Wrap any execution callback — metrics happen automatically:

```ts
async observeQuery<T>(
  meta: DbQueryMeta,
  execute: () => Promise<T>,
): Promise<T> {
  const start = performance.now();
  try {
    return await execute();                     // the real work
  } catch (error) {
    counter('db.query.error').increment();
    throw error;
  } finally {
    histogram('db.query.duration').record(performance.now() - start);
    counter('db.query.count').increment();
  }
}
```

Callers just wrap their query in a lambda. Metrics are invisible to them.

**B) Caching Proxy — `MemoryCacheService.memCached`**

For views flagged `isMemCached`: build a cache key → on hit, return cached rows → on miss, run the real query and cache the result:

```ts
// Caller asks for query results — doesn't know if cache was used
const rows = await memoryCacheService.memCached(cacheKey, () =>
  dbService.query(sql)
);
```

The `() => dbService.query(sql)` lambda is only called on a cache miss. The caller is unaware of the cache decision.

**C) Service-layer Proxy — `DBService.query`**

`DBService` itself acts as a proxy over the raw DB pool. It centralizes: SQL preparation, Sentry span creation, metrics observation via `observeQuery`, and error mapping. Feature modules call `dbService.query()` — they never touch the pool directly.

**D) HTTP Proxies — Nest Interceptors**

- `TimeoutInterceptor` — enforces a maximum request duration; the route handler is unmodified
- `CacheInterceptor` — caches HTTP responses when metadata flags are present

All four follow the same principle: **add behavior by wrapping, not by modifying**.

### Key Insight

> Proxy is the pattern for cross-cutting concerns — things that need to happen everywhere but belong to no single domain. Cache, metrics, and timeouts are infrastructure concerns. Proxy keeps them out of business logic entirely.

### Interview Talking Points

- _"Proxy adds behavior around the real subject without the caller knowing — cache, metrics, timeout."_
- _"The distinction from Decorator matters: Decorator is about stacking same-interface behavior; Proxy is about controlling access to something."_
- _"`observeQuery` is a textbook instrumentation proxy. Any query function passed to it gets timed and counted."_

---

# Part E — How Patterns Compose

Real systems don't use patterns in isolation. Here are the three clearest composition stories from this codebase.

---

## Story 1 — Post-fetch Row Enrichment

**Patterns: Factory + Registry + Chain of Responsibility**

**The situation:** After the SQL query returns rows, some views need S3 signed URLs. Others need identity validation. More transforms will be added. `FormatService` shouldn't know about any of them.

**The design:**

```
PIPELINES config (declarative class lists per pipeline name)
         │
         ▼
PipelineFactory.onModuleInit()
  ├── Resolves each class from Nest DI
  ├── Links them with setNext (CoR chain)
  └── Stores the head in PipelineRegistry
         │
         ▼
FormatService (at request time)
  └── registry.get(pipelineKey).transform(row)
```

**The outcome:** Adding a new row transform = write a new handler class + add it to the `PIPELINES` config. `FormatService` never changes. The chain order is controlled by the config array order.

---

## Story 2 — Logging Destinations

**Patterns: Dependency Inversion + Adapter + Decorator**

**The situation:** Services need logging. The logging destination changes per environment. Vendor SDKs should not appear in business logic.

**The design:**

```
ILogger (the contract — defined once, never changes)
   │
   ├── LoggerAdapter        (translates ILogger → Nest Logger API)
   ├── SentryLoggerAdapter  (translates ILogger → Sentry SDK API)
   └── CompositeLogger      (implements ILogger, fans out to both)

Nest DI tokens:
  CONSOLE_LOGGER   → LoggerAdapter
  REMOTE_LOGGER    → SentryLoggerAdapter
  COMPOSITE_LOGGER → CompositeLogger([Sentry, Console])
```

**The outcome:** Services inject a token and call `ILogger`. The DI wiring controls what happens. New logging destination = new adapter + one token registration. No service changes.

---

## Story 3 — Nested-selection SQL Generation

**Patterns: Builder + Strategy + Factory + Layer Pipeline**

**The situation:** Ordering and filtering across nested relations needs different SQL shapes — DISTINCT, LATERAL, GROUP BY — and a multi-phase context preparation before SQL is rendered.

**The design:**

```
RelationLayer pipeline (CoR-like, ordered steps)
  └── Each layer reads/writes a shared RelationContext
         │
         ▼
OrderByRelationFactory
  └── Picks Strategy (Distinct | Lateral | GroupBy) by relation type
         │
         ▼
Strategy uses fluent QueryBuilder
  └── .select().from().renderJoins().addFilter().addOrderBy().build()
         │
         ▼
WhereClauseBuilder
  └── Assembles EXISTS subqueries + basic predicates into WHERE (...)
```

**The outcome:** A new SQL ordering style = one new strategy class + one factory branch. Layers and builders stay untouched.

---

# Quick Reference

## Pattern Summary Table

|Pattern|What It Does|The Pain Without It|Where I Used It|
|---|---|---|---|
|**DIP**|High-level code depends on abstractions, not implementations|Changing a vendor touches every service|`ILogger`, `IMetricsProvider`, all strategy interfaces|
|**Adapter**|Translates one interface to another|Incompatible vendor APIs leak into callers|`LoggerAdapter`, `SentryLoggerAdapter`, `SentryMetricServiceAdapter`|
|**Decorator**|Wraps same-interface objects to stack behavior|Callers need environment-aware branching|`CompositeLogger` — console + Sentry in one|
|**Strategy**|Swappable algorithms behind one interface|`if/else` trees that grow with every new case|Hashing, WHERE, ORDER BY, metadata, validation|
|**Registry**|Named map of implementations, resolved at runtime|Hard-coded lookups in every consumer|`PipelineRegistry`, `IndexRegistry`, all strategy registries|
|**Factory**|Centralizes object creation and wiring|Construction complexity leaks to call sites|`PipelineFactory`, `OrderByRelationFactory`, `IteratorService`|
|**CoR**|Request passes through a chain of independent handlers|All transforms pile up in one formatter|`post-fetch-operations/` + `FormatService`|
|**Builder**|Complex object assembled step by step|String concatenation SQL — fragile and untestable|Fluent `QueryBuilder`, DI `Buildable` pieces|
|**Iterator**|Unified interface for walking structures|Every feature reimplements its own traversal|`BfsIterator`, `DfsIterator`, `IteratorService`|
|**Proxy**|Wraps a real object to add cache, metrics, or access control|Cross-cutting concerns duplicated across every caller|`memCached`, `observeQuery`, `DBService`, interceptors|

---

## STAR Interview Template

Use this frame for any pattern when answering "tell me about a time you improved a system":

> **Situation:** Describe the pain — what was growing, breaking, or becoming hard to change.
> 
> **Task:** What property did the system need? (Extensibility, testability, isolation from vendors, etc.)
> 
> **Action:** Name the pattern, where you introduced it, and what patterns it was composed with.
> 
> **Result:** Quantify the improvement — "new X = one class + one line, no changes to existing code."

---

## Suggested Review Order

For interview prep, read in this sequence — each builds on the previous:

1. **Part A end-to-end:** `ILogger` interface → `LoggerAdapter` → `SentryLoggerAdapter` → `CompositeLogger` → `logger.module.ts`
2. **Post-fetch story:** `PostFetchAbstractClass` → `PipelineFactory` → `PipelineRegistry` → one concrete handler → `FormatService.chainPipeline`
3. **One Strategy area you know best:** either hashing or order-by SQL
4. **Fluent `QueryBuilder`** + one RelationLayer render file
5. **Rehearse the three Part E composition stories out loud** — two minutes each

---

_Personal engineering reference — built during work on Zeustra Advanced Filters (Troodon)._ _Every pattern here was introduced to solve a real, visible problem. None were added speculatively._