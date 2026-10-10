# Module 8: Owned Entities and Complex Types

In Domain-Driven Design (DDD), we frequently use **Value Objects**—small, cohesive objects that describe properties but don't have a unique identity of their own. For example, an `Address` (Street, City, ZipCode) is just a collection of properties describing where a `User` lives. 

Entity Framework Core provides two distinct ways to map these concepts to a relational database: **Owned Entities** and the newly introduced **Complex Types** (EF Core 8+).

---

## 1. The Core Problem

Imagine we have a `User` class, and we want to store their `Address`.

```csharp
public class User
{
    public Guid Id { get; set; }
    public string Name { get; set; }
    
    // We want this to be a cohesive object in C#!
    public Address HomeAddress { get; set; } 
}

public class Address
{
    public string Street { get; set; }
    public string City { get; set; }
    public string ZipCode { get; set; }
}
```

If we just leave this as-is, EF Core will try to treat `Address` as a completely separate Entity with its own table, and will crash because `Address` doesn't have a Primary Key (`Id`). 

We don't want a separate `Addresses` table. We want the `Street`, `City`, and `ZipCode` columns to live inside the `Users` table (a concept called **Table Splitting**).

---

## 2. Approach 1: Owned Entities (`OwnsOne` / `OwnsMany`)

Introduced in earlier versions of EF Core, **Owned Entities** tell EF Core that the `Address` type is entirely "owned" by the `User` type. It cannot exist without a User.

### How to Configure `OwnsOne`

```csharp
public class UserConfiguration : IEntityTypeConfiguration<User>
{
    public void Configure(EntityTypeBuilder<User> builder)
    {
        builder.HasKey(u => u.Id);

        // Tell EF Core that User OWNS the Address
        builder.OwnsOne(u => u.HomeAddress, addressBuilder =>
        {
            // We can configure the nested properties here!
            addressBuilder.Property(a => a.Street).HasColumnName("HomeStreet").HasMaxLength(100);
            addressBuilder.Property(a => a.City).HasColumnName("HomeCity").HasMaxLength(50);
            addressBuilder.Property(a => a.ZipCode).HasColumnName("HomeZipCode").HasMaxLength(20);
        });
    }
}
```

### The Database Schema Result
EF Core will create a **single** `Users` table through "Table Splitting":

| Id (PK) | Name | HomeStreet | HomeCity | HomeZipCode |
| :--- | :--- | :--- | :--- | :--- |
| `1` | `Sanjeev` | `123 Tech Ln` | `Delhi` | `110001` |

### The `OwnsMany` Collection
Owned entities also support collections. If a user has multiple `ShippingAddresses`, you can use `.OwnsMany()`.

```csharp
public class User
{
    public Guid Id { get; set; }
    public ICollection<Address> ShippingAddresses { get; set; }
}

// In Fluent API:
builder.OwnsMany(u => u.ShippingAddresses, addressBuilder =>
{
    // Because it's a collection, EF Core MUST create a separate table for this.
    addressBuilder.ToTable("UserShippingAddresses");
});
```

### The "Hidden Identity" Problem of Owned Entities
While `OwnsOne` looks perfect, it has a major architectural quirk. EF Core is fundamentally an Entity tracker. Therefore, behind the scenes, **EF Core secretly assigns a hidden Primary Key** to every Owned Entity to track it. 

Because it's secretly tracked as an entity:
1.  **Immutability Issues:** If you try to replace a user's address by assigning a completely new `Address` object (`user.HomeAddress = new Address(...)`), EF Core sometimes throws tracking exceptions because it thinks you are trying to delete one tracked entity and insert a new one with a conflicting key.
2.  **Performance Overhead:** The Change Tracker wastes memory tracking the hidden keys of these objects.
3.  **Not true Value Objects:** In DDD, Value Objects have no identity. Owned Entities pretend they don't, but secretly do.

---

## 3. Approach 2: Complex Types (The EF Core 8+ Solution)

