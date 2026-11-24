---
title: "Building a Data Seeding System in ASP.NET Core"
description: "Creating a reusable JSON-based data seeding system for ASP.NET Core applications."
pubDate: "Nov 24 2026"
heroImage: "../../assets/aspnet-core-data-seeding.jpg"
---

In my previous article, [Using AI to Generate Realistic Sample Data for an ASP.NET Core Storefront](/blog/using-ai-to-generate-realistic-sample-data-for-an-aspnet-core-storefront/), I explored how AI could be used to generate a realistic catalogue for the vintage clothing storefront I'm currently building. The resulting dataset contained hundreds of products complete with categories, sizes, colours, pricing information and inventory levels.

Generating the data was only part of the challenge, however. To make the catalogue useful, it needed to be imported into the application and persisted to the database in a way that was both reliable and maintainable.

In this article, I'll look at how I built a reusable data seeding system for ASP.NET Core, allowing structured JSON data to be loaded into the application's database during startup. Although the examples focus on products, the same approach can be applied to any type of reference or seed data.

## 1. Why Build a Data Seeding System?

In the previous article, I used AI to generate a richer, more realistic catalogue of vintage clothing products.

As part of that process, the application's domain model evolved to accommodate the various edge cases and inconsistencies exposed by the generated data. New concepts such as `BaseColour` were introduced to support filtering, while sizing rules became more sophisticated to cater for the differing requirements of clothing, footwear and accessories.

By the end of the exercise, I had a catalogue containing hundreds of realistic products represented as JSON. The next challenge was finding a reliable and maintainable way for the application to consume that data and populate the database.

In *Building Your First Shopping Basket*, the storefront is initially seeded using a small number of products defined directly within the `SeedData` class. While this approach works well for a simple demonstration application, it was no longer suitable for managing a catalogue containing hundreds of products.

Rather than continually expanding the existing seeding logic, I decided to build a more flexible data seeding system capable of importing structured data from external JSON files.

The first step was ensuring that the application's domain model reflected the richer catalogue generated during the previous article.

---

## 2. Extending the Domain Model

Before importing the catalogue, I needed to revisit the application's product model.

The version of the storefront developed in *Building Your First Shopping Basket* was intentionally simple. Products contained only the fields required to demonstrate the core shopping basket functionality.

The catalogue generated in the previous article was considerably richer and exposed several gaps in the original model.

Products now needed to support additional attributes such as audience, condition, size and colour information. During the modelling process, it also became apparent that some concepts required more structure than a simple string property could provide.

For example, a descriptive colour such as "Deep Plum" works well when presenting a product to customers, but it is less useful when implementing filtering and search functionality. This led to the introduction of a separate `BaseColour` field, allowing products to retain their descriptive colours whilst also belonging to a manageable set of filterable colour categories.

Similarly, categories, audiences and conditions were promoted to enum types. This provided stronger validation and ensured that imported data remained consistent across the application.

The resulting product model was considerably more expressive than the version used in the original storefront whilst still remaining straightforward to work with.

The following enums were introduced to support the richer catalogue structure:

```csharp
public enum ProductCategory
{
    Tops,
    Bottoms,
    DressesAndJumpsuits,
    Outerwear,
    LayeringAndKnitwear,
    Footwear,
    Accessories
}

public enum Audience
{
    Mens,
    Womens,
    Boys,
    Girls,
    Unisex
}

public enum BaseColour
{
    Black,
    Blue,
    Brown,
    Green,
    Grey,
    Orange,
    Pink,
    Purple,
    Red,
    White,
    Yellow,
    Beige
}
```

With the domain model updated, the next task was designing a way to load the JSON catalogue into the application.

---

## 3. Building the Seeding Infrastructure

After tackling the domain model, the next challenge was designing a way to load the JSON catalogue into the application.

Although it would have been possible to deserialize the JSON directly into the `Product` entity, I wanted to introduce an intermediate layer between the external data source and the application's domain model.

