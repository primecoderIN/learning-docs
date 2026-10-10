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

*   **How to configure (Fluent API):** You explicitly define the relationship but leave the `WithMany()` method empty, exactly as seen in Normora's `TenantsDbContext`:
    
    ```csharp
    modelBuilder.Entity<TenantInvitation>()
        .HasOne(i => i.Tenant)
        .WithMany() // <-- Left empty!
        .HasForeignKey(i => i.TenantId)
        .OnDelete(DeleteBehavior.Cascade);
    ```

*   **What exactly happens when `.WithMany()` is left empty?**
    1. **"The Parent has Many Children, but I don't want to track them."** You are telling EF Core: "A Tenant can have many Invitations, but my C# `Tenant` class does NOT have an `ICollection<TenantInvitation>` property."
    2. **Database Level:** **Nothing changes.** EF Core still correctly creates a One-to-Many relationship in the SQL database, including the `TenantId` Foreign Key column on the `TenantInvitations` table.
    3. **C# Code Level:** Because the `Tenant` class has no collection property, you **cannot navigate** from the Parent to the Child in memory (e.g., `tenant.Invitations` doesn't exist). You can only navigate from Child to Parent (`invitation.Tenant`).

*   **Why is this a Best Practice?** Use this to prevent massive, accidental data loading ("Object Graph Pollution"). If a `Tenant` has 100,000 invitations over its lifetime, and you provided a navigation property, a developer might accidentally loop through `tenant.Invitations`, instantly loading all 100,000 rows into RAM and crashing the API.
*   **How to query instead:** Always use Unidirectional relationships when the "Many" side contains a massive amount of data (e.g., Audit Logs, Invitations). Query those entities directly using the `DbContext` with pagination:
    ```csharp
    // Developers MUST do this instead:
    var invites = await dbContext.TenantInvitations
        .Where(i => i.TenantId == tenantId)
        .Take(50) // Safe pagination!
        .ToListAsync();
    ```

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

### Which entity holds the Foreign Key in One-to-One?
The Foreign Key always goes on the **Dependent** entity (the one that cannot exist without the other). `TenantBranding` cannot exist without a `Tenant`, so `TenantBranding.TenantId` is the FK. You must explicitly specify which entity holds the FK via `HasForeignKey<TDependentEntity>()` — EF Core cannot guess this for One-to-One relationships.

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

### Three Ways to Fix Circular References

| Approach | How | Best for |
|---|---|---|
| `[JsonIgnore]` on nav property | Attribute on one side of the relationship | Simple apps; quick fix |
| DTOs + `.Select()` projection | Never return raw EF entities from API | Enterprise apps; recommended |
| Global `IgnoreCycles` setting | Configure JSON serializer in `Program.cs` | Quick global fix; less control |

```csharp
// Global fix (Program.cs)
builder.Services.AddControllers().AddJsonOptions(options =>
{
    options.JsonSerializerOptions.ReferenceHandler = ReferenceHandler.IgnoreCycles;
});
```

---

## 7. Master List: All Navigation Property Combinations

Based on the concepts in this module, relationships are built by combining a **Relationship Type** (One-to-Many, One-to-One, Many-to-Many) with a **Navigation Direction** (Bidirectional or Unidirectional).

Here is a comprehensive list of all possible combinations, how they look in C#, how to configure them in the Fluent API, and when you should use them (with examples from Normora).

### 1. One-to-Many Combinations

This is the most common relationship type. One parent has multiple children. The Foreign Key is always on the child table.

#### A. Bidirectional (The Standard)
Both sides know about each other. You can navigate `Parent.Children` and `Child.Parent`.
*   **C# Code:** Parent has `ICollection<Child>`. Child has `Parent`.
*   **Fluent API:** `.HasOne(c => c.Parent).WithMany(p => p.Children)`
*   **When to use:** This is the default. Use it when your business logic requires you to frequently query the relationship from both angles.
*   **Normora Example:** `Tenant` and `User`. You frequently need to know what users belong to a tenant (`tenant.Users`), and what tenant a user belongs to (`user.Tenant`).

