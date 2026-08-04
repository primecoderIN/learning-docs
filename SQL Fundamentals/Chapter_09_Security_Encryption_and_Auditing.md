# Chapter 9 – Security, Encryption, and Auditing

## 1. Concept Overview

A database is the most valuable asset a company owns. Consequently, it is the primary target for malicious actors. Database security is not a single feature; it is an architectural philosophy called **Defense in Depth**, layered across four distinct pillars:

1.  **Authentication (Who are you?):** Verifying the identity of the user or application connecting to the database.
2.  **Authorization (What can you do?):** Ensuring that an authenticated user only has access to the exact tables and rows they need (Principle of Least Privilege).
3.  **Encryption (Is the data safe?):** Protecting data *In Transit* (as it moves across the network) and *At Rest* (when it sits on the physical hard drive).
4.  **Auditing (Who did what, and when?):** Maintaining an immutable log of access and modifications to prove compliance with legal frameworks (GDPR, HIPAA, PCI-DSS).

## 2. History

In the early days of the web (late 1990s), database security was virtually non-existent. Applications connected to databases using the ultimate superuser account (`sa` in SQL Server, `postgres` in PostgreSQL). Passwords were sent across the network in plaintext. In 1998, a hacker named Jeff Forristal published the first documented **SQL Injection** attack in Phrack Magazine. Over the next two decades, SQL Injection became the most devastating vulnerability in software history, leading to billion-dollar corporate data breaches and forcing RDBMS vendors to completely overhaul their security architectures.

## 3. Real-world analogy

Think of the database as a **High-Security Bank**.

*   **Firewall / Network Security:** The heavily fortified outer fence. Only armored trucks from approved routes are allowed in.
*   **Authentication (Logins):** The security guard at the front door checking your ID card.
*   **Authorization (Roles):** You are allowed into the lobby, but your ID card doesn't open the vault door.
*   **Row-Level Security (RLS):** Once inside the vault, you can only open *your specific* safe deposit box, even though millions of others are in the same room.
*   **Encryption at Rest (TDE):** The actual cash inside the box is shredded into a cipher. Even if a thief steals the entire physical box and takes it home, it's useless without the master decryption key.

## 4. Business problem solved

*   **Compliance & Legal:** Operating without rigorous security is illegal in many sectors. Security features allow businesses to pass rigorous SOC2 and ISO 27001 audits.
*   **Multi-Tenant Data Leaks:** Row-Level Security ensures that Organization A can never accidentally query Organization B's data, even if they share the exact same physical table.
*   **Insider Threats:** Dynamic Data Masking prevents rogue internal employees (like junior developers or customer support reps) from viewing raw credit card numbers or Social Security Numbers.

---

## 5. Microsoft SQL Server explanation

SQL Server is deeply integrated into the Windows ecosystem. Its security model is hierarchical: Server-Level Logins map to Database-Level Users, which are assigned to Roles. 

Microsoft strongly recommends **Windows Authentication** (Kerberos/NTLM) over SQL Server Authentication. With Windows Auth, passwords are not stored in the database, and applications use Active Directory Service Accounts to connect, entirely eliminating passwords from connection strings.

SQL Server offers advanced enterprise security features out of the box:
*   **TDE (Transparent Data Encryption):** Encrypts the physical MDF/LDF/Backup files.
*   **Always Encrypted:** The data is encrypted *inside the application* before it is sent to SQL Server. The DBA can query the table, but they only see cipher text. They cannot decrypt it.
*   **Dynamic Data Masking (DDM):** Masks sensitive data on the fly (e.g., displaying `XXXX-XXXX-XXXX-1234`) for unprivileged users.
*   **Row-Level Security (RLS):** Uses a hidden predicate function to automatically append a `WHERE` clause to every query, hiding rows based on the user's execution context.

## 6. SQL Server syntax

