# SQL injection UNION attack, retrieving data from other tables

**Date:** July 2026<br>
**Author:** ShahinSecLab<br>
**Category:** SQL Injection<br>
**Vulnerability:** UNION-based SQL Injection<br>
**Difficulty:** Easy<br>
**Platform:** PortSwigger Web Security Academy<br>
**Database:** PostgreSQL<br>
**Tools:** Burp Suite Community Edition, Firefox

## Table of Contents

* [Introduction](#introduction)
* [Attack Flow](#attack-flow)
* [Why This Attack Works](#why-this-attack-works)
* [Lab Setup](#lab-setup)
* [Tools Used](#tools-used)
* [Prerequisites](#prerequisites)
* [Step 1 - Launching the Lab and Identifying the Injection Point](#step-1---launching-the-lab-and-identifying-the-injection-point)
* [Step 2 - Determining the Number of Columns](#step-2---determining-the-number-of-columns)
* [Step 3 - Identifying String-Compatible Columns](#step-3---identifying-string-compatible-columns)
* [Step 4 - Retrieving Data from Another Table](#step-4---retrieving-data-from-another-table)
* [Step 5 - Identifying Administrator Credentials](#step-5---identifying-administrator-credentials)
* [Step 6 - Logging In as the Administrator](#step-6---logging-in-as-the-administrator)
* [How Defenders Can Catch This](#how-defenders-can-catch-this)
* [How to Prevent It](#how-to-prevent-it)
* [References](#references)
* [Lessons Learned](#lessons-learned)

## Introduction

SQL Injection is one of the most common and dangerous vulnerabilities in web applications. It happens when a web application includes user input in an SQL query without checking or handling it properly.

One type of SQL Injection is **UNION-based SQL Injection**. In this attack, the attacker uses the SQL `UNION` operator to combine the original query with another `SELECT` query. If both queries have the same structure, the database returns the results from both queries together.

As a result, the attacker can access data from database tables that are normally hidden from the application’s users.

## Attack Flow
```
User selects a product category
            │
            ▼
The application sends a SQL query to the database
            │
            ▼
The category parameter is vulnerable to SQL injection
            │
            ▼
The attacker adds a UNION SELECT payload
            │
            ▼
The database runs the modified query
            │
            ▼
Data from another table is returned
            │
            ▼
The application displays the returned data
            │
            ▼
Sensitive data becomes visible in the response
```

## Why This Attack Works

This attack works because the application puts user input directly into a SQL query without checking it properly.

When a user selects a product category, the application builds a SQL query to get matching products from the database. If the input is not validated, an attacker can change the original query by adding SQL code.

The `UNION` operator allows the results of two `SELECT` statements to be combined into one result. If the injected query has the same number of columns and compatible data types as the original query, the database treats both queries as valid and returns the combined results.

Instead of showing only product data, the application may also return data from another table, such as usernames or passwords, if the database user has permission to read those tables.

The database does not know which part of the SQL query was written by the application and which part was injected by the attacker. It simply executes the final SQL query it receives, which allows the injected query to return additional data.

For a UNION attack to work, a few conditions must be met:

- The application must be vulnerable to SQL injection.
- The original query must return data to the page.
- The injected query must return the same number of columns as the original query.
- The data types of the matching columns must be compatible.
- The database user must have permission to read the target table.

If all of these conditions are true, the attacker can access data from database tables that are normally hidden from the application’s users.

## Lab Setup

| Item | Details |
|------|---------|
| **Platform** | PortSwigger Web Security Academy |
| **Category** | SQL Injection |
| **Technique** | UNION-based SQL Injection |
| **Injection Point** | Product `category` parameter |
| **Operating System** | Kali Linux |
| **Browser** | Firefox |
| **Proxy Tool** | Burp Suite Community Edition |

## Tools Used

| Tool | Purpose |
|------|---------|
| **Burp Suite Community Edition** | Intercepted, modified, and replayed HTTP requests using Proxy and Repeater |
| **Firefox** | Accessed and tested the target application |

## Prerequisites

| Requirement | Why It Is Needed |
|-------------|------------------|
| **Basic SQL** | To understand how SQL queries work |
| **SELECT Statement** | To understand how data is retrieved from a database |
| **WHERE Clause** | To understand how records are filtered |
| **UNION Operator** | To understand how two query results can be combined |
| **Basic SQL Injection** | To understand how user input can change a SQL query |
| **HTTP Requests and Responses** | To understand how the application communicates with the server |
| **Burp Suite Basics** | To intercept and modify HTTP requests |
| **Database Tables and Columns** | To understand where the retrieved data comes from |

## Step 1 - Launching the Lab and Identifying the Injection Point

First, I launched the lab from **PortSwigger Web Security Academy** and opened the target application in Firefox.

The application is an online shop where products can be filtered by category. I configured Firefox to work with Burp Suite and enabled Intercept to capture the HTTP requests.

After selecting a product category, I captured the request in Burp Suite and sent it to **Repeater** for testing.

**Captured Request:**

```http
GET /filter?category=Gifts HTTP/2
Host: 0ae6002704b03b508417e66b007600cf.web-security-academy.net
```

The category value was sent as a request parameter, so I started testing this parameter to check whether the application was handling user input safely.

I added a single quote (`'`) at the end of the `category` value:

```http
category=Gifts'
```

After sending the request, the application returned a database error instead of the normal response. This happened because the additional quote affected the SQL query structure created by the application.

<p align="center">
  <img src="images/step1-1.png" width="600">
</p>

I also tested the input with additional quotes (`''`). This time, the application returned a normal response instead of an SQL error. This showed that the application was directly using the user input inside an SQL query.

<p align="center">
  <img src="images/step1-2.png" width="600">
</p>

Based on these responses, I identified the `category` parameter as a possible SQL injection point and continued further testing.

## Step 2 - Determining the Number of Columns

After identifying the possible SQL injection point, the next step was to find the number of columns returned by the original SQL query.

This is required because a UNION query only works when both `SELECT` statements return the same number of columns.

I tested the parameter with a UNION-based query and used different numbers of `NULL` values to match the columns returned by the original query.

Example:

```sql
Gift' UNION SELECT NULL,NULL --
```
**Breakdown**

| Part | Description |
|--------------|-------------|
| `Gift` | The original category value. |
| `'` | Closes the original string in the SQL query. |
| `UNION` | Combines the results of another `SELECT` query with the original query. |
| `SELECT` | Specifies the values to return. |
| `NULL, NULL` | Placeholder values used to match the number of columns in the original query. |
| `--` | Comments out the rest of the original SQL query so it is ignored by the database. |

After sending the request, the payload worked when I used two `NULL` values. This confirmed that the original SQL query returned two columns, so I moved on to the next step.

<p align="center">
  <img src="images/step2-1.png" width="600">
</p>

## Step 3 - Identifying String-Compatible Columns

After determining the number of columns, the next step was to identify which columns could accept string values.

This is important because the data I wanted to retrieve from the database, such as usernames and passwords, is stored as text. For a `UNION` query to work, the data types in both queries must be compatible.

To test this, I replaced the `NULL` values with simple string values and observed the application's response.

Payload used:

```sql
Gifts' UNION SELECT 'a','a' --
```
**Breakdown**

| Payload Part | Description |
|--------------|-------------|
| `Gifts'` | Closes the original SQL query after the `Gifts` category value. |
| `UNION SELECT` | Combines the original query with a new query.|
| `'a'` | Tests whether the first column accepts and displays text data. |
| `'a'` | Tests whether the second column accepts and displays text data. |
| `--` | Comments out the rest of the original SQL query to prevent syntax errors. |

After sending the request, the application returned a normal response without any SQL errors. This confirmed that both columns accepted string values, allowing me to retrieve text data in the next step.

<p align="center">
  <img src="images/step3-1.png" width="600">
</p>


## Step 4 - Retrieving Data from Another Table

After identifying the number of columns and the columns that supported string values, the next step was to test whether data from another database table could be retrieved.

The application was originally displaying product information, but by using a `UNION` query, I attempted to retrieve data from the `users` table.

Payload used:

```sql
Gifts' UNION SELECT username, password FROM users --
```
<p align="center">
  <img src="images/step4-1.png" width="600">
</p>

After sending the request, the application returned the contents of the `users` table instead of only product data. The response included usernames and passwords, confirming that the `UNION` query successfully retrieved data from another database table.

## Step 5 - Identifying Administrator Credentials

After successfully retrieving data from the `users` table, the next step was to review the returned data and identify the administrator account.

The response contained usernames and their corresponding passwords. Among them was the `administrator` account and its password.

<p align="center">
  <img src="images/step5-1.png" width="600">
</p>

At this point, I had successfully extracted the administrator's credentials from the database. This demonstrated the impact of the SQL injection vulnerability, as sensitive authentication data could be accessed without authorization.

## Step 6 - Logging In as the Administrator

Using the extracted administrator credentials, I attempted to log in to the application.

The login was successful, confirming that the retrieved credentials were valid.

<p align="center">
  <img src="images/step6-1.png" width="600">
</p>
<p align="center">
  <img src="images/step6-2.png" width="600">
</p>

Successfully logging in as the administrator demonstrated the full impact of the SQL injection vulnerability. By retrieving credentials directly from the database, an attacker could gain unauthorized access to administrative functionality.

## How Defenders Can Catch This

There are several ways defenders can detect or identify UNION-based SQL injection attempts before they cause serious damage.

1. Monitor Web Requests

Check web server logs for suspicious input containing SQL keywords or special characters, such as:

- `UNION`
- `SELECT`
- `FROM`
- `--`
- `'`

Example:
```
category=Gifts' UNION SELECT username,password FROM users --
```

2. Monitor Database Activity

Watch for unusual database queries, especially those accessing sensitive tables like users or returning large amounts of data.

3. Use a Web Application Firewall (WAF)

A properly configured WAF can detect and block many common SQL injection payloads before they reach the application.

4. Monitor Database Errors

Unexpected SQL errors or repeated database exceptions may indicate someone is testing for SQL injection vulnerabilities.

5. Perform Regular Security Testing

Regular penetration testing, code reviews, and vulnerability scans can help identify SQL injection vulnerabilities before they are exploited.

## How to Prevent It

The best way to prevent UNION-based SQL injection is to ensure that user input is never treated as part of an SQL query.

### 1. Use Parameterized Queries (Prepared Statements)

Parameterized queries keep user input separate from the SQL statement, preventing attackers from changing the query.

**Unsafe:**

```sql
SELECT * FROM products WHERE category = 'Gifts'
```

**Safe:**

```sql
SELECT * FROM products WHERE category = ?
```

### 2. Validate User Input

Only accept input that matches the expected format. For example, allow only valid category names and reject unexpected values whenever possible.

### 3. Apply the Principle of Least Privilege

Give the application's database account only the permissions it needs. Avoid granting administrative privileges or unnecessary access to sensitive tables.

### 4. Hide Database Error Messages

Do not expose database errors to users. Instead, return a generic error message.

**Instead of:**

```text
SQL syntax error near UNION SELECT
```

**Show:**

```text
Something went wrong. Please try again later.
```

### 5. Perform Regular Security Testing

Regular code reviews, vulnerability assessments, and penetration testing can help identify and fix SQL injection vulnerabilities before they are exploited.


## References

- PortSwigger Web Security Academy - SQL Injection Labs  
  https://portswigger.net/web-security/sql-injection

- OWASP Top 10 - Injection  
  https://owasp.org/www-project-top-ten/

- OWASP SQL Injection Prevention Cheat Sheet  
  https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html

- PostgreSQL Documentation  
  https://www.postgresql.org/docs/

## Lessons Learned

This lab helped me understand how a UNION-based SQL injection vulnerability can be used to retrieve data from database tables that were never intended to be exposed.

Some of the key things I learned were:

- How to identify a UNION-based SQL injection point.
- Why the number of columns must match before a `UNION` query works.
- How to determine which columns can display string data.
- How a `UNION` query can be used to retrieve data from another database table.
- Why parameterized queries are one of the most effective ways to prevent SQL injection.

Overall, this lab gave me a better understanding of how UNION-based SQL injection works in practice and why secure database queries are important.