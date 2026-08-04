# Volume 3, Part 3: Healthcare & Pharma Scenarios (Questions 201-300)

**Domain Context:** You are the Lead Data Analyst for a regional hospital network. 
**Core Schema:**
*   `Patients` (PatientID, FirstName, LastName, DateOfBirth, Gender, SSN)
*   `Doctors` (DoctorID, FirstName, LastName, Specialty, DepartmentID)
*   `Appointments` (ApptID, PatientID, DoctorID, ApptDate, Status, Diagnosis)
*   `Prescriptions` (ScriptID, ApptID, DrugName, Dosage, Refills, IssueDate)
*   `Departments` (DepartmentID, DeptName, Floor)

---

## Section 1: Patient and Appointment Management (Questions 201-220)

**Q201: Find all patients born after the year 2000.**
*   **Solution:** `SELECT PatientID, FirstName, LastName FROM Patients WHERE YEAR(DateOfBirth) >= 2000;`

**Q202: Calculate the exact age (in years) of every patient.**
*   **Solution (SQL Server):** `SELECT PatientID, DATEDIFF(YEAR, DateOfBirth, GETDATE()) AS Age FROM Patients;` *(Note: This is an approximation; a more accurate way checks the month/day).*
*   **Solution (Postgres):** `SELECT PatientID, EXTRACT(YEAR FROM age(CURRENT_DATE, DateOfBirth)) AS Age FROM Patients;`

**Q203: Find all upcoming appointments for the 'Cardiology' department.**
*   **Solution:** 
    ```sql
    SELECT a.ApptID, a.ApptDate 
    FROM Appointments a 
    JOIN Doctors d ON a.DoctorID = d.DoctorID 
    JOIN Departments dept ON d.DepartmentID = dept.DepartmentID 
    WHERE dept.DeptName = 'Cardiology' AND a.ApptDate > GETDATE();
    ```

**Q204: Identify patients who missed their appointments (Status = 'No-Show').**
*   **Solution:** `SELECT p.FirstName, p.LastName, a.ApptDate FROM Patients p JOIN Appointments a ON p.PatientID = a.PatientID WHERE a.Status = 'No-Show';`

**Q205: Count the number of 'No-Shows' per doctor.**
*   **Solution:** `SELECT DoctorID, COUNT(*) AS MissedAppts FROM Appointments WHERE Status = 'No-Show' GROUP BY DoctorID ORDER BY MissedAppts DESC;`

**Q206: Find patients who have never had an appointment.**
*   **Solution:** `SELECT p.PatientID FROM Patients p LEFT JOIN Appointments a ON p.PatientID = a.PatientID WHERE a.ApptID IS NULL;`

**Q207: Find the most common diagnosis across the entire hospital network.**
*   **Solution:** `SELECT TOP 1 Diagnosis, COUNT(*) FROM Appointments WHERE Diagnosis IS NOT NULL GROUP BY Diagnosis ORDER BY COUNT(*) DESC;`

**Q208: Calculate the percentage of appointments that result in a prescription.**
*   **Solution:** 
    ```sql
    SELECT (CAST(COUNT(p.ScriptID) AS FLOAT) / COUNT(a.ApptID)) * 100 AS RxRate
    FROM Appointments a LEFT JOIN Prescriptions p ON a.ApptID = p.ApptID;
    ```

**Q209: Find patients who have had appointments with more than 3 different doctors.**
*   **Solution:** `SELECT PatientID FROM Appointments GROUP BY PatientID HAVING COUNT(DISTINCT DoctorID) > 3;`

**Q210: List all appointments that happened on weekends (Saturday or Sunday).**
*   **Solution (SQL Server):** `SELECT ApptID FROM Appointments WHERE DATENAME(dw, ApptDate) IN ('Saturday', 'Sunday');`

*(Questions 211-220 involve identifying conflicting appointments where a patient booked two doctors at the exact same time, and filtering patients by gender/age demographics for clinical trials).*

---

## Section 2: Doctor Utilization and Scheduling (Questions 221-240)

