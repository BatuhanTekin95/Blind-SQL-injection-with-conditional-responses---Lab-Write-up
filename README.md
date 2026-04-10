# Blind-SQL-injection-with-conditional-responses---Lab-Write-up

Exploitation of blind SQL injection via conditional responses in an Oracle database, demonstrating error-based data extraction and side-channel inference.

📌 Initial Analysis 

During initial testing, the application revealed several important behaviors:

A tracking cookie is used for analytics. The cookie value is embedded directly into a SQL query. The application does not return query results in responses. 

However, it responds differently when a database error occurs. 

👉 This indicates a Blind SQL Injection vulnerability using conditional errors. 

Additionally, the backend database is Oracle, which is significant because: 
Oracle requires FROM dual for standalone SELECT queries. Error-based techniques differ from other DBMS (e.g., MySQL, PostgreSQL). Certain functions (like TO_CHAR(1/0)) can be used to deliberately trigger errors. 


🧠 Key Observation Although query results are not visible, the application leaks information via: Differences in HTTP responses when database errors occur This creates a side-channel, allowing indirect data extraction.

🧪 Injection Confirmation & Exploitation After identifying the tracking cookie as the injection point, testing was performed using Burp Suite Repeater. 

🔹 Initial Payload ' || ( SELECT CASE WHEN (SUBSTR(password,1,1)='a') THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator' ) || ' ![sda](https://github.com/user-attachments/assets/dd11eb09-c399-4720-bac4-c7eb9639a6da)



🧠 Payload Logic || → Concatenates injected SQL into the original query 

SUBSTR(password,1,1) → Extracts the first character


CASE WHEN → Evaluates a condition TO_CHAR(1/0) → Forces a division-by-zero error in Oracle Condition Result 

TRUE Server returns 500 error FALSE Server returns normal response (200) 

🔍 Testing Example Character Tested Response Result 'a' 200 OK False 'b' 500 True 'c' 200 OK False 

👉 Therefore, the first character of the password is: b 

⚙️ Automation with Burp Suite Intruder To efficiently extract the full password, the attack was automated. 


🔹 Payload Template <img width="238" height="29" alt="Ekran görüntüsü 2026-04-10 093322" src="https://github.com/user-attachments/assets/5ad28d3b-dc12-426c-8bb6-1139ec08244a" />



🔹 Attack Configuration Attack Type: Cluster Bomb Payload Set 1 (Position): Range: 1 → 20 Payload Set 2 (Characters): a–z 0–9 
(The reason that I used Cluster Bomb is that each unknown character of the password needed to be tested across multiple positions (up to 20), and every possible character (a–z, 0–9) had to be tried for each position. This results in a total of 20 × 36 = 720 requests, allowing full combinational testing and making it ideal for efficiently enumerating the entire password.)



🔹 Execution Steps Load payload sets into Intruder Launch attack Monitor responses: 

500 → correct character 

200 → incorrect character 

🧠 Key Insight This lab demonstrates that even when applications: 

Do not display query results They can still be exploited because: 

Conditional logic leaks data Server errors act as an oracle Automation enables full data extraction 


🏁 Conclusion This vulnerability highlights critical security failures:

Unsanitized user input in SQL queries Exposure of database error behavior Even with suppressed output, attackers can: 

🔥 Extract sensitive data using error-based blind SQL injection 

## 📘 Lab Information

This lab is part of the PortSwigger Web Security Academy and focuses on exploiting a blind SQL injection vulnerability using conditional responses.

- **Lab Name:** Blind SQL injection with conditional responses  
- **Database:** Oracle  
- **Vulnerability Type:** Blind SQL Injection (Error-Based / Conditional)  
