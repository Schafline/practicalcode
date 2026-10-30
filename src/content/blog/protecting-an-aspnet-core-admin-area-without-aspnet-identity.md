---
title: "Protecting an ASP.NET Core Admin Area Without ASP.NET Identity"
description: "Learn how to secure a Razor Pages admin area using password hashing and cookie authentication without the complexity of ASP.NET Core Identity."
pubDate: "Oct 28 2026"
heroImage: "../../assets/admin-authentication.png"
---

Most ASP.NET Core applications eventually need some form of administrative interface. In the storefront application developed throughout my eBook, *Building Your First Shopping Basket*, the admin area allows products to be added, edited and removed.

While the storefront doesn't contain highly sensitive customer information, the admin pages would allow anyone to change product and order information, so I needed a simple way to prevent unauthorised access.

ASP.NET Core offers several industrial strength
authentication options, including full user accounts with ASP.NET Core
Identity or external identity providers such as Microsoft, Google and
GitHub. These are excellent choices for applications that store
sensitive data or require fine‑grained user management.

For my own project, the requirements were much simpler:

*  A single administrator
*  No user registration
*  No password reset functionality
*  No user management screens
*  No role-based access control

I simply needed a secure way to prevent unauthorised users from accessing the admin pages.

With those requirements in mind, adopting the full ASP.NET Core Identity stack felt like unnecessary complexity. Instead, I chose a lightweight approach based on a single administrator password, secure password hashing and cookie based authentication.

In this article, I'll show how I protected the storefront's admin area using ASP.NET Core's built in authentication features without introducing ASP.NET Core Identity. Along the way, we'll discuss the trade offs involved, why password hashing matters and when a lightweight solution like this is appropriate.

## 1 Storing a Hashed Password Instead of a Plaintext Password

Before building the login page, we need to decide how the administrator's password will be stored.

A common mistake in small applications is to store passwords directly in configuration files. Even if those files aren't committed to source control, storing plaintext passwords is never a good idea.

Instead, applications should store a one-way hash of the password.

A one-way hash is a value generated using a cryptographic algorithm. The original password can be transformed into a hash but the hash cannot be reversed to reveal the original password.

This means that when a user enters a password, the application doesn't compare the password itself. Instead, it hashes the user's input and compares the resulting hash with the stored value.

For this implementation, we'll store a hashed administrator password in `appsettings.Development.json` using a simple `Admin` section:

```json
{
  "Admin": {
    "PasswordHash": ""
  }
}
```

During development, I stored this value in `appsettings.Development.json`, while in production it was supplied through environment variables.

This approach means the application never stores or retrieves the administrator's actual password; only the hashed value is retained.

---

## 2 Generating the Password Hash

For the storefront application described in my eBook, *Building Your First Shopping Basket*, I created a temporary Razor Page that generated password hashes during development. Once the hash had been generated and copied into configuration, the page was removed from the application.

The page model looked like this:

``` csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Identity;
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.RazorPages;

[AllowAnonymous]
public class GenerateHashModel : PageModel
{
    [BindProperty]
    public string Password { get; set; } = string.Empty;

    public string? Hash { get; set; }

    public void OnPost()
    {
        if (!string.IsNullOrWhiteSpace(Password))
        {
            var hasher = new PasswordHasher<object>();

            Hash = hasher.HashPassword(
                null,
                Password);
        }
    }
}
```

This was purely a development utility and was never deployed to production. Its sole purpose was to generate a password hash that could be copied into configuration.

Once generated, only the hash is stored. The original password is never written to disk and is never committed to source control.

---

## 3 Creating the Login Page

With a password hash stored in configuration, the next step is to create a login page where the administrator can enter their password.

The page itself is deliberately simple. We only need a single password field and a button to submit the form. Any validation or authentication logic will be handled by the page model.

The markup for the page looks like this:

```csharp
@page
@model Storefront.Pages_Admin.LoginModel

<form method="post">
  <div>
    <label for="password">Password</label><br />
    <input
      type="password"
      id="password"
      name="Password" />
  </div>

  <button type="submit">Log In</button>

  @if (!string.IsNullOrEmpty(Model.ErrorMessage))
  {
    <p style="color:red">
      @Model.ErrorMessage
    </p>
  }
</form>
```
---

