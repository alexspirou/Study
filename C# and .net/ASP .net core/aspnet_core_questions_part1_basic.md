# 📚 ASP.NET MVC Core – Interview Questions (Part 1)

---

### **1. What is ASP .NET MVC Core?**  
ASP.NET MVC Core is an **open-source, cross-platform framework by Microsoft** for building web apps. It runs on **Windows, Linux, and macOS**, is **lightweight**, and has **high performance** compared to earlier ASP.NET versions.

---

### **2. Differentiate between ASP .NET WebForms vs MVC vs MVC Core**
- **WebForms (2003):** Windows-only, heavy (ViewState, Page Life Cycle), poor performance, limited HTML control.  
- **MVC (MVC 5):** Windows-only, better separation of concerns, no ViewState, medium performance, Web.config is complex.  
- **MVC Core (2016+):** Cross-platform, cloud-ready, lightweight DLLs, simple `appsettings.json`, built-in DI, best performance.

**Key differences:**  
Cross-platform ✅, Performance ✅, Simplification ✅, Cloud readiness ✅.

---

### **3. Explain MVC Architecture**
- **Model:** Domain/business logic (Customer, Supplier, Payroll).  
- **View:** UI (HTML, CSS, Razor).  
- **Controller:** Connects Model and View, handles requests, sends responses.  
➡ Separation improves maintainability.

---

### **4. Why do we have wwwroot folder?**
`wwwroot` is the **special folder for static content** (CSS, JS, images, HTML). Anything here is directly served to the client.

---

### **5. Explain the importance of appsettings.json**
- Stores **configuration data** like connection strings, keys, version numbers.  
- **Simple JSON (key-value)** compared to heavy `web.config` from MVC5.

---

### **6. How to read configurations from appsettings.json**
- Inject **`IConfiguration`** into the constructor.  
- Use `configuration["Key"]` or `configuration["Section:SubKey"]` for nested JSON.  
- `IConfiguration` is automatically dependency-injected by Core.

---

### **7. What is dependency injection (DI)?**
DI is the practice of **providing dependent objects from outside** instead of creating them inside a class. Example: inject `CustomerDbContext` via constructor instead of `new`.

---

### **8. Why do we need dependency injection?**
- Removes **tight coupling**  
- Easier to **swap implementations** (e.g., DbContext v1 → DbContext v2)  
- Improves **testability** and **maintainability**  
➡ Leads to a **decoupled architecture**.

---

### **9. How do we implement dependency injection?**
- In `Startup.cs → ConfigureServices()`  
- Register with `services.AddScoped<T>()`, `services.AddTransient<T>()`, or `services.AddSingleton<T>()`.  
- Framework will inject automatically wherever needed.

---

### **10. What is the use of Middleware?**
Middleware allows inserting **pre-processing logic in the request pipeline** before requests hit controllers (e.g., logging, authentication, security).

---

### **11. How to create a Middleware?**
1. Add a **middleware class** with `Invoke()` method.  
2. Write your logic in `Invoke()`.  
3. Register in pipeline via `app.UseMiddleware<YourMiddleware>()` inside `Configure()`.

---

### **12. What does Startup.cs file do?**
- Configures **Dependency Injection** (in `ConfigureServices`)  
- Configures **Middleware pipeline** (in `Configure`)  

---

### **13. ConfigureServices vs Configure method**
- **ConfigureServices:** Register dependencies (DI).  
- **Configure:** Register middlewares (request pipeline).  

---

### **14. Explain the different Ways of doing DI**
- **AddScoped:** One instance per HTTP request.  
- **AddTransient:** New instance every time requested.  
- **AddSingleton:** Single shared instance for the whole app.

---

### **15. Explain Scoped vs Transient vs Singleton**
- **Scoped:** One instance per request.  
- **Transient:** New instance each time injected.  
- **Singleton:** Same instance shared across entire application.

---

### **16. What is Razor?**
Razor is a **view engine & markup syntax** that lets you mix **C# code + HTML** inside views. Syntax starts with `@`.

---

### **17. How to pass Model data to a View?**
Use `return View("ViewName", modelObject);`

---

### **18. What is the use of Strongly typed views?**
Strongly typed views provide **IntelliSense support** in Razor pages by defining `@model TypeName` at the top. Helps catch errors at compile time.

---

### **19. Explain the concept of ViewModel in MVC**
- A **wrapper class** that combines multiple models into one object for a View.  
- Example: `CustomerViewModel` contains both `Customer` and `Products`.

---

### **20. What is Kestrel Web Server?**
Kestrel is the **default lightweight web server** for ASP.NET Core. Cross-platform, open source, fast, and included in ASP.NET Core setup.

---

### **21. Why Kestrel when we have IIS server?**
- IIS runs **only on Windows**.  
- ASP.NET Core is **cross-platform**, so Kestrel is required.  
- In production: Kestrel is usually combined with IIS, Apache, or Nginx as **reverse proxy**.

---

### **22. What is the concept of Reverse proxy?**
- Request first hits **IIS/Apache/Nginx** (strong, secure server).  
- Then forwarded to **Kestrel**, which runs the ASP.NET Core app.  
- Response goes back through proxy.  
➡ Combines security + performance.

---

### **23. What are cookies?**
- **Small text files** stored on the client’s machine by a website.  
- Store user-specific data like preferences, login sessions, etc.

---

### **24. What is the need for session management?**
- HTTP is **stateless** (doesn’t remember users).  
- To **maintain state across requests** (user login, shopping cart), we need **session management**.

---

### **25. What are the various ways of doing Session management in ASP.NET?**
1. **Session Variables** (`HttpContext.Session`)  
2. **ViewData / ViewBag** (pass data between controller & view in one request)  
3. **TempData** (persists data between requests until read)

---
