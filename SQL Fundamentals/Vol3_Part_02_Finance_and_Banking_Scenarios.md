# Volume 3, Part 2: Finance & Banking Scenarios (Questions 101-200)

**Domain Context:** You are the Lead Database Architect for a multinational retail bank. 
**Core Schema:**
*   `Customers` (CustomerID, FirstName, LastName, TaxID, RiskScore)
*   `Accounts` (AccountID, CustomerID, AccountType, Balance, Status)
*   `Transactions` (TransactionID, AccountID, TxDate, Amount, TxType, IsFlagged) *(Amount is positive for deposits, negative for withdrawals)*
*   `Loans` (LoanID, CustomerID, PrincipalAmount, InterestRate, StartDate, EndDate)
*   `Branches` (BranchID, BranchName, City)

---

## Section 1: Balance and Ledger Queries (Questions 101-120)

**Q101: Find all active 'Checking' accounts with a balance under $50.**
*   **Solution:** `SELECT AccountID, Balance FROM Accounts WHERE AccountType = 'Checking' AND Balance < 50 AND Status = 'Active';`

**Q102: Calculate the total deposits held by the entire bank.**
*   **Solution:** `SELECT SUM(Balance) FROM Accounts;`

**Q103: List the names of customers who have a negative balance in any of their accounts.**
*   **Solution:** `SELECT DISTINCT c.FirstName, c.LastName FROM Customers c JOIN Accounts a ON c.CustomerID = a.CustomerID WHERE a.Balance < 0;`

**Q104: Find the total number of transactions processed on January 15, 2026.**
*   **Solution:** `SELECT COUNT(*) FROM Transactions WHERE CAST(TxDate AS DATE) = '2026-01-15';`

**Q105: Identify customers who hold both a 'Checking' and a 'Savings' account.**
*   **Solution (Using INTERSECT):** 
    ```sql
    SELECT CustomerID FROM Accounts WHERE AccountType = 'Checking'
    INTERSECT
    SELECT CustomerID FROM Accounts WHERE AccountType = 'Savings';
    ```

**Q106: The ledger is corrupted. Calculate what the balance *should* be for Account #500 based solely on its transaction history.**
*   **Solution:** `SELECT SUM(Amount) AS CalculatedBalance FROM Transactions WHERE AccountID = 500;`

**Q107: Find accounts where the stored `Balance` does NOT match the sum of their `Transactions`.**
*   **Solution:**
    ```sql
    SELECT a.AccountID, a.Balance, SUM(t.Amount) AS TxSum
    FROM Accounts a JOIN Transactions t ON a.AccountID = t.AccountID
    GROUP BY a.AccountID, a.Balance
    HAVING a.Balance <> SUM(t.Amount);
    ```

**Q108: Calculate the total net money movement (Deposits vs Withdrawals) for yesterday.**
*   **Solution (SQL Server):** `SELECT SUM(Amount) FROM Transactions WHERE CAST(TxDate AS DATE) = CAST(DATEADD(DAY, -1, GETDATE()) AS DATE);`

**Q109: Identify the largest single withdrawal ever made.**
*   **Solution:** `SELECT MIN(Amount) FROM Transactions WHERE TxType = 'Withdrawal';` *(Assuming withdrawals are negative numbers).*

**Q110: Find customers who have an account but have NEVER made a transaction.**
*   **Solution:** `SELECT a.CustomerID FROM Accounts a LEFT JOIN Transactions t ON a.AccountID = t.AccountID WHERE t.TransactionID IS NULL;`

*(Questions 111-120 focus on complex grouping by Branch and identifying dormant accounts).*

---

## Section 2: Fraud Detection and Anomalies (Questions 121-140)

**Q121: Find all transactions flagged by the automated fraud system (`IsFlagged = 1`).**
*   **Solution:** `SELECT * FROM Transactions WHERE IsFlagged = 1;`