To achieve this, I created a dedicated `ProductSeedDto` type representing the structure of the data contained within the JSON file.

The DTO mirrors the structure of the JSON document whilst remaining separate from the `Product` entity used by Entity Framework Core.

Using a dedicated DTO follows the Data Transfer Object (DTO) pattern, creating a clear boundary between imported data and the application's domain model. This provides a convenient place to perform validation, mapping and enum conversion before entities are persisted to the database.

The catalogue itself was stored as a JSON document within the application's `Data` folder:

```text
Data/
└── products.json
```

Separating the data from the application's code makes the catalogue easier to maintain and update. New products can be added simply by modifying the JSON file without needing to make changes to the seeding logic itself.

The seeding process follows a straightforward pipeline:

```text
products.json
        ↓
ProductSeedDto
        ↓
Product
        ↓
Database
```

JSON data is first deserialized into a collection of `ProductSeedDto` instances. These DTOs are then mapped to the application's domain entities before being persisted to the database.

With the import structure in place, the final step was to load the catalogue during application startup and create the corresponding product records.

---

## 4. Seeding the Database

Before product data can be imported, the application's category data must already exist.

Unlike products, categories represent a small and relatively stable set of reference data. For this reason, the storefront continues to seed categories directly within the `SeedData` class:

```csharp
context.Categories.AddRange(
    new Category { Name = "Tops" },
    new Category { Name = "Bottoms" },
    new Category { Name = "Dresses & Jumpsuits" },
    new Category { Name = "Outerwear" },
    new Category { Name = "Layering & Knitwear" },
    new Category { Name = "Footwear" },
    new Category { Name = "Accessories" }
);
```

Once the category data has been created, the product import process can begin.

The product import process begins by checking whether products already exist in the database. If products have already been loaded, the import routine exits immediately, preventing duplicate catalogue entries from being created.

```csharp
if (context.Products.Any())
{
    return;
}
```

With the prerequisite data in place, the JSON catalogue can be loaded and deserialized into a collection of `ProductSeedDto` instances:

```csharp
    var json =
      File.ReadAllText(
        "Data/products.json");

    var options = new JsonSerializerOptions
    {
        PropertyNameCaseInsensitive = true,
        Converters = { new JsonStringEnumConverter() }
    };

    var seedProducts =
      JsonSerializer.Deserialize<
        List<ProductSeedDto>>(json, options);
```

Because products reference categories, a lookup dictionary is created from the existing category records:

```csharp
var categories =
    context.Categories
        .ToDictionary(
            c => c.Name,
            c => c);
```

The lookup dictionary allows imported products to be associated with existing category records without repeatedly querying the database.

At this point, all the information required to construct the application's domain entities is available. Each `ProductSeedDto` can therefore be mapped to a corresponding `Product` entity before being added to the database context:

```csharp
foreach (var seedProduct in seedProducts)
{
    context.Products.Add(
        new Product
        {
            Name = seedProduct.Name,
            Description = seedProduct.Description,
            Price = seedProduct.Price,
            Audience = seedProduct.Audience,
            BaseColour = seedProduct.BaseColour,
            Category =
                categories[
                    seedProduct.Category]
        });
}
```

Whilst the actual implementation maps every property, the overall process remains straightforward. The JSON catalogue is deserialized into DTOs, each DTO is converted into a domain entity, and the resulting entities are added to the database context.

Finally, the changes are persisted:

```csharp
context.SaveChanges();
```

At this point, the application is populated with a complete catalogue sourced entirely from the external JSON document.

---

Moving the catalogue into an external JSON file has transformed what was once a simple demonstration seeding mechanism into a far more flexible and maintainable data seeding system.

As the storefront continues to evolve, new products can be added, existing products can be updated and entire catalogues can be regenerated without requiring changes to the application's code. More importantly, the seeding process now mirrors the way real-world data is managed, with content and application logic kept clearly separated.

With a realistic catalogue now flowing into the database, the next challenge is making that data available to customers through product listings, search, filtering and other storefront features.




