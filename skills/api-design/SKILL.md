---
name: api-design
description: Design public and cross-boundary API contracts without implementing them. Covers HTTP/REST, GraphQL, public package interfaces, exported types, component contracts, and service/module boundaries. Produces a concise design artifact under .api-design/, prioritizes repository conventions and compatibility, and leaves implementation to another workflow.
disable-model-invocation: true
---

# API Design

Use this skill to design or review a public or cross-boundary contract before implementation.

This includes:
- HTTP / REST APIs
- GraphQL contracts
- public C# interfaces and classes
- exported TypeScript APIs
- package/library contracts
- SDK interfaces
- component props/events
- service/module boundaries
- other contracts consumed across ownership or trust boundaries.

Private/internal implementation details are normally out of scope unless they materially constrain the public contract.

This skill is analysis/design only.

It must not implement the design.

## Purpose

The goal is to answer:

```text
What should consumers be able to do?
What behavior are we promising them?
What errors and edge cases are part of the contract?
What compatibility constraints will exist once this is public?
```

It should not answer:

```text
Which files should be created?
How should DI be wired?
Which storage implementation should be used?
How should this be coded?
```

Use `plan-implementation` or `delegate-task` for execution.

## Core workflow

```text
Understand requested capability
        ↓
Inspect existing repository conventions
        ↓
Identify public/cross-boundary contract
        ↓
Design minimal usable surface
        ↓
Define behavior / errors / compatibility
        ↓
Challenge misuse and ambiguity
        ↓
Record unresolved product/domain decisions
        ↓
Persist .api-design/<topic>.md
```

## Repository conventions first

Before proposing a contract, read applicable repository instructions such as:
- `AGENTS.md`
- `CLAUDE.md`
- architecture/design notes that govern the affected boundary.

Then inspect relevant existing conventions.

Examples:
- naming
- endpoint/resource structure
- status codes
- error shapes
- DTO/request/response patterns
- serialization formats
- pagination
- filtering/sorting conventions
- authentication/authorization patterns
- versioning
- public interface naming
- package export patterns
- component props/events
- compatibility rules.

Prefer repository consistency over generic best practices unless existing conventions create a concrete problem.

Do not redesign unrelated contracts merely because another style is preferred.

## HTTP / REST design

When the contract is HTTP-based, explicitly consider where relevant:

- HTTP method semantics
- route/resource shape
- request body/query/path parameters
- response shapes
- status codes
- headers
- authentication expectations
- authorization expectations
- filtering
- sorting
- pagination
- caching semantics
- conditional requests
- idempotency/retry behavior
- concurrency/conflict behavior
- versioning when genuinely necessary.

Treat common REST conventions as defaults, not laws.

Do not blindly enforce:
- plural resource names
- `PATCH` over `PUT`
- pagination on every list
- one universal status-code mapping
- versioning from day one.

Actual semantics and repository consistency win.

## Non-HTTP APIs

For library/package/interface contracts, consider where relevant:

- public surface size
- naming
- type safety
- nullability
- synchronous vs asynchronous behavior
- cancellation
- ownership/lifetime
- error/exception semantics
- extension points
- compatibility for external implementers
- serialization or ABI concerns
- misuse resistance.

Do not impose language-specific patterns unless they materially improve the actual contract.

## Compatibility

Treat public behavior as a contract.

When relevant, classify a design change as:

```text
Additive
Breaking
Potentially breaking
```

Consider compatibility for:
- existing consumers
- external implementers
- serialized data
- stored data
- plugins/extensions
- SDKs
- public package exports
- HTTP clients
- component consumers.

Prefer additive evolution when it satisfies the requirements.

Do not hide breaking changes behind implementation language.

## Errors and failure behavior

Every public operation should have clear failure semantics where relevant.

Follow the repository's existing error conventions.

For HTTP APIs, define:
- status semantics
- response/error shape
- client vs server failures
- conflict behavior where relevant.

For code/library APIs, define:
- exceptions
- result/error types
- cancellation
- invalid-input behavior
- missing-value behavior.

Do not invent a new error model when the repository already has one.

## Validation

Prefer validation at trust/system boundaries.

Examples:
- HTTP requests
- message consumers
- deserialized external data
- public package input
- cross-service contracts
- user-controlled configuration.

Do not duplicate validation throughout internal layers without demonstrated need.

Internal invariant checks are still valid when they protect meaningful assumptions.

## Idempotency and retries

Keep idempotency design at the contract level.

When relevant, define:
- whether retries are safe
- what duplicate requests mean
- whether equivalent repeated input returns the original result or another outcome
- concurrency behavior
- conflicting reuse behavior
- what accepting an idempotency key promises to callers.

Do not design:
- Redis storage
- database tables
- endpoint filters
- locks
- middleware internals
- implementation-specific persistence.

Those belong to implementation planning.

## Concurrency and conflicts

