blind-sqli-time-delays-info-retrieval


## Step 1: Setting Up the Lab and Intercepting the Request

I opened the "Blind SQL injection with time delays and information retrieval" lab from PortSwigger Web Security Academy. Before touching anything, I turned on Intercept in Burp Suite's Proxy tab, then visited the lab's front page.

Once the front page loaded, Burp caught the request going out to the server. Inside this request, I could see the `Cookie` header carrying a `TrackingId` value along with a `session` value:

```bash
Cookie: TrackingId=Q3SWxllcbjggnqJ0; session=LOKQKFvZhhwXB5C847y0LYVn0qFsQrrV
```
I picked `TrackingId` as my target because shopping sites often use cookies like this to track visitors, and the value usually comes straight from a database lookup — which makes it worth testing for SQL injection.

I sent this request to Burp Repeater so I could edit and resend it as many times as I wanted, without reloading the page every time.

<p align="center">
  <img src="images/step1-1.png" width="600">
</p>
<p align="center">
  <img src="images/step1-2.png" width="600">
</p>
<p align="center">
  <img src="images/step1-3.png" width="600">
</p>

## Step 2: Confirming the SQL Injection Point via Time Delay

Inside Burp Repeater, I changed the `TrackingId` cookie value to this payload, to see if the server would actually run SQL code I put in it:

```bash
Q3SWxllcbjggnqJ0' %3BSELECT CASE WHEN (1=1) THEN pg_sleep(5) ELSE pg_sleep(0) END --
```
**Breakdown**

| Part                          | Description                                                    |
|----------------------------------|-------------------------------------------------------------------|
| `Q3SWxllcbjggnqJ0'`                    | Random tracking string followed by a single quote to close the original string value so my own SQL can start right after |
| `%3B`                             | URL-encoded semicolon (`;`) — ends the current statement and starts a new one |
| `SELECT CASE WHEN (1=1)`          | A condition that's always true, just to check if my code runs at all |
| `THEN pg_sleep(5)`                | If the condition is true, make PostgreSQL wait 5 seconds          |
| `ELSE pg_sleep(0)`                | If false, don't wait at all                                       |
| `END --`                          | Closes the CASE block and comments out the rest of the original query |


I clicked Send, and the response came back after **5,302 ms** — about 5 seconds, just like I told it to wait. This showed me the cookie value was going directly into a SQL query on the backend, and the server was running whatever I put there. This confirmed the `TrackingId` cookie was vulnerable to **time-based blind SQL injection**.

<p align="center">
  <img src="images/step2-1.png" width="600">
</p>