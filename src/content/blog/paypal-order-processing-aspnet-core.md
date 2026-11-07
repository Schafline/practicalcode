---
title: "Processing Completed PayPal Orders in ASP.NET Core"
description: "Learn how to process completed PayPal payments in ASP.NET Core by creating orders, generating verification codes, sending confirmation emails and redirecting customers to an order confirmation page."
pubDate: "Nov 12 2026"
heroImage: "../../assets/paypal-order-processing-aspnet-core.png"
---

The [previous article](https://www.practicalcode.dev/blog/accepting-paypal-payments-in-aspnet-core-using-the-sandbox-environment/) described how to integrate PayPal Sandbox into an ASP.NET Core application and capture a successful payment.

However, processing a payment is only part of the checkout process. Once PayPal confirms a transaction, we still need to create an order, persist its contents and notify the customer that everything has completed successfully.

The storefront application developed throughout my eBook, *Building Your First Shopping Basket*, also generates a unique verification code for each order. Rather than requiring customers to create accounts, this code can later be used to retrieve order information and track order status.

In this article, I'll show how I implemented that workflow using ASP.NET Core Razor Pages.

---

## 1 What Happens After a Successful Payment?

The previous article concluded when the customer successfully approved a PayPal payment.

In reality, however, this is where some of the most important application logic begins. Once PayPal confirms that a payment has been captured, the application still needs to perform several additional tasks before the checkout process can be considered complete.

In the storefront application developed throughout *Building Your First Shopping Basket*, the workflow looks like this:

```text
Customer Approves Payment
          ↓
Create Order
          ↓
Persist Order Items
          ↓
Generate Verification Code
          ↓
Send Confirmation Email
          ↓
Display Confirmation Page
```

Before reaching PayPal checkout, my storefront app collects the customer's shipping information, including the email address used for order confirmations. This means all of the information required to create the order is already available when PayPal confirms the payment.

The remainder of this article focuses on how that workflow is implemented once PayPal has successfully captured a payment.

---

## 2 Updating the PayPal Callback

In the previous article, a successful payment resulted in the customer being redirected directly to a confirmation page:

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

While this is fine in the context of demonstrating a successful PayPal integration, it isn't sufficient for a real storefront application as no order records have been created, no confirmation email has been sent and no verification code has been generated.

To support these additional requirements, we can update the callback to invoke a Razor Pages handler responsible for processing the completed order:

```javascript
onApprove: function (data, actions) {
  return actions.order
    .capture()
    .then(function (details) {
      return fetch(
        '/Basket?handler=CompleteOrder',
        {
          method: 'POST',
          headers: {
            'Content-Type':
              'application/json'
          },
          body: JSON.stringify({
            orderId: data.orderID,
            payerId: data.payerID
          })
        })
        .then(response =>
          response.json())
        .then(result => {
          window.location =
            `/Confirmation?id=${result.orderId}`;
        });
    });
}
```

After PayPal confirms the payment, the browser sends the PayPal order details to the `CompleteOrder` handler within the basket page model.

This handler is responsible for:

* Creating the order record.
* Persisting the associated order items.
* Generating an order verification code.
* Sending the confirmation email.
* Returning the newly created order identifier.

Rather than redirecting immediately, the application now waits for the order processing workflow to complete successfully before navigating to the confirmation page.

This change ensures that the customer only sees the confirmation page once the corresponding order has been created successfully.

The `CompleteOrder` handler forms the heart of the order processing workflow. In the next section, we'll examine its implementation in more detail and see how the storefront creates orders, generates verification codes and sends confirmation emails after a successful PayPal payment.

---

## 3 Introducing the CompleteOrder Handler

The updated PayPal callback delegates the responsibility for order processing to a Razor Pages handler named `CompleteOrder`.

The implementation begins with the following method:

```csharp
public async Task<IActionResult>
  OnPostCompleteOrderAsync(
    [FromBody] PayPalOrderInfo info)
```

The `OnPost` prefix indicates that this handler responds to HTTP POST requests, while the `CompleteOrder` portion of the method name corresponds directly to the handler specified in the JavaScript callback:

```javascript
fetch(
  '/Basket?handler=CompleteOrder',
  ...
)
```

When the payment has been captured successfully, the browser sends the PayPal order information to this handler, allowing the server to perform the remaining checkout tasks.

Before creating the order itself, the handler retrieves two important pieces of information:

```csharp
var cart =
  _basketService.GetBasket();

var shippingInfoId =
  HttpContext.Session.GetInt32(
    SessionKeys.ShippingInfoIdKey);
```

The basket contains the products that the customer has purchased, while the shipping information was collected earlier in the checkout process and stored in the user's session.

At this point, the application has everything required to create an order:

* The items being purchased.
* The customer's shipping details.
* Confirmation that the PayPal payment has been completed.

If you'd like to explore the complete implementation, the full storefront application source code is available via the GitHub repository linked from the site header.

The next step is to construct and persist the order itself.

---

## 4 Creating the Order

Before continuing, it's worth noting that the storefront application already contains the `Order`, `OrderItem` and `ShippingInfo` models, together with the corresponding Entity Framework migrations and database schema.

The code that follows assumes these models already exist and that the database has been created and migrated successfully.

Your own implementation may differ. For example, applications that require customer accounts may associate orders with a customer record rather than a standalone shipping information record.

With the basket contents and shipping information available, the first responsibility of the `CompleteOrder` handler is to create a new order record.

The handler begins by constructing an `Order` object:

```csharp
var order = new Order
{
  OrderDate =
    DateTime.UtcNow,

  ShippingInfoId =
    shippingInfoId
};
```

At this point, the order contains only the information common to every purchase. The individual products are added in the next step by iterating through the basket contents and creating the associated order items:

```csharp
foreach (var product in cart)
{
  order.Items.Add(
    new OrderItem
    {
      ProductId = product.Id,
      Quantity = 1,
      ProductName = product.Name,
      Price = product.Price
    });
}
```

> The storefront developed throughout *Building Your First Shopping Basket* treats each basket entry as a single purchased item and therefore sets the quantity to 1. More sophisticated basket implementations may track product quantities separately and populate this value accordingly.

Each product in the basket becomes an `OrderItem` associated with the newly created `order`.

Once all items have been added, additional order metadata can be populated:

```csharp
order.Total =
  order.Items.Sum(i => i.Price);

order.OrderStatus =
  OrderStatusConstants.Paid;
```

Because the payment has already been captured successfully by PayPal, the order can immediately be marked as paid.

The storefront also generates a verification code that can later be used for retrieving order information and checking order status:

```csharp
order.VerificationCode =
  Random.Shared.Next(
    100000,
    999999)
  .ToString();
```

This approach generates a simple six digit code that is easy to implement and understand.

Larger production systems may choose to use longer random identifiers or GUID based values to reduce the possibility of collisions. For example:

```csharp
order.VerificationCode =
  Guid.NewGuid()
    .ToString("N")
    .Substring(0, 8)
    .ToUpper();
```

For the purposes of the storefront application developed throughout *Building Your First Shopping Basket*, however, the simpler six digit implementation was sufficient. It allows the verification code concept to be demonstrated without introducing additional complexity or distracting from the overall order processing workflow.

With the order now fully populated, it can be added to the Entity Framework context and persisted:

```csharp
_context.Orders.Add(order);

await _context
  .SaveChangesAsync();
```
For clarity, the examples shown here omit some of the error handling code present in the full storefront implementation. The complete source code is available in the GitHub repository linked from the site header.

At this point, the order has been written to the database and assigned its primary key value. This identifier will be used later when redirecting the customer to the confirmation page.

The next stage of the workflow is to notify the customer that the order has been received successfully.

---

## 5 Sending the Confirmation Email

With the order saved, the next step is to notify the customer that their purchase has been completed successfully.

The storefront application uses an email service built on top of Brevo's HTTP API. If you're interested in the implementation details of that service, see my earlier article:

[Sending Transactional Emails from ASP.NET Core Using Brevo's HTTP API](https://www.practicalcode.dev/blog/sending-transactional-emails-from-aspnet-core-using-brevo-http-api/)

The exact mechanism used to retrieve the customer's email address will vary depending on the application's design. In the storefront application developed throughout *Building Your First Shopping Basket*, this information was collected earlier in the checkout process and stored as part of the customer's shipping information.

Once the email address is available, the confirmation email can be sent:

```csharp
await _emailService.SendEmailAsync(
  shippingInfo.Email,
  "Order Confirmation",
  "Thank you for your order! " +
  "Your verification code is: " +
  order.VerificationCode);
```

The confirmation email includes the verification code generated during order creation. Customers can use this code later to retrieve order information and check order status.

The example shown here intentionally keeps the email content simple so that the focus remains on the order processing workflow. In a production application, you might also include information such as:

* The order number.
* A summary of purchased items.
* The order total.
* Delivery or fulfilment information.

With the order saved and the confirmation email sent, only one step remains: returning the newly created order identifier and redirecting the customer to the confirmation page.

---

## 6 Displaying the Confirmation Page

Once the order has been created successfully, the handler returns the newly generated order identifier to the browser:

```csharp
return new JsonResult(
  new { orderId = order.Id });
```

Back in `paypal.js`, this value is used to redirect the customer to the confirmation page:

```javascript
.then(result => {
  window.location =
    `/Confirmation?id=${result.orderId}`;
});
```

This differs from the implementation shown in the previous article.

There, the customer was redirected to a static confirmation page:

```javascript
window.location.href =
  '/Confirmation';
```

While this was sufficient for demonstrating a successful PayPal integration, it provided no information about the order that had just been created.

The confirmation page in the storefront application accepts the order identifier as a query string parameter and uses it to retrieve the corresponding order from the database.

This allows the page to display useful information about the completed purchase, such as the order number, order total and verification code.

By returning the order identifier from the `CompleteOrder` handler and including it in the redirect, the application can provide a much richer confirmation experience than a simple success message.

---

## 7 Testing the Complete Workflow

With all of the pieces now in place, it's time to verify that the complete checkout process behaves as expected.

To test the workflow:

1. Add one or more products to the basket.
2. Enter the required shipping information, including email address.
3. Complete the PayPal checkout process.
4. Wait for the payment to be captured successfully.

A successful checkout should result in several things happening automatically:

* A new order record should be created in the database.
* The purchased items should be persisted as order items.
* A verification code should be generated.
* A confirmation email should be sent to the customer's email address.
* The customer should be redirected to the confirmation page.
* The confirmation page should display information about the newly created order.

Verifying each of these steps provides confidence that the complete order processing workflow is functioning correctly.

If you're following along with the storefront application developed throughout *Building Your First Shopping Basket*, I also recommend inspecting the generated database records and confirmation email to ensure that the persisted order details match the contents of the customer's basket.

---

In the previous article, we successfully integrated PayPal Sandbox into an ASP.NET Core application and captured customer payments.

In this article, we've built on that foundation by introducing the server side processing required to turn a successful payment into a completed order.

The techniques shown here provide a practical example of how payment processing, order creation and customer communication can be combined into a cohesive checkout workflow.

In the next article, we'll make use of the verification code generated during checkout to allow customers to retrieve order information and track order status without the need for customer accounts.
