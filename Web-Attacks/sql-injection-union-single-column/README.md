# SQL injection UNION attack, retrieving multiple values in a single column

## Table of Contents

- [Introduction](#introduction)
- [Attack Flow](#attack-flow)
- [Why This Attack Works](#why-this-attack-works)
- [Lab Setup](#lab-setup)
- [Tools Used](#tools-used)
- [Prerequisites](#prerequisites)
- [Step 1 - Finding the Number of Columns](#step-1---finding-the-number-of-columns)
- [Step 2 - Finding a Column That Accepts Text](#step-2---finding-a-column-that-accepts-text)
- [Step 3 - Combining the Username and Password into One Column](#step-3---combining-the-username-and-password-into-one-column)
- [Step 4 - Retrieving Usernames and Passwords](#step-4---retrieving-usernames-and-passwords)
- [Step 5 - Logging in as Administrator](#step-5---logging-in-as-administrator)
- [How to Detect This Attack](#how-to-detect-this-attack)
- [How to Prevent It](#how-to-prevent-it)
- [Lessons Learned](#lessons-learned)
- [References](#references)