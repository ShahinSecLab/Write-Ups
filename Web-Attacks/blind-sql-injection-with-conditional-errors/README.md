# Blind SQL injection with conditional errors

**Date:** July 2026<br>
**Author:** ShahinSecLab<br>
**Category:** SQL Injection<br>
**Vulnerability:** Blind SQL Injection<br>
**Difficulty:** Easy<br>
**Platform:** PortSwigger Web Security Academy<br>
**Database:** PostgreSQL<br>
**Tools:** Burp Suite Community Edition, Firefox

## Table of Contents

* [Introduction](#introduction)
* [Why This Attack Works](#why-this-attack-works)
* [Lab Setup](#lab-setup)
* [Prerequisites](#prerequisites)
* [Attack Flow](#attack-flow)
* [Step-by-Step Walkthrough](#step-by-step-walkthrough)
   * [Step 1 — Confirm the injection point](#step-1--confirm-the-injection-point)
   * [Step 2 — Confirm it's a SQL syntax error](#step-2--confirm-its-a-sql-syntax-error)
   * [Step 3 — Confirm the query is actually running](#step-3--confirm-the-query-is-actually-running)
   * [Step 4 — Check if the users table exists](#step-4--check-if-the-users-table-exists)
   * [Step 5 — Trigger a conditional error](#step-5--trigger-a-conditional-error)
   * [Step 6 — Check if the administrator user exists](#step-6--check-if-the-administrator-user-exists)
   * [Step 7 — Find the password length](#step-7--find-the-password-length)
   * [Step 8 — Extract the password with Burp Intruder](#step-8--extract-the-password-with-burp-intruder)
   * [Step 9 — Log in as administrator](#step-9--log-in-as-administrator)
* [Detection](#detection)
* [Prevention](#prevention)
* [References](#references)
* [Key Takeaways](#key-takeaways)

## Introduction

This lab demonstrates a blind SQL injection vulnerability that uses conditional errors.

Unlike other blind SQL injection techniques, the application does not show different page content or use a time delay. Instead, it returns an error only when a SQL condition is true. When the condition is false, the page loads normally.

By comparing these two responses, it is possible to test SQL conditions and slowly gather information from the database.

In this lab, the backend database is Oracle, so the payloads use Oracle-specific functions such as `CASE WHEN`, `TO_CHAR()`, and the `dual` table.

## Attack Flow
```
Check if TrackingId is vulnerable
│
▼
Find the SQL error
│
▼
Test database queries
│
▼
Find the users table
│
▼
Check if administrator user exists
│
▼
Find password length
│
▼
Extract password characters
│
▼
Login as administrator
```

## Why This Attack Works

The application uses the `TrackingId` cookie directly in an SQL query without properly handling user input.

Because the database is Oracle, a `CASE WHEN` statement can be used to create an error only when a condition is true. In this lab, `TO_CHAR(1/0)` is used to trigger a divide-by-zero error.

When the condition is true, the application returns an error page. When the condition is false, the page loads normally.

By comparing these responses, it is possible to test SQL conditions and slowly retrieve information from the database without seeing the actual data.

