# SQL injection UNION attack, retrieving multiple values in a single column

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
* [Step 6 - Logging in as Administrator](#step-6---logging-in-as-administrator)
* [How to Detect This Attack](#how-to-detect-this-attack)
* [How to Prevent It](#how-to-prevent-it)
* [Lessons Learned](#lessons-learned)
* [References](#references)







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
After sending the request, the application returned a normal response without any SQL errors. This confirmed that both columns accepted string values, allowing me to retrieve text data in the next step.

<p align="center">
  <img src="images/step3-1.png" width="600">
</p>