#### B. Unidirectional: Child -> Parent (Protects Memory)
The child knows about the parent, but the parent does *not* have a list of children.
*   **C# Code:** Parent has NO collection. Child has `Parent`.
*   **Fluent API:** `.HasOne(c => c.Parent).WithMany()` *(Notice `WithMany` is empty)*
*   **When to use:** Use this when a parent has a massive amount of children. It prevents developers from accidentally loading millions of records into RAM by traversing the object graph.
*   **Normora Example:** `Tenant` and `TenantInvitation`. You query invitations directly (`dbContext.TenantInvitations.Where(x => x.TenantId == id)`), rather than calling `tenant.Invitations`.

#### C. Unidirectional: Parent -> Child (Strict Encapsulation)
The parent has a list of children, but the child does *not* have a navigation property back to the parent (it just has the Foreign Key `ParentId`).
*   **C# Code:** Parent has `ICollection<Child>`. Child has `Guid ParentId`, but NO `Parent` object.
*   **Fluent API:** `.HasMany(p => p.Children).WithOne()` *(Notice `WithOne` is empty)*
*   **When to use:** Use this in Domain-Driven Design (DDD) when the child is a "sub-entity" that cannot exist or be reasoned about outside the context of the parent.

### 2. One-to-One Combinations

One entity is the Principal (Parent), and the other is the Dependent (Child). The Foreign Key must live on the Dependent.

#### D. Bidirectional (The Standard)
Both sides point to each other.
*   **C# Code:** `EntityA` has `EntityB`. `EntityB` has `EntityA`. 
*   **Fluent API:** `.HasOne(a => a.B).WithOne(b => b.A)`
*   **When to use:** When both entities are equally important and frequently need to reference each other.
*   **Normora Example:** `Tenant` and `TenantBranding`. If you have the branding, you can easily find the tenant it belongs to, and vice-versa.

#### E. Unidirectional: Principal -> Dependent
The main entity knows about the dependent details, but the details don't link back.
*   **C# Code:** `User` (Parent) has `UserProfile` (Child). `UserProfile` has NO `User` property (only `UserId`).
*   **Fluent API:** `.HasOne(u => u.Profile).WithOne()`
*   **When to use:** When the dependent entity is purely supplemental data. If you load a `UserProfile`, you probably don't need to traverse back up to the `User` because you already know who the user is.

### 3. Many-to-Many Combinations

These require a Join Table in the database. Modern EF Core can handle this invisibly, or you can manage it explicitly.

#### F. Bidirectional (Implicit Join Table)
Both sides have collections, and EF Core handles the join table behind the scenes.
*   **C# Code:** `Student` has `ICollection<Course>`. `Course` has `ICollection<Student>`.
*   **Fluent API:** `.HasMany(s => s.Courses).WithMany(c => c.Students)`
*   **When to use:** When the relationship is simple and you don't need to track *when* the record was created or store extra data on the join itself.

#### G. Unidirectional (Implicit Join Table)
One side has a collection, the other side doesn't care.
*   **C# Code:** `Post` has `ICollection<Tag>`. `Tag` has NO collection of Posts.
*   **Fluent API:** `.HasMany(p => p.Tags).WithMany()`
*   **When to use:** When you only traverse in one direction.

#### H. Explicit Join Entity (The "Two One-to-Manys" Method)
You manually create the Join Class in C# because you need to store extra data on the relationship itself.
*   **C# Code:** You create a third class: `JoinEntity`. Entity A has `ICollection<JoinEntity>`, Entity B has `ICollection<JoinEntity>`, JoinEntity points to both A and B.
*   **Fluent API:** You don't use `.WithMany()`. Instead, you configure two One-to-Many relationships.
*   **When to use:** Whenever the relationship itself has properties (dates, roles, statuses, etc.). 
*   **Normora Example:** `TenantMembership` and `Department`. Since users can join departments with specific roles or start dates, Normora explicitly creates `MembershipDepartment` to store that extra join data.

---

## 8. Demystifying Delete Behaviors (`OnDelete`)

When configuring relationships via the Fluent API, deciding what happens to "child" records when the "parent" is deleted is critical for data integrity. 

> **Crucial Rule:** The `OnDelete()` behavior always dictates what happens to the **Child (Dependent) table** (the one holding the Foreign Key) when a record in the **Parent (Principal) table** is deleted. The trigger is the parent's deletion; the victim is the child table.

EF Core provides several `DeleteBehavior` options.

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

### Delete Behavior Quick Reference

