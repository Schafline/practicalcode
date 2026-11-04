---
title: "Accepting PayPal Payments in ASP.NET Core Using the Sandbox Environment"
description: "Learn how to integrate PayPal Checkout into an ASP.NET Core Razor Pages application using the PayPal Sandbox environment for safe testing."
pubDate: "Nov 05 2026"
heroImage: "../../assets/paypal-sandbox-aspnet-core.png"
---

For the storefront application developed throughout my eBook, *Building Your First Shopping Basket*, I chose PayPal as the payment provider. It is widely recognised, straightforward to integrate and allows developers to test payment flows safely using a dedicated sandbox environment.

This article assumes that you already have a PayPal account. If you don't, creating one takes only a few minutes and is free.

We'll also be working with a PayPal Sandbox app, which provides a safe environment for testing payments without using real money. PayPal also provides sandbox testing accounts that can be used to simulate both the merchant and customer sides of a transaction.

In this article, I'll show how I integrated the PayPal Sandbox into an ASP.NET Core Razor Pages application. Along the way, we'll look at creating a sandbox app, securely storing your PayPal Client ID using .NET user secrets and testing the checkout flow without spending any real money.

---

## 1 Why Use the PayPal Sandbox?

There are many payment providers available to ASP.NET Core developers, including Stripe, Square and PayPal. Any of these could be used to power a storefront checkout experience.

For this project, I chose PayPal largely because it is one of the most widely recognised online payment platforms. Many customers already have a PayPal account and those who don't are often familiar with the PayPal checkout experience.

PayPal also provides a dedicated Sandbox environment, allowing developers to test payment workflows safely without processing real transactions.

During development, you'll typically need to test scenarios such as:

* Successful payments
* Cancelled checkouts
* Redirects back to your application
* Basket totals and order calculations
* End to end purchase workflows

The Sandbox environment allows all of these scenarios to be tested without handling real money.

For the storefront application developed throughout *Building Your First Shopping Basket*, this provided a safe and convenient way to verify that the basket, checkout and payment confirmation process all behaved as expected before considering a production deployment.

Once everything is working correctly in the Sandbox, moving to a live PayPal environment becomes a much smaller and less risky step.

---

## 2 Creating a PayPal Sandbox App

Before we can integrate PayPal, we need a Sandbox application and its associated Client ID.

Start by signing in to the PayPal Developer Dashboard:

