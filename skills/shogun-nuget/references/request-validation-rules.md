# Request validation, rule catalogue

Every rule exists twice under the same name: as an attribute
(`[Required]`) and as a fluent method (`RuleFor(x => x.Field).Required()`).
Two rules are fluent only: `AnyOf` and `Password`.

Read `asp-net-request-validation.md` first for how rules are registered, merged
and reported. This file is the lookup.

Message keys resolve through `IMessageResolver`, so every key below needs an
entry in `messages/{language}/validation.json`. A missing key renders as the key
itself.

Rules are grouped here by what they do. That grouping is this file's own and does
not mirror the page split under `docs/asp-net-request-validation/validation-rules/`,
so do not expect the two to line up section for section.

## Reading the tables

- **Field arguments** are a `string` field name on an attribute and an expression
  on the fluent method: `[Same("Password")]` against `.SameAs(x => x.Password)`.
- A **sized** key ends in `.string`, `.numeric`, `.array` or `.file`, chosen from
  what the value is. All four subkeys need a message.
- Most rules **pass on an empty value**; presence rules are what make a field
  mandatory.

## Presence

| Rule | Arguments | Message key |
| --- | --- | --- |
| `Required` | | `validation.required` |
| `Filled` | | `validation.filled` |
| `Present` | | `validation.present` |
| `Missing` | | `validation.missing` |
| `Prohibited` | | `validation.prohibited` |
| `Prohibits` | other fields | `validation.prohibits` |

Presence rules run before every other rule on the field, whatever order they are
declared in.

## Conditional presence

Each family takes the same shapes. `If` and `Unless` take a field plus the values
that trigger them; `With` and `WithAll` take one or more other fields;
`IfAccepted` and `IfDeclined` take a field holding a boolean-like value.

| Family | Variants | Message keys |
| --- | --- | --- |
| `Required*` | `If`, `Unless`, `With`, `WithAll`, `Without`, `WithoutAll`, `IfAccepted`, `IfDeclined` | `validation.required.if`, `.unless`, `.with`, `.with-all`, `.without`, `.without-all`, `.if-accepted`, `.if-declined` |
| `Present*` | `If`, `Unless`, `With`, `WithAll` | `validation.present.if`, `.unless`, `.with`, `.with-all` |
| `Missing*` | `If`, `Unless`, `With`, `WithAll` | `validation.missing.if`, `.unless`, `.with`, `.with-all` |
| `Prohibited*` | `If`, `Unless`, `IfAccepted`, `IfDeclined` | `validation.prohibited.if`, `.unless`, `.if-accepted`, `.if-declined` |

## Sizes

One family, eight modes, measured as string length, numeric magnitude, item count
or file size in kilobytes.

| Rule | Arguments | Message key |
| --- | --- | --- |
| `Min` | value | `validation.min.<sized>` |
| `Max` | value | `validation.max.<sized>` |
| `Between` | min, max | `validation.between.<sized>` |
| `Size` | value | `validation.size.<sized>` |
| `GreaterThan` | value | `validation.greater-than.<sized>` |
| `GreaterThanOrEqual` | value | `validation.greater-than-or-equal.<sized>` |
| `LessThan` | value | `validation.less-than.<sized>` |
| `LessThanOrEqual` | value | `validation.less-than-or-equal.<sized>` |

A value whose size cannot be measured fails with `validation.numeric`.

## Numbers

| Rule | Arguments | Message key |
| --- | --- | --- |
| `Numeric` | | `validation.numeric` |
| `Integer` | | `validation.integer` |
| `Decimal` | places | `validation.decimal` |
| `MultipleOf` | value | `validation.multiple-of` |
| `Digits` | digits | `validation.digits` |
| `DigitsBetween` | min, max | `validation.digits.between` |
| `MinDigits` | min | `validation.min.digits` |
| `MaxDigits` | max | `validation.max.digits` |

## Dates

| Rule | Arguments | Message key |
| --- | --- | --- |
| `Date` | | `validation.date` |
| `DateFormat` | format | `validation.date.format` |
| `DateEquals` | date or field | `validation.date.equals` |
| `After` | date or field | `validation.after` |
| `AfterOrEqual` | date or field | `validation.after.or-equal` |
| `Before` | date or field | `validation.before` |
| `BeforeOrEqual` | date or field | `validation.before.or-equal` |
| `Timezone` | | `validation.timezone` |

The attribute form takes a `string target` plus a `ComparisonTarget`, defaulting
to `ComparisonTarget.Value`, so pass `ComparisonTarget.Field` to compare against
another property. The fluent form has an overload per shape: `DateTime`,
`DateTimeOffset`, or an expression.

## Field comparison

