# Volume 3, Part 5: Logistics & Supply Chain Scenarios (Questions 401-500)

**Domain Context:** You are the Senior Data Architect for a global logistics and freight company. 
**Core Schema:**
*   `Warehouses` (WarehouseID, City, Capacity, CurrentStock)
*   `Shipments` (ShipmentID, Origin_WarehouseID, Dest_WarehouseID, Status, Weight, DispatchDate, DeliveryDate)
*   `Vehicles` (VehicleID, VehicleType, MaxWeight, LastMaintenanceDate)
*   `Drivers` (DriverID, FirstName, LastName, LicenseClass, IsActive)
*   `Routes` (RouteID, ShipmentID, VehicleID, DriverID, MilesDriven)

---

## Section 1: Inventory Distribution (Questions 401-420)

**Q401: Find all Warehouses that are currently operating at 95% capacity or higher.**
*   **Solution:** `SELECT WarehouseID, City FROM Warehouses WHERE (CAST(CurrentStock AS FLOAT) / Capacity) >= 0.95;`

**Q402: Calculate the total weight of all shipments currently marked as 'In Transit'.**
*   **Solution:** `SELECT SUM(Weight) FROM Shipments WHERE Status = 'In Transit';`

**Q403: Which City shipped out the most freight (by weight) in 2025?**
*   **Solution:**
    ```sql
    SELECT TOP 1 w.City, SUM(s.Weight) AS TotalWeight
    FROM Shipments s JOIN Warehouses w ON s.Origin_WarehouseID = w.WarehouseID
    WHERE YEAR(s.DispatchDate) = 2025
    GROUP BY w.City ORDER BY TotalWeight DESC;
    ```

**Q404: Identify shipments that were dispatched from, and delivered to, the EXACT same warehouse.**
*   **Solution:** `SELECT ShipmentID FROM Shipments WHERE Origin_WarehouseID = Dest_WarehouseID;`

**Q405: Find Warehouses that received exactly 0 shipments in the last 30 days.**
*   **Solution (Using NOT EXISTS):**
    ```sql
    SELECT WarehouseID FROM Warehouses w
    WHERE NOT EXISTS (
        SELECT 1 FROM Shipments s 
        WHERE s.Dest_WarehouseID = w.WarehouseID 
          AND s.DeliveryDate > DATEADD(DAY, -30, GETDATE())
    );
    ```

**Q406: Calculate the average weight of a shipment for each specific `VehicleType`.**
*   **Solution:** `SELECT v.VehicleType, AVG(s.Weight) FROM Vehicles v JOIN Routes r ON v.VehicleID = r.VehicleID JOIN Shipments s ON r.ShipmentID = s.ShipmentID GROUP BY v.VehicleType;`

*(Questions 407-420 focus on resolving unbalanced stock (Origins sending 5x more than they receive) using complex GROUP BY and HAVING clauses).*

---

## Section 2: Delivery Times and SLA Tracking (Questions 421-440)

**Q421: Calculate the exact transit time (in days) for every completed shipment.**
*   **Solution:** `SELECT ShipmentID, DATEDIFF(DAY, DispatchDate, DeliveryDate) AS TransitDays FROM Shipments WHERE Status = 'Delivered';`

**Q422: The SLA (Service Level Agreement) states all shipments must be delivered within 5 days. Find the SLA Breach Rate (Percentage of late deliveries).**
*   **Solution:**
    ```sql
    SELECT 
        (CAST(SUM(CASE WHEN DATEDIFF(DAY, DispatchDate, DeliveryDate) > 5 THEN 1 ELSE 0 END) AS FLOAT) / COUNT(*)) * 100 AS SLABreachPct
    FROM Shipments WHERE Status = 'Delivered';
    ```

**Q423: Find the specific `DriverID` who has the highest number of SLA breaches.**
*   **Solution:**
    ```sql
    SELECT TOP 1 r.DriverID, COUNT(*) AS Breaches
    FROM Routes r JOIN Shipments s ON r.ShipmentID = s.ShipmentID
    WHERE DATEDIFF(DAY, s.DispatchDate, s.DeliveryDate) > 5
    GROUP BY r.DriverID ORDER BY Breaches DESC;
    ```

**Q424: Identify "Lost" shipments (Dispatched more than 14 days ago, but Status is still 'In Transit').**
*   **Solution:** `SELECT ShipmentID FROM Shipments WHERE Status = 'In Transit' AND DispatchDate < DATEADD(DAY, -14, GETDATE());`

**Q425: Calculate the average transit time per Origin-Destination pair.**
*   **Solution:** `SELECT Origin_WarehouseID, Dest_WarehouseID, AVG(DATEDIFF(DAY, DispatchDate, DeliveryDate)) FROM Shipments WHERE Status = 'Delivered' GROUP BY Origin_WarehouseID, Dest_WarehouseID;`

*(Questions 426-440 explore calculating median delivery times, filtering out weekends and holidays from transit calculations using custom UDFs).*

---

## Section 3: Fleet Management and Maintenance (Questions 441-460)

**Q441: Find all Vehicles that have not had maintenance in over 6 months.**
*   **Solution:** `SELECT VehicleID, VehicleType FROM Vehicles WHERE LastMaintenanceDate < DATEADD(MONTH, -6, GETDATE());`

