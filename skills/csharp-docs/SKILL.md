---
name: csharp-docs
description: >-
  Author, update, or independently verify C# XML documentation against a strict
  accuracy-first standard. Default mode writes or updates /// docs. Use --verify
  for a read-only semantic/completeness/shape audit, and --verify --fix to
  review findings before applying approved documentation-only corrections.
  Covers public/private members, tag completeness/order/grouping, verified
  semantics, escaping exceptions, remarks, inheritdoc, cref usage, extension
  receivers, wrapping, author/since metadata, XML-doc compiler failures, and
  stale cross-file claims. Never change implementation code merely to make
  documentation true.
argument-hint: "[path ...] [--verify [--fix]]"
disable-model-invocation: true
---

# C# Docs

Author and update C# XML documentation.

This skill governs `///` XML documentation, not ordinary explanatory comments. A general "avoid comments unless needed" rule does not suppress XML API documentation where the project documents its members.

## Modes

### Default: author/update

```text
csharp-docs
csharp-docs path/to/Project
csharp-docs path/to/File.cs
csharp-docs path/A path/B
```

Write or update XML documentation in the requested scope.

### `--verify`: independent audit

```text
csharp-docs --verify
csharp-docs path/to/Project --verify
csharp-docs path/to/File.cs --verify
csharp-docs path/A path/B --verify
```

Audit existing XML documentation against the code and project standard.

This mode is read-only. A clean compiler result proves only part of the
contract; semantic claims can still be false or stale.

Verification must remain independent of authoring conclusions. For large or
partitionable targets, use fresh isolated verifier sub-agents where practical
rather than trusting a previous authoring pass.

### `--verify --fix`: audit, then approved documentation fixes

```text
csharp-docs --verify --fix
csharp-docs path/to/Project --verify --fix
```

`--fix` is valid only together with `--verify`.

The workflow is:

1. perform the complete verification pass;
2. show the findings;
3. obtain explicit approval for the documentation corrections;
4. apply only approved documentation-only fixes;
5. verify the corrected scope again.

Invoking `--fix` expresses intent to fix findings, but does not skip review of
the findings or approval of the actual documentation corrections.

If the user asks for fixes after a normal `--verify` report, treat that as the
same fix phase.

## Target resolution

For both authoring and verification:

- file → operate on that file;
- directory → operate on `.cs` files beneath it, excluding generated/build
  output such as `bin` and `obj`;
- several paths → handle each independently where practical;
- no argument → resolve the relevant C# project from the working repository;
- multiple plausible projects with no clear target → ask rather than guessing.

## Precedence

When rules conflict, resolve them in this order:

1. **Accuracy** — every sentence must be true of the current code.
2. **Completeness** — every required member and applicable tag must be present.
3. **Depth** — say what the reader cannot learn from the signature alone.
4. **Shape** — follow the required tag order, grouping, wrapping, and voice.

Never sacrifice a higher rule to satisfy a lower one.

## Determine the project standard

Before writing documentation, read applicable repository instructions such as:
- `AGENTS.md`
- `CLAUDE.md`
- `.editorconfig`
- project-specific documentation conventions

If the project documents members, document every member in scope to the same standard, including:
- public and private members;
- methods;
- properties;
- fields;
- constructors;
- interfaces;
- attributes;
- enum members;
- extension-block receivers.

Private members are not exempt.

Use the project's prescribed `<author>` and `<since>` values. When the project has no more specific value but follows this standard, use:

```xml
<author>Almighty-Shogun</author>
<since>Unreleased</since>
```

## Verify before writing

Every sentence is a claim about the code.

Before documenting a behavior:
- read the member body;
- inspect callers when the claim depends on callers;
- inspect collaborators when the claim depends on another type;
- inspect runtime/framework behavior only when necessary and verifiable.

Write only what was established.

If something cannot be established, leave it out. A short correct block is finished documentation; do not pad it to resemble neighboring blocks.

Never invent:
- constraints;
- failure modes;
- rationale;
- guarantees;
- ownership semantics;
- platform behavior.

## Substance

Never merely restate the member or parameter name.

Bad:

