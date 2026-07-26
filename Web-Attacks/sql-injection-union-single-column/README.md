# SQL injection UNION attack, retrieving multiple values in a single column

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
* [Step 4 - Combining the Username and Password into One Column](#step-4---combining-the-username-and-password-into-one-column)
* [Step 5 - Retrieving Usernames and Passwords](#step-5---retrieving-usernames-and-passwords)
* [Step 6 - Logging In as the Administrator](#step-6---logging-in-as-administrator)
* [How Defenders Can Catch This](#how-defenders-can-catch-this)
* [How to Prevent It](#how-to-prevent-it)
* [References](#references)
* [Lessons Learned](#lessons-learned)

## Introduction

This lab demonstrates how a UNION-based SQL injection can be used to retrieve multiple values in a single column.

The application is vulnerable because it directly includes user input in an SQL query without proper validation. Since the application only displays one text column, the `username` and `password` values must be combined into a single string before they can be returned.

In this write-up, I show how I identified the SQL injection point, determined the query structure, combined two database columns into one, extracted user credentials, and logged in as the administrator.

## Attack Flow

```text
User selects a product category
            │
            ▼
The application sends a SQL query to database
            │
            ▼
The category parameter is vulnerable to SQL injection
            │
            ▼
Determine the number of columns
            │
            ▼
Identify String-Compatible Columns
            │
            ▼
The username and password are combined into one column
            │
            ▼
A UNION SELECT query is used
            │
            ▼
The database returns usernames and password
            │
            ▼
The administrator credentials are found
            │
            ▼
The administrator account is accessed
```
## Why This Attack Works

The application builds SQL queries by directly using the value supplied in the `category` parameter.

Because the input is not properly validated, an attacker can inject a` UNION SELECT` statement into the original query.

The application only displays one text column, so the `username` and `password` values are combined into a single string using PostgreSQL's `||` operator. This allows both values to be returned in the same column and displayed in the application's response.

## Lab Setup

| Item | Details |
|------|---------|
| **Platform** | PortSwigger Web Security Academy |
| **Category** | SQL Injection |
| **Technique** | UNION-based SQL Injection |
| **Injection Point** | Product category parameter |
| **Operating System** | Kali Linux |
| **Browser** | Firefox |
| **Proxy Tool** | Burp Suite Community Edition |

## Tools Used

| Tool | Purpose |
|------|---------|
| **Burp Suite Community Edition** | Intercepted, modified, and replayed HTTP requests using Proxy and Repeater |
| **Firefox** | Accessed and tested the target application |

## Prerequisites

Before starting this lab, you should understand:

- Basic SQL queries
- How `UNION SELECT` works
- HTTP requests and responses
- Using Burp Suite Proxy and Repeater
- Basic knowledge of PostgreSQL string concatenation (`||`)

## Step 1 - Launching the Lab and Identifying the Injection Point

First, I launched the lab from **PortSwigger Web Security Academy** and opened the target application in Firefox.

The application is an online shop where products can be filtered by category. I configured Firefox to work with Burp Suite and enabled Intercept to capture the HTTP requests.

After selecting a product category, I captured the request in Burp Suite and sent it to **Repeater** for testing.

**Captured Request:**

```http
GET /filter?category=Gifts HTTP/2
Host: 0a1c0059032dfc4185ccf83600820098.web-security-academy.net
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
Gifts' UNION SELECT NULL,'string' --
```
**Breakdown**

| Part | Description |
|--------------|-------------|
| `Gifts'` | Closes the original SQL query after the `Gifts` category value. |
| `UNION SELECT` | Combines the original query with a new query. |
| `NULL` | Placeholder for the first column. It is used because no text value is needed in this column. |
| `'string'` | Tests whether the second column accepts and displays text data. If `string` appears in the response, the column is string-compatible. |
| `--` | Comments out the rest of the original SQL query to prevent syntax errors. |

After sending the request, the application returned a normal response without any SQL errors. This confirmed that both columns accepted string values, allowing me to retrieve text data in the next step.

<p align="center">
  <img src="images/step3-1.png" width="600">
</p>

## Step 4 - Combining the Username and Password into One Column

The application only displayed one column that could contain text, but the information I wanted to retrieve was stored in two separate columns: username and password.

To return both values in a single column, I used PostgreSQL's string concatenation operator (`||`). I also added the `:` character between the username and password to make the output easier to read.

```sql
Gift' UNION SELECT NULL, username||':'||password FROM users--
```
**Breakdown**

| Part | Description |
|------|-------------|
| `Gift'` | Closes the original string in the SQL query after the category value. |
| `UNION SELECT` | Combines the original query with a new query. |
| `NULL` | Placeholder for the first column. |
| `username` | Retrieves the username from the `users` table. |
| `||` | PostgreSQL string concatenation operator used to join values together. |
| `':'` | Adds a separator between the username and password, making the output easier to read. |
| `||` | Concatenates the next value to the existing string. |
| `password` | Retrieves the password from the `users` table. |
| `FROM users` | Specifies that the data should be retrieved from the `users` table. |
| `--` | Comments out the rest of the original SQL query to prevent syntax errors. |

This payload combines the username and password into a single value, such as **administrator~password**, allowing both values to be returned in the same column.

This made it easy to identify each username and its corresponding password. In the next step, I used this payload to retrieve all usernames and passwords from the `users` table.

<p align="center">
  <img src="images/step4-1.png" width="600">
</p>

## Step 5 - Retrieving Usernames and Passwords

After sending the payload from the previous step, the application returned the data from the users table.

Each result contained a username and password separated by the `:` character, making it easy to identify which password belonged to each user.

**Output:**
```
administrator:mgp.................
carlos:2xx.................
wiener:z2y.................
```
The response contained the credentials for all users, including the administrator account.

<p align="center">
 <img src="images/step5-1.png" width="600"> 
 </p>

From the results, I located the administrator username and copied its password. These credentials were then used in the final step to log in as the administrator user and complete the lab.

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

- Monitor for SQL keywords such as `UNION`, `SELECT`, and comment characters (`--`) in user input.
- Watch for repeated requests with different SQL payloads.
- Review web server and application logs for unusual requests.
- Use a Web Application Firewall (WAF) to detect and block common SQL injection payloads.
- Monitor for unexpected database queries and failed SQL statements.

## How to Prevent It

- Use parameterized queries or prepared statements.
- Never build SQL queries by directly concatenating user input.
- Validate and filter user input.
- Apply the principle of least privilege to database accounts.
- Return generic error messages instead of database errors.
- Keep the application and database software up to date.
- Test the application regularly for SQL injection vulnerabilities.

## References

- [PortSwigger Web Security Academy](https://portswigger.net/web-security)
- [Lab: SQL injection UNION attack, retrieving multiple values in a single column](https://portswigger.net/web-security/sql-injection/union-attacks/lab-retrieve-multiple-values-in-single-column)
- [PostgreSQL Documentation - String Functions and Operators](https://www.postgresql.org/docs/current/functions-string.html)
- [OWASP SQL Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)
- [OWASP Web Security Testing Guide](https://owasp.org/www-project-web-security-testing-guide/)

## Lessons Learned

During this lab, I learned how a UNION-based SQL injection can return multiple database values in a single column.

- A `UNION` query must return the same number of columns as the original query.
- It is important to identify which columns accept text values before retrieving data.
- PostgreSQL's `||` operator can be used to combine multiple values into a single string.
- Adding a separator such as `:` makes the returned data easier to read.
- Exposed SQL injection vulnerabilities can allow attackers to retrieve sensitive information, including usernames and passwords.