```sql
-- SQL SERVER SYNTAX
USE NextEventDB;
GO

-- 1. Create a specific, restricted Role
CREATE ROLE AppDataEntry;
GRANT SELECT, INSERT, UPDATE ON SCHEMA::Core TO AppDataEntry;
-- Explicitly deny DELETE
DENY DELETE ON SCHEMA::Core TO AppDataEntry;
GO

-- 2. Dynamic Data Masking (Masking an Email Address)
ALTER TABLE Core.Users 
ALTER COLUMN Email ADD MASKED WITH (FUNCTION = 'email()');
GO
-- A user without the 'UNMASK' permission will query the table and see 'aXXX@XXXX.com'

-- 3. Row-Level Security (RLS)
-- Step A: Create the filter logic
CREATE FUNCTION Security.fn_TenantAccessPredicate(@TenantID INT)
    RETURNS TABLE
    WITH SCHEMABINDING
AS
    RETURN SELECT 1 AS fn_accessResult 
    WHERE @TenantID = CAST(SESSION_CONTEXT(N'TenantID') AS INT);
GO

-- Step B: Bind it to the table
CREATE SECURITY POLICY Security.TenantPolicy
    ADD FILTER PREDICATE Security.fn_TenantAccessPredicate(TenantID) 
    ON Core.Tickets
    WITH (STATE = ON);
GO
```

## 7. SQL Server internals

**TDE Internals:** TDE operates at the I/O layer. When the Buffer Manager writes an 8KB page from RAM to the physical disk, it encrypts the page using an AES-256 Database Encryption Key (DEK). When reading the page back into RAM, it decrypts it. The data in RAM (Buffer Pool) is unencrypted. This protects against physical theft of the hard drives or backup tapes, but does not protect against a hacker who successfully gains `sa` access via SQL Injection.

## 8. SQL Server execution

**RLS Execution:**
When a developer queries `SELECT * FROM Core.Tickets;`
The Parser and Algebrizer detect the Security Policy attached to the table. The Optimizer silently rewrites the query, injecting the predicate function as an `INNER JOIN`. 
The execution plan actually runs: 
`SELECT * FROM Core.Tickets WHERE TenantID = CAST(SESSION_CONTEXT(N'TenantID') AS INT);`
Because this happens deep inside the relational engine, it is impossible for the application developer to bypass it.

## 9. SQL Server enterprise examples

*   **Healthcare (HIPAA):** Hospitals use SQL Server's **Always Encrypted** for patient medical records. The encryption keys are stored in an Azure Key Vault, accessible only by the application tier. Even if a rogue Database Administrator dumps the entire database, the patient data remains mathematically unreadable.
*   **Financial Auditing:** SQL Server Server Audits are configured to write directly to the Windows Security Event Log. This log is monitored by a separate SIEM (Security Information and Event Management) team, ensuring DBAs cannot cover their tracks by deleting their own audit trails.

## 10. SQL Server performance considerations

*   **RLS Overhead:** Because RLS executes a function for every single row evaluated, it can destroy performance if the predicate function is complex (e.g., executing subqueries to check a permissions table). RLS functions must be incredibly simple, ideally comparing a column directly to `SESSION_CONTEXT`.
*   **TDE Overhead:** Modern CPUs have AES-NI hardware acceleration. Enabling TDE usually incurs a negligible 1-3% CPU overhead. It should be enabled on all enterprise databases by default.

## 11. SQL Server common mistakes

*   **Using `sa` (System Administrator) in Connection Strings:** This grants the web application God-mode over the entire database server. If the application has a SQL Injection vulnerability, the hacker can use `xp_cmdshell` to drop down to the Windows OS layer and format the C: drive.
*   **Failing to backup the TDE Certificate:** If a server dies, and you have the `.bak` files but you lost the Master Key / Certificate used to encrypt them, your backups are permanently useless.

## 12. SQL Server best practices

*   Always practice the Principle of Least Privilege. Create a specific user for the Web Application that only has `EXECUTE` permissions on specific Stored Procedures, and zero direct `SELECT/UPDATE` access to tables.
*   Store connection strings and passwords in a secure vault (Azure Key Vault or HashiCorp Vault), never in application source code.