**Q221: Find all doctors who do not have any appointments scheduled for tomorrow.**
*   **Solution:**
    ```sql
    SELECT d.DoctorID, d.LastName 
    FROM Doctors d 
    LEFT JOIN Appointments a ON d.DoctorID = a.DoctorID AND CAST(a.ApptDate AS DATE) = CAST(DATEADD(DAY, 1, GETDATE()) AS DATE)
    WHERE a.ApptID IS NULL;
    ```

**Q222: Calculate the average number of patients seen per day for each doctor.**
*   **Solution (Using CTE):**
    ```sql
    WITH DailyVisits AS (
        SELECT DoctorID, CAST(ApptDate AS DATE) AS VisitDay, COUNT(PatientID) AS PatientsSeen
        FROM Appointments WHERE Status = 'Completed' GROUP BY DoctorID, CAST(ApptDate AS DATE)
    )
    SELECT DoctorID, AVG(PatientsSeen) AS AvgPatientsPerDay FROM DailyVisits GROUP BY DoctorID;
    ```

**Q223: Which Department handles the highest volume of patients?**
*   **Solution:** `SELECT TOP 1 dept.DeptName, COUNT(a.ApptID) FROM Departments dept JOIN Doctors d ON dept.DepartmentID = d.DepartmentID JOIN Appointments a ON d.DoctorID = a.DoctorID GROUP BY dept.DeptName ORDER BY COUNT(a.ApptID) DESC;`

**Q224: Identify Doctors who are overworked (Scheduled for more than 15 appointments in a single day).**
*   **Solution:** `SELECT DoctorID, CAST(ApptDate AS DATE) FROM Appointments GROUP BY DoctorID, CAST(ApptDate AS DATE) HAVING COUNT(*) > 15;`

**Q225: The hospital is merging the 'Neurology' and 'Neurosurgery' departments into 'Neuroscience'. Write an UPDATE statement to reflect this.**
*   **Solution:** `UPDATE Departments SET DeptName = 'Neuroscience' WHERE DeptName IN ('Neurology', 'Neurosurgery');`