**Q122: Find accounts that had more than 5 withdrawals within a single 24-hour period.**
*   **Solution:** 
    ```sql
    SELECT AccountID, CAST(TxDate AS DATE), COUNT(*) 
    FROM Transactions 
    WHERE TxType = 'Withdrawal' 
    GROUP BY AccountID, CAST(TxDate AS DATE) 
    HAVING COUNT(*) > 5;
    ```

**Q123: Identify "Structuring" (Smurfing): Customers making multiple deposits just under the $10,000 reporting threshold (e.g., $9,000 to $9,999).**
*   **Solution:** `SELECT AccountID, COUNT(*) FROM Transactions WHERE Amount BETWEEN 9000 AND 9999 GROUP BY AccountID HAVING COUNT(*) > 2;`

**Q124: Calculate the total amount of money involved in flagged transactions this month.**
*   **Solution:** `SELECT SUM(ABS(Amount)) FROM Transactions WHERE IsFlagged = 1 AND MONTH(TxDate) = MONTH(GETDATE()) AND YEAR(TxDate) = YEAR(GETDATE());`

**Q125: Find customers with a high RiskScore (> 80) who opened an account in the last 7 days.**
*   **Solution:** `SELECT c.CustomerID FROM Customers c JOIN Accounts a ON c.CustomerID = a.CustomerID WHERE c.RiskScore > 80 AND a.Status = 'New';`

**Q126: Using Window Functions, find transactions that are 3x larger than the account's average transaction size.**
*   **Solution:**
    ```sql
    WITH AccountAverages AS (
        SELECT AccountID, TransactionID, Amount, 
        AVG(ABS(Amount)) OVER(PARTITION BY AccountID) AS AvgTx
        FROM Transactions
    )
    SELECT * FROM AccountAverages WHERE ABS(Amount) > (AvgTx * 3);
    ```

**Q127: Identify Rapid Movement: Accounts that received a deposit > $50,000 and then had a withdrawal of the exact same amount within 1 hour.**
*   **Solution (Using SELF JOIN):**
    ```sql
    SELECT t1.AccountID
    FROM Transactions t1
    JOIN Transactions t2 ON t1.AccountID = t2.AccountID
    WHERE t1.Amount > 50000 
      AND t2.Amount = (t1.Amount * -1)
      AND t2.TxDate > t1.TxDate 
      AND DATEDIFF(HOUR, t1.TxDate, t2.TxDate) <= 1;
    ```

*(Questions 128-140 involve geographical tracking, matching TaxIDs to OFAC sanction lists, and detecting duplicate transactions submitted by buggy payment gateways).*

---

## Section 3: Loan and Interest Calculations (Questions 141-160)

**Q141: Find all customers who currently have a Loan.**
*   **Solution:** `SELECT DISTINCT c.FirstName, c.LastName FROM Customers c JOIN Loans l ON c.CustomerID = l.CustomerID;`

**Q142: Calculate the total Outstanding Principal across all active loans.**
*   **Solution:** `SELECT SUM(PrincipalAmount) FROM Loans WHERE EndDate > GETDATE();`

**Q143: Find the customer with the highest total loan debt.**
*   **Solution:** `SELECT TOP 1 CustomerID, SUM(PrincipalAmount) FROM Loans GROUP BY CustomerID ORDER BY SUM(PrincipalAmount) DESC;`

**Q144: The bank is offering a rate reduction. Write an UPDATE to lower all interest rates > 15% down to 14.5%.**
*   **Solution:** `UPDATE Loans SET InterestRate = 14.5 WHERE InterestRate > 15.0;`

**Q145: Calculate the Simple Interest earned on each loan for exactly 1 year (Principal * Rate).**
*   **Solution:** `SELECT LoanID, PrincipalAmount * (InterestRate / 100.0) AS OneYearInterest FROM Loans;`

**Q146: Find loans that are maturing (EndDate) within the next 30 days.**
*   **Solution (SQL Server):** `SELECT LoanID FROM Loans WHERE EndDate BETWEEN GETDATE() AND DATEADD(DAY, 30, GETDATE());`