To solve the "hidden identity" problem, Microsoft introduced **Complex Types** in EF Core 8. 

Complex Types are true Value Objects. They have **no identity** (no hidden primary keys), they are not tracked as separate entities, and they must always be stored in the same table as their parent (Table Splitting).

### How to Configure a Complex Type

There are two ways to define a Complex Type.

**Method 1: Data Annotations**
```csharp
// Tell EF Core this is NOT an entity, it's just a Complex Type!
[ComplexType]
public class Address
{
    public string Street { get; set; }
    public string City { get; set; }
}
```

**Method 2: Fluent API (Preferred)**
```csharp
public class UserConfiguration : IEntityTypeConfiguration<User>
{
    public void Configure(EntityTypeBuilder<User> builder)
    {
        // Notice we use .ComplexProperty() instead of .OwnsOne()!
        builder.ComplexProperty(u => u.HomeAddress, addressBuilder =>
        {
            addressBuilder.Property(a => a.City).HasColumnName("HomeCity");
        });
    }
}
```

### The "No Primary Key" Advantage (Change Tracking)
The biggest and most important difference between a Complex Type and an Owned Entity (`OwnsOne`) is that **Complex Types do not create or receive a primary key**. 

Because it has no primary key, **EF Core does not track it as an entity.**

*   **Owned Entities (`OwnsOne`):** Secretly have a Primary Key. They are tracked as entities. This tracking overhead can sometimes cause exceptions if you try to completely replace the object (e.g., `user.HomeAddress = new Address()`), because EF Core gets confused about whether you are deleting the old tracked key and inserting a new one.
*   **Complex Types (`[ComplexType]`):** Have NO Primary Key. They are pure structural data. EF Core treats the `Address` object exactly the same way it treats a simple `string`. If you do `user.HomeAddress = new Address { City = "Delhi" }`, EF Core just says, *"Okay, the user's City property changed,"* and executes a simple `UPDATE` without throwing tracking exceptions.

### Complex Types vs Owned Entities

| Feature | Owned Entity (`OwnsOne`) | Complex Type (`ComplexProperty`) |
| :--- | :--- | :--- |
| **Has an Identity (PK)?** | Yes (Secretly managed by EF Core) | **No** (True Value Object) |
| **Supports Collections?** | Yes (`OwnsMany`) | **No** (Cannot do `List<Address>`) |
| **Can be mapped to a separate table?**| Yes (via `.ToTable()`) | **No** (Must be table-split) |
| **Allows null values?** | Yes | **No** (Complex Type instances cannot be null in memory) |
| **Reassignment Behavior** | Can cause tracking errors | Flawless (Treats it like replacing a string) |

### The "Cannot Be Null" Rule for Complex Types
The most important quirk of Complex Types is that **the object itself cannot be null**.

```csharp
// 🚨 ERROR: If you save this, EF Core will throw an exception!
var user = new User { Name = "Sanjeev", HomeAddress = null }; 
```

Even if every column (`HomeCity`, `HomeStreet`) is nullable in the database, EF Core requires the C# object to be instantiated. If the database row has all `null` columns for the address, EF Core will automatically instantiate a blank `new Address()` object when reading the row.

---

## 4. Value Converters vs Complex Types

We learned about Value Conversions in Module 7. When should you use a Converter vs a Complex Type?

*   **Use a Value Converter** when you want to take a complex C# object (like `List<string>` or `EmailAddress`) and flatten it into a **single column** in the database (e.g., a JSON string or a `VARCHAR`).
*   **Use a Complex Type** when you want to take a cohesive C# object (like `Address`) and flatten it into **multiple columns** in the database (e.g., `Street` column, `City` column).

---

## 5. Querying: Do I need `.Include()`?

A common question is whether you need to use `.Include()` to fetch Owned Entities or Complex Types. **The answer is NO.**

EF Core considers Owned Types and Complex Types to be fundamental parts of the parent entity, not separate entities that require explicit joining.

