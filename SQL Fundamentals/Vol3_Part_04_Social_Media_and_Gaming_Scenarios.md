# Volume 3, Part 4: Social Media & Gaming Scenarios (Questions 301-400)

**Domain Context:** You are the Lead Database Engineer for a massive social network that also features integrated multiplayer gaming. 
**Core Schema:**
*   `Users` (UserID, Username, Email, JoinDate, IsBanned)
*   `Posts` (PostID, UserID, Content, PostDate, LikesCount, ViewsCount)
*   `Followers` (FollowerID, FolloweeID, FollowDate) *(User A follows User B)*
*   `GameSessions` (SessionID, UserID, GameID, StartTime, EndTime, Score)
*   `Games` (GameID, Title, Genre)

---

## Section 1: User Engagement and Activity (Questions 301-320)

**Q301: Find the 10 most viewed posts of all time.**
*   **Solution:** `SELECT TOP 10 PostID, ViewsCount FROM Posts ORDER BY ViewsCount DESC;`

**Q302: Calculate the average number of likes per post for the user 'xX_Gamer_Xx'.**
*   **Solution:** `SELECT AVG(p.LikesCount) FROM Posts p JOIN Users u ON p.UserID = u.UserID WHERE u.Username = 'xX_Gamer_Xx';`

**Q303: Find the total number of users who joined in the last 24 hours.**
*   **Solution (SQL Server):** `SELECT COUNT(*) FROM Users WHERE JoinDate >= DATEADD(HOUR, -24, GETDATE());`

**Q304: Identify "Lurkers": Users who have joined but have NEVER made a post.**
*   **Solution:** `SELECT u.Username FROM Users u LEFT JOIN Posts p ON u.UserID = p.UserID WHERE p.PostID IS NULL;`

**Q305: Find the date when the platform reached its 1-millionth user.**
*   **Solution (Using CTE & ROW_NUMBER):**
    ```sql
    WITH UserSeq AS (
        SELECT UserID, JoinDate, ROW_NUMBER() OVER(ORDER BY JoinDate ASC) AS SignupNumber
        FROM Users
    )
    SELECT JoinDate FROM UserSeq WHERE SignupNumber = 1000000;
    ```

**Q306: Which day of the week has the highest volume of posts?**
*   **Solution (Postgres):** `SELECT EXTRACT(DOW FROM PostDate) AS DayOfWeek, COUNT(*) FROM Posts GROUP BY EXTRACT(DOW FROM PostDate) ORDER BY COUNT(*) DESC LIMIT 1;`

**Q307: Calculate the overall "Engagement Rate" (Total Likes / Total Views) across the whole platform.**
*   **Solution:** `SELECT CAST(SUM(LikesCount) AS FLOAT) / SUM(ViewsCount) AS EngagementRate FROM Posts WHERE ViewsCount > 0;`

**Q308: Find users who posted more than 50 times in a single day.**
*   **Solution:** `SELECT UserID, CAST(PostDate AS DATE), COUNT(*) FROM Posts GROUP BY UserID, CAST(PostDate AS DATE) HAVING COUNT(*) > 50;`

*(Questions 309-320 cover calculating DAU (Daily Active Users) vs MAU (Monthly Active Users), tracking viral spikes, and isolating hashtag text from post content using SUBSTRING).*

---

## Section 2: Social Graph and Friend Networks (Questions 321-340)

**Q321: Calculate the total number of followers for UserID = 5.**
*   **Solution:** `SELECT COUNT(*) FROM Followers WHERE FolloweeID = 5;`

**Q322: Calculate the total number of people UserID = 5 is following.**
*   **Solution:** `SELECT COUNT(*) FROM Followers WHERE FollowerID = 5;`

**Q323: Find "Mutuals" (Users who mutually follow each other). Specifically, check if User 5 and User 10 are mutuals.**
*   **Solution:**
    ```sql
    SELECT 'Yes' AS IsMutual
    WHERE EXISTS (SELECT 1 FROM Followers WHERE FollowerID = 5 AND FolloweeID = 10)
      AND EXISTS (SELECT 1 FROM Followers WHERE FollowerID = 10 AND FolloweeID = 5);
    ```

**Q324: Find the most followed user on the platform (The Top Influencer).**
*   **Solution:**
    ```sql
    SELECT TOP 1 u.Username, COUNT(f.FollowerID) AS FollowerCount 
    FROM Users u JOIN Followers f ON u.UserID = f.FolloweeID 
    GROUP BY u.Username ORDER BY FollowerCount DESC;
    ```

**Q325: Suggest Friends: Find users who follow the exact same people that UserID = 1 follows.**
*   **Solution (Complex Set theory / Relational Division):**
    ```sql
    -- A simplified approach finding users with high overlap
    SELECT f2.FollowerID, COUNT(*) AS MutualFollows
    FROM Followers f1
    JOIN Followers f2 ON f1.FolloweeID = f2.FolloweeID
    WHERE f1.FollowerID = 1 AND f2.FollowerID <> 1
    GROUP BY f2.FollowerID
    ORDER BY MutualFollows DESC;
    ```

**Q326: Find "Unrequited Followers": Users who follow someone, but that person does NOT follow them back.**
*   **Solution (Using LEFT JOIN on the same table):**
    ```sql
    SELECT f1.FollowerID, f1.FolloweeID
    FROM Followers f1
    LEFT JOIN Followers f2 ON f1.FollowerID = f2.FolloweeID AND f1.FolloweeID = f2.FollowerID
    WHERE f2.FollowerID IS NULL;
    ```

*(Questions 327-340 dive deep into graph traversal, finding 2nd-degree connections ("Friends of Friends"), and analyzing follower churn rates over time).*

