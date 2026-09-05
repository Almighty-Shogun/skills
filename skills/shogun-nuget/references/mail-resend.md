# AlmightyShogun.Mail.Resend

Sending through Resend with reusable HTML and plain-text templates. A mail is a
class deriving from `BaseMailTemplate`; the package renders it into three HTML
files on disk and sends both bodies.

Docs: https://nuget.docs.shogun.ms/mail-resend/

Depends on `AlmightyShogun.Utils`, the `Resend` client, and
`Microsoft.Extensions.Http` with the standard resilience handler.

## What is where

| Need | Reach for |
| --- | --- |
| Register it | `AddResendEmail(configuration)` |
| Send to one address | `IResendMailService.SendAsync(email, mail)` |
| Send with cc, bcc, reply-to or attachments | `IResendMailService.SendAsync(mail, options)` |
| Render without sending | `IResendMailService.PreviewAsync(mail)` |
| Write a mail | derive from `BaseMailTemplate` |
| A call to action | `MailButton(label, url)` |

## Templates on disk

Three files must exist in a `mail` directory next to the built assembly
(`AppContext.BaseDirectory`):

```text
mail/BaseEmailTemplate.html
mail/BaseEmailParagraph.html
mail/BaseEmailButton.html
```

`AddResendEmail` checks for them and throws `InvalidOperationException` naming
what is missing, so a project must copy them to the output directory.

Placeholders the base template can use: `{{DocumentTitle}}`, `{{Title}}`,
`{{Greeting}}`, `{{BodyHtml}}`, `{{ButtonsHtml}}`, `{{LogoUrl}}`,
`{{BrandName}}`, `{{AppUrl}}`, `{{CopyrightText}}`, `{{FooterLinkText}}`,
`{{IgnoreTextHtml}}`, plus one per key in `AdditionalValues`. The paragraph file
uses `{{Paragraph}}`; the button file `{{ButtonUrl}}` and `{{ButtonLabel}}`.

## Writing a mail

```csharp
public sealed class ResetPasswordMail(string link) : BaseMailTemplate
{
    public override string Subject => "Reset your password";

    protected override string Title => "Password reset";

    protected override string Greeting => "Hi,";

    protected override IReadOnlyList<string> Paragraphs =>
        ["Use the button below to choose a new password.", "The link expires in an hour."];

    protected override IReadOnlyList<MailButton> Buttons => [new("Choose a new password", link)];
}
```

`Subject` is public, the rest are protected. `AdditionalValues` is the escape
hatch for extra placeholders.

Both bodies come from the same class: `Paragraphs` become paragraph blocks in
HTML and blank-line separated text, `Buttons` become buttons in HTML and
`label: url` lines in text.

## Traps

**Every value is HTML encoded.** Paragraphs, title, greeting, brand name and the
`AdditionalValues` entries are encoded on the way in, so a template cannot inject
markup through them; put markup in the HTML files instead.

**URLs must be absolute `http`, `https` or `mailto`.** `MailButton` throws
`ArgumentException` on anything else, and a `LogoUrl` or `AppUrl` that fails the
same check is replaced with an empty string rather than rendered.

**Templates are cached in memory after the first read.** Editing an HTML file at
runtime changes nothing until the process restarts.

**A template name that escapes the `mail` directory throws.** The loader resolves
the path and rejects anything outside the root.

**Sending never throws for a Resend failure.** `SendAsync` returns
`MailSendResult.Failure(message)` and logs; only a cancellation propagates. Check
`IsSuccess`.

**An empty `To` is a failure result, not an exception**, with the message "No
recipient was supplied."

**An idempotency key is generated per send** when `MailOptions.IdempotencyKey` is
null, using a version 7 GUID, so a retry that reuses the same `MailOptions`
instance is only deduplicated if you set the key yourself.

**`From` is composed from configuration**, as `FromName <FromEmail>` when a name
is set and the bare address otherwise.

**`{app_name}` and `{app_url}` are substituted in the three template settings
only**, case-insensitively, and `{app_url}` becomes empty when `AppUrl` fails the
URL check.

**`Email:Links` is bound but not rendered by the base template**, so it is
available to your own placeholder handling and nothing else.

## Public surface

Registration: `AddResendEmail(IConfiguration)` on `IServiceCollection`.

Service: `IResendMailService` (`SendAsync(string, BaseMailTemplate, ...)`,
`SendAsync(BaseMailTemplate, MailOptions, ...)`,
`PreviewAsync(BaseMailTemplate, ...)`), registered transient.

Authoring: `BaseMailTemplate` (`Subject`, `Title`, `Greeting`, `Paragraphs`,
`Buttons`, `AdditionalValues`), `MailButton(label, url)`.

Models: `MailOptions` (`To`, `Cc`, `Bcc`, `ReplyTo`, `Attachments`,
`IdempotencyKey`), `MailAttachment` (`FileName`, `Content`, `ContentType`),
`MailPreview` (`Html`, `Text`), `MailSendResult` (`IsSuccess`, `MessageId`,
`Error`).

Configuration: `EmailSettings` (`ApiToken`, `FromEmail`, `FromName`,
`BrandName`, `LogoUrl`, `AppUrl`, `Links`, `Template`, computed `From`),
`EmailTemplateSettings` (`CopyrightTextTemplate`, `FooterLinkText`,
`IgnoreText`).