### 1. `OwnsOne` and `ComplexType` (Table Splitting)
Because these are stored in the exact same database table as the parent entity, when you query `dbContext.Users.ToList()`, EF Core naturally pulls all columns from the `Users` table (including the `HomeCity` and `HomeStreet` columns) and automatically populates your `user.HomeAddress` object. 

### 2. `OwnsMany` (Separate Tables)
Even though an `OwnsMany` collection requires a separate database table, EF Core **still automatically loads them**. 

Because you told EF Core that the `User` "owns" those shipping addresses, EF Core assumes the `User` is incomplete without them. Therefore, when you query `dbContext.Users.ToList()`, EF Core secretly performs the `JOIN` and eagerly loads the `OwnsMany` collection behind the scenes, without you ever typing `.Include()`. 

*(Note: If your `OwnsMany` collection is absolutely massive and you want to prevent EF Core from automatically loading it every time you fetch a User, you usually have to redesign the architecture to use a standard `HasMany()` navigation relationship instead of `OwnsMany()`, so that you regain manual control over when it loads.)*

---

## 6. Real-World Scenarios

### Scenario A: True Value Objects in DDD (Complex Types)
In Normora, a `Money` value object consists of an `Amount` and a `Currency`. It is a perfect candidate for a Complex Type.

```csharp
public class Money
{
    public decimal Amount { get; set; }
    public string Currency { get; set; }
}

public class Invoice
{
    public Guid Id { get; set; }
    public Money TotalPrice { get; set; } = new(); // MUST be instantiated!
}

// Configuration
builder.ComplexProperty(i => i.TotalPrice, m => 
{
    m.Property(p => p.Amount).HasColumnType("decimal(18,2)");
    m.Property(p => p.Currency).HasMaxLength(3);
});
```

### Scenario B: Audit Metadata (Complex Types)
Instead of adding `CreatedAt` and `CreatedBy` directly to every entity, group them into an `AuditInfo` complex type.

```csharp
[ComplexType]
public class AuditInfo
{
    public DateTime CreatedAt { get; set; }
    public Guid CreatedBy { get; set; }
}

public class Tenant
{
    public Guid Id { get; set; }
    public AuditInfo Audit { get; set; } = new();
}
```
In the database, the `Tenants` table will have `Audit_CreatedAt` and `Audit_CreatedBy` columns automatically.

### Scenario C: Unbounded Child Records (`OwnsMany`)
When a Tenant has a list of allowed IP configurations, and we want them stored in a separate table but managed entirely through the `Tenant` aggregate root.

```csharp
public class Tenant
{
    public Guid Id { get; set; }
    // We cannot use ComplexType for collections. We MUST use Owned Entities.
    public ICollection<IpConfiguration> AllowedIps { get; set; }
}

builder.OwnsMany(t => t.AllowedIps, ip => 
{
    ip.ToTable("TenantIpConfigurations"); // Creates a separate table
    ip.Property(i => i.IpAddress).HasMaxLength(45);
});
```

---

## 7. Seeding Data (`HasData`)

When you want to seed data (using `.HasData()`) for an entity that contains a Complex Type or an Owned Entity, EF Core treats them very differently.

### 1. Seeding Data with Complex Types (EF Core 8+)
Because Complex Types are true value objects (they are just properties of the parent entity, not separate entities themselves), **you seed them directly inside the parent's object.** It feels very natural, exactly like writing normal C# code:

```csharp
public class UserConfiguration : IEntityTypeConfiguration<User>
{
    public void Configure(EntityTypeBuilder<User> builder)
    {
        builder.HasKey(u => u.Id);
        
        // Define it as a complex type
        builder.ComplexProperty(u => u.HomeAddress);

        // SEEDING: Just instantiate the complex type directly inside the parent!
        builder.HasData(
            new User 
            { 
                Id = Guid.Parse("11111111-1111-1111-1111-111111111111"),
                Name = "Sanjeev",
                HomeAddress = new Address 
                { 
                    Street = "123 Tech Ln", 
                    City = "Delhi"
                } 
            }
        );
    }
}
```