---

## 13. PostgreSQL explanation

PostgreSQL utilizes a fundamentally different security architecture, deeply rooted in its Unix/Linux heritage. 

Authentication is strictly governed by a physical file: **`pg_hba.conf` (Host-Based Authentication)**. This acts as an internal firewall, dictating exactly which IP addresses can connect, to which databases, using which users, and requiring which cryptographic authentication method (e.g., SCRAM-SHA-256).

PostgreSQL treats Users and Groups identically as **Roles**. A Role can inherit privileges from other Roles, creating a highly flexible RBAC (Role-Based Access Control) hierarchy.

## 14. PostgreSQL syntax

```sql
-- POSTGRESQL SYNTAX
-- Connect to next_event_db

-- 1. Create Roles and apply Least Privilege
CREATE ROLE app_user WITH LOGIN PASSWORD 'super_secure_pwd';
GRANT CONNECT ON DATABASE next_event_db TO app_user;
GRANT USAGE ON SCHEMA core TO app_user;
GRANT SELECT, INSERT, UPDATE ON ALL TABLES IN SCHEMA core TO app_user;

-- 2. Row-Level Security (RLS)
-- Step A: Enable RLS on the table
ALTER TABLE core.tickets ENABLE ROW LEVEL SECURITY;

-- Step B: Create the Policy
CREATE POLICY tenant_isolation_policy ON core.tickets
    FOR ALL
    TO app_user
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);

-- 3. Data Encryption (pgcrypto)
-- Hash a password securely in the database
CREATE EXTENSION IF NOT EXISTS pgcrypto;
INSERT INTO core.users (email, password_hash) 
VALUES ('test@test.com', crypt('plaintext_pwd', gen_salt('bf')));
```

## 15. PostgreSQL internals

**The `pg_hba.conf` File:**
When a connection request arrives at port 5432, the Postmaster daemon reads `pg_hba.conf` from top to bottom. It stops at the first matching rule. 
```text
# TYPE  DATABASE        USER            ADDRESS                 METHOD
host    next_event_db   app_user        192.168.1.50/32         scram-sha-256
host    all             all             0.0.0.0/0               reject
```
If the IP doesn't match, Postgres drops the connection before even asking for a password.

## 16. PostgreSQL execution

When executing an RLS-protected query, Postgres leverages its Optimizer beautifully.
If `app_user` runs `SELECT * FROM core.tickets WHERE event_id = 1;`
Postgres rewrites the query tree internally:
`SELECT * FROM core.tickets WHERE event_id = 1 AND tenant_id = current_setting('app.current_tenant_id')::UUID;`
Because this happens *before* the planner chooses indexes, Postgres can fully utilize an index on `(tenant_id, event_id)` to execute the query safely and optimally.

## 17. PostgreSQL enterprise examples

*   **PGAudit:** Native Postgres logging is somewhat chaotic for enterprise compliance. Financial institutions install the `pgaudit` extension, which hooks deep into the execution engine to produce highly structured JSON audit logs detailing exactly what statement was executed, by whom, and what objects were touched, satisfying strict GDPR/PCI-DSS auditors.

## 18. PostgreSQL performance considerations

*   **Connection Encryption (TLS/SSL):** Enforcing SSL (parameter `ssl = on` and `hostssl` in pg_hba) adds CPU overhead for encryption/decryption on every packet. For high-throughput internal microservices, architects often offload SSL termination to a sidecar proxy (like Envoy or PgBouncer) to save database CPU cycles.
*   **Cryptographic Functions:** Using `pgcrypto` to hash or encrypt data on the fly during `INSERT`/`SELECT` requires heavy CPU math. It is vastly superior to hash passwords in the application layer (e.g., using Node.js `bcrypt`) before sending them to the database, distributing the CPU load across the web tier.