## 4 Verifying the Password and Signing In the User

The page model is responsible for handling the login request. When the administrator submits the form, the application reads the stored password hash from configuration and compares it against the password that was entered.

If the password is correct, an authentication cookie is issued and the user is redirected to the admin area. Otherwise, an error message is displayed and the login page is shown again.

The complete page model is shown below:

```csharp
using Microsoft.AspNetCore.Identity;
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.RazorPages;
using Microsoft.AspNetCore.Authentication;
using Microsoft.AspNetCore.Authentication.Cookies;
using System.Security.Claims;

namespace Storefront.Pages_Admin;

public class LoginModel : PageModel
{
  private readonly IConfiguration _config;

  public LoginModel(IConfiguration config)
  {
    _config = config;
  }

  [BindProperty]
  public string Password { get; set; } = string.Empty;

  public string? ErrorMessage { get; set; }

  public async Task<IActionResult> OnPostAsync()
  {
    var storedHash = _config["Admin:PasswordHash"];

    if (string.IsNullOrWhiteSpace(storedHash))
    {
      ErrorMessage =
        "Admin password has not been configured.";
      return Page();
    }

    var hasher = new PasswordHasher<object>();

    var result = hasher.VerifyHashedPassword(
      null,
      storedHash,
      Password
    );

    if (result == PasswordVerificationResult.Success)
    {
      var claims = new List<Claim>
      {
        new Claim(ClaimTypes.Name, "AdminUser")
      };

      var identity = new ClaimsIdentity(
        claims,
        CookieAuthenticationDefaults.AuthenticationScheme
      );

      var principal = new ClaimsPrincipal(identity);

      await HttpContext.SignInAsync(
        CookieAuthenticationDefaults.AuthenticationScheme,
        principal
      );

      return RedirectToPage("/Admin/Index");
    }

    ErrorMessage = "Incorrect password.";
    return Page();
  }
}
```

Although the page model contains several moving parts, the authentication process itself is straightforward.

First, the stored password hash is loaded from configuration:

```csharp
var storedHash =
    _config["Admin:PasswordHash"];
```

Next, ASP.NET Core's `PasswordHasher<T>` is used to verify the submitted password:

```csharp
var result =
    hasher.VerifyHashedPassword(
        null,
        storedHash,
        Password);
```

Notice that the application never compares plaintext passwords. Instead, the submitted password is processed by the same hashing algorithm that was used to generate the stored hash.

If verification succeeds, a `ClaimsPrincipal` is created and ASP.NET Core issues an authentication cookie:

```csharp
await HttpContext.SignInAsync(
    CookieAuthenticationDefaults
        .AuthenticationScheme,
    principal);
```

From this point onwards, ASP.NET Core recognises the user as authenticated and allows access to protected pages. If the password is incorrect, the user remains unauthenticated and an error message is displayed instead.

---

## 5 Protecting the Admin Area

At this point we have a login page that can verify the administrator's password and issue an authentication cookie. However, ASP.NET Core still needs to be configured to use cookie-based authentication.

Because this application uses Razor Pages, the easiest approach is to protect the entire /Admin folder and then explicitly allow anonymous access to the login page.

Add the following configuration:

```csharp
builder.Services.AddRazorPages(options =>
{
  options.Conventions
    .AuthorizeFolder("/Admin");

  options.Conventions
    .AllowAnonymousToPage(
      "/Admin/Login");
});
```

The `AuthorizeFolder()` call ensures that every page within the */Admin* folder requires an authenticated user.

However, if every page in the folder required authentication, users would never be able to reach the login page itself. The call to `AllowAnonymousToPage()` solves this problem by making */Admin/Login* accessible to unauthenticated users.

With these conventions in place, any attempt to access a protected admin page will trigger ASP.NET Core's authorisation system.

---

## 6 Configuring Cookie Authentication

When the administrator successfully enters their password, the login page creates a `ClaimsPrincipal` and calls `SignInAsync()`. To support this, ASP.NET Core must be configured to issue and validate authentication cookies.

Add the following code to `Program.cs`:

```csharp
builder.Services
  .AddAuthentication(
    CookieAuthenticationDefaults
      .AuthenticationScheme)
  .AddCookie(options =>
  {
    options.LoginPath =
      "/Admin/Login";
  });
```
This registers ASP.NET Core's cookie authentication handler and specifies that unauthenticated users should be redirected to */Admin/Login*.

