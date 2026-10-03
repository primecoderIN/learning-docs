# Module 5: Modeling Relationships and Navigation Properties

In a relational database, tables rarely exist in isolation. They relate to one another. Entity Framework Core provides an incredibly powerful, albeit sometimes confusing, system for managing these relationships. 

To master relationships in EF Core, you must understand how **Navigation Properties**, the **Fluent API**, and **JSON Serialization** work together.

---

## 1. The Three Layers of EF Core (The Mental Model)

The most important mental model to hold when building relationships is that EF Core operates across three distinct layers. When you run into an issue, you must identify *which layer* is causing the problem.

```text
              C# OBJECT MODEL (Layer 1)
                     │
              Navigation Props (.User, .Tenants)
                     │
                     ▼
                EF CORE MODEL (Layer 2)
                     │
                Fluent API (.HasOne, .WithMany)
                     │
                     ▼
              DATABASE RELATIONSHIP (Layer 2.5)
                     │
               Foreign Keys (TenantId)
                     │
                     ▼
           API SERIALIZATION (Layer 3)
                     │
         [JsonIgnore] / DTO Mapping
                     │
                     ▼
               JSON RESPONSE
```

### Layer 1: C# Object Model (Navigation Properties)
Navigation properties represent how you navigate your objects in C#. They exist purely to express the relationship in the C# object graph.
*   **Example:** `tenant.Users` or `user.Department`

### Layer 2: EF Core Mapping (Fluent API)
The Fluent API tells EF Core *how* those C# object navigation properties map to the relational database.
*   **Example:** `.HasOne().WithMany().HasForeignKey()`

### Layer 3: API / JSON Representation (`[JsonIgnore]`)
Attributes like `[JsonIgnore]` control what gets exposed during JSON serialization when returning data from your API. They have **absolutely nothing** to do with how the database is configured.

> **Golden Rule:** Navigation properties describe how you want to navigate between entities in C#. The Fluent API tells EF Core how that relationship actually maps to the database tables and foreign keys.

---

## 2. Defining Navigation Properties (Bidirectional vs. Unidirectional)

When defining a relationship, you have to decide if you want to navigate it from **both sides** or just **one side**.

### 1. Navigating from Both Sides (Bidirectional)
This is the most common approach. Both entities have a navigation property pointing to each other.

```csharp
public class Tenant { public ICollection<User> Users { get; set; } }
public class User { public Tenant Tenant { get; set; } }
```
*   **Why use it:** It gives you maximum flexibility in your C# code. You can easily do `user.Tenant.Name` AND `tenant.Users.Count()`.
*   **Fluent API:** Both properties are mentioned: `.HasOne(u => u.Tenant).WithMany(t => t.Users)`

### 2. Navigating from One Side Only (Unidirectional)
You are **not required** to define navigation properties on both sides. What happens if you only add it to one side?

**Example:** A `User` belongs to a `Tenant`, but you never intend to load all `Users` directly from the `Tenant` object. 

```csharp
public class Tenant { /* No Users collection! */ }
public class User { public Tenant Tenant { get; set; } }
```
*   **What happens in EF Core?** EF Core is perfectly fine with this! The foreign key in the database (`TenantId` on the `Users` table) is created exactly the same way. The relationship still exists in the database.
*   **Why use it:** It keeps your domain classes cleaner and prevents massive accidental data loading if you serialize a `Tenant`. 
*   **Fluent API:** You simply leave the second part empty! `.HasOne(u => u.Tenant).WithMany()`

---

## 3. One-to-Many Relationships

This is the most common relationship. Let's look at `Tenant` and `Users` in Normora.

**Rule:** One Tenant can have many Users.

```csharp
public class Tenant
{
    public Guid Id { get; set; }
    public string Name { get; set; }

    // Collection Navigation Property
    public ICollection<User> Users { get; set; } = new List<User>();
}

public class User
{
    public Guid Id { get; set; }
    public string Name { get; set; }

    // Foreign Key Property
    public Guid TenantId { get; set; }

    // Reference Navigation Property
    public Tenant Tenant { get; set; }
}
```

### Why put the Foreign Key on `User`?
Because there are many users. Each user needs to answer: *"Which tenant do I belong to?"* Therefore, `User.TenantId` is the foreign key. You don't put `Tenant.UserId` because a Tenant has multiple users.

### The Fluent API Mapping
Even though EF Core can often guess the relationship via conventions, using the Fluent API gives you explicit control over delete behaviors and exact column mappings:

