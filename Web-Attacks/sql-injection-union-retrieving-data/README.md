# SQL injection UNION attack, retrieving data from other tables

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
* [Step 4 - Identifying the Target Table](#step-4---identifying-the-target-table)
* [Step 5 - Identifying the Required Columns](#step-5---identifying-the-required-columns)
* [Step 6 - Retrieving Data from Another Table](#step-6---retrieving-data-from-another-table)
* [Step 7 - Extracting the Administrator Credentials](#step-7---extracting-the-administrator-credentials)
* [Step 8 - Logging In as the Administrator](#step-8---logging-in-as-the-administrator)
* [How Defenders Can Catch This](#how-defenders-can-catch-this)
* [How to Prevent It](#how-to-prevent-it)
* [References](#references)
* [Lessons Learned](#lessons-learned)

## Introduction

SQL Injection is one of the most common and dangerous vulnerabilities in web applications. It happens when a web application includes user input in an SQL query without checking or handling it properly.

One type of SQL Injection is **UNION-based SQL Injection**. In this attack, the attacker uses the SQL `UNION` operator to combine the original query with another `SELECT` query. If both queries have the same structure, the database returns the results from both queries together.

This allows the attacker to view data from database tables that the application was never meant to show.

## Attack Flow
```
User selects a product category
            │
            ▼
Application builds a SQL query
            │
            ▼
The category parameter is vulnerable to SQL injection
            │
            ▼
A UNION query is added to the original SQL statement
            │
            ▼
The database executes the combined query
            │
            ▼
Results from another database table are returned
            │
            ▼
The application displays the combined results
            │
            ▼
Sensitive data becomes visible in the response
```

## Why This Attack Works

This attack works because the application puts user input directly into a SQL query without checking it properly.

When a user selects a product category, the application builds a SQL query to get matching products from the database. If the input is not validated, an attacker can change the original query by adding SQL code.

The `UNION` operator allows the results of two `SELECT` statements to be combined into one result. If the injected query has the same number of columns and compatible data types as the original query, the database treats both queries as valid and returns the combined results.

Instead of showing only product data, the application may also return data from another table, such as usernames or passwords, if the database user has permission to read those tables.

This happens because the database cannot tell the difference between the original query and the injected one. It simply runs the final SQL statement it receives from the application.

For a UNION attack to work, a few conditions must be met:

- The application must be vulnerable to SQL injection.
- The original query must return data to the page.
- The injected query must return the same number of columns as the original query.
- The data types of the matching columns must be compatible.
- The database user must have permission to read the target table.

If all of these conditions are true, the attacker can make the application display data that was never meant to be exposed.

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
| **Burp Suite Community Edition** | Intercepted and modified HTTP requests |
| **Firefox** | Accessed the target application |

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

The application is an online shop where products can be filtered by category. Before testing anything, I configured Firefox to send traffic through Burp Suite and turned Intercept on.

After browsing the website, I clicked one of the product categories and captured the request in Burp Suite. I noticed that the selected category was sent as a request parameter.

Since category filters are commonly used in SQL queries to fetch products from a database, I decided to test this parameter for SQL injection in the next step.

**Captured Request:**

```http
GET /filter?category=Corporate+gifts HTTP/1.1
Host: 0a9200580392419380b2535e002d003d.web-security-academy.net
```

**What I observed:**

- The application displayed products based on the selected category.
- The category value was included in the request.
- This parameter looked like a good place to start testing for SQL injection.

<p align="center">
  <img src="images/step1-1.png" width="600">
</p>