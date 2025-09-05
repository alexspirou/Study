# 📚 Chapter 2 – Session Management Q&A

---

## Q26: What exactly is a session?
A session is the collection of user interactions with a website over a period of time. An interaction means a request sent to the server and a response received.  
Example: Opening a browser, logging in, adding products, checking out, and closing the browser — all these interactions together form one session.

---

## Q27: Explain "HTTP is a stateless protocol."
HTTP does not remember any state between requests. Each request–response pair is treated as a completely independent unit. The server does not retain information about previous interactions unless we explicitly maintain state.

---

## Q28: What are various ways of doing session management?
In ASP.NET Core, there are three primary ways to maintain state (i.e., make HTTP “stateful”):
1. **Session variables**  
2. **TempData**  
3. **ViewData / ViewBag**

---

## Q29: Are sessions enabled by default?
No. In ASP.NET Core, sessions are **not enabled by default**. This is intentional to keep the framework lightweight.  
(In classic ASP.NET WebForms or MVC5, sessions were enabled by default.)

---

## Q30: How to enable sessions in MVC Core?
1. In `ConfigureServices`, call **`AddSession()`**.  
2. In `Configure`, call **`UseSession()`**.  
3. Use `HttpContext.Session.Set...` to store values and `HttpContext.Session.Get...` to retrieve them.

---

## Q31: Are session variables shared (global) between users?
No. Session variables are **specific to each user**.  
Every user who visits the site gets a separate set of session variables (e.g., opening in incognito creates a fresh session).

---

## Q32: Do session variables use cookies?
Yes. By default, session variables use cookies.  
A cookie (like `.learn`) is created per website. If it is deleted, the session resets, and a new cookie is issued.

---

## Q33: What is a cookie?
A cookie is a small text file stored on the end user’s computer.  
- Cookies are specific to a website.  
- They hold small pieces of data (like session IDs).  
- They help in maintaining state across requests.

---

## Q34: Explain idle timeout in sessions.
Idle timeout defines how long a session can remain inactive before it expires.  
It is set in `services.AddSession(options => options.IdleTimeout = TimeSpan.FromSeconds(10));`.  
If no activity occurs within that time, the session is cleared/reset.

---

## Q35: What does a Context mean in HTTP?
An **HTTP context** represents **one request–response pair**.  
It contains:  
- Request data  
- Response data  
- Session variables  
- Cookies  
- Headers  
and other related information.  
Whenever we access values (like `HttpContext.Session` or `HttpContext.Request.Form`), we are working within that specific HTTP context.