## 19. PostgreSQL security considerations

*   **No Native TDE:** The biggest gap between Postgres and commercial databases is the lack of native Transparent Data Encryption. To encrypt data at rest, Postgres administrators must rely on OS-level encryption (LUKS/dm-crypt on Linux) or Cloud Provider encryption (AWS EBS volume encryption). 

## 20. PostgreSQL common mistakes

*   **Using `md5` authentication:** Historically, Postgres defaulted to `md5` for password hashing. MD5 is completely broken and vulnerable to collision and rainbow table attacks. Always update `pg_hba.conf` and `password_encryption` in `postgresql.conf` to use **SCRAM-SHA-256**.
*   **Leaving `public` schema permissions wide open:** By default, older versions of Postgres allowed any user to create tables in the `public` schema. A malicious user could create a table named `Users` to intercept traffic. Always revoke `CREATE` on the `public` schema.

## 21. PostgreSQL best practices

*   Always use `pg_hba.conf` to whitelist specific application IP addresses and explicitly reject `0.0.0.0/0`.
*   Leverage `current_setting()` to pass application variables (like Tenant ID) into the database for robust Row-Level Security.

---

## 22. SQL Server vs PostgreSQL comparison

| Feature | Microsoft SQL Server | PostgreSQL | Architect's Note |
| :--- | :--- | :--- | :--- |
| **Authentication** | Active Directory / SQL Auth | `pg_hba.conf` / Roles | Postgres's `pg_hba.conf` provides a strict, IP-level firewall that SQL Server lacks natively. |
| **Data At Rest (TDE)** | Native, Highly Integrated | Requires OS-level (LUKS/EBS) | SQL Server wins for native compliance. Enterprise Postgres forks (like EDB) add TDE, but community Postgres does not. |
| **Row-Level Security** | Yes (Security Policies) | Yes (Policies) | Both engines implement RLS efficiently at the query rewrite level. |
| **Data Masking** | Dynamic Data Masking (Native) | Requires 3rd party extensions | SQL Server provides excellent native masking for non-privileged users. |

## 23. Architect recommendations

**The SQL Injection Defense Standard**
SQL Injection happens when an application concatenates raw user input into a SQL string.
```javascript
// CATASTROPHIC ANTI-PATTERN
db.query("SELECT * FROM Users WHERE Email = '" + userInput + "'");
// If userInput = "'; DROP TABLE Users; --", the database is destroyed.
```
As an Architect, you must ban string concatenation for database queries. 
You must mandate **Parameterized Queries** (Prepared Statements) or Stored Procedures.
```javascript
// ARCHITECTURAL STANDARD
db.query("SELECT * FROM Users WHERE Email = @email", { email: userInput });
```
When parameterized, the database engine treats the input strictly as a literal string value, not as executable code. A hacker's attempt to drop a table is simply treated as a very strange email address.

## 24. DBA recommendations

*   **The Principle of Least Privilege:** If a web application only needs to read Events and insert Tickets, create a Role that specifically grants `SELECT` on Events and `INSERT` on Tickets. Do not grant `UPDATE` or `DELETE`. If a hacker compromises the web application, the blast radius is physically constrained by the database engine.

## 25. Developer recommendations

*   When building multi-tenant SaaS applications, do not rely on application `if` statements to isolate tenant data. Developers make mistakes. Implement RLS at the database layer. If a developer forgets a `WHERE TenantID = 1` clause in their code, RLS will safely step in and enforce it at the database engine level.

## 26. Production case study

**The NextEvent Right-To-Be-Forgotten (GDPR) Architecture**

*Scenario:* Under GDPR, a user has the "Right to be Forgotten." If they request account deletion, you must scrub their Personal Identifiable Information (PII) from all active databases and backups. However, finding and modifying historical backup tapes is physically impossible.