```xml
/// <summary>
/// Configures the service.
/// </summary>
/// <param name="serviceCollection">The service collection.</param>
```

Documentation should add information a reader cannot reliably infer from the signature.

### Tag responsibilities

| Tag | Content |
|---|---|
| `<summary>` | What the member does and, when supported by the code, the important constraint, failure mode, or usage distinction a caller could otherwise miss. |
| `<typeparam>` | What the type argument represents and what its constraint provides when that matters. |
| `<param>` | What the value represents and what it changes; for flags, describe the behavior of both sides when meaningful. |
| `<returns>` | What is returned, including meaningful null, empty, or failure cases. |
| `<exception>` | Every exception that can escape and, where one exists, the non-throwing alternative or caller action. |
| `<remarks>` | Verified reasoning, non-obvious consequences, compatibility constraints, or load-bearing implementation details. |

These are not quotas. Include detail only when the implementation supports it.

A member whose name, type, and attributes already state everything true is finished with one short sentence.

A short block is correct when there is nothing more verifiable to say. Do not reach into a consumer for depth the member itself does not have; documentation borrowed from elsewhere is the failure this standard is most likely to produce, because it reads exactly like verified prose.

## Cross-file promises

Distinguish local facts from promises about other code.

A claim such as:
- "the caller already validates this";
- "the dispatcher always supplies X";
- "this is registered as a singleton";
- "the runtime leaves the stream open";

depends on another component.

Open and verify that component before writing the claim.

When useful, point to the enforcing type/member with `<see cref="..."/>` so the next reader can follow the guarantee.

## Whose contract is it

Before writing a sentence about a member, ask whether it would need editing if a different type changed.

If it would, the sentence documents that other type. Put it there, and reference it from here with `<see cref="..."/>` when the reader needs the pointer.

Duplicating another member's contract creates two copies of one fact in two files, with nothing keeping them in step. The copy that does not sit next to the behavior is the one that goes stale, and no build reports it.

### Data carriers

A DTO, request, response, options object, or entity property carries a value. It does not perform behavior.

Document:
- what the value is;
- the constraints that live on the property itself: validation attributes, format, allowed values, nullability, whether it is required.

Do not document:
- what a service does with the value once it receives it;
- the conditions under which a consumer rejects it;
- the order in which another member validates it.

That behavior belongs on the member performing it, where it can be checked against an implementation.

Where the type as a whole has a contract worth stating, such as which rules an endpoint enforces and which it leaves to the caller, state it once in the type's own `<summary>` rather than repeating a fragment of it on every property.

## Exceptions

`<exception>` is required for every exception that can escape the member.

Do not infer exceptions from names.

Trace:
- direct throws;
- relevant callees;
- wrappers such as reflection/invocation layers;
- validation helpers;
- parsing/serialization calls.

Where a non-throwing alternative exists, name it concisely.

## Fixed tag order

Use this order and omit tags that do not apply:

```text
<summary>
<typeparam>
<param>
<returns>
<exception>
<remarks>
<author>
<since>
```

Never reorder what remains.

`<typeparam>` always precedes `<param>`.

`<param>` tags must follow signature order.

## Grouping

Separate groups with a single bare `///` line.

Rules:
- `<typeparam>` and `<param>` form one group.
- `<author>` and `<since>` form the final group.
- Every other tag type opens its own group.

Example:

```csharp
/// <summary>
/// Registers concrete types that inherit from or implement <typeparamref name="T"/>.
/// </summary>
///
/// <typeparam name="T">The base type or interface to search for.</typeparam>
/// <param name="serviceLifetime">The lifetime applied to matching registrations.</param>
///
/// <returns>The <see cref="IServiceCollection"/> instance with matching implementations registered.</returns>
///
/// <author>Almighty-Shogun</author>
/// <since>Unreleased</since>
```

## Wrapping

Wrap at the project's `.editorconfig` `max_line_length`.

Count:
- indentation;
- `///`;
- spaces;
- XML tags;
- text.

When a one-line tag would exceed the limit, use:

```csharp
/// <param name="addType">
/// Whether to register each implementation under <typeparamref name="T"/> instead of its concrete type.
/// </param>
```