*(Questions 147-160 focus on amortization schedule calculations, compound interest formulas using `POWER()`, and cross-referencing high-risk loans with negative bank balances).*

---

## Section 4: Advanced Window Functions & Running Balances (Questions 161-180)

**Q161: Generate a Bank Statement (Running Balance) for Account #500.**
*   **Solution (Crucial Interview Question):** 
    ```sql
    SELECT 
        TxDate, 
        Amount, 
        SUM(Amount) OVER(PARTITION BY AccountID ORDER BY TxDate ROWS UNBOUNDED PRECEDING) AS RunningBalance
    FROM Transactions 
    WHERE AccountID = 500;
    ```

**Q162: Find the End-of-Day balance for Account #500 for the last 30 days.**
*   **Solution:** *(Requires creating a calendar CTE, then joining it to the transactions and using `LAST_VALUE()` or cumulative sums).*

**Q163: Identify the account with the highest *average* daily balance this month.**
*   **Solution:** *(Requires calculating the running balance per day using Window Functions, then wrapping it in a CTE to average those daily balances, then ordering by the highest average).*

**Q164: For each customer, rank their accounts by Balance from highest to lowest.**
*   **Solution:** `SELECT CustomerID, AccountID, Balance, RANK() OVER(PARTITION BY CustomerID ORDER BY Balance DESC) as Rnk FROM Accounts;`

**Q165: Calculate the 7-day Moving Average of total deposits to the bank.**
*   **Solution:**
    ```sql
    WITH DailyTotal AS (
        SELECT CAST(TxDate AS DATE) as Dt, SUM(Amount) as DailySum FROM Transactions WHERE Amount > 0 GROUP BY CAST(TxDate AS DATE)
    )
    SELECT Dt, DailySum, 
    AVG(DailySum) OVER(ORDER BY Dt ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS MovingAvg
    FROM DailyTotal;
    ```

*(Questions 166-180 cover percentile rankings of wealth managers (NTILE), finding the gap between a customer's largest and smallest deposit, and year-over-year loan default rates).*

---

## Section 5: Transaction Concurrency and Rollbacks (Questions 181-200)

**Q181: Write a stored procedure to transfer $500 from Account 1 to Account 2, ensuring ACID compliance.**
*   **Solution:**
    ```sql
    CREATE PROCEDURE TransferMoney (@FromAcc INT, @ToAcc INT, @Amount DECIMAL) AS
    BEGIN
        BEGIN TRY
            BEGIN TRANSACTION;
            
            -- Deduct
            UPDATE Accounts SET Balance = Balance - @Amount WHERE AccountID = @FromAcc;
            INSERT INTO Transactions (AccountID, Amount) VALUES (@FromAcc, -@Amount);
            
            -- Add
            UPDATE Accounts SET Balance = Balance + @Amount WHERE AccountID = @ToAcc;
            INSERT INTO Transactions (AccountID, Amount) VALUES (@ToAcc, @Amount);
            
            COMMIT;
        END TRY
        BEGIN CATCH
            ROLLBACK;
            THROW;
        END CATCH
    END;
    ```

**Q182: A developer ran `UPDATE Accounts SET Balance = 0;` without a WHERE clause inside an uncommitted transaction. What is the impact?**
*   **Solution:** Because the transaction is uncommitted, it holds an Exclusive lock on every single row in the `Accounts` table. All other bank systems attempting to read or update balances will freeze (blocking), effectively taking the entire bank offline until a DBA kills the query or it rolls back.

*(Questions 183-200 are architectural scenarios detailing Optimistic vs Pessimistic Concurrency Control, Deadlock resolution in high-frequency trading databases, and isolating dirty reads using Transaction Isolation Levels).*

---
**This concludes Part 2 (Finance & Banking). In Part 3, we will tackle Healthcare & Pharmaceuticals!**
