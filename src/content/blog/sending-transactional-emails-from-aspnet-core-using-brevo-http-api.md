---
title: "Sending Transactional Emails from ASP.NET Core Using Brevo's HTTP API"
description: "Learn how to send transactional emails from ASP.NET Core using Brevo's HTTP API, User Secrets and a lightweight email service implementation."
pubDate: "Oct 21 2026"
heroImage: "../../assets/brevo-http-api.png"
---

Most ASP.NET Core applications eventually need to send email. Common examples include order confirmations, password reset links and account notifications. For my storefront application, I needed a simple and affordable way to send transactional emails as customers progressed through the checkout process.

After evaluating a few options, I chose Brevo. Its free plan was a particularly good fit for both my eBook, *Building Your First Shopping Basket*, and many of the small online sellers and hobby developers the book is aimed at. At the time of writing, Brevo's free tier allows up to 300 emails per day, which is more than enough for development, testing and many low-volume applications.

Rather than using SMTP, this article demonstrates how to send emails using Brevo's HTTP API. This approach avoids some of the deployment challenges associated with SMTP, particularly in cloud-hosted environments where outbound SMTP traffic may be restricted or blocked.

This article assumes that you already have a Brevo account, a verified sender address and a valid API key. The complete account setup process is covered in *Building Your First Shopping Basket*. Here, we'll focus purely on the ASP.NET Core integration.

---

## 1 Storing the Brevo API Key Securely with User Secrets

Before any email can be sent, the application needs access to a Brevo API key. While it may be tempting to store this value directly in `appsettings.json` or hard code it into your application, neither approach is recommended.

API keys should be treated as sensitive credentials and should never be committed to source control. Even if your repository is private today, storing secrets in code increases the risk of accidental exposure in the future.

For local development, ASP.NET Core provides a built-in solution called User Secrets. User Secrets allow configuration values to be stored securely on your development machine without becoming part of the project files.

### 1.1 Check That User Secrets Are Enabled

Before storing the API key, verify that your project supports User Secrets.

Open your `.csproj` file and look for a `UserSecretsId` entry:


```XML

<PropertyGroup>
  <TargetFramework>net9.0</TargetFramework>
  <UserSecretsId>
    5f8c8d2a-4e2a-4ef0-9b3b-123456789abc
  </UserSecretsId>
</PropertyGroup>

```

If a `UserSecretsId` is already present, User Secrets are enabled and you can continue to the next step.

If the entry is missing, initialise User Secrets from your project folder by running:


``` bash
dotnet user-secrets init
```

This will add a `UserSecretsId` to your project file and enable the User Secrets feature.

### 1.2 Add the Brevo API Key

From your project folder, run:

``` bash
dotnet user-secrets set "Email:ApiKey" "your-api-key-here"
```

Replace "your-api-key-here" with the API key generated from your Brevo dashboard.

You can verify that the value has been stored correctly by running:

``` bash
dotnet user-secrets list
```

You should see output similar to:

``` bash
Email:ApiKey = xkeysib-xxxxxxxxxxxxxxxxxxxxxxxx
```

### 1.3 Load User Secrets

If your application does not already load User Secrets, add the following code immediately after creating the application builder in `Program.cs`:

``` csharp
var builder = WebApplication.CreateBuilder(args);

builder.Configuration
    .AddUserSecrets<Program>();
```

This makes the API key available through ASP.NET Core's configuration system and allows it to be injected wherever it is needed.

We'll use this configuration value later when configuring and implementing the email service.

---

## 2 Creating an Email Service Interface

The implementation shown in my book keeps things deliberately simple for readers who are new to ASP.NET Core. In this article, we'll take a slightly more production oriented approach by introducing an email service abstraction and using dependency injection throughout.

Rather than sending emails directly from page models or controllers, we'll encapsulate all email-related functionality inside a dedicated service. This keeps email logic in a single location, reduces duplication and makes it easier to switch providers in the future if requirements change.

We'll start by defining a simple interface that describes the functionality our email service will provide:

``` csharp
namespace Storefront.Services;

public interface IEmailService
{
    Task SendEmailAsync(
        string toEmail,
        string subject,
        string body);
}
```

With the interface in place, we're ready to create a concrete implementation that uses Brevo's HTTP API to send email messages.

---

## 3 Configuring Email Settings

Before we implement our email service, we need somewhere to store the configuration values required by Brevo. Rather than scattering these values throughout the application, we'll group them together in a strongly-typed configuration class and bind them to a dedicated configuration section.

This approach keeps the API key, sender address and sender name in a single place and allows them to be accessed cleanly through ASP.NET Core's configuration system.

Create a new class named `EmailSettings`:

``` csharp
namespace Storefront.Settings;

public class EmailSettings
{
    public string ApiKey { get; set; }

    public string From { get; set; }

    public string FromName { get; set; }
}
```

In addition to the API key, our service also requires a sender address and display name. Unlike the API key, these values are not considered sensitive and can safely be stored in your application's configuration files.

Add an Email section to `appsettings.Development.json`:

``` JSON
{
  "Email": {
    "From": "store@example.com",
    "FromName": "Storefront"
  }
}
```

As discussed earlier, the API key should not be stored in `appsettings.Development.json`. Instead, it should be supplied via User Secrets during development and environment variables in production.