Do not reflow text beyond what the project's line limit requires.

## Voice

Summaries use third-person singular:

```text
Registers
Deserializes
Adds
Gets
Gets or sets
```

A read-only property uses `Gets`. A property with a setter uses `Gets or sets`, matching the .NET convention, so the accessor shape is readable without checking the signature.

Never use:
- imperative voice;
- "This method...";
- filler that repeats the member name.

## Defaults

Do not restate a default value that is already visible in the signature.

Instead explain what the relevant value causes.

Prefer:

```text
If none are provided, the calling assembly is used.
```

over repeating `null`, `false`, or another signature default.

## Returns

`<returns>` uses a noun phrase rather than a prose sentence.

For fluent receiver returns, keep one consistent shape, for example:

```text
The <see cref="IServiceCollection"/> instance with matching implementations registered.
```

Do not invent stylistic variants such as "The updated..." merely for variety.

## References

Use:
- `<see cref="..."/>` for resolvable types and members;
- `<paramref name="..."/>` for the current member's parameters;
- `<typeparamref name="..."/>` for the current member's type parameters;
- `<c>` for literals or for a parameter/name that belongs to a different overload and therefore cannot use `paramref`.

For overloaded methods, use a fully disambiguated cref parameter list when needed to avoid `CS0419`.

Crefs into extension blocks may resolve; verify with the generated XML/build rather than assuming.

## Remarks

`<remarks>` is where verified design reasoning belongs when source documentation needs to preserve it.

Use it for:
- why a non-obvious behavior exists;
- compatibility constraints;
- external protocol/spec requirements;
- ownership/lifetime consequences;
- implementation details a future editor might otherwise "simplify" away.

Do not use `<remarks>` to praise the design or assert vague qualities such as "clean", "maintainable", or "flexible".

## Extension blocks

Document the receiver of an `extension` block with a `<param>` on the block itself.

Describe:
- what the receiver represents;
- ownership/lifetime behavior when relevant;
- consequences such as whether a stream remains open.

## Changing documented code

Whenever a documented member's body changes:

1. Re-read its entire XML block.
2. Correct every sentence the change invalidated.
3. Re-check summary, remarks, rationale, returns, exceptions, and parameter effects.
4. Do not limit updates to compiler-forced tags.

When a member's behavior changes, search documentation in other files for references to that member/type and update stale cross-file claims.

Deleting a stale sentence is always allowed.

Do not invent a replacement merely to preserve block length.

## Code is the fact

When code and documentation disagree, the documentation is wrong for the purpose of a documentation update.

Never change code merely to make an existing doc block true.

If the documented behavior appears preferable:
- report the mismatch;
- explain why the alternative behavior might be better on its own merits;
- require explicit user approval before changing code;
- treat the behavior change as a separate development task.

A documentation pass does not silently resolve design disagreements.

## Build verification

After authoring or updating XML docs, build the relevant project when practical.

Treat these as documentation failures:

```text
CS1573
CS1591
CS1574
CS0419
```

Also inspect generated XML where necessary to confirm cref resolution or extension-block documentation.

## Repository safety

Default authoring mode may edit XML documentation blocks only unless the user
explicitly requests broader code changes through another workflow.

`--verify` is read-only until documentation fixes have been explicitly approved.

Do not:
- change implementation behavior to satisfy documentation;
- stage, commit, or push;
- create or delete branches;
- amend or rebase;
- reset, stash, clean, or checkout over unrelated user work;
- overwrite unrelated changes.

Protect unrelated uncommitted work.

## Token efficiency

- Read only the code needed to verify each claim.
- Reuse repository conventions instead of restating them repeatedly.
- Do not generate prose to fill empty tags.
- Do not explain obvious signatures.
- Inspect cross-file dependencies only when a documentation claim depends on them.
- Keep documentation as short as correctness and useful depth permit.

# Verification mode

The following rules apply when `--verify` is active. They extend the shared authoring standard above rather than defining a second documentation standard.

## Scale and sub-agents

Estimate target size before exhaustive reading.