*(Questions 226-240 deal with shift scheduling, finding gaps in a doctor's schedule using `LEAD` to calculate the time difference between appointments).*

---

## Section 3: Prescriptions and Drug Interactions (Questions 241-260)

**Q241: List the top 5 most prescribed drugs.**
*   **Solution:** `SELECT TOP 5 DrugName, COUNT(*) FROM Prescriptions GROUP BY DrugName ORDER BY COUNT(*) DESC;`

**Q242: Find all patients who were prescribed 'Amoxicillin' with '0' refills.**
*   **Solution:** `SELECT a.PatientID FROM Prescriptions p JOIN Appointments a ON p.ApptID = a.ApptID WHERE p.DrugName = 'Amoxicillin' AND p.Refills = 0;`

**Q243: Identify "Doctor Shopping": Find patients who were prescribed the same drug (e.g., 'Oxycodone') by 3 or more different doctors in the last 30 days.**
*   **Solution:**
    ```sql
    SELECT a.PatientID 
    FROM Appointments a 
    JOIN Prescriptions p ON a.ApptID = p.ApptID 
    WHERE p.DrugName = 'Oxycodone' AND a.ApptDate > DATEADD(DAY, -30, GETDATE())
    GROUP BY a.PatientID 
    HAVING COUNT(DISTINCT a.DoctorID) >= 3;
    ```

**Q244: Calculate the total number of refills authorized by the 'Pediatrics' department.**
*   **Solution:** `SELECT SUM(p.Refills) FROM Prescriptions p JOIN Appointments a ON p.ApptID = a.ApptID JOIN Doctors d ON a.DoctorID = d.DoctorID JOIN Departments dept ON d.DepartmentID = dept.DepartmentID WHERE dept.DeptName = 'Pediatrics';`

**Q245: Identify potential harmful drug interactions: Find any patient who was prescribed both 'Warfarin' and 'Aspirin' at any time.**
*   **Solution (Using INTERSECT):**
    ```sql
    SELECT a.PatientID FROM Appointments a JOIN Prescriptions p ON a.ApptID = p.ApptID WHERE p.DrugName = 'Warfarin'
    INTERSECT
    SELECT a.PatientID FROM Appointments a JOIN Prescriptions p ON a.ApptID = p.ApptID WHERE p.DrugName = 'Aspirin';
    ```

*(Questions 246-260 cover calculating average dosages, flagging prescriptions issued by inactive doctors, and tracking medication adherence).*

---

## Section 4: Advanced Grouping and Readmissions (Questions 261-280)

**Q261: Identify "Readmissions": Find patients who were discharged (Status = 'Completed') but returned for another appointment within 30 days.**
*   **Solution (Using LEAD):**
    ```sql
    WITH PatientVisits AS (
        SELECT PatientID, ApptDate, LEAD(ApptDate, 1) OVER(PARTITION BY PatientID ORDER BY ApptDate ASC) AS NextAppt
        FROM Appointments WHERE Status = 'Completed'
    )
    SELECT DISTINCT PatientID FROM PatientVisits WHERE DATEDIFF(DAY, ApptDate, NextAppt) <= 30;
    ```

**Q262: Rank doctors within their specific department based on how many appointments they've completed.**
*   **Solution:**
    ```sql
    SELECT d.DoctorID, d.DepartmentID, COUNT(a.ApptID) as TotalAppts,
    RANK() OVER(PARTITION BY d.DepartmentID ORDER BY COUNT(a.ApptID) DESC) as DeptRank
    FROM Doctors d JOIN Appointments a ON d.DoctorID = a.DoctorID
    WHERE a.Status = 'Completed'
    GROUP BY d.DoctorID, d.DepartmentID;
    ```

**Q263: Find the longest streak (in days) a doctor went without an appointment.**
*   **Solution (Requires LAG to find the gap between current and previous appt).**
    ```sql
    WITH ApptGaps AS (
        SELECT DoctorID, ApptDate, LAG(ApptDate, 1) OVER(PARTITION BY DoctorID ORDER BY ApptDate ASC) AS PrevAppt
        FROM Appointments
    )
    SELECT DoctorID, MAX(DATEDIFF(DAY, PrevAppt, ApptDate)) AS MaxGapDays FROM ApptGaps GROUP BY DoctorID;
    ```

*(Questions 264-280 explore cohort analysis of patient survival rates, calculating the rolling 12-month average of ER visits, and segmenting patients by age deciles using NTILE).*

---

## Section 5: Data Privacy (HIPAA) and Data Scrubbing (Questions 281-300)

**Q281: Write a query to permanently delete all appointment history for patients who have exercised their "Right to be Forgotten", ensuring no orphaned prescriptions are left behind.**
*   **Solution:**
    ```sql
    -- Assuming a CTE or list of target patients. If ON DELETE CASCADE is not set, must delete children first.
    BEGIN TRANSACTION;
    DELETE FROM Prescriptions WHERE ApptID IN (SELECT ApptID FROM Appointments WHERE PatientID = @TargetID);
    DELETE FROM Appointments WHERE PatientID = @TargetID;
    DELETE FROM Patients WHERE PatientID = @TargetID;
    COMMIT;
    ```

**Q282: Create a View that masks Patient SSNs (e.g., 'XXX-XX-1234') for billing clerks.**
*   **Solution (SQL Server):**
    ```sql
    CREATE VIEW vw_BillingPatients AS
    SELECT PatientID, FirstName, LastName, 
    'XXX-XX-' + RIGHT(SSN, 4) AS MaskedSSN 
    FROM Patients;
    ```

**Q283: Find any Patient whose `DateOfBirth` is magically recorded as being in the future (Data validation).**
*   **Solution:** `SELECT PatientID, DateOfBirth FROM Patients WHERE DateOfBirth > GETDATE();`

**Q284: An auditor wants to know if any doctor has modified an old prescription record. How would you design the database to track this?**
*   **Solution:** You cannot solve this with a simple query unless an Audit architecture exists. You must add an `UpdatedAt` column to the `Prescriptions` table, and create an `AFTER UPDATE` Trigger to copy the `OLD` data into a `PrescriptionAuditHistory` table whenever a change occurs.

*(Questions 285-300 dive into handling overlapping time zones for telehealth appointments, encrypting SSNs using `ENCRYPTBYKEY`, and standardizing gender/race string formats using `REPLACE` and `UPPER` for CDC compliance reporting).*

---
**This concludes Part 3 (Healthcare). In Part 4, we will tackle Social Media & Gaming!**
