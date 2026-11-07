---
name: api-design
description: Design public and cross-boundary API contracts without implementing them. Covers HTTP/REST, GraphQL, public package interfaces, exported types, component contracts, and service/module boundaries. Produces a concise design artifact under .api-design/, prioritizes repository conventions and compatibility, and leaves implementation to another workflow.
---

# API Design

Design or review a contract consumed across an ownership or trust boundary
before it is implemented: HTTP and GraphQL APIs, public C# and TypeScript
types, package and SDK interfaces, component props and events, and
service or module boundaries. Private implementation detail is out of scope
unless it constrains the contract.

The design answers what consumers can do, what behavior is promised, which
errors and edge cases are part of the contract, and what compatibility
constraints follow once it is public. Which files to create, how to wire DI,
and which storage to use belong to `plan-implementation` or `delegate-task`.

## Repository conventions first

Read `AGENTS.md`, `CLAUDE.md`, and architecture notes for the affected
boundary, then the existing conventions it touches: naming, resource
structure, status codes, error shapes, DTO patterns, serialization,
pagination, filtering, auth, versioning, export patterns, and compatibility
rules. Repository consistency beats generic best practice unless a convention
causes a concrete problem. Do not redesign unrelated contracts.

## What to decide

**HTTP**: method semantics, routes and resources, request and response shapes,
status codes, headers, authentication and authorization expectations,
filtering, sorting, pagination, caching and conditional requests,
idempotency and retries, concurrency and conflicts, and versioning only when
genuinely needed. Common REST conventions are defaults, not laws: do not force
plural resources, `PATCH` over `PUT`, pagination on every list, one universal
status mapping, or versioning from day one.

**Libraries and interfaces**: surface size, naming, type safety, nullability,
sync versus async, cancellation, ownership and lifetime, exception semantics,
extension points, external implementers, serialization or ABI concerns, and
misuse resistance. Impose language-specific patterns only where they improve
the contract.

**Errors**: every operation gets clear failure semantics in the repository's
existing error model: status semantics, error shape, and client versus server
failures for HTTP; exceptions or result types, cancellation, invalid input, and
missing values for code. Do not invent a new error model when one exists.

**Validation** belongs at trust boundaries: requests, message consumers,
deserialized external data, public package input, user-controlled
configuration. Internal invariant checks stay valid when they protect real
assumptions.

**Idempotency, retries, and concurrency**: define what callers observe:
whether retries are safe, what a duplicate request means, what an idempotency
key promises, and how optimistic concurrency, stale updates, and conflicting
mutations behave. Leave storage, locks, and middleware to implementation
planning.

**Compatibility**: classify changes as additive, breaking, or potentially
breaking, for consumers, external implementers, serialized and stored data,
plugins, SDKs, package exports, HTTP clients, and component consumers. Prefer
additive evolution, and never hide a breaking change behind implementation
language.

**Surface**: the smallest coherent surface that meets current requirements.
Challenge redundant operations, ambiguous convenience methods, overlapping
parameters, leaked implementation detail, and two ways to do one thing. Add
versioning, pagination, hooks, abstraction layers, optional parameters, or
generic escape hatches only for current requirements or realistic known
evolution.

## Before finalizing

Challenge the design: can callers misuse it easily, is any behavior ambiguous,
are failure, retry, and concurrency behavior defined where they matter, is
compatibility clear, is validation placed sensibly, and does any abstraction
exist only for a hypothetical future? Revise where the answer exposes a real
weakness.

Never invent product or domain decisions. Record one the design cannot settle
under `Open questions` (`Should duplicate submissions return the original
result or a conflict?`) and keep specifying the independent parts.

Use authoritative evidence when framework, protocol, or specification behavior
affects the design, through `source-research` for anything substantial. Use
sub-agents only for genuinely independent boundaries, such as an HTTP contract
alongside an SDK surface.

## Artifact

Write to `.api-design/<descriptive-topic>.md`, such as
`.api-design/idempotency.md`. If a design for the topic exists, update it:
keep still-valid decisions, replace outdated assumptions, and never overwrite
an unrelated design.

Use only the sections that help, typically Goal, Proposed contract, Behavior,
Errors, Compatibility, Design decisions, and Open questions. Show concrete
contract detail: methods, routes, shapes, headers, and statuses for HTTP;
public signatures or type shapes for packages.

```text
POST /sessions
Idempotency-Key: <opaque value>
```

```csharp
public interface ICacheStore
{
    ValueTask<T?> GetAsync<T>(...)
}
```

Stop at the contract. "Create Foo.cs, register service X, add migration Y" is
an implementation plan. Keep rationale short and concrete, without API-design
essays, attribution, or commentary on the design being clean or minimal.

## Boundaries

The only persistent change is the `.api-design/` file. Leave source, tests,
configuration, and Git state alone, and protect unrelated uncommitted work.

## Final summary

The topic, the compatibility classification when relevant, the important open
questions, and the artifact path, without repeating the design.
