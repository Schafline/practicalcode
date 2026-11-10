---
title: "Implementing Order Status Tracking in ASP.NET Core"
description: "Learn how to implement order status tracking in ASP.NET Core using verification codes, allowing customers to retrieve order information without creating an account."
pubDate: "Nov 10 2026"
heroImage: "../../assets/order-status-tracking-aspnet-core.png"
---

In the [previous article](https://www.practicalcode.dev/blog/paypal-order-processing-aspnet-core/), we generated a unique verification code for each order and included it in the confirmation email. In this article, we'll use that code to allow customers to retrieve order information and track order status without requiring customer accounts.

The snippets shown throughout this article have been extracted from the storefront application developed throughout *Building Your First Shopping Basket*. If you'd like to explore the complete implementation, the full source code is available in the GitHub repository linked from the site header.

## 1 Why Verification Codes?

Many e-commerce applications allow customers to view previous orders by signing into a customer account.

The storefront application developed throughout *Build  ing Your First Shopping Basket* takes a simpler approach. Each order is assigned a unique verification code during checkout and that code is included in the order confirmation email.

Customers can later use that code to retrieve information about their order without needing to create an account or remember additional login credentials.

The first step is providing customers with a simple page where they can enter that verification code and begin the lookup process.

---

## 2 Creating the Order Lookup Page

The first step is providing customers with a page where they can enter the verification code received in their confirmation email.

To implement this functionality, I created a new Razor Page named `OrderLookup`.

The page contains a simple form with a single input field bound to a `VerificationCode` property, together with a submit button:

```csharp
@page
@model OrderLookupModel

<h2>Find Your Order</h2>

@if (!string.IsNullOrEmpty(
  Model.ErrorMessage))
{
  <div class="alert alert-danger">
    @Model.ErrorMessage
  </div>
}

<form method="post">
  <div class="form-group">
    <label for="VerificationCode">
      Verification Code
    </label>

    <input type="text" id="VerificationCode" name="VerificationCode" class="form-control" asp-for="VerificationCode" />

    <span asp-validation-for="VerificationCode" class="text-danger">
    </span>
  </div>

  <button type="submit" class="btn btn-primary">
    View Order
  </button>
</form>

@section Scripts {
  <partial name="_ValidationScriptsPartial" />
}
```

The form shown above is backed by a page model located in `OrderLookup.cshtml.cs`.

This introduces a bindable `VerificationCode` property together with some basic validation to ensure that a value is supplied before the lookup process begins:

```csharp
[BindProperty]
[Required(
  ErrorMessage =
    "Please enter your " +
    "verification code.")]
public string VerificationCode
{
  get;
  set;
}
```

Once the form has been submitted successfully, the page model can attempt to retrieve the corresponding order from the database.

---

## 3 Retrieving the Order

With the lookup page in place, the next step is retrieving the corresponding order from the database.

To accomplish this, I injected the application's `ShopContext` into the page model:

```csharp
private readonly ShopContext
  _context;

public OrderLookupModel(
  ShopContext context)
{
  _context = context;
}
```

`ShopContext` provides access to the storefront application's Entity Framework models, including the `Orders` table.

Once the submitted verification code has passed validation, it can be normalised and used to query the database:

```csharp
var code = VerificationCode
  .Trim()
  .ToUpperInvariant();

var order = _context.Orders
  .FirstOrDefault(o =>
    o.VerificationCode ==
      code);
```

If no matching order is found, the customer remains on the lookup page and is presented with a validation message:

```csharp 
if (order == null) 
{
     ErrorMessage =
      "We couldn't find an "
       + "order with that "
       + "verification code.";
     return Page(); 
}
```

When a matching order is found, the workflow can continue by redirecting the customer to the order details page.

---

## 4 Creating the Order Details Page

Once a matching order has been found, the final step is displaying the order's details to the customer.

Rather than rendering the information directly on the lookup page, I chose to redirect the customer to a dedicated `OrderDetails` page:

```csharp
return RedirectToPage(
  "OrderDetails",
  new
  {
    code =
      VerificationCode
  });
```

This keeps the responsibilities of each page clearly separated. The lookup page is responsible for locating the order, while the order details page is responsible for displaying it.

The corresponding page model accepts the verification code as a route parameter and uses it to retrieve the matching order:

```csharp
public IActionResult OnGet(
  string code)
{
  if (string.IsNullOrWhiteSpace(
    code))
  {
    return RedirectToPage(
      "OrderLookup");
  }

  var normalisedCode = code
    .Trim()
    .ToUpperInvariant();

  Order = _context.Orders
    .Include(o => o.Items)
    .FirstOrDefault(o =>
      o.VerificationCode ==
        normalisedCode);

  if (Order == null)
  {
    return RedirectToPage(
      "OrderLookup",
      new { notfound = true });
  }

  return Page();
}
```

With the order successfully retrieved, the final task is presenting the information back to the customer.

---

## 5 Displaying Order Information

To display order information, I created a dedicated `OrderDetails` page that displays three categories of information:

* An order summary.
* The purchased items.
* The shipping information associated with the order.

The order summary displays key information such as the order number, order date, verification code and current order status:

```csharp
<dt class="col-sm-3">Order Number</dt>
<dd class="col-sm-9">
  @Model.Order.Id
</dd>

<dt class="col-sm-3">Order Date</dt>
<dd class="col-sm-9">
  @(Model.Order.OrderDate
    .ToString("dd MMM yyyy"))
</dd>

<dt class="col-sm-3">
  Verification Code
</dt>
<dd class="col-sm-9">
  @Model.Order.VerificationCode
</dd>

<dt class="col-sm-3">Status</dt>
<dd class="col-sm-9">
  @Model.Order.OrderStatus
</dd>
```

Beneath the summary, the page displays the items associated with the order:

```csharp
@foreach (var item in
  Model.Order.Items)
{
  <tr>
    <td>@item.ProductName</td>
    <td class="text-end">
      @item.Quantity
    </td>
    <td class="text-end">
      @(item.Price
        .ToString("C"))
    </td>
  </tr>
}
```

The page also displays the shipping information captured during checkout:

```csharp
<p>
  <strong>Name:</strong>
  @(Model.Order.ShippingInfo
    .FullName)
</p>

<p>
  <strong>Address:</strong>
  @(Model.Order.ShippingInfo
    .AddressLine1)
</p>

@if (!string.IsNullOrWhiteSpace(
      Model.Order.ShippingInfo
        .AddressLine2))
{
  <p>
    @(Model.Order.ShippingInfo
      .AddressLine2)
  </p>
}

<p>
  @Model.Order.ShippingInfo.City
</p>

<p>
  @Model.Order.ShippingInfo.Postcode
</p>
```

Together, the order summary, items table and shipping information provide customers with a complete view of their purchase without requiring them to create an account or sign in.

---

## 6 Testing the Workflow

With both pages in place, it's time to verify that the complete lookup workflow behaves as expected.

To test the implementation:

1. Complete an order and make a note of the verification code included in the confirmation email.
2. Navigate to the Order Lookup page.
3. Enter the verification code and submit the form.
4. Confirm that the application redirects to the Order Details page.

A successful lookup should display:

* The order summary.
* The purchased items.
* The shipping information.
* The current order status.

It's also worth testing a few failure scenarios.

For example, submitting an empty verification code should trigger the validation message defined earlier, while entering an invalid code should leave the customer on the lookup page and display an error message indicating that no matching order could be found.

Verifying both the successful and unsuccessful paths helps ensure that the lookup workflow behaves predictably and provides useful feedback when problems occur.

---

## Conclusion

In the previous article, we generated a unique verification code for each order and included it in the confirmation email.

In this article, we've used that verification code to allow customers to retrieve order information and view the details of their purchase without requiring customer accounts or authentication.

The result is a simple order lookup workflow that provides customers with access to order and shipping information long after the checkout process has completed.