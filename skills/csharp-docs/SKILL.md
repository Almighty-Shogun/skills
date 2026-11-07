---
name: csharp-docs
description: >-
  Write, update, or verify C# XML documentation (///) in a compact,
  accuracy-first style. Default mode writes or updates docs; --verify audits
  them read-only; --verify --fix applies approved documentation-only fixes.
  Never changes code to make docs true, and never documents a defect as
  designed behavior.
argument-hint: "[path ...] [--verify [--fix]]"
---

# C# Docs

Author, update, and verify C# XML documentation. This governs `///` API
documentation, not ordinary comments: a general "avoid comments" rule does not
suppress XML docs where the project documents its members.

## Modes

```text
csharp-docs [path ...]                  write or update docs
csharp-docs [path ...] --verify         read-only audit
csharp-docs [path ...] --verify --fix   audit, then apply approved fixes
```

`--fix` is valid only with `--verify`. It expresses intent to fix, but the
findings are still shown and the corrections approved before anything is
written. A user asking for fixes after a plain `--verify` report enters the
same fix phase.

Targets: a file is that file; a directory is every `.cs` file beneath it,
excluding generated and build output such as `bin` and `obj`; no argument resolves the relevant project from the
repository, and several plausible projects with no clear target means asking.

## Precedence

1. **Accuracy**: every sentence is true of the current code.
2. **Completeness**: every required member and applicable tag is present.
3. **Brevity**: the shortest block that is still true and complete.
4. **Shape**: tag order, grouping, wrapping, and voice.

Never sacrifice a higher rule for a lower one. Brevity never removes a required
tag, nor the one clause a caller needs to avoid misusing the member.

## Project standard

Read `AGENTS.md`, `CLAUDE.md`, `.editorconfig`, and any documentation
conventions, then a few documented files in the same package. Where this skill
allows a choice of phrasing, follow the form the package already uses.

When the project documents members, document every member in scope to the same
standard, private ones included: types, methods, properties, fields,
constructors and primary-constructor parameters, interfaces, attributes, enum
members, and extension-block receivers.

Use the project's `<author>` and `<since>` values. Without a more specific
value, use `<author>Almighty-Shogun</author>` and `<since>Unreleased</since>`.
`<since>` records when a member was added; changing its signature or body
leaves it alone.

## Verify before writing

Every sentence is a claim about the code. Read the member body, and open the
callers, collaborators, or framework behavior a claim depends on before making
it. Write only what was established; a short correct block is finished
documentation. Never invent constraints, failure modes, rationale, guarantees,
ownership semantics, or platform behavior.

A claim such as "the caller already validates this" or "this is registered as a
singleton" depends on another component: verify it there, or leave it out.

## House style

Documentation is compact and focused. Each tag states one fact in one short
line. A block grows only when a caller would otherwise misuse the member, and
that detail goes in a short `<remarks>`, not in the summary or the parameters.

| Tag | Content |
|---|---|
| `<summary>` | One sentence: what the member does, or what the value is. |
| `<typeparam>` | A short noun phrase: `The application's user entity.` |
| `<param>` | A short noun phrase: `The cancellation token.` Add a clause only for behavior the caller must know: `The refresh token; unknown tokens are ignored.` |
| `<returns>` | A short noun phrase: `The service collection.` Name a meaningful null or empty case in the same phrase: `The resolved application, or null when scoping is disabled.` |
| `<exception>` | The condition, as a short clause: `The account is locked.` |
| `<remarks>` | One to three plain sentences, only for what the other tags cannot hold. |

A plain noun phrase is finished documentation for a parameter whose role is
obvious, such as a cancellation token, service collection, configuration, HTTP
context, or logger.