### 2. Seeding Data with Owned Entities (`OwnsOne`)
Seeding Owned Entities is much trickier! Because EF Core secretly treats the Owned Entity as a separate tracked object with a hidden Foreign Key linking it to the parent, **you cannot seed it directly inside the parent object.**

You must seed the parent first, and then seed the Owned Entity separately using an **Anonymous Object** so you can hardcode the hidden Foreign Key (e.g., `UserId`) to link them together!

```csharp
public class UserConfiguration : IEntityTypeConfiguration<User>
{
    public void Configure(EntityTypeBuilder<User> builder)
    {
        var userId = Guid.Parse("11111111-1111-1111-1111-111111111111");

        // 1. Seed the Parent FIRST
        builder.HasData(new User { Id = userId, Name = "Sanjeev" });

        // 2. Configure the Owned Entity
        builder.OwnsOne(u => u.HomeAddress, addressBuilder =>
        {
            // 3. SEED the Owned Entity SEPARATELY using an anonymous object!
            // Notice how we must manually provide the hidden Foreign Key ('UserId')
            addressBuilder.HasData(
                new 
                { 
                    UserId = userId, // Hidden FK linking back to the User!
                    Street = "123 Tech Ln", 
                    City = "Delhi"
                }
            );
        });
    }
}
```

---

## 8. Interview Questions

**Q1: What is Table Splitting in EF Core?**
> Table Splitting is a technique where multiple C# classes are mapped to a single table in the database. For example, if a `User` class has an `Address` property (a Complex Type or Owned Entity), EF Core flattens the `Address` properties into columns within the `Users` table, preventing the need for an expensive SQL `JOIN` when querying.

**Q2: What is the primary difference between an Owned Entity (`OwnsOne`) and a Complex Type in EF Core 8?**
> The fundamental difference is **Identity**. An Owned Entity is secretly tracked as an entity by EF Core and is assigned a hidden primary key. A Complex Type is a true Value Object; it has no identity, no hidden key, and ignores the Change Tracker's entity lifecycle. Complex types are generally preferred for pure value objects because they avoid tracking overhead and reassignment exceptions.

**Q3: Can a Complex Type be mapped to its own separate database table?**
> No. Complex Types are strictly structural types and must always be mapped to the same table as their owning entity (Table Splitting). If you need the child object to exist in a separate table, you must use an Owned Entity (`OwnsOne` with `.ToTable()`) or a standard navigation relationship (`HasOne`).

**Q4: Can you use a Complex Type for a collection property (e.g., `List<Address>`)?**
> No. Complex Types do not support collections. If you need a collection of child objects that are wholly owned by the parent, you must use the `OwnsMany()` configuration, which will automatically create a separate database table with a hidden foreign key pointing back to the parent.

**Q5: What happens if you assign `null` to a Complex Type property on an entity?**
> EF Core will throw an exception when you attempt to save or track the entity. Complex Type properties must always be instantiated in memory (e.g., `= new Address()`), even if all the underlying columns in the database are completely null. This is a key restriction of Complex Types compared to Owned Entities.

**Q6: When would you choose a Value Converter over a Complex Type?**
> You use a Value Converter when you want to map a complex C# object (like a `List<string>` or a custom `Email` class) to a **single database column** (like a JSON `NVARCHAR` or a regular string). You use a Complex Type when you want to map a cohesive C# object (like `Address`) across **multiple distinct database columns** (`Street`, `City`, `Zip`).

---

## Summary
In this module, we explored how to map Domain-Driven Design Value Objects to relational databases. We learned that while **Owned Entities** (`OwnsOne`/`OwnsMany`) are powerful and support collections and separate tables, they come with hidden identity tracking overhead. EF Core 8's **Complex Types** provide a strict, high-performance alternative for pure Value Objects that require no identity and are mapped via table splitting.