```csharp
modelBuilder.Entity<User>()
    .HasOne(u => u.Tenant)             // User has ONE Tenant
    .WithMany(t => t.Users)            // Tenant has MANY Users
    .HasForeignKey(u => u.TenantId)    // User.TenantId is the Foreign Key
    .OnDelete(DeleteBehavior.Restrict); // Prevent deleting a Tenant if Users exist
```

---

## 4. One-to-One Relationships

A strict one-to-one relationship means both sides point to a single entity. 

**Rule:** A `Tenant` has exactly one `TenantBranding` profile, and that `TenantBranding` belongs exclusively to that `Tenant`.

```csharp
public class Tenant
{
    public Guid Id { get; set; }
    public TenantBranding Branding { get; set; }
}

public class TenantBranding
{
    public Guid Id { get; set; }
    public string PrimaryColor { get; set; }
    
    // Foreign Key
    public Guid TenantId { get; set; }
    public Tenant Tenant { get; set; }
}
```

### The Fluent API Mapping
```csharp
modelBuilder.Entity<Tenant>()
    .HasOne(t => t.Branding)
    .WithOne(b => b.Tenant)
    .HasForeignKey<TenantBranding>(b => b.TenantId)
    .OnDelete(DeleteBehavior.Cascade); // If Tenant is deleted, delete Branding!
```

---

## 5. Many-to-Many Relationships

**Rule:** A `User` can be part of many `Department`s, and a `Department` can contain many `User`s.

In a relational database, you cannot store this directly using a single foreign key. You need a **Join Table**. Modern EF Core can handle this automatically for you.

```csharp
public class User
{
    public Guid Id { get; set; }
    public ICollection<Department> Departments { get; set; } = new List<Department>();
}

public class Department
{
    public Guid Id { get; set; }
    public ICollection<User> Users { get; set; } = new List<User>();
}
```

### The Fluent API Mapping
```csharp
modelBuilder.Entity<User>()
    .HasMany(u => u.Departments)
    .WithMany(d => d.Users)
    .UsingEntity(j => j.ToTable("UserDepartments")); // Explicitly name the hidden join table
```

### Many-to-Many with an Explicit Join Entity
If the relationship itself has additional data (e.g., we want to know what `Role` a user has in a specific department, or the date they joined the department), we must create a join entity.

```csharp
public class UserDepartment
{
    public Guid UserId { get; set; }
    public User User { get; set; }

    public Guid DepartmentId { get; set; }
    public Department Department { get; set; }

    public string Role { get; set; } // Extra data on the relationship!
    public DateTime JoinedDate { get; set; }
}
```

When you do this, you are actually creating **two One-to-Many relationships** that form a Many-to-Many:

```csharp
// Define the composite primary key
modelBuilder.Entity<UserDepartment>()
    .HasKey(ud => new { ud.UserId, ud.DepartmentId });

// Map User -> UserDepartment
modelBuilder.Entity<UserDepartment>()
    .HasOne(ud => ud.User)
    .WithMany(u => u.UserDepartments)
    .HasForeignKey(ud => ud.UserId);

// Map Department -> UserDepartment
modelBuilder.Entity<UserDepartment>()
    .HasOne(ud => ud.Department)
    .WithMany(d => d.UserDepartments)
    .HasForeignKey(ud => ud.DepartmentId);
```

---

## 6. The Crucial Role of `[JsonIgnore]`

If you fetch a `Tenant` and its `Users` and return it directly from an API controller, the JSON serializer will try to serialize the `Tenant`. 
1. It sees the `Users` collection and starts serializing the users.
2. Inside each `User`, it sees the `Tenant` navigation property.
3. It starts serializing the `Tenant` again... which has `Users`... which has a `Tenant`...

This causes an infinite loop known as a **Circular Reference Exception**.

To fix this, you apply `[JsonIgnore]` to the "child" navigation property.

```csharp
public class User
{
    public Guid Id { get; set; }
    public Guid TenantId { get; set; }

    [JsonIgnore]
    public Tenant Tenant { get; set; }
}
```

### The Key Distinction
`[JsonIgnore]` **does not configure or remove the EF Core relationship.** 
It only tells the JSON serialization layer: *"Don't serialize this navigation property."* EF Core will still fully track the foreign keys and allow you to write `.Include(u => u.Tenant)` in your C# code!

---

## 7. The Fluent API Cheat Sheet

*   `.HasOne().WithMany()` -> **One-to-Many** (One entity has a single reference; the other has a collection).
*   `.HasOne().WithOne()` -> **One-to-One** (Both sides point to a single reference).
*   `.HasMany().WithMany()` -> **Many-to-Many** (Both sides have collections).
