# AlmightyShogun.AspNet.RequestValidation

Request models validated through attributes, fluent rules, or both. Every failure
on a request is collected and returned as one `422` with localized messages.

Docs: https://nuget.docs.shogun.ms/asp-net-request-validation/

Depends on `AlmightyShogun.AspNet.Core` and `AlmightyShogun.AspNet.Localization`.

For the rule catalogue, attribute by attribute with its fluent equivalent and
message key, read `request-validation-rules.md`.

## What is where

| Need | Reach for |
| --- | --- |
| Register it | `AddAspNetValidation()` or `AddAspNetValidation(assemblies)` |
| Handle a malformed JSON body | `app.UseAspNetValidation()` |
| Validate a minimal API endpoint | `.UseAspNetValidation()` on the route handler or group |
| Declare rules on the model | attributes such as `[Required]`, `[Email]`, `[Min(8)]` |
| Declare rules in code | `Validator<TRequest>` with `RuleFor(x => x.Property)` |
| A rule with dependencies | `ICustomValidationRule<TRequest, TProperty>` plus `[CustomRule<T>]` or `.CustomRule<T>()` |
| Fail from inside an action | `ValidationErrorResult.Create(messageResolver, field, key, parameters)` |
| Read the declared rules | `IValidationRuleDescriber.Describe<TRequest>()` |

## The response

```json
{
  "code": 422,
  "error": "validation_error",
  "errorDescription": "...",
  "errors": {
    "email": { "code": 790803925, "error": "validation_email", "errorDescription": "..." }
  }
}
```

`ValidationErrorResponse` extends `HttpErrorResponse` from `AspNet.Core`, so the
envelope matches every other error. Per field, `code` is a stable numeric hash of
the message key, `error` is the key snake_cased, and `errorDescription` is the
resolved message.

Field names are camelCase, taken from `[JsonPropertyName]` when present.
**Only the first failure per field is reported**, and once a field has an error
its remaining rules are skipped.

## Messages

Keys are `validation.<rule>`, with dotted subkeys for variants, resolved through
`IMessageResolver`. Put them in `messages/{language}/validation.json`:

```json
{
    "required": "This field is required.",
    "date": { "default": "Must be a date.", "format": "Must match {0}." },
    "min": { "string": "Must be at least {0} characters.", "numeric": "Must be at least {0}." }
}
```

Size-style rules pick a subkey by the value's kind: `string`, `numeric`, `array`
or `file`. A missing key renders as the key itself, so an incomplete
`validation.json` shows up as `validation.min.string` in the response.

## Task recipes

### Attributes

```csharp
public sealed record SignupRequest
{
    [Required]
    [Email]
    public string Email { get; init; } = "";

    [Required]
    [Min(12)]
    [PasswordSecure]
    public string Password { get; init; } = "";

    [RequiredWith(nameof(Password))]
    public string PasswordConfirmation { get; init; } = "";
}
```

The attributes live in this package's namespace, not
`System.ComponentModel.DataAnnotations`. A `using AlmightyShogun.AspNet.RequestValidation;`
changes what `[Required]` means in that file.

### Fluent rules

```csharp
public sealed class SignupValidator : Validator<SignupRequest>
{
    protected override void Rules()
    {
        RuleFor(request => request.Email).Required().Email();
        RuleFor(request => request.Password).Required().Min(12).PasswordSecure();
    }
}
```

Validators are discovered from the assemblies passed to `AddAspNetValidation`,
instantiated per use, and merged with the attribute rules on the same type. One
request type may have at most one validator, and a validator must have a public
parameterless constructor; both are enforced at startup with an explicit
exception.

### A rule that needs services

```csharp
public sealed class UniqueEmailRule(AppDbContext database) : ICustomValidationRule<SignupRequest, string>
{
    public async Task<ValidationRuleResult> ValidateAsync(
        SignupRequest request,
        string? value,
        CancellationToken cancellationToken = default
    ) => await database.Users.AnyAsync(user => user.Email == value, cancellationToken)
        ? ValidationRuleResult.Failure("validation.email-taken")
        : ValidationRuleResult.Success();
}
```

Attach it with `[CustomRule<UniqueEmailRule>]` or `.CustomRule<UniqueEmailRule>()`.
The rule is resolved from the request's service provider, so it can take scoped
dependencies even though the validator itself cannot.

## Traps

**`AddAspNetValidation()` scans the calling assembly.** Validators and custom
rules in another assembly need the explicit overload.

**Registration changes MVC behavior**, deliberately: `ThrowOnBadRequest` on
route handlers, an `InvalidModelStateResponseFactory` that renders this
package's body, `SuppressImplicitRequiredAttributeForNonNullableReferenceTypes`,
and two MVC filters. A project that configures those itself will fight it.

**Minimal API endpoints are not covered by the MVC filters.** Add
`.UseAspNetValidation()` per route handler or per route group.

**`UseAspNetValidation()` on the app is a different thing** from the route
builder extension of the same name: the middleware turns a malformed request
body into the package's error body.

**Rules run in priority order, presence first**, and a field stops at its first
failure. Ordering attributes on the property does not change which message a user
sees.

**Only the first error per field reaches the response**, even though a field can
collect several internally.

**A validator taking constructor dependencies fails at startup**, by design: rules
are declared once, outside any request scope. Dependencies belong in a custom
rule.

**Two validators for one request type fail at startup** with a message naming
both.

## Public surface

Registration: `AddAspNetValidation()`, `AddAspNetValidation(Assembly[])` on
`IServiceCollection`; `UseAspNetValidation()` on `IApplicationBuilder`,
`RouteHandlerBuilder` and `RouteGroupBuilder`.

Authoring: `Validator<TRequest>` with `Rules()` and `RuleFor(expression)`,
`RuleBuilder<TRequest, TProperty>`, `ICustomValidationRule<TRequest, TProperty>`,
`ValidationRuleResult.Success()`, `ValidationRuleResult.Failure(key, parameters)`.

Responses: `ValidationErrorResponse` (`Errors`), `ValidationRuleError` (`Code`,
`Error`, `ErrorDescription`), `ValidationErrorResult.Create(...)`.

Describing: `IValidationRuleDescriber.Describe<TRequest>()` returning field to
`ValidationRuleDescription` (`Rule`, `Arguments`).

Types: `ComparisonTarget`, which tells a comparison rule whether its argument is
a literal value or another field.