The authentication middleware must also be added to the request pipeline:

```csharp
app.UseRouting();

app.UseAuthentication();
app.UseAuthorization();
```

Notice that `UseAuthentication()` appears before `UseAuthorization()`. ASP.NET Core must first establish who the user is before it can determine whether they are authorised to access a protected resource.

With this configuration in place, the authentication cookie issued by the login page will be recognised on subsequent requests, allowing authenticated users to access the protected admin area without having to repeatedly enter their password.

---

## 7 How the Authentication Flow Works

At this point, all of the pieces are in place. When a user attempts to access a page within the */Admin* folder, ASP.NET Core automatically applies the authorisation rules we configured earlier.

The authentication process works as follows:

1. The user attempts to access a page within the */Admin* folder.

2. ASP.NET Core checks for a valid authentication cookie.

3. If no cookie is present, the user is redirected to */Admin/Login*.

4. The user enters the administrator password and submits the login form.

5. The application verifies the submitted password against the stored hash.

6. If verification succeeds, ASP.NET Core issues an authentication cookie and redirects the user to the admin area.

7. On subsequent requests, ASP.NET Core validates the cookie and allows access to the protected pages without requiring the password to be entered again.

Because only a password hash is stored, the application never needs to know the administrator's actual password. Authentication is based entirely on comparing the submitted password against the stored hash and issuing a cookie when the comparison succeeds.

The result is a lightweight authentication system that requires very little code while still preventing unauthorised access to the admin pages.

---

## 8 When This Approach Is Appropriate

Like most architectural decisions, the suitability of this approach depends entirely on the requirements of the application.

For my storefront application, the admin area exists primarily to manage products and update order statuses. It is used by a single administrator and does not provide customer-facing account functionality.

Although the application stores customer email addresses and order information, it does not store payment card details, banking information, phone numbers or other highly sensitive personal data. Payment processing is delegated to a third-party provider and the admin area itself is relatively small in scope.

Given these requirements, a lightweight authentication system based on a single administrator password was a reasonable choice.

This approach can work well when:

*  A single administrator requires access to the application.
*  There is no user registration process.
*  There are no customer accounts.
*  Role-based authorisation is not required.
*  The administrative functionality is relatively small and self-contained.
*  You want to minimise complexity while still preventing unauthorised access.

For small business applications, hobby projects, internal tools and simple administration portals, a hashed password and cookie-based authentication can provide an effective level of protection without introducing the overhead of a full identity management system.

The important thing is to choose a solution that matches the complexity of the problem. For this storefront application, the goal was simply to prevent public access to the admin pages while keeping the application easy to understand and maintain.

---

## 9 When You Should Use ASP.NET Core Identity Instead

Although this approach worked well for my storefront application, it isn't the right solution for every project.

As the complexity of an application grows, the requirements around authentication often grow with it. At some point, managing those requirements yourself becomes more work than adopting a dedicated identity solution.

ASP.NET Core Identity is often the better choice when you need:

*  Multiple users
*  User registration
*  Password reset functionality
*  Role-based authorisation
*  Account lockout policies
*  Email confirmation workflows
*  Multi-factor authentication (MFA)
*  Auditing and account management features

Identity is designed to solve these problems and has been extensively tested and hardened over time. If your application needs any of these capabilities, it is usually better to adopt Identity rather than attempting to build them yourself.

For larger applications, external identity providers can also be a good option. Services such as Microsoft, Google and GitHub can authenticate users on your behalf using industry-standard protocols such as OAuth and OpenID Connect.

In those scenarios, your application no longer needs to manage passwords directly. Instead, authentication is delegated to a trusted provider and ASP.NET Core receives information about the authenticated user after the sign-in process completes.

For my storefront application, none of these capabilities were required. The goal was simply to prevent unauthorised access to a small administrative area used by a single administrator.

A lightweight solution based on password hashing and cookie authentication met those requirements while keeping the application simple and easy to maintain. However, if the application were to grow to support multiple administrators, customer accounts or additional security requirements, migrating to ASP.NET Core Identity or an external identity provider would be a natural next step.