We now have our configuration class and configuration values, but ASP.NET Core doesn't automatically bind them together. To make the Email configuration section available through `IOptions<EmailSettings>`, add the following code to `Program.cs`:

``` csharp
builder.Services.Configure<EmailSettings>(
    builder.Configuration
        .GetSection("Email"));
```
This binds the Email configuration section to our `EmailSettings` class. ASP.NET Core will automatically populate an `EmailSettings` object using the values found in configuration and make it available through `IOptions<EmailSettings>`, which we'll use in the email service shortly.

---

## 4 Implementing the Brevo Email Service

Now that we've defined our interface, we can create a concrete implementation that sends emails using Brevo's HTTP API.

The service uses:

*  `IOptions<T>` to access configuration values.
*  `IHttpClientFactory` to create and manage `HttpClient` instances.
*  Brevo's `/smtp/email` endpoint to send emails over HTTPS.

Create a new class named `BrevoEmailService`:

``` csharp
using System.Net.Http;
using System.Text;
using System.Text.Json;
using Microsoft.Extensions.Options;
using Storefront.Settings;

namespace Storefront.Services
{
  public class BrevoEmailService : IEmailService
  {
    private readonly EmailSettings _settings;
    private readonly HttpClient _http;

    public BrevoEmailService(
      IOptions<EmailSettings> settings,
      IHttpClientFactory httpFactory)
    {
      _settings = settings.Value;
      _http = httpFactory.CreateClient();
    }

    public async Task SendEmailAsync(
      string toEmail,
      string subject,
      string body)
    {
      var payload = new
      {
        sender = new
        {
          name = _settings.FromName,
          email = _settings.From
        },
        to = new[]
        {
          new { email = toEmail }
        },
        subject = subject,
        htmlContent = body
      };

      var json = JsonSerializer
        .Serialize(payload);

      var content = new StringContent(
        json,
        Encoding.UTF8,
        "application/json");

      _http.DefaultRequestHeaders.Clear();
      _http.DefaultRequestHeaders.Add(
        "api-key",
        _settings.ApiKey);

      var response = await _http.PostAsync(
        "https://api.brevo.com/v3/smtp/email",
        content);

      if (!response.IsSuccessStatusCode)
      {
        var error = await response
          .Content
          .ReadAsStringAsync();

        throw new Exception(
          $"Brevo API error: " +
          $"{response.StatusCode} - " +
          $"{error}");
      }
    }
  }
}
```

The service constructs a JSON payload containing the sender, recipient, subject and HTML content before posting it to Brevo's transactional email endpoint. Configuration values such as the API key and sender details are supplied through the EmailSettings class, while `IHttpClientFactory` is used to obtain a managed `HttpClient` instance.

---

## 5 Registering the Email Service with Dependency Injection

The final step is to register the email service with ASP.NET Core's dependency injection container.

Add the following code to `Program.cs`:

``` csharp
  builder.Services.AddHttpClient<
    IEmailService,
    BrevoEmailService>();
```

This tells ASP.NET Core to use `BrevoEmailService` whenever an `IEmailService` is requested.

The `AddHttpClient()` method also registers the service with the `HttpClientFactory` infrastructure, allowing ASP.NET Core to create and manage `HttpClient` instances efficiently.

---

## 6 Using the Email Service



With the service registered, it can now be injected anywhere within the application.

The example below shows a Razor Page model requesting an `IEmailService` through constructor injection:

``` csharp
    public class OrderConfirmationModel
        : PageModel
    {
        private readonly IEmailService
            _emailService;

        public OrderConfirmationModel(
            IEmailService emailService)
        {
            _emailService = emailService;
        }
    }
```

Once injected, sending an email becomes as simple as:

``` csharp
await _emailService.SendEmailAsync(
    customerEmail,
    "Order Confirmation",
    htmlBody);
```

At this point, the `BrevoEmailService` takes over. It constructs the JSON request payload, sends the request to Brevo's API and throws an exception if the request fails.

The calling code doesn't need to know anything about HTTP requests, API keys or the Brevo API itself. Its only responsibility is to supply the recipient, subject and message content.

---

## 7 Why I Chose the HTTP API Over SMTP

SMTP has been the traditional approach to sending email from applications for many years. Most email providers, including Brevo, offer SMTP credentials that allow applications to send messages using the standard SMTP protocol.

However, SMTP can introduce deployment challenges that aren't always obvious during development.

For example:

*  Some firewalls allow the TCP connection but silently block SMTP traffic.
*  Cloud hosting providers may restrict outbound SMTP connections on common ports such as 25.
*  SMTP configuration often requires additional troubleshooting when moving an application between hosting environments.

In contrast, Brevo's HTTP API uses standard HTTPS requests over port 443. Since HTTPS traffic is already required by most applications, it is significantly less likely to be blocked or restricted by hosting providers, firewalls or network policies.

Another advantage is simplicity. The entire email delivery process can be implemented using familiar ASP.NET Core components such as HttpClient, configuration classes and dependency injection, without introducing additional SMTP specific libraries or configuration.

For my eBook storefront application, the HTTP API proved to be a simple and reliable solution that worked consistently in both development and production environments.

At this point, your application is capable of sending transactional emails through Brevo using a clean service abstraction, strongly typed configuration and standard ASP.NET Core dependency injection patterns.

More importantly, the implementation remains largely independent of Brevo itself. Should you decide to switch providers in the future, much of the surrounding application can remain unchanged.