Keep out of the docs: rationale and design argument ("so a token minted for one
application cannot authenticate another"); framework internals beyond the one
clause that explains a non-obvious choice; package wiring, registration order,
and setup walkthroughs, which belong on the documentation site; lists of
callers; what a consumer does with a value; defaults already visible in the
signature; and provenance, such as how a defect was found or which version it
failed under.

Verbose, the style this standard replaces:

```csharp
/// <summary>
/// Resolves the app for the current request, refusing rather than guessing when the host is unknown.
/// </summary>
///
/// <returns>The resolved app, or <c>null</c> when scoping is off and there is nothing to resolve.</returns>
///
/// <exception cref="UnknownAppException">
/// Scoping is active and the request cannot be attributed: either the host maps to no configured application, or
/// there is no request in flight at all, as when a background job mints a token. The second carries a <c>null</c>
/// host rather than the one that failed.
/// </exception>
```

Compact:

```csharp
/// <summary>
/// Resolves the application for the current request.
/// </summary>
///
/// <returns>The resolved application, or null when scoping is disabled.</returns>
///
/// <exception cref="UnknownAppException">
/// Scoping is enabled and the application cannot be resolved.
/// </exception>
```

### Remarks

`<remarks>` holds what a caller or maintainer needs and the other tags cannot
carry: a caller obligation (`Callers must validate account status before
creating a session.`), a transaction or concurrency consequence (`This method
saves changes but does not start its own transaction.`), a non-obvious edge
case, the one-clause reason for a choice a future editor might undo (`Uses the
exception handler feature because exception middleware may clear the current
endpoint.`), or a pointer to the alternative (`Use <see cref="RegisterRequest"/>
for public registration.`). One plain sentence per point; never praise the
design as clean, maintainable, or flexible.

### Whose contract is it

Before writing a sentence, ask whether it would need editing if a different
type changed. If so, it documents that type: put it there and reference it with
`<see cref="..."/>` when the reader needs the pointer. A copy of another
member's contract goes stale with no build to report it.

A data carrier (DTO, request, response, options, entity property) documents
what the value is, plus the constraints on the property itself, folded into the
summary: `The access token lifetime in minutes (1–1440).` or `The device
identified from the user agent, if available.` What a service does with the
value belongs on that service. A contract of the type as a whole, such as which
rules an endpoint enforces, is stated once in the type's `<summary>`, not in
fragments on every property.

### Exceptions

`<exception>` is required for every exception that can escape. Trace direct
throws, callees, reflection or invocation wrappers, validation helpers, and
parsing calls; do not infer exceptions from names. Use the inline form when it
fits on one line, and put several conditions for one exception in a single tag.

### Voice

- Methods use third-person singular: `Registers`, `Resolves`; a `Try` method
  `Attempts to resolve...`.
- Types: `Represents...` for records, entities, settings, and requests;
  `Provides...` for extension classes and blocks; `Defines...` for enums;
  `Thrown when...` for exception types; what they do for services and handlers.
- Properties state the value with no `Gets`/`Sets` prefix; a boolean property
  starts `Indicates whether...`.
- A boolean-returning method summary starts `Determines whether...`; its
  `<returns>` is `Whether...` or `True if...; otherwise, false.`, whichever the
  package uses.
- No imperative voice, no "This method..." opening, no second summary sentence.

### Returns and defaults

`<returns>` is a noun phrase. Fluent receiver returns use one fixed phrase per
receiver: `The service collection.` For a result-less `Task`, follow the
package: omit `<returns>`, or write `A task that completes when the response
has been written.` For `Task<T>`, describe the result.

Do not restate a visible signature default. When a default changes behavior the
caller cannot infer, say what it causes: `or null for one second`.

### References

`<see cref>` for resolvable types and members, `<paramref>` and
`<typeparamref>` for the current member's parameters, `<c>` for literals or a
name from another overload. Disambiguate overloaded crefs to avoid `CS0419`,
and confirm crefs into extension blocks with the build or generated XML.

Write `/// <inheritdoc />` alone, with no `<author>` or `<since>` below it.

Document an `extension` block's receiver with a `<param>` on the block: `The
HTTP request.`, plus an ownership consequence such as whether a stream stays
open, when a caller must know it.

## Shape

Tag order, omitting what does not apply:

```text
<summary> <typeparam> <param> <returns> <exception> <remarks> <author> <since>
```

`<param>` tags follow signature order. Separate groups with a bare `///` line:
`<typeparam>` and `<param>` form one group, `<author>` and `<since>` the last,
and every other tag type its own.

```csharp
/// <summary>
/// Creates an authenticated session and issues access and refresh tokens.
/// </summary>
///
/// <param name="user">The user receiving the session.</param>
/// <param name="app">The application scope, or null when unscoped.</param>
/// <param name="cancellationToken">The cancellation token.</param>
///
/// <returns>The issued tokens and their associated user.</returns>
///
/// <remarks>
/// This method saves changes but does not start its own transaction.
/// </remarks>
///
/// <author>Almighty-Shogun</author>
/// <since>4.0.0</since>
```

Use the one-line form whenever it fits within `.editorconfig`
`max_line_length`, counting indentation, `///`, and tags; otherwise put the
text on its own line between the opening and closing tag. A tag that needs
several wrapped lines is usually too long: shorten it before wrapping. Do not
reflow existing text beyond what the line limit requires.

## Changing documented code

When a documented member's body changes, re-read its whole block and correct
every sentence the change falsified, not only what the compiler forces. Search
other files for docs that reference the member and fix stale claims there.
Deleting a stale sentence is always allowed; do not invent a replacement to
preserve block length.

## Code is the fact

When code and documentation disagree, the documentation is wrong. Never change
code to make a doc block true. If the documented behavior looks preferable,
report the mismatch, argue for the change on its own merits, and treat it as a
separate task that needs explicit approval.

## A defect is not a design

A doc block describes what a member does; it is never where a defect is
recorded. A verified sentence can still be the wrong sentence: when documenting
a member reveals behavior no one chose, do not write the sentence that
legitimizes it. Describing broken behavior in the register of intended behavior
turns a bug into a specification, and the next reader, including a later bug
hunt, cannot tell the two apart.

Signals: the sentence explains why something can never work, why a value never
arrives, why a supported-looking input is refused, or reads as rationale for
behavior a caller would report as broken.

Leave the block short and report the defect to the user. Silently documenting
it is wrong, and so is silently dropping it.

## Build

Build the relevant project after writing docs, and treat `CS1573`, `CS1591`,
`CS1574`, and `CS0419` as documentation failures, not tolerable warnings.
Inspect the generated XML when cref resolution is in doubt. A clean build does
not prove a sentence is true.

## Boundaries

Authoring edits XML documentation blocks only; `--verify` edits nothing until
fixes are approved. Leave implementation code and Git state alone, and protect
unrelated uncommitted work.

# Verification mode

These rules extend the authoring standard for `--verify`.

Verification stays independent of authoring: never trust a previous authoring
pass's conclusions, and prefer fresh verifier sub-agents for that reason. Split
large targets by project or directory across isolated sub-agents, giving
each only this standard and its scope. Spot-check their actionable findings and
downgrade anything without evidence to `UNCHECKED`. Never describe a partial
pass as complete.

## Dimensions

**Accuracy.** Read each block against the implementation it sits on and judge
each claim separately. Ask two questions: is it true, and does it describe
something anyone intended.

| Verdict | Meaning |
|---|---|
| `CONFIRMED` | Checked and true. Not normally reported. |
| `FALSE` | Contradicted by the code. |
| `STALE` | Describes behavior that no longer exists. |
| `UNINTENDED` | True of the code, but describes a defect as designed behavior. |
| `JUSTIFICATION` | Argues the design is good without a checkable behavior. Reasoning that contradicts the code is `FALSE` or `STALE` instead. |
| `UNVERIFIABLE` | The codebase cannot settle a checkable claim. |
| `UNCHECKED` | Verification did not inspect what the claim depends on. |

`UNVERIFIABLE` and `UNCHECKED` are different and never merged.

**Completeness.** `MISSING DOC` for a member that requires docs and has none;
`MISSING TAG` for an absent applicable tag, including escaping exceptions,
null/empty/failure returns, and parameters. List missing docs individually, not
as a count.

**Brevity.** `VERBOSE` for a block carrying what the house style keeps out: a
multi-sentence summary, rationale, wiring or caller lists, consumer behavior on
a data carrier, a restated default, a tag running to several lines of
explanation, or narrating remarks. A plain noun phrase such as `The
cancellation token.` is the house style, not a finding.

**Shape.** Tag order, grouping, parameter order, one-line form and wrapping,
voice, noun-phrase returns, reference tags, `<author>`/`<since>` (none below
`<inheritdoc />`), the extension receiver `<param>`, and correctly targeted
crefs.

## Before reporting FALSE or STALE

A false positive gets a correct doc "fixed" into a wrong one. First ask what
would make the claim true and check it: an earlier guard, a caller or factory
enforcing it, a different code path, or wording that is merely imprecise.
Report only after that defense fails, with the decisive evidence.

`<inheritdoc />` members stay in scope: resolve the inherited contract, check
each implementation honors it, and report a divergence against the
implementation, naming the inherited source. For cross-component claims, open the other
component. Check exceptions in both directions, flags in both states, and that
a cref which resolves still points at the right member after a refactor. Git
history is a signal, not proof.

## Report

Group by file, ordered: `FALSE`, `STALE`, `UNINTENDED`, `JUSTIFICATION`,
`UNVERIFIABLE`, `UNCHECKED`, `MISSING DOC`, `MISSING TAG`, `VERBOSE`, then shape
and compiler findings. Within a category, lead with what is most likely to
mislead callers or maintainers.

```text
path/Thing.cs:60  FALSE
  Claim:     "Builds the dispatch table by resolving every registered command once."
  Code:      Commands are resolved per invocation instead.
  Checked:   Thing.cs:41-58, Thing.cs:96-120.
  Ruled out: no factory or alternate constructor pre-resolves them.

path/Thing.cs:88  MISSING TAG
  Member:   Parse(string value)
  Missing:  <exception cref="FormatException">
  Checked:  Parse calls Parser.Parse at Parser.cs:31, which can propagate FormatException.

path/Thing.cs:112  VERBOSE
  Member:   AddAuth(IConfiguration configuration, bool registerExceptionHandler)
  Excess:   <param name="registerExceptionHandler"> explains package wiring and registration order.
  Compact:  "Whether to register the authentication exception handler."

path/Thing.cs:140  UNCHECKED
  Claim:    "The dispatcher has already validated the payload."
  Status:   Not established.
  Needed:   The dispatcher lies outside the approved verification scope.
```

Close with files read, files skipped and why, counts per category, build
status, and whether the pass was complete. Say plainly when the docs are
accurate throughout; never invent marginal findings.

## `--fix`

Without `--fix`, stop after the report. With it, apply only the approved
corrections: re-read the code first, treat every replacement sentence as a new
claim, delete stale or unverifiable text when no honest replacement exists,
follow the authoring standard, then rebuild and verify the corrected scope
again. A finding documentation alone cannot fix stays a finding.
