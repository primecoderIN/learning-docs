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

When defining a relationship, you have to decide if you want to navigate it from **both sides** or just **one side**. Navigation properties are the primary way you traverse the object graph in C# to read or manipulate related data.

### 1. Navigating from Both Sides (Bidirectional)
This is the most common approach. Both entities have a navigation property pointing to each other, allowing you to traverse the relationship from either direction.

```csharp
public class Tenant 
{ 
    public ICollection<User> Users { get; set; } = new List<User>(); 
}

public class User 
{ 
    public Tenant Tenant { get; set; } 
}
```

*   **How to configure (Fluent API):** Both properties are explicitly mentioned in the configuration: 
    `.HasOne(u => u.Tenant).WithMany(t => t.Users)`
*   **When to use:** Use this when your application's business logic frequently requires querying from both angles. 
*   **Examples in C#:** 
    *   *Angle 1:* `var tenantName = user.Tenant.Name;` (Finding what tenant a user belongs to)
    *   *Angle 2:* `var userCount = tenant.Users.Count;` (Finding how many users a tenant has)
*   **Best Practice:** Initialize collection navigation properties (like `new List<User>()`) in the class constructor or at the property level to avoid `NullReferenceException`s when adding items to a newly instantiated object.

### 2. Navigating from One Side Only (Unidirectional)
You are **not required** to define navigation properties on both sides. If you never need to traverse from a parent down to its children (or vice-versa) in your C# code, you can leave one side out.

**Example from Normora:** A `TenantInvitation` belongs to a `Tenant`. You will frequently query `invitation.Tenant` to find out what tenant the user is being invited to. However, you will almost *never* want to load `tenant.Invitations` directly via the object graph because a tenant could have thousands of pending, accepted, or revoked invitations over time!

```csharp
public class Tenant 
{ 
    // Best Practice: Omit the collection if it clutters the domain or could be dangerously large!
    // Notice there is NO 'public ICollection<TenantInvitation> Invitations' here.
}

public class TenantInvitation 
{ 
    public Guid TenantId { get; set; }
    public Tenant Tenant { get; set; } 
    
    public string Email { get; set; }
    public string Status { get; set; } // Pending, Accepted, Revoked
}
```

*   **What happens in EF Core?** EF Core is perfectly fine with this! The foreign key (`TenantId` on the `TenantInvitations` table) is created exactly the same way. 
*   **How to configure (Fluent API):** You simply leave the second part of the mapping empty, exactly as seen in Normora's `TenantsDbContext`:
    
    ```csharp
    modelBuilder.Entity<TenantInvitation>()
        .HasOne(i => i.Tenant)
        .WithMany() // <-- Left empty!
        .HasForeignKey(i => i.TenantId)
        .OnDelete(DeleteBehavior.Cascade);
    ```

*   **When to use:** Use this to prevent massive, accidental data loading ("Object Graph Pollution") and to keep your domain models focused strictly on the traversal paths you actually need.
*   **Best Practice:** Always use Unidirectional relationships when the "Many" side contains a massive amount of data (e.g., Audit Logs, Invitations, Transactions). Query those entities directly using `DbContext.TenantInvitations.Where(i => i.TenantId == id)` with pagination, rather than relying on a `.Invitations` navigation property.

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

### Many-to-Many with an Explicit Join Entity (Normora Example)
In Normora, a User doesn't just join a Department. A user has a `TenantMembership`, and that membership is assigned to a `Department`. We need to track exactly when they joined the department, or perhaps their specific role in it. We use an explicit join entity called `MembershipDepartment`.

```csharp
public class MembershipDepartment
{
    // Composite Primary Key (TenantMembershipId + DepartmentId)
    public Guid TenantMembershipId { get; set; }
    public TenantMembership TenantMembership { get; set; }

    public Guid DepartmentId { get; set; }
    public Department Department { get; set; }
}
```

When you do this, you are actually creating **two One-to-Many relationships** that form a Many-to-Many. Here is exactly how Normora configures this in `TenantsDbContext`:

```csharp
modelBuilder.Entity<MembershipDepartment>(entity =>
{
    // 1. Define the composite primary key
    entity.HasKey(md => new { md.TenantMembershipId, md.DepartmentId });

    // 2. Map TenantMembership -> MembershipDepartment
    entity.HasOne(md => md.TenantMembership)
          .WithMany(m => m.MembershipDepartments)
          .HasForeignKey(md => md.TenantMembershipId)
          .OnDelete(DeleteBehavior.Cascade); // If membership is removed, remove from department

    // 3. Map Department -> MembershipDepartment
    entity.HasOne(md => md.Department)
          .WithMany(d => d.MembershipDepartments)
          .HasForeignKey(md => md.DepartmentId)
          .OnDelete(DeleteBehavior.Restrict); // PREVENT deleting a department if members still exist in it!
});
```

**Best Practice:** Notice the `.OnDelete(DeleteBehavior.Restrict)` above. Explicit join entities allow you to control delete cascading precisely. In this case, Normora ensures you cannot accidentally delete a `Department` if people are still actively assigned to it!

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

---

## 8. Demystifying Delete Behaviors (`OnDelete`)

When configuring relationships via the Fluent API, deciding what happens to "child" records when the "parent" is deleted is critical for data integrity. EF Core provides several `DeleteBehavior` options.

### 1. `DeleteBehavior.Cascade` (The Default for Required Relationships)
If the parent is deleted, all related child records are automatically deleted as well. 
*   **How it works:** `DELETE FROM Tenants WHERE Id = 1` will also secretly issue a `DELETE FROM Users WHERE TenantId = 1`.
*   **When to use:** Use this when the child record absolutely cannot exist without the parent. 
*   **Normora Example:** Deleting a `Tenant` should permanently delete its `TenantBranding` and its `Users`.

### 2. `DeleteBehavior.Restrict`
Prevents the parent from being deleted if any child records still reference it. An exception will be thrown.
*   **How it works:** The database enforces a strict foreign key constraint. If you try to delete the parent, SQL Server will block it.
*   **When to use:** Use this to prevent accidental, catastrophic data loss of critical sub-entities, or for Many-to-Many join tables.
*   **Normora Example:** You cannot delete a `Department` if there are still `MembershipDepartment` records pointing to it. You must unassign the users first!

### 3. `DeleteBehavior.SetNull`
Instead of deleting the child records, the Foreign Key on the child records is set to `NULL`. 
*   **How it works:** If you delete a `Department`, all users in that department are kept in the database, but their `DepartmentId` is set to `NULL`.
*   **When to use:** Use this for optional relationships where the child can continue to exist independently of the parent. 
*   **Requirement:** The foreign key property (e.g., `Guid? DepartmentId`) **must be nullable** in C#. 

### 4. `DeleteBehavior.NoAction`
EF Core does not perform any action regarding the dependent entities. It assumes the database is handling it (via a SQL trigger or a cascade defined manually in SQL), or it just lets the database throw a standard foreign key constraint error.
*   **How it works:** EF Core simply issues the `DELETE` for the parent and ignores the children.
*   **When to use:** Rarely used in code-first applications. Use it only if you have custom database triggers managing your cascading logic.

### 5. `DeleteBehavior.ClientCascade` / `ClientSetNull`
These behave exactly like `Cascade` and `SetNull`, but the cascading action happens **in memory** for entities currently being tracked by the `DbContext`, rather than relying on the database's foreign key constraint to cascade it at the database level.
*   **When to use:** Generally avoided unless you are using an obscure database provider that doesn't support database-level cascading.