---

## Section 3: Content Moderation and Banning (Questions 341-360)

**Q341: Find all posts containing the banned word "HackTool".**
*   **Solution:** `SELECT PostID, UserID FROM Posts WHERE Content LIKE '%HackTool%';`

**Q342: Write a query to flag/ban any user who has posted the banned word more than 3 times.**
*   **Solution:**
    ```sql
    UPDATE Users SET IsBanned = 1 
    WHERE UserID IN (
        SELECT UserID FROM Posts WHERE Content LIKE '%HackTool%' GROUP BY UserID HAVING COUNT(*) > 3
    );
    ```

**Q343: A user (UserID = 99) was just banned for spam. Write a transaction to delete all their posts and remove all their follower connections.**
*   **Solution:**
    ```sql
    BEGIN TRANSACTION;
    DELETE FROM Posts WHERE UserID = 99;
    DELETE FROM Followers WHERE FollowerID = 99 OR FolloweeID = 99;
    -- UPDATE Users SET IsBanned = 1 WHERE UserID = 99; -- (If not actually deleting the user)
    COMMIT;
    ```

**Q344: Identify "Bot" behavior: Users who have made exactly the same post content 5 times in a row.**
*   **Solution:** `SELECT UserID, Content, COUNT(*) FROM Posts GROUP BY UserID, Content HAVING COUNT(*) >= 5;`

*(Questions 345-360 focus on building automated triggers to quarantine posts with too many negative reports, and calculating the false-positive ban rate by joining against an `Appeals` table).*

---

## Section 4: Gaming Analytics and Leaderboards (Questions 361-380)

**Q361: Generate a global Leaderboard showing the Top 10 highest scores for the game "CyberStrike".**
*   **Solution:**
    ```sql
    SELECT TOP 10 u.Username, gs.Score
    FROM GameSessions gs JOIN Users u ON gs.UserID = u.UserID JOIN Games g ON gs.GameID = g.GameID
    WHERE g.Title = 'CyberStrike'
    ORDER BY gs.Score DESC;
    ```

**Q362: We only want to show each user ONCE on the leaderboard (Their Personal Best).**
*   **Solution (Using CTE & ROW_NUMBER):**
    ```sql
    WITH RankedScores AS (
        SELECT u.Username, gs.Score,
        ROW_NUMBER() OVER(PARTITION BY gs.UserID ORDER BY gs.Score DESC) as rn
        FROM GameSessions gs JOIN Users u ON gs.UserID = u.UserID JOIN Games g ON gs.GameID = g.GameID
        WHERE g.Title = 'CyberStrike'
    )
    SELECT TOP 10 Username, Score FROM RankedScores WHERE rn = 1 ORDER BY Score DESC;
    ```

**Q363: Calculate the total playtime (in minutes) for each user across all games.**
*   **Solution:** `SELECT UserID, SUM(DATEDIFF(MINUTE, StartTime, EndTime)) AS TotalPlayMinutes FROM GameSessions GROUP BY UserID;`

**Q364: Find the most popular game genre based on total session count.**
*   **Solution:** `SELECT TOP 1 g.Genre, COUNT(gs.SessionID) FROM Games g JOIN GameSessions gs ON g.GameID = gs.GameID GROUP BY g.Genre ORDER BY COUNT(gs.SessionID) DESC;`

**Q365: Identify "Rage Quitters": Users whose game sessions consistently end after less than 1 minute.**
*   **Solution:**
    ```sql
    SELECT UserID, COUNT(*) AS ShortSessions 
    FROM GameSessions 
    WHERE DATEDIFF(SECOND, StartTime, EndTime) < 60 
    GROUP BY UserID HAVING COUNT(*) > 10;
    ```

*(Questions 366-380 cover Elo rating calculations, detecting win-trading by analyzing IP addresses, and finding the peak concurrent player count).*

---

## Section 5: Advanced Time-Series and Sessionization (Questions 381-400)

**Q381: Calculate the "Sticky Factor" (DAU / MAU) for the month of July 2025.**
*   **Solution:** *(Requires creating two CTEs: one calculating the distinct users on a given day, and one calculating the distinct users for the whole month, then dividing them).*

**Q382: Sessionization: A user's "Play Session" is defined as a series of games played with less than 15 minutes of idle time between them. Find the length of User 1's longest session.**
*   **Solution (A notoriously difficult Data Engineering problem):**
    ```sql
    -- Step 1: Use LAG to find time between games.
    -- Step 2: Use a SUM(CASE WHEN gap > 15 THEN 1 ELSE 0) OVER (ORDER BY time) to create a "Session_ID" grouping integer.
    -- Step 3: Group by that Session_ID and calculate MIN(StartTime) and MAX(EndTime).
    ```

**Q383: Find the Median game score. (SQL does not have a native MEDIAN aggregate function).**
*   **Solution (Using PERCENTILE_CONT):**
    ```sql
    SELECT DISTINCT PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY Score) OVER () AS MedianScore 
    FROM GameSessions;
    ```

**Q384: Find the hour of the day (0-23) when the servers experience the highest load (most active sessions).**
*   **Solution:** `SELECT DATEPART(HOUR, StartTime) AS HourOfDay, COUNT(*) FROM GameSessions GROUP BY DATEPART(HOUR, StartTime) ORDER BY COUNT(*) DESC;`

*(Questions 385-400 involve gap-and-island problems to calculate longest consecutive daily login streaks for rewarding players).*

---
**This concludes Part 4 (Social & Gaming). In our final installment, Part 5, we will conquer Logistics & Supply Chain!**