When concurrent operations can affect correctness, define observable contract behavior.

Examples:
- duplicate create requests
- optimistic concurrency
- stale updates
- conflicting mutations
- retry races.

Do not leave externally visible concurrency semantics accidental when they matter.

## Public surface minimization

Prefer the smallest surface that satisfies current requirements cleanly.

Challenge:
- redundant operations
- convenience methods that create ambiguity
- overlapping parameters
- unnecessary extension points
- leaked implementation details
- duplicate ways to achieve the same behavior.

Do not optimize for fewer methods blindly. Optimize for a clear, coherent contract.

## No speculative future-proofing

Do not add:
- versioning
- pagination
- extension hooks
- abstraction layers
- optional parameters
- polymorphism
- generic escape hatches

solely because they might be useful later.

Design for current requirements and realistic known evolution.

## External research

Do not perform broad external research by default.

If framework, protocol, language, or specification behavior materially affects the design, use authoritative evidence.

Prefer `source-research` for substantial external investigation rather than turning this workflow into a research session.

## Sub-agents

Do not spawn sub-agents for a small single-boundary design by default.

Use isolated sub-agents only when the task contains genuinely independent public boundaries.

Example:

```text
HTTP API contract → sub-agent A
SDK/package API → sub-agent B
```

Keep handoffs concise.

## Challenge pass

Before finalizing the design, challenge it.

Ask:

- Can callers misuse this contract easily?
- Is any behavior ambiguous?
- Are failure modes clear?
- Is the public surface larger than necessary?
- Does it expose implementation details?
- Is retry behavior defined where relevant?
- Is concurrency behavior defined where relevant?
- Is compatibility clear?
- Are validation responsibilities placed sensibly?
- Are there multiple ways to perform the same operation unnecessarily?
- Is any abstraction included only for hypothetical future needs?

Revise the design when these checks reveal a real weakness.

## Unresolved decisions

Do not invent product/domain decisions.

If the design cannot be finalized responsibly without a real decision, record it under `Open questions`.

Example:

```text
Open question:
Should duplicate submissions return the original result or a conflict?
```

Do not block the rest of the design when independent parts can still be specified.

## Persistence

Persist designs under:

```text
.api-design/<descriptive-topic>.md
```

Examples:

```text
.api-design/idempotency.md
.api-design/cache-store.md
.api-design/pagination.md
.api-design/public-event-contract.md
```

Use concise descriptive slugs.

## Existing designs

Before creating a new design file, check whether the same topic already exists.

If matching design material exists:
- update it when clearly appropriate
- preserve still-valid decisions
- replace outdated assumptions
- retain compatibility/history context when useful.

Do not create duplicate design files unnecessarily.

Do not overwrite unrelated designs.

## Report structure

Use only sections that materially help.

Typical structure:

```md
# <API / contract topic>

## Goal
...

## Proposed contract
...

## Behavior
...

## Errors
...

## Compatibility
...

## Design decisions
...

## Open questions
...
```

For HTTP APIs, include concrete contract details such as methods, routes, request/response shapes, headers, and status behavior where useful.

For package/interface APIs, include concrete public signatures or type shapes where useful.

Do not turn the artifact into an implementation plan.

## Suggested direction vs implementation

Concrete contract examples are allowed.

For example:

```text
POST /sessions
Idempotency-Key: <opaque value>
```

or:

```csharp
public interface ICacheStore
{
    ValueTask<T?> GetAsync<T>(...)
}
```

But do not continue into:

```text
Create Foo.cs
Register service X
Add migration Y
Implement middleware Z
```

Implementation belongs to another workflow.

## Writing style

Follow the standard concise, human style used by the other skills.

Prefer:
- short direct sections
- concrete behavior
- concise rationale
- explicit compatibility notes
- clear unresolved questions.

Avoid:
- giant boilerplate templates
- generic API-design essays
- implementation diaries
- repeated best-practice explanations
- unnecessary headings
- generated-by/tool attribution
- narrating that the design is minimal, focused, clean, or elegant.

## Repository modification

This skill is analysis-only.

The only persistent repository changes are files under:

```text
.api-design/
```

Do not:
- modify source code
- modify tests
- modify configuration
- commit
- push
- create or delete branches
- reset
- stash
- clean
- amend
- rebase
- refactor existing implementation.

Protect unrelated user work and never overwrite unrelated changes.

## Final summary

Keep the parent-facing result compact.

Useful information:
- design topic
- compatibility classification when relevant
- important unresolved decisions
- artifact path.

Do not repeat the entire design when it already exists in the artifact.

## Token efficiency

- Inspect only repository conventions relevant to the contract.
- Avoid broad codebase exploration.
- Use sub-agents only for genuinely independent boundaries.
- Reuse existing design artifacts.
- Keep rationale concise.
- Do not perform external research unless it materially affects correctness.
- Do not turn design analysis into implementation planning.
