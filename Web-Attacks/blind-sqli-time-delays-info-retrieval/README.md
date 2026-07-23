blind-sqli-time-delays-info-retrieval


## Step 1: Setting Up the Lab and Intercepting the Request

I opened the "Blind SQL injection with time delays and information retrieval" lab from PortSwigger Web Security Academy. Before touching anything, I turned on Intercept in Burp Suite's Proxy tab, then visited the lab's front page.

Once the front page loaded, Burp caught the request going out to the server. Inside this request, I could see the `Cookie` header carrying a `TrackingId` value along with a `session` value:

```bash
Cookie: TrackingId=Q3SWxllcbjggnqJ0; session=LOKQKFvZhhwXB5C847y0LYVn0qFsQrrV
```
This `TrackingId` cookie is the parameter I'll be testing for SQL injection, since it's a common spot where shopping sites track visitors using values pulled straight from a database query which makes it a good target for this kind of attack.

I forwarded this request to Burp Repeater so I could freely modify and resend it as many times as I needed without reloading the page each time.

<p align="center">
  <img src="images/step1-1.png" width="600">
</p>
<p align="center">
  <img src="images/step1-2.png" width="600">
</p>
<p align="center">
  <img src="images/step1-3.png" width="600">
</p>