*Architectural Fix (Cryptographic Erasure):* We architected the `Users` table differently. Instead of storing PII in plaintext, we generated a unique AES encryption key for *every single user*. We stored this key in a central, highly secure KMS (Key Management Service) table. The PII in the main tables was encrypted with this key.
When a user requested deletion, we didn't search the massive database or backups. We simply deleted their 1 specific row in the KMS table (destroying their key). Instantly, all their data across the live database and all historical backups became mathematically undecipherable, successfully satisfying the GDPR requirement in milliseconds.

## 27. ASCII diagrams wherever helpful

**Defense in Depth Architecture**

```text
       [ HACKER / INTERNET ]
               |
               v
+=============================================+
| 1. NETWORK FIREWALL (Blocks bad IPs)        |
+=============================================+
               |
               v
+=============================================+
| 2. pg_hba.conf / Active Directory (Auth)    |
|    - SCRAM-SHA-256 or Kerberos tickets      |
+=============================================+
               |
               v
+=============================================+
| 3. ROLE-BASED ACCESS CONTROL (Authorization)|
|    - AppUser only has SELECT/INSERT         |
+=============================================+
               |
               v
+=============================================+
| 4. ROW-LEVEL SECURITY (RLS)                 |
|    - Can only see TenantID = 99             |
+=============================================+
               |
               v
+=============================================+
| 5. TDE / DISK ENCRYPTION (Data at Rest)     |
|    - Physical files are AES-256 encrypted   |
+=============================================+
               |
        [ ACTUAL DATA ]
```
*If a hacker breaches Layer 1, they hit Layer 2. If an insider bypasses Layer 2, they are stopped by Layer 3. Security requires overlapping physical and logical barriers.*

## 28. Enterprise design discussion

**Database Auditing vs. Performance**

Auditing every single `SELECT` statement in an enterprise database generates terabytes of log data per day, crippling disk I/O and crashing the server.
*Architectural Standard:* Never audit `SELECT` statements globally. 
Audit `SELECT` statements *only* on highly sensitive tables (e.g., Credit Cards, Medical Records). Audit all DDL (Schema changes, `DROP TABLE`, `CREATE USER`). Audit all Failed Login attempts. Push these logs asynchronously to a centralized logging server (Splunk/Elasticsearch) so they do not impact OLTP performance.

## 29. Hands-on exercises

1. In Postgres, open your `pg_hba.conf` file. Add a line to reject connections from your specific IP address. Restart the service and try to connect. Observe the hard firewall rejection.
2. In SQL Server, create a new Login and User. Grant them `SELECT` on the `Events` table, but `DENY` access to the `Tickets` table. Log in as that user and attempt to query both.

## 30. Coding exercises

1. Write a SQL Server script to implement Dynamic Data Masking on a `PhoneNumbers` column, showing only the last 4 digits (e.g., `XXX-XXX-1234`).
2. Write a Postgres script to enable Row-Level Security on the `Venues` table, allowing a venue manager to only see rows where the `manager_id` equals their `current_user` ID.

## 31. Mini project

**Objective:** Secure the NextEvent Payment Gateway.
1. Create a `CreditCards` table. 
2. Ensure you never store the raw credit card number. Store a Hash (for fast exact-match lookups if needed) and an Encrypted blob (for retrieval).
3. Create a specific database Role named `PaymentProcessor`. Grant this role access to the `CreditCards` table.
4. Implement Row-Level Security so the `PaymentProcessor` can only query cards belonging to the Tenant currently processing the checkout.

## 32. Quiz

1. Explain the difference between Authentication and Authorization.
2. What is the fundamental mechanism that enables SQL Injection, and how do Parameterized Queries prevent it?
3. How does Transparent Data Encryption (TDE) protect data, and what specific attack vector does it *not* protect against?

## 33. Interview questions

**Entry Level (Developer)**
*   **Q:** What is SQL Injection?
    *   **A:** SQL Injection is an attack where malicious SQL statements are inserted into entry fields for execution. It happens when untrusted user input is directly concatenated into a SQL string. We prevent it by using Parameterized Queries (Prepared Statements) or ORMs that parameterize automatically.

