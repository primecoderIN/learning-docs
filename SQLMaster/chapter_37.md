# Chapter 37: Geospatial Data

## Learning Objectives
By the end of this chapter, you will be able to:
*   Identify the performance and mathematical flaws of storing coordinates as `FLOAT` columns.
*   Implement the SQL Server `GEOGRAPHY` data type to accurately model data on the curved surface of the Earth.
*   Execute Spatial Queries (like `STDistance` and `STIntersects`) directly within the database engine.
*   Optimize geospatial searches using **Spatial Indexes** and Tessellation grids.
*   Configure Entity Framework Core to utilize `NetTopologySuite` for seamless C# geospatial mapping.

---

## 37.1 The "Find Nearest" Problem

In our EV SaaS platform, a mobile app user opens the map and clicks *"Find Charging Stations within 10 miles."*

**The Anti-Pattern:** 
A junior developer creates two columns: `Latitude FLOAT` and `Longitude FLOAT`. 
To find the distance between the user and the stations, they write a C# loop that downloads all stations and calculates the **Haversine formula** (the math to calculate distance on a sphere) in memory. 
If the application scales to 500,000 stations, this requires downloading half a million rows to the web server just to display 3 nearby stations. This will crash the API instantly.

Alternatively, they try to put the Haversine math directly into the SQL `WHERE` clause. This forces SQL Server to perform massive trigonometric calculations on every single row in the table (a Full Table Scan), obliterating the CPU.

---

## 37.2 The `GEOGRAPHY` Data Type

To solve this, SQL Server provides the **`GEOGRAPHY`** data type (and its flat-earth counterpart, `GEOMETRY`).

Instead of two floats, we store the location as a single object that natively understands the curvature of the Earth.

```sql
-- Altering the table to support true spatial data
ALTER TABLE core.Stations ADD Coordinates GEOGRAPHY;
```

To insert data, we provide a Well-Known Text (WKT) string representing a `POINT(Longitude, Latitude)`, and we specify the **SRID (Spatial Reference Identifier)**. 
**SRID 4326** is the GPS standard (WGS 84) used by Google Maps, Apple Maps, and almost all mobile devices.

```sql
-- Insert a station located in New York City
-- Note: SQL Server expects POINT(Longitude Latitude)
UPDATE core.Stations 
SET Coordinates = geography::STGeomFromText('POINT(-73.935242 40.730610)', 4326)
WHERE StationId = '...';
```

---

## 37.3 Querying Distances (`STDistance`)

Once the data is stored natively, we can use built-in spatial functions. The most common is `STDistance()`, which calculates the shortest distance between two points in *Meters* (because SRID 4326 uses meters as its default unit of measurement).

```sql
-- Find all stations within 10 miles (16,093.4 meters) of the User's GPS location
DECLARE @UserLocation GEOGRAPHY = geography::Point(40.730610, -73.935242, 4326);

SELECT TOP 10 
    Name, 
    Coordinates.STDistance(@UserLocation) AS DistanceInMeters
FROM core.Stations
WHERE Coordinates.STDistance(@UserLocation) <= 16093.4
ORDER BY Coordinates.STDistance(@UserLocation) ASC;
```
This is drastically more readable and accurate than writing custom trigonometry. However, without an index, this is still a Full Table Scan.

---

## 37.4 Spatial Indexes (Tessellation)

You cannot put a standard B-Tree index on a `GEOGRAPHY` column. How do you sort a 2D sphere?
SQL Server uses **Spatial Indexes** via a process called **Tessellation**.

Imagine throwing a giant grid net over the Earth. If a station is in New York, SQL Server figures out which grid square it belongs to. When you search for stations within 10 miles of New York, SQL Server only checks the specific grid squares near New York, completely ignoring the grids covering Europe and Asia.

```sql
CREATE SPATIAL INDEX SPATIAL_Stations_Coordinates 
   ON core.Stations(Coordinates)
   USING GEOGRAPHY_GRID
   WITH (
       GRIDS = (HIGH, HIGH, HIGH, HIGH), 
       CELLS_PER_OBJECT = 64
   );
```
With this index in place, the `STDistance` query executes in microseconds, safely isolating the search radius.

---

## 37.5 Architect Perspective: SQL Native vs Redis Geo

While SQL Server Spatial Indexes are incredibly powerful, they are heavily tied to disk I/O.
If you are building an application like Uber or Lyft, where 50,000 drivers are updating their GPS coordinates every 3 seconds, writing those coordinates to SQL Server will destroy the Transaction Log and cause massive Page Latch contention.

*   **Static Data (Stations, Buildings, City Boundaries):** Store these in SQL Server `GEOGRAPHY`. The data rarely moves, and you can join it directly against billing or tenant tables.
*   **Hyper-Active Data (Live Cars, Delivery Drivers):** Store these in **Redis (GeoHashes)**. Redis holds spatial data completely in RAM, allowing for millions of real-time coordinate updates and radius queries per second, completely offloading the SQL database.

---

## 37.6 The Code: NetTopologySuite in EF Core

Entity Framework Core does not support `GEOGRAPHY` out of the box. You must install a third-party, mathematically precise library called **NetTopologySuite (NTS)**.