| Behavior | Child records deleted? | Parent can be deleted? | FK set to NULL? |
|---|---|---|---|
| `Cascade` | ✅ Yes (automatically) | ✅ Yes | No |
| `Restrict` | No | ❌ No (throws exception) | No |
| `SetNull` | No | ✅ Yes | ✅ Yes |
| `NoAction` | No action taken by EF Core | Database decides | No |

---

## 9. Conventions vs. Fluent API (Do I have to use both?)

A very common question is: *"If I configure navigation properties in my C# Models, am I strictly required to also configure them in the Fluent API?"*

The answer is **No**. 

EF Core has a powerful feature called **Conventions** (often called "EF Core Magic"). If you name your properties following standard conventions, EF Core can automatically figure out the relationship just by looking at your C# Models (Layer 1).

If you write this in your C# Models:

```csharp
public class Tenant 
{ 
    public ICollection<User> Users { get; set; } 
}

public class User 
{ 
    public Guid TenantId { get; set; } // EF Core sees this matches the 'Tenant' navigation property!
    public Tenant Tenant { get; set; } 
}
```

If you do absolutely nothing in the Fluent API, EF Core will **automatically** figure out:
1. This is a One-to-Many relationship.
2. `TenantId` is the foreign key.
3. Because `TenantId` is a non-nullable `Guid`, it will automatically apply `DeleteBehavior.Cascade`.

### So why do enterprise apps (like Normora) use the Fluent API?

While conventions are great for simple apps, enterprise applications explicitly use the Fluent API for a few critical reasons:

1. **Controlling Delete Behaviors:** EF Core's convention defaults to `Cascade` for required relationships. But as we saw with `Department`, you often want `DeleteBehavior.Restrict` to prevent catastrophic data loss. You *must* use the Fluent API to change this.
2. **Unconventional Names:** If your foreign key doesn't perfectly match the class name (e.g., you have a `CreatedByUserId` foreign key pointing to the `User` table), EF Core's magic breaks. The Fluent API tells it exactly how they map.
3. **Composite Keys:** EF Core conventions cannot automatically guess composite primary keys (like in `MembershipDepartment`). You must use `.HasKey(md => new { ... })`.
4. **Explicitness (Best Practice):** Relying on "magic" guessing can lead to unexpected database migrations if a junior developer accidentally renames a property. Explicitly writing `.HasOne().WithMany()` guarantees the database schema matches exactly what the architect intended.

---

## 10. Change Tracking and Object Graphs

One of the most powerful features of EF Core is how the **Change Tracker** watches navigation properties to automatically manage relationships in the database. EF Core tracks entire "Object Graphs", not just single rows.

### 1. Automatically Inserting Children (Graph Insertion)
If you fetch a tracked parent entity, you can simply add a new object to its collection navigation property. You **do not** need to explicitly add the child to the `DbContext` or manually set its Foreign Key.

```csharp
// 1. Fetch a tracked tenant
var tenant = await dbContext.Tenants.Include(t => t.Users).FirstAsync(t => t.Id == 1);

// 2. Add a brand new user to the navigation property
tenant.Users.Add(new User { Name = "Sanjeev" }); 

// 3. Save!
await dbContext.SaveChangesAsync();
```
**What happens:** EF Core scans the `tenant.Users` collection, sees the new `User`, automatically sets its `TenantId` Foreign Key to `1`, and executes an `INSERT` statement.

### 2. Changing Relationships (Re-parenting)
You can update foreign keys in the database simply by assigning a different tracked object to a reference navigation property.

```csharp
var user = await dbContext.Users.FirstAsync(u => u.Id == 5);
var newDepartment = await dbContext.Departments.FirstAsync(d => d.Name == "IT");

// Re-assign the navigation property
user.Department = newDepartment;

await dbContext.SaveChangesAsync();
```
**What happens:** The Change Tracker notices `user.Department` changed. It automatically updates the underlying `user.DepartmentId` column to match `newDepartment.Id` and executes an `UPDATE` statement.

### 3. "Fixing Up" the Graph (Navigation Fixup)
If you insert a new record, EF Core will automatically populate the navigation properties of other tracked objects to reflect reality, even before you query the database again.