**Intermediate Level (Backend Developer / DBA)**
*   **Q:** What is Row-Level Security (RLS) and why is it useful in a multi-tenant application?
    *   **A:** RLS is a database feature that automatically restricts row access based on the execution context of the user (like a Tenant ID variable). It is useful because it enforces data isolation at the engine layer, acting as an absolute safety net even if a backend developer forgets to include a `WHERE TenantID = x` clause in their application code.
*   **Q:** In Postgres, what does the `pg_hba.conf` file do?
    *   **A:** It stands for Host-Based Authentication. It acts as an internal firewall and authentication router, controlling which IP addresses can connect, to which databases, using which users, and enforcing the specific cryptographic authentication method required.

**Advanced / Tricky (Performance Engineer / Architect)**
*   **Q:** You enabled TDE (Transparent Data Encryption) on SQL Server. Later, a developer writes an unparameterized query and a hacker successfully exploits a SQL Injection vulnerability, selecting all data from the `Users` table. The CEO asks why TDE didn't stop them. How do you explain this?
    *   **A:** TDE encrypts data *at rest* (on the physical disk) to protect against someone stealing the hard drives or backup tapes. When the SQL Server engine processes a query, it decrypts the 8KB pages into the memory Buffer Pool. Because the hacker successfully authenticated (via the hijacked application connection) and executed a query through the SQL Engine, the engine performed exactly as designed—it read the unencrypted data from memory and returned it. TDE does not protect against SQL Injection; that requires Defense in Depth (AppSec, WAF, Least Privilege, and Parameterization).

## 34. Chapter summary

### Learning Summary
We explored the critical importance of Database Security through the lens of Defense in Depth. We learned how to secure the perimeter (pg_hba.conf / Active Directory), enforce the Principle of Least Privilege (Roles), and isolate multi-tenant data transparently at the engine level (Row-Level Security). We addressed the existential threat of SQL Injection, the mechanics of Data at Rest encryption (TDE), and strategies for complying with complex legal frameworks like GDPR using Cryptographic Erasure.

### Key Takeaways
*   Security requires overlapping layers: Authentication, Authorization, Encryption, and Auditing.
*   SQL Injection is eradicated entirely by Parameterized Queries. Never use string concatenation.
*   Row-Level Security (RLS) moves multi-tenant isolation logic from the flawed application tier to the bulletproof database engine.
*   Transparent Data Encryption (TDE) protects physical files (disks/backups), but does not protect against malicious queries executed via valid connections.
*   Never grant applications `sa` or `postgres` superuser access.

### Glossary
*   **Defense in Depth:** A security strategy utilizing multiple overlapping layers of defense.
*   **SQL Injection:** An exploit caused by concatenating untrusted input into a SQL command.
*   **TDE:** Transparent Data Encryption (Encrypting physical files on disk).
*   **RLS:** Row-Level Security (Applying hidden `WHERE` clauses to restrict row access).
*   **Cryptographic Erasure:** Deleting an encryption key to instantly render massive amounts of data permanently unreadable.

### Common Mistakes
*   Using simple passwords and `md5` hashing in Postgres.
*   Auditing every `SELECT` statement and crashing the server via I/O exhaustion.
*   Believing that TDE solves all security problems.

### Best Practices
*   Use Windows Authentication (Active Directory) for SQL Server whenever possible to eliminate passwords from connection strings.
*   Use SCRAM-SHA-256 for PostgreSQL authentication.
*   Implement Dynamic Data Masking to protect PII from internal employees who require database access.

### Preparation for Next Chapter
In Chapter 10, we will conclude our architectural journey by mastering **Database Monitoring, Performance Tuning, and Troubleshooting**. We will learn how to diagnose a slow database when the CPU is at 100%, how to read Dynamic Management Views (DMVs) and Wait Statistics, how to find runaway queries, and how to survive a production outage with calm, analytical precision.