| Rule | Arguments | Message key |
| --- | --- | --- |
| `SameAs` | other field | `validation.same` |
| `Different` | other field | `validation.different` |
| `Confirmed` | optional field | `validation.confirmed` |
| `InArray` | field holding the array | `validation.in.array` |
| `InArrayKeys` | keys | `validation.in.array-keys` |
| `RequiredArrayKeys` | keys | `validation.required.array-keys` |

`Confirmed` with no argument looks for `{Name}Confirmation` and then
`Confirm{Name}` on the request type, and throws when neither exists.

## Sets and substrings

| Rule | Arguments | Message key |
| --- | --- | --- |
| `In` | values | `validation.in` |
| `NotIn` | values | `validation.not.in` |
| `Contains` | values | `validation.contains` |
| `DoesNotContain` | values | `validation.does-not.contain` |
| `StartsWith` | prefixes | `validation.starts-with` |
| `EndsWith` | suffixes | `validation.ends-with` |
| `DoesNotStartWith` | prefixes | `validation.does-not.start-with` |
| `DoesNotEndWith` | suffixes | `validation.does-not.end-with` |

## Strings

| Rule | Arguments | Message key |
| --- | --- | --- |
| `String` | | `validation.string` |
| `Alpha` | | `validation.alpha` |
| `AlphaDash` | | `validation.alpha.dash` |
| `AlphaNumeric` | | `validation.alpha.num` |
| `Ascii` | | `validation.ascii` |
| `Lowercase` | | `validation.lowercase` |
| `Uppercase` | | `validation.uppercase` |
| `Regex` | pattern, options, description, timeout | `validation.regex` |
| `NotRegex` | pattern, options, description, timeout | `validation.not.regex` |

The regex rules take `RegexOptions`, an optional `description` passed to the
message as `{0}`, and a match timeout in seconds defaulting to 1.

## Formats

| Rule | Arguments | Message key |
| --- | --- | --- |
| `Email` | | `validation.email` |
| `Url` | | `validation.url` |
| `Uuid` | | `validation.uuid` |
| `Ulid` | | `validation.ulid` |
| `Json` | | `validation.json` |
| `HexColor` | | `validation.hex-color` |
| `MacAddress` | | `validation.mac-address` |
| `Ip` | | `validation.ip` |
| `Ipv4` | | `validation.ip.ipv4` |
| `Ipv6` | | `validation.ip.ipv6` |
| `Enum` | enum type, generic or `Type` | `validation.enum` |

## Booleans

| Rule | Arguments | Message key |
| --- | --- | --- |
| `Boolean` | | `validation.boolean` |
| `Accepted` | | `validation.accepted` |
| `AcceptedIf` | field, values | `validation.accepted.if` |
| `Declined` | | `validation.declined` |
| `DeclinedIf` | field, values | `validation.declined.if` |

## Collections and files

| Rule | Arguments | Message key |
| --- | --- | --- |
| `Array` | | `validation.array` |
| `List` | | `validation.list` |
| `Distinct` | | `validation.distinct` |
| `File` | | `validation.file` |
| `Uploaded` | | `validation.uploaded` |
| `Image` | | `validation.image` |
| `Extensions` | extensions | `validation.extensions` |
| `Mimes` | mime shorthands | `validation.mimes` |
| `MimeTypes` | mime types | `validation.mimetypes` |
| `Dimensions` | width, height | `validation.dimensions` |
| `MinDimensions` | width, height | `validation.dimensions` |
| `MaxDimensions` | width, height | `validation.dimensions` |

All three dimension rules share one message key, so the message cannot say which
bound failed. `Image` and the dimension rules read the file signature and header
rather than trusting the content type.

## Passwords

| Rule | Message key |
| --- | --- |
| `PasswordSecure` | `validation.password.secure` |
| `PasswordLetters` | `validation.password.letters` |
| `PasswordMixed` | `validation.password.mixed` |
| `PasswordNumbers` | `validation.password.numbers` |
| `PasswordSymbols` | `validation.password.symbols` |

`Password(letters: true, mixed: true, numbers: true, symbols: true)` is fluent
only and composes the individual requirements.

## Composition and custom

| Rule | Arguments | Message key |
| --- | --- | --- |
| `AnyOf` | rule sets, fluent only | `validation.any-of` |
| `CustomRule<TRule>` | the rule type | whatever the rule returns |

`AnyOf` passes when at least one of its rule sets passes, and reports one
message for the field rather than the failures of each branch. A custom rule
chooses its own key through `ValidationRuleResult.Failure(key, parameters)`.

## Keys not tied to one rule

| Key | Used for |
| --- | --- |
| `validation.invalid` | a value the pipeline could not bind or interpret |
| `validation.invalid-body` | a request body that is not readable JSON |
| `http-error.422` | the envelope's `errorDescription` on a validation response |
