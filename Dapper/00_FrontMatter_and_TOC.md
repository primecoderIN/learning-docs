# Chapter 0: Dapper Basics and Getting Started

## 1. Learning Objectives

By the end of this chapter, you will be able to:
*   Install Dapper and the required SQL Server database drivers via NuGet.
*   Understand how to configure connection strings and instantiate a database connection.
*   Execute basic CRUD (Create, Read, Update, Delete) operations using Dapper.
*   Use parameterized queries to prevent SQL Injection attacks.
*   Familiarize yourself with Dapper's core utility methods (`Query` and `Execute`).

## 2. Introduction to Dapper

Before diving into the advanced architectural patterns and performance optimizations discussed in later chapters, we must cover the absolute basics. Dapper is a **Micro-ORM** (Object-Relational Mapper). Unlike Entity Framework, which tries to hide the database behind a layer of C# objects and LINQ translations, Dapper expects you to write raw SQL (Structured Query Language).

Dapper's only job is to take the result of your raw SQL query and map it (copy the data) into your C# objects as fast as possible. It achieves this by extending the standard `IDbConnection` interface provided by ADO.NET.

## 3. Installation and Setup

To use Dapper in a modern .NET application, you need two things:
1. The Dapper library itself.
2. A database driver (usually for SQL Server).

Open your terminal or Package Manager Console and run the following commands in your project directory:

```bash
# Install Dapper
dotnet add package Dapper

# Install the modern SQL Server driver
dotnet add package Microsoft.Data.SqlClient
```

*Note: Always use `Microsoft.Data.SqlClient` instead of the legacy `System.Data.SqlClient` for SQL Server.*

## 4. Connecting to the Database

Dapper does not manage your database connections for you. You must create and open the connection yourself using ADO.NET.

### The Connection String
Your application needs to know where the database is located. This is defined in a Connection String, typically stored in `appsettings.json`:

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=EVPlatform;User Id=sa;Password=StrongPassword123;TrustServerCertificate=True;"
  }
}
```

### Instantiating the Connection
To talk to the database, you create a `SqlConnection` object. Because connections hold onto unmanaged network resources, you **must** wrap them in a `using` block so they are properly closed and returned to the connection pool when you are done.

```csharp
using System.Data.SqlClient;
// OR in modern .NET: using Microsoft.Data.SqlClient;
using Dapper;

string connectionString = "Server=localhost;Database=EVPlatform;User Id=sa;Password=StrongPassword123;TrustServerCertificate=True;";

// The 'using' keyword ensures the connection is closed automatically
using (var connection = new SqlConnection(connectionString))
{
    // You must open the connection before Dapper can use it
    connection.Open();
    
    // Dapper code goes here...
}
```

## 5. Basic CRUD Operations

Dapper provides extension methods on the `connection` object. The two most fundamental methods are `Query` (for reading data) and `Execute` (for modifying data).

### Parameterization (Security First!)
Before showing the methods, you must learn the golden rule of Dapper: **Never concatenate strings to build SQL.** Always use parameters (`@VariableName`) and pass an anonymous C# object to Dapper. Dapper will safely bind these variables, preventing SQL Injection attacks.

```csharp
// DANGEROUS - SQL INJECTION RISK:
string badSql = $"SELECT * FROM Users WHERE Email = '{userInput}'";

// SAFE - THE DAPPER WAY:
string goodSql = "SELECT * FROM Users WHERE Email = @Email";
var parameters = new { Email = userInput };
```

### 1. READ (Selecting Data)
To read data, use the `Query<T>` method, where `T` is the C# class you want to map the data into.

```csharp
public class User
{
    public int Id { get; set; }
    public string Name { get; set; }
    public string Email { get; set; }
}

// Inside your using block:
string sql = "SELECT Id, Name, Email FROM Users WHERE Id = @UserId";

// QueryFirstOrDefault returns one object, or null if not found
User user = connection.QueryFirstOrDefault<User>(sql, new { UserId = 1 });

if (user != null)
{
    Console.WriteLine($"Found user: {user.Name}");
}

// To get multiple rows, use Query<T> which returns an IEnumerable<T>
string listSql = "SELECT * FROM Users WHERE IsActive = @IsActive";
IEnumerable<User> activeUsers = connection.Query<User>(listSql, new { IsActive = true });
```

### 2. CREATE (Inserting Data)
To insert data, use the `Execute` method. It returns an `int` representing the number of rows affected.

```csharp
string sql = "INSERT INTO Users (Name, Email, IsActive) VALUES (@Name, @Email, @IsActive)";

var newUser = new 
{ 
    Name = "Sanjeev", 
    Email = "sanjeev@example.com", 
    IsActive = true 
};

int rowsAffected = connection.Execute(sql, newUser);
Console.WriteLine($"Inserted {rowsAffected} row(s).");
```

### 3. UPDATE (Modifying Data)
Updating uses the exact same `Execute` method.

```csharp
string sql = "UPDATE Users SET Name = @NewName WHERE Id = @Id";

int rowsAffected = connection.Execute(sql, new { NewName = "Sanjeev Kumar", Id = 1 });
```

### 4. DELETE (Removing Data)
Deleting also uses the `Execute` method.

```csharp
string sql = "DELETE FROM Users WHERE Id = @Id";

int rowsAffected = connection.Execute(sql, new { Id = 1 });
```

## 6. Asynchronous Methods (Async/Await)

In modern web applications, you should always use the asynchronous versions of these methods (`QueryAsync`, `ExecuteAsync`) to ensure your application remains responsive under heavy load.

```csharp
public async Task<User> GetUserAsync(int userId)
{
    using var connection = new SqlConnection(_connectionString);
    await connection.OpenAsync();
    
    return await connection.QueryFirstOrDefaultAsync<User>(
        "SELECT * FROM Users WHERE Id = @Id", 
        new { Id = userId }
    );
}
```

## 7. Summary

Dapper is incredibly easy to get started with. The workflow is always the same:
1. Create a C# class to represent your data.
2. Open a `SqlConnection` inside a `using` block.
3. Write a raw SQL string using `@` parameters.
4. Call `connection.Query<T>()` to read data, or `connection.Execute()` to modify data.
5. Pass an anonymous object containing your parameter values.

Now that you understand the absolute basics of connecting and querying, the rest of this book will explore how Dapper works internally, how to map complex relationships, and how to architect these simple methods into enterprise-grade, high-performance applications.