**Q442: Prevent Overloading: Find any Route where the Shipment `Weight` exceeds the Vehicle's `MaxWeight`.**
*   **Solution:**
    ```sql
    SELECT r.RouteID, s.Weight, v.MaxWeight
    FROM Routes r 
    JOIN Shipments s ON r.ShipmentID = s.ShipmentID 
    JOIN Vehicles v ON r.VehicleID = v.VehicleID
    WHERE s.Weight > v.MaxWeight;
    ```

**Q443: Calculate the total miles driven by each Driver in the year 2026.**
*   **Solution:** `SELECT r.DriverID, SUM(r.MilesDriven) FROM Routes r JOIN Shipments s ON r.ShipmentID = s.ShipmentID WHERE YEAR(s.DispatchDate) = 2026 GROUP BY r.DriverID;`

**Q444: Identify Drivers who drove more than 3,000 miles in a single week.**
*   **Solution:** `SELECT DriverID, DATEPART(WEEK, s.DispatchDate), SUM(MilesDriven) FROM Routes r JOIN Shipments s ON r.ShipmentID = s.ShipmentID GROUP BY DriverID, DATEPART(WEEK, s.DispatchDate) HAVING SUM(MilesDriven) > 3000;`

**Q445: Find the vehicle that has completed the highest number of distinct routes.**
*   **Solution:** `SELECT TOP 1 VehicleID, COUNT(DISTINCT RouteID) FROM Routes GROUP BY VehicleID ORDER BY COUNT(DISTINCT RouteID) DESC;`

*(Questions 446-460 deal with driver fatigue tracking, cross-referencing `LicenseClass` (e.g., CDL-A) with `VehicleType` (e.g., 18-Wheeler) to find compliance violations).*

---

## Section 4: "Final Boss" Scenarios - Recursive CTEs (Questions 461-480)

**Domain Shift:** We now have a `WarehouseHierarchy` table: `(WarehouseID, Parent_WarehouseID)`. A Regional Hub acts as the parent to Local Hubs.

**Q461: Write a Recursive CTE to find all downstream warehouses that report to Hub #1.**
*   **Solution (The quintessential Recursive CTE interview question):**
    ```sql
    WITH RecursiveTree AS (
        -- Anchor Member (The Root)
        SELECT WarehouseID, Parent_WarehouseID, 1 AS Level
        FROM WarehouseHierarchy
        WHERE WarehouseID = 1
        
        UNION ALL
        
        -- Recursive Member (The Loop)
        SELECT wh.WarehouseID, wh.Parent_WarehouseID, rt.Level + 1
        FROM WarehouseHierarchy wh
        INNER JOIN RecursiveTree rt ON wh.Parent_WarehouseID = rt.WarehouseID
    )
    SELECT * FROM RecursiveTree;
    ```

**Q462: Using the Recursive CTE above, calculate the total combined `CurrentStock` of Hub #1 AND all of its downstream child hubs.**
*   **Solution:** *(Take the CTE from Q461, and `JOIN` it to the `Warehouses` table, performing a `SUM(CurrentStock)`).*

**Q463: Write a Recursive CTE to trace a package's journey.** 
(If a `ShipmentTracking` table has `(TrackingID, Prev_TrackingID, Location)`).
*   **Solution:** *(Similar to the hierarchy tree, anchor on the final destination `TrackingID` where `Prev_TrackingID IS NULL`, and recurse backwards to map the entire route).*

*(Questions 464-480 dive deeply into Bill of Materials (BOM) explosions, finding cyclical routing loops where Warehouse A sends to B, B to C, and C back to A).*

---

## Section 5: Architecture and Indexing Scenarios (Questions 481-500)

**Q481: The query `SELECT * FROM Shipments WHERE Status = 'Delivered'` is taking 10 minutes. The table has 500 million rows. What is your first step?**
*   **Solution:** Check the Execution Plan. It is likely performing a Full Table Scan because there is no Non-Clustered index on the `Status` column.

**Q482: You add an index on `Status`. The query still takes 10 minutes. Why?**
*   **Solution:** Because 99% of the shipments are 'Delivered'. The Query Optimizer realizes that using the index to look up 495 million individual row pointers is actually *slower* than just scanning the whole table. The index is not selective enough.

**Q483: How do you fix the above problem if users only care about querying 'In Transit' or 'Lost' shipments?**
*   **Solution:** Create a **Filtered Index**. `CREATE INDEX idx_ActiveShipments ON Shipments(Status) WHERE Status IN ('In Transit', 'Lost');`. This creates a tiny, lightning-fast index that completely ignores the 495 million delivered rows.

**Q484: A Deadlock occurs every night at 2:00 AM between the `InventoryUpdate` stored procedure and the `NightlyReporting` stored procedure. How do you resolve it?**
*   **Solution:** Ensure both procedures access tables in the exact same alphabetical order. If Procedure A locks `Warehouses` then `Shipments`, and Procedure B locks `Shipments` then `Warehouses`, they will inevitably deadlock. Order matters. Alternatively, use `WITH (NOLOCK)` or Snapshot Isolation for the reporting query if dirty reads are acceptable.

*(Questions 485-500 conclude the workbook with scenarios on Table Partitioning by Year, dealing with Parameter Sniffing in complex dynamic routing queries, and scaling databases horizontally using Shards).*

---

# Congratulations!
**You have successfully completed the 10-Chapter Architecture Textbook, the 15-Chapter Developer Fundamentals, and the 500-Question Scenario Workbook.** 

You are now equipped with the syntax, the design principles, and the architectural foresight required to build, query, and maintain enterprise-grade relational databases!