For large or naturally partitioned verification targets:
- split by project, package, directory, or coherent area;
- use isolated sub-agents per independent chunk;
- avoid one giant context;
- keep each agent's findings evidence-based and citable.

Do not promote sub-agent findings blindly. Spot-check actionable findings and downgrade anything lacking evidence to `UNCHECKED`.

Report coverage accurately. Never describe a partial pass as complete.

## Verification dimensions

### 1. Accuracy

For each documented member, read the XML block and the implementation it sits on.

Judge each semantic claim independently.

Possible verdicts:

| Verdict | Meaning |
|---|---|
| `CONFIRMED` | Checked and true. Not normally reported. |
| `FALSE` | Contradicted by the code. |
| `STALE` | Describes behavior that no longer exists. |
| `JUSTIFICATION` | Argues that the design is good without conveying a checkable behavior/constraint. |
| `UNVERIFIABLE` | The codebase cannot settle a checkable claim. |
| `UNCHECKED` | Verification did not inspect what the claim depends on. |

`UNVERIFIABLE` and `UNCHECKED` are different and must never be merged.

### 2. Completeness

Where the project's standard requires XML documentation, verify every in-scope member, including private members when required.

Report:
- `MISSING DOC` — member requires documentation but has none;
- `MISSING TAG` — an applicable required tag is absent;
- missing `<exception>` coverage for escaping exceptions;
- missing meaningful null/empty/failure behavior in `<returns>` where required;
- missing parameter/type-parameter coverage.

Do not reduce missing documentation to a count only when the standard requires every member.

### 3. Depth

Documentation should say what a reader cannot reliably infer from the signature.

Report `FILLER` when a block merely restates:
- the member name;
- the parameter name;
- an obvious type;
- a visible default value;

without conveying meaningful behavior.

Do not demand extra prose merely because neighboring members are longer.

A short block is correct when there is nothing more verifiable to say.

Do not invent missing "depth" findings from stylistic preference.

### 4. Shape

Verify the required project shape, including when applicable:
- tag order;
- grouping;
- `<typeparam>` before `<param>`;
- parameter order matching the signature;
- line wrapping against `.editorconfig` `max_line_length`;
- summary voice;
- noun-phrase `<returns>`;
- correct `paramref` / `typeparamref` / `see` / `c` usage;
- `<author>` / `<since>`;
- extension receiver `<param>`;
- resolvable and correctly targeted crefs.

Shape findings come after accuracy, completeness, and depth.

## Try to prove a claim true before reporting it false

A false positive can cause a correct doc to be "fixed" into something wrong.

Before reporting `FALSE` or `STALE`, ask what would make the claim true and check it.

Examples:

- Is the supposedly invalid branch unreachable because of an earlier guard?
- Is a guarantee enforced by a caller, constructor, factory, or type constraint?
- Does the sentence refer to a different code path?
- Is the issue merely imprecise wording rather than behavior that would mislead a reader?

Only report the finding after this defense fails.

Include decisive evidence.

## Resolve `<inheritdoc />`

A member using `<inheritdoc />` inherits a contract and remains in scope.

Resolve the inherited documentation to its source, then verify the current implementation against it.

When several implementations share an inherited contract, check whether each implementation honors that contract.

Report divergence against the implementation and identify the inherited source.

## High-yield semantic checks

### Cross-component claims

Claims such as:
- "already validated by the caller";
- "the dispatcher supplies this";
- "already logged";
- "registered as a singleton";

depend on other files.

Open those files before judging the claim.

### Rewritten behavior

Compare summaries and remarks against current behavior, especially where Git history shows implementation changes.

History is a signal, not proof.

### Exceptions

Check both directions:
- every documented exception can actually escape;
- every exception that can escape is documented.

Trace relevant callees and wrappers.

### Returns

Verify:
- null behavior;
- empty behavior;
- failure behavior;
- fluent receiver return claims.

### Parameters

Verify the stated effect.

For flags, check both meaningful states.

Do not treat a restated signature default as useful documentation.

### Remarks

Distinguish:

**Explanation**  
Checkable reasoning tied to actual behavior. Verify normally.