```csharp
var tenant = await dbContext.Tenants.FirstAsync(t => t.Id == 1);

var newUser = new User { Name = "Alice", TenantId = 1 };
dbContext.Users.Add(newUser); 

// At this EXACT moment (even before SaveChanges), EF Core looks at the TenantId.
// It realizes Tenant 1 is already tracked in memory.
// It automatically adds 'newUser' into 'tenant.Users'!

Console.WriteLine(tenant.Users.Contains(newUser)); // This will print TRUE!
```
The Change Tracker ensures that your C# object graph perfectly matches the Foreign Keys you've assigned in memory.

---

## 11. Advanced Relationships: Alternate Keys (`HasPrincipalKey`)

By default, in relational databases and EF Core, a **Foreign Key always points to the Primary Key** of the parent table. 

If your `Genre` table has a primary key called `Id`, and your `Movie` table has a foreign key called `MainGenreId`, EF Core **automatically** knows they link together. Therefore, calling `.HasPrincipalKey(genre => genre.Id)` in the Fluent API is technically **redundant and unnecessary**. 

### When do you actually NEED `HasPrincipalKey`?
You use `HasPrincipalKey` when you want a Foreign Key to point to a unique column that is **NOT** the Primary Key. This is called an **Alternate Key**.

### A Real-World Use Case (Normora)
Every `Tenant` in Normora has a `Guid Id` as its Primary Key. However, for white-label routing, every `Tenant` also has a unique string called a `Slug` (e.g., `"acme-corp"`).

Imagine building a public API for Normora so third-party systems can push `SupportTickets`. Those third-party systems might not know the internal `Guid Id` of the tenant, but they *do* know the `Slug`.

```csharp
public class Tenant
{
    public Guid Id { get; set; } // Primary Key
    public string Slug { get; set; } // Alternate Unique Key ("acme-corp")
    
    public ICollection<SupportTicket> SupportTickets { get; set; }
}

public class SupportTicket
{
    public Guid Id { get; set; }
    public string Issue { get; set; }
    
    // The Foreign Key is the string Slug, NOT the Guid Id!
    public string TenantSlug { get; set; } 
    public Tenant Tenant { get; set; }
}
```

If we try to configure this without `HasPrincipalKey`, EF Core will crash because it will try to link the `string TenantSlug` to the `Guid Id` (the default Primary Key), and the data types don't match.

We fix this by explicitly pointing the relationship to the Alternate Key:

```csharp
builder.Entity<SupportTicket>()
    .HasOne(ticket => ticket.Tenant)
    .WithMany(tenant => tenant.SupportTickets)
    .HasForeignKey(ticket => ticket.TenantSlug) // 1. The FK is the string property
    .HasPrincipalKey(tenant => tenant.Slug);    // 2. We explicitly tell EF Core to point it to the Slug, NOT the Id!
```

**Rule of Thumb:**
*   If your Foreign Key points to the parent's **Primary Key** (like `Id`), you **do not** need `HasPrincipalKey`.
*   If your Foreign Key points to a unique **Alternate Key** (like an Email, a Slug, or a Social Security Number), you **must** use `HasPrincipalKey`.

---

## 12. Real-World Scenarios

### Scenario A: Designing the Normora Relationship Map
```text
Tenant (1) ──────────────── (Many) User
Tenant (1) ──────────────── (1) TenantBranding
Tenant (1) ──────────────── (Many) Department
User (1) ────────────────── (1) TenantMembership
TenantMembership (M) ────── (M) Department  [via MembershipDepartment join table]
Tenant (1) ──────────────── (Many) TenantInvitation  [unidirectional - no nav on Tenant side]
```

### Scenario B: Loading a User With All Related Data
```csharp
var user = await dbContext.Users
    .Include(u => u.Tenant)                        // Tenant details
        .ThenInclude(t => t.Branding)              // Tenant's branding
    .Include(u => u.TenantMembership)              // Membership record
        .ThenInclude(m => m.MembershipDepartments) // Department assignments
            .ThenInclude(md => md.Department)      // Department details
    .SingleOrDefaultAsync(u => u.Id == userId);
```
This generates a single SQL query with multiple JOINs — no N+1 problem.