[PayPal Developer Dashboard](https://developer.paypal.com/dashboard)

At the time of writing, creating a sandbox application involves selecting
**Apps & Credentials** from the navigation menu, ensuring **Sandbox**
mode is active and then clicking **Create App**.

Because PayPal regularly updates its dashboard, I've chosen not to
include screenshots directly in this article. Instead, I maintain an
accompanying set of screenshots and navigation notes here:

[PayPal UI Reference](https://github.com/Schafline/Storefront/tree/main/external-ui-updates/paypal)

When creating the application:

* Enter a name for your application.
* Select **Merchant** when prompted for the application type.
* Choose the sandbox business account that will receive test payments.

At the time of writing, PayPal-created sandbox business accounts can often be identified by email addresses in the form:

```text
xxxx@business.example.com
```

Once the application has been created, PayPal generates a **Client ID**. This identifier allows your application to load the PayPal SDK and connect it to your sandbox environment.

At the time of writing, the Client ID can be viewed from the application's details page within the **Apps & Credentials** section of the PayPal Developer Dashboard.

You'll need to make a copy of this value as we'll use it in the next section when configuring ASP.NET Core to load the PayPal SDK securely during development.

At this stage, it can be tempting to simply copy the Client ID directly into your markup or source code. While this works, it creates two problems:

* The value becomes tied to source controlled files.
* Credentials become harder to manage across different environments.

Instead, we'll store the Client ID using .NET user secrets and load it through ASP.NET Core's configuration system.

Doing so keeps the Client ID out of source control while making it available to the application during development.

---

## 3 Storing the PayPal Client ID Securely

Now that we have a PayPal sandbox application, we need a way to make its Client ID available to our ASP.NET Core application.

While it would be possible to paste the Client ID directly into a Razor Page or JavaScript file, doing so would tightly couple the value to source controlled code and make it difficult to manage across different environments.

Instead, I chose to store the Client ID using .NET User Secrets.

If you're unfamiliar with User Secrets, see my earlier article:

[Sending Transactional Emails from ASP.NET Core Using Brevo's HTTP API](https://www.practicalcode.dev/blog/sending-transactional-emails-from-aspnet-core-using-brevo-http-api/)

where I cover configuring ASP.NET Core to load secrets during development.

With User Secrets already configured, adding the PayPal Client ID is as simple as running:

```bash
dotnet user-secrets set \
"PayPal:SandboxClientId" \
"<your-client-id>"
```

Replace `<your-client-id>` with the Client ID generated when you created your sandbox application.

This stores the value outside of source control while still making it available through ASP.NET Core's configuration system.

In the next section, we'll load this value into our basket page and use it to initialise the PayPal SDK.

## 4 Loading the Client ID into ASP.NET Core

With the Client ID stored securely in the user secrets store, the next step is making it available to the part of the application responsible for rendering the PayPal checkout UI.

In the storefront application developed throughout *Building Your First Shopping Basket*, that responsibility belongs to the basket page. Your own application may load the Client ID elsewhere, but the overall approach remains the same.

To do this, we can inject ASP.NET Core's `IConfiguration` service into the page model and expose a property that can be accessed from the Razor Page.

The following additions were made to `Basket.cshtml.cs`:

```csharp
private readonly IConfiguration _config;

public BasketModel(IConfiguration config)
{
    _config = config;
}

public string PayPalClientId
{
    get;
    private set;
}
```

This allows the page model to retrieve values from ASP.NET Core's configuration system, including values stored using User Secrets.

Next, the OnGet() method is updated to read the Client ID from configuration:

```csharp
PayPalClientId =
    _config["PayPal:SandboxClientId"];
```

The value is assigned to the `PayPalClientId` property, making it available to the Razor Page.

In production applications, consider validating that a Client ID has been supplied at startup and failing fast if the configuration value is missing. This prevents configuration issues from surfacing later when the PayPal SDK is initialised.

At this point, the application no longer needs to know where the Client ID came from. Whether the value originates from User Secrets during development or another configuration source in the future, the page model simply reads it from ASP.NET Core's configuration system.

In the next section, we'll use this value to initialise the PayPal JavaScript SDK and render the checkout buttons on the basket page.

---

## 5 Loading the PayPal SDK

With the Client ID available, we can now use it to load the PayPal JavaScript SDK.

In the storefront application, the SDK is loaded dynamically using the Client ID retrieved from configuration:

```csharp
@{
  var url =
    "https://www.sandbox.paypal.com/sdk/js?client-id=" +
    Model.PayPalClientId;
}
```

The resulting value is then used to load the PayPal SDK and the application's own checkout code:

```html
  <div id="paypal-button-container" class="mt-3 text-end"></div>

  <script src="@url"></script>
  <script src="~/js/paypal.js"></script>
```

The `paypal-button-container` element provides a location in the page where PayPal can render its checkout buttons.

You'll see this same identifier referenced later in `paypal.js`:

```javascript
.render(
  '#paypal-button-container'
);
```

When the PayPal SDK executes this code, it locates the element with the matching ID and inserts the checkout buttons into that part of the page.

---

## 6 Creating and Capturing Payments

With the PayPal SDK loaded, the next step is configuring the checkout buttons themselves.

In the storefront application, this logic lives inside `paypal.js`:

```javascript
paypal.Buttons({
  createOrder: function (data, actions) {
    const totalElement =
      document.getElementById
        ('basket-total');

    const total =
      totalElement
        ? totalElement.dataset.total
        : "0.00";

    return actions.order
      .create({
        purchase_units: [{
          amount: {
            value: total
          }
        }]
      });
  },

  onApprove: function (data, actions) {
    return actions.order
      .capture()
      .then(function (details) {
        window.location.href =
          '/Confirmation';
      });
  }
})
  .render(
    '#paypal-button-container'
  );
```

Although the code may look intimidating at first glance, there are really only two important callbacks: `createOrder()` and `onApprove()`.

### 6.1 Creating the Order

```javascript
  createOrder: function (data, actions) {
    const totalElement =
      document.getElementById
        ('basket-total');

    const total =
      totalElement
        ? totalElement.dataset.total
        : "0.00";

    return actions.order
      .create({
        purchase_units: [{
          amount: {
            value: total
          }
        }]
      });
  }
```

The `createOrder()` callback retrieves the basket total from the page and creates a PayPal order for that amount.

The call to `create()` is provided by PayPal's JavaScript SDK and is responsible for creating the order before the customer sees the checkout experience.

`purchase_units` is a PayPal SDK concept that describes the items being purchased. In my book's storefront application, the entire basket is treated as a single purchase unit whose value is equal to the basket total.

### 6.2 Capturing the Payment

```javascript
onApprove: function (data, actions) {
  return actions.order
    .capture()
    .then(function () {

      window.location.href =
        '/Confirmation';
    });
}
```

The `onApprove()` callback executes after the customer authorises the payment.

The call to `capture()` is provided by PayPal's JavaScript SDK. It finalises the transaction and instructs PayPal to complete the payment using the order that was previously created by `createOrder()`.

For simplicity, the example above redirects directly to a confirmation page after the payment has been captured.

In a real world application, this is typically the point at which the application would create an order record, persist transaction details and perform any additional fulfilment processing.

## 7 Testing a Sandbox Transaction

At this point, all of the pieces are in place. We've created a sandbox application, configured a Client ID, loaded the PayPal SDK and implemented the JavaScript required to create and capture payments.

The next step is to verify that everything is working correctly by completing a test transaction.

If your configuration is correct, navigating to the basket page should display a PayPal checkout button beneath the basket contents. If the button doesn't appear, double-check the Client ID, browser console and JavaScript configuration before continuing.

To complete a test transaction, you'll need to use one of the sandbox buyer accounts provided through the PayPal Developer Dashboard. These accounts allow PayPal to simulate the behaviour of a real customer while keeping all transactions within the sandbox environment.

With an item in your basket, the checkout flow should follow a process similar to this:

```text
Add Product To Basket
          ↓
Click PayPal Button
          ↓
Sign In Using Sandbox Buyer Account
          ↓
Approve Payment
          ↓
Return To Application
          ↓
Confirmation Page
```

After approving the payment, PayPal should return the user to your application and display the confirmation page.

A successful transaction demonstrates that:

* The PayPal SDK has been loaded correctly.
* The sandbox Client ID is valid.
* The order can be created successfully.
* The payment can be captured successfully.
* The application responds correctly after payment approval.

At this point, you've verified the complete end to end payment flow without processing any real transactions or spending any real money.

---

## 8 Common Sandbox Pitfalls

While the PayPal Sandbox is invaluable for testing, it isn't always predictable. During development, I encountered a few issues that are worth being aware of before you spend too much time debugging your own integration.

### 8.1 Session Conflicts

One of the most confusing problems occurs when PayPal reuses an existing login session.

For example, if you're already logged into a sandbox seller account and then attempt to complete a purchase as a sandbox buyer, PayPal may display an error indicating that you're attempting to purchase your own item.

To avoid this, try to keep buyer and seller activities separate. Using different browsers can help, as can private browsing windows.

### 8.2 Use Incognito or Private Browsing

If PayPal starts behaving unexpectedly during testing, open a new incognito/private browsing session and try again.

The Sandbox environment relies heavily on cookies and existing sessions, which can sometimes cause login confusion and account crossover issues.

### 8.3 Avoid Multiple PayPal Tabs

Where possible, avoid having multiple PayPal-related tabs open simultaneously.

PayPal shares authentication information across tabs, which can sometimes result in the wrong sandbox account being used during checkout.

### 8.4 Regional and Sandbox Variations

The PayPal experience can vary slightly depending on your region and the type of sandbox account being used.

For example:

* Different payment methods may be displayed.
* Additional buttons may appear.
* A PayPal balance option may not always be available.

As long as you can complete the payment successfully and return to your application's confirmation page, these differences are usually not a cause for concern.

### 8.5 When in Doubt, Create Fresh Test Resources

If you're convinced your code is correct but the Sandbox still behaves unexpectedly, creating a new sandbox buyer account or even a new sandbox application can often resolve difficult-to-diagnose testing issues.

While this shouldn't be necessary often, it can sometimes be quicker than trying to recover a sandbox environment that has become confused after extensive testing.

---

Integrating PayPal Sandbox into an ASP.NET Core application provides a safe way to develop and test payment functionality without processing real transactions.

By combining a sandbox application, a securely stored Client ID and PayPal's JavaScript SDK, we've created a complete end to end checkout flow that can be validated before moving to a production environment.

In a later PayPal article, I'll show how to move from the sandbox environment to a live PayPal integration capable of accepting real customer payments.