**Justification**  
Statements such as "this keeps the API clean" or "this is the most maintainable approach". Report as `JUSTIFICATION`.

**Contradicted reasoning**  
Reasoning that describes behavior the implementation does not have. Report as `FALSE` or `STALE`, not merely justification.

### Cross-references

A cref can resolve yet point to the wrong member after a refactor. Verify both resolution and semantic target.

For overloaded methods, confirm disambiguation when needed.

For extension blocks, confirm resolution through build/generated XML where necessary.

## Build/compiler verification

Run the relevant build when practical.

Treat these as documentation failures:

```text
CS1573
CS1591
CS1574
CS0419
```

Do not dismiss them as tolerable warnings when the project follows this standard.

A clean build does not prove semantic accuracy.

## Code and documentation disagreement

When code and docs disagree, report that the documentation is wrong.

Do not change code to make the documentation true.

If the documented behavior appears preferable, record that only as a separate design observation and require explicit user approval before any behavior change.

The argument for changing code must stand on its own merits, not on the existence of an old doc sentence.

## Report ordering

Group findings by file and order them by the documentation precedence.

Recommended order:

1. `FALSE`
2. `STALE`
3. `JUSTIFICATION`
4. `UNVERIFIABLE`
5. `UNCHECKED`
6. `MISSING DOC`
7. `MISSING TAG`
8. `FILLER`
9. shape/compiler findings

Within a category, prioritize findings most likely to mislead callers or maintainers.

## Finding format

For semantic findings:

```text
path/Thing.cs:60  FALSE
  Claim:    "Builds the dispatch table by resolving every registered command once."
  Code:     Commands are resolved per invocation instead.
  Checked:  Thing.cs:41-58, Thing.cs:96-120.
  Ruled out: no factory or alternate constructor pre-resolves them.
```

For completeness/shape findings:

```text
path/Thing.cs:88  MISSING TAG
  Member:   Parse(string value)
  Missing:  <exception cref="FormatException">
  Checked:  Parse calls Parser.Parse at Parser.cs:31, which can propagate FormatException.
```

For unchecked claims:

```text
path/Thing.cs:88  UNCHECKED
  Claim:    "The dispatcher has already validated the payload."
  Status:   Not established.
  Needed:   The dispatcher lies outside the approved verification scope.
```

Every actionable finding should identify the evidence that established it.

Do not report a semantic finding you cannot support.

## Coverage summary

Close with:
- files read;
- files skipped and why;
- counts by verdict/category;
- build status if run;
- whether verification was complete or partial.

State plainly when documentation is accurate throughout.

Do not invent marginal findings to demonstrate effort.

## `--fix`

Without `--fix`, stop after the report.

With `--verify --fix`, or when the user asks after reviewing the report:

1. Apply only approved documentation corrections.
2. Re-read the code before writing replacement text.
3. Treat every replacement sentence as a new claim requiring verification.
4. Delete stale or unverifiable text when no honest replacement is needed.
5. Replace design praise with verified behavior or remove it.
6. Follow this skill's authoring standard for all rewritten blocks.
7. Never modify implementation code.
8. Rebuild afterward and verify no XML-doc compiler failures remain.

If a finding cannot be corrected through documentation alone, leave it as a finding.

## Safety

- Never edit before reporting. `--verify --fix` still requires review of findings and explicit approval of the documentation corrections.
- Never change code to satisfy documentation.
- Never commit, stage, push, amend, rebase, or create/delete branches.
- Never reset, stash, clean, or checkout over unrelated user work.
- Never call an unchecked claim confirmed.
- Never call an unchecked claim unverifiable.
- Never report a claim false without trying to establish it as true.
- Never describe skipped coverage as complete.
- Protect unrelated uncommitted work.

## Token efficiency

- Split large targets into isolated sub-agent scopes.
- Mechanically inspect repeated structures before reading every file serially.
- Give sub-agents only the documentation standard and scope they need.
- Keep findings compact and evidence-backed.
- Do not repeat confirmed claims in the report.
- Use Git history only when it helps settle a claim.
- Prefer decisive code evidence over lengthy prose analysis.