### Scenario C: Transferring a User Between Departments
```csharp
// Normora: Move user from Engineering to Product
var membership = await dbContext.TenantMemberships
    .Include(m => m.MembershipDepartments)
    .SingleAsync(m => m.UserId == userId);

// Remove old department assignment
var oldAssignment = membership.MembershipDepartments
    .First(md => md.DepartmentId == engineeringDeptId);
dbContext.MembershipDepartments.Remove(oldAssignment);

// Add new department assignment
membership.MembershipDepartments.Add(new MembershipDepartment
{
    TenantMembershipId = membership.Id,
    DepartmentId = productDeptId
});

await dbContext.SaveChangesAsync();
```

---

## 13. Interview Questions

**Q1: What is the difference between a bidirectional and unidirectional navigation property? When would you use each?**
> Bidirectional means both entities have navigation properties pointing to each other (e.g., `Tenant.Users` and `User.Tenant`). Unidirectional means only one side has the navigation property. Use bidirectional when you genuinely need to traverse from both directions in code. Use unidirectional (omitting the collection on the parent) when the parent could have an unbounded number of children — to prevent developers from accidentally loading millions of records by accessing the collection.

**Q2: What happens at the database level when you use `.WithMany()` with an empty argument?**
> Nothing changes at the database level. EF Core still creates the Foreign Key column on the child table and establishes the One-to-Many relationship in the SQL schema. The only effect is at the C# code level — the parent entity simply doesn't have a collection navigation property, so you can't traverse from parent to children using the object graph. You must query children directly via `DbContext`.

**Q3: Explain the difference between `DeleteBehavior.Cascade`, `Restrict`, and `SetNull`.**
> - `Cascade`: When the parent is deleted, all child records are automatically deleted too. Use for dependent entities that can't exist without the parent.
> - `Restrict`: Prevents deleting the parent if any children reference it. An exception is thrown. Use to protect against accidental data loss.
> - `SetNull`: When the parent is deleted, the child's FK is set to `NULL`. The child record is kept. Use for optional relationships where children can exist independently. Requires the FK to be nullable.

**Q4: What causes a circular reference exception when returning EF Core entities from an API? How do you fix it?**
> When two entities have bidirectional navigation properties (e.g., `Tenant.Users` and `User.Tenant`), the JSON serializer enters an infinite loop — it serializes `Tenant`, then its `Users`, then each user's `Tenant`, which again has `Users`, and so on. Fixes: (1) Use DTOs/projections to never return raw entities (best practice), (2) Apply `[JsonIgnore]` to one side of the navigation property, or (3) Configure `ReferenceHandler.IgnoreCycles` globally in the JSON serializer options.

**Q5: What is an Explicit Join Entity in Many-to-Many relationships? When should you use it instead of EF Core's implicit join table?**
> An Explicit Join Entity is a manually-defined C# class that represents the join table. Use it when the relationship itself needs to store extra data — for example, when a user joins a department with a specific `RoleId`, `JoinedAt` date, or other metadata. EF Core's implicit join table (via `.HasMany().WithMany()`) is suitable only when the relationship has no extra properties.

**Q6: What is Navigation Fixup in EF Core?**
> Navigation Fixup is a Change Tracker behavior where EF Core automatically updates in-memory navigation properties to reflect FK assignments — even before `SaveChanges` is called. For example, if you set `newUser.TenantId = 1` and `Tenant 1` is already tracked in memory, EF Core automatically adds `newUser` to `tenant.Users`. This ensures the C# object graph is always consistent with the foreign keys in memory.

**Q7: Why should you always explicitly configure relationships in the Fluent API rather than relying on EF Core conventions in enterprise apps?**
> Conventions work well for simple apps but break down in enterprise scenarios: (1) EF Core's default `DeleteBehavior.Cascade` may not be appropriate for all relationships — you often need `Restrict` to prevent data loss. (2) Non-standard FK names (like `CreatedByUserId` pointing to `Users`) break convention-based detection. (3) Composite primary keys must always be explicitly configured. (4) Explicit configuration documents intent — a junior developer renaming a navigation property won't silently change the database schema.

---

## Summary
In this module, we mastered all three relationship types (One-to-Many, One-to-One, Many-to-Many), understood the difference between bidirectional and unidirectional navigation, learned how delete behaviors protect data integrity, explored how the Change Tracker automatically manages object graphs, and discovered alternate keys for advanced scenarios. In the next module, we will deep-dive into querying, projections, eager loading, and global query filters.