1.  Install the NuGet package: `Microsoft.EntityFrameworkCore.SqlServer.NetTopologySuite`
2.  Enable it in your `DbContext` configuration:
    ```csharp
    optionsBuilder.UseSqlServer(connectionString, x => x.UseNetTopologySuite());
    ```
3.  Map the property in your Entity:
    ```csharp
    using NetTopologySuite.Geometries;

    public class Station
    {
        public Guid StationId { get; set; }
        public string Name { get; set; }
        
        // This maps perfectly to SQL Server's GEOGRAPHY type
        public Point Location { get; set; } 
    }
    ```
4.  Write LINQ queries using native NTS methods. EF Core will seamlessly translate them to `STDistance` in SQL!
    ```csharp
    var userLocation = new Point(-73.935242, 40.730610) { SRID = 4326 };

    var nearbyStations = await _context.Stations
        .Where(s => s.Location.Distance(userLocation) <= 16093.4) // Translates to STDistance
        .OrderBy(s => s.Location.Distance(userLocation))
        .Take(10)
        .ToListAsync();
    ```

---

## 37.7 Performance & Security Analysis

### Performance Analysis: SARGability
A common mistake that prevents the Query Optimizer from using the Spatial Index is wrapping the column in a function before the comparison.
*   **Bad (Scan):** `WHERE @UserLocation.STDistance(Coordinates) < 1000`
*   **Good (Seek):** `WHERE Coordinates.STDistance(@UserLocation) < 1000`
You must put the Table Column on the left side of the method call, and the User Parameter inside the method call, to ensure the query remains SARGable (Chapter 19).

### Security Implications
*   **Location Tracking & PII:** While the locations of public Charging Stations are public data, storing the live `GEOGRAPHY` coordinates of *Users* falls under strict privacy regulations (GDPR/CCPA). If you store user locations to power a "Find friends nearby" feature, that data must be highly secured, subject to automatic data-retention purging jobs, and should ideally be stored with an obfuscated precision (e.g., rounding the decimal places so the exact street address cannot be derived).

---

## 37.8 Common Mistakes & Production Pitfalls

1.  **Longitude vs Latitude Order:** GPS systems and Google Maps list coordinates as `(Latitude, Longitude)`. However, the spatial specification (WKT) demands standard X/Y Cartesian coordinates, which means `(Longitude, Latitude)`. If you accidentally write `POINT(40.730610 -73.935242)`, SQL Server will place your New York station in Antarctica.
2.  **Missing SRID:** If you create a `Point` in C# using NetTopologySuite but forget to explicitly set `{ SRID = 4326 }`, EF Core defaults to SRID 0 (an unknown Euclidean plane). When it sends the query to SQL Server, it will crash with a mismatch error, because you cannot compare SRID 0 to SRID 4326.

---

## 37.9 Production Checklist

*   [ ] Coordinate pairs (`Latitude`/`Longitude`) are migrated from `FLOAT` columns to the native `GEOGRAPHY` data type.
*   [ ] The GPS standard SRID 4326 is consistently applied to all database and application-layer geometries.
*   [ ] Spatial Indexes are deployed to optimize `STDistance` and bounding-box queries, preventing full table scans.
*   [ ] Entity Framework Core is extended with `UseNetTopologySuite()` to translate LINQ spatial queries securely without client-side memory evaluation.

---

## 37.10 Exercises

1.  **Code Correction:** A junior developer writes this query to find if a car is currently inside a geofenced Polygon: `SELECT * FROM Geofences WHERE PolygonArea.STIntersects(geography::Point(Lat, Long, 4326)) = 1`. What are the two syntax/logic errors in the `Point` constructor that will cause this query to fail or return wildly inaccurate results?
2.  **Architectural Choice:** The business wants to track the live location of 100,000 delivery drivers in real-time (updated every 5 seconds) to build a live dashboard. Would you store this data in a SQL Server `GEOGRAPHY` table with a Spatial Index? Explain your reasoning.

---

## 37.11 Interview Questions

**Q1: Why is it an anti-pattern to store Latitude and Longitude as separate `FLOAT` columns when building a "Search Nearby" feature?**
*Answer:* Storing them as separate floats prevents the database from natively calculating distance. The application is forced to either download the entire table into C# memory to calculate the Haversine formula, or write the complex trigonometric math into the SQL `WHERE` clause. Both approaches prevent the use of indexes, forcing a Full Table Scan on every search, which will destroy CPU performance as the table grows.

**Q2: How does a Spatial Index differ from a standard B-Tree index in SQL Server?**
*Answer:* A standard B-Tree index sorts scalar data (like strings or numbers) in a 1-dimensional hierarchy. It cannot sort 2-dimensional or 3-dimensional shapes. A Spatial Index uses a technique called Tessellation, which divides the geographic space into a grid hierarchy. When querying for points within a radius, the Spatial Index quickly eliminates entire massive grid sections (e.g., entire continents) that do not overlap the search area, vastly reducing the number of rows the engine has to evaluate mathematically.

---
**Next up in Chapter 38:** We will close out the technical chapters of this book by diving into the most complex operational challenge: Database Migrations in CI/CD. We will explore how to achieve Zero-Downtime deployments using the Expand-and-Contract pattern.
