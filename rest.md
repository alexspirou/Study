
## 1) HTTP caching with ETag — how it benefits clients and how to implement
**Answer:**  
- Client gets an ETag (version hash).  
- On subsequent requests, it sends `If-None-Match`.  
- If unchanged, server returns **304 Not Modified** (no body) → faster, less bandwidth.

**Generate ETag from a list and handle conditional GET (ASP.NET Core):**
```csharp
using System.Security.Cryptography;
using System.Text;
using System.Text.Json;

[HttpGet("users/{username}/wantlist")]
public async Task<IActionResult> GetWantList(string username, int page = 1, int perPage = 10)
{
    // 1) Load data (mocked here)
    IList<WantListItem> response = await LoadWantListAsync(username, page, perPage);

    // 2) Serialize deterministically (stable order) and hash to produce weak ETag
    string json = JsonSerializer.Serialize(response, new JsonSerializerOptions
    {
        PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
        WriteIndented = false
    });

    string etag = $"W/\"{Convert.ToBase64String(MD5.HashData(Encoding.UTF8.GetBytes(json)))}\"";

    // 3) If-None-Match handling (client revalidation)
    if (Request.Headers.TryGetValue("If-None-Match", out var inm) && inm == etag)
    {
        Response.Headers.ETag = etag;
        return StatusCode(StatusCodes.Status304NotModified);
    }

    // 4) Normal 200 with ETag
    Response.Headers.ETag = etag;
    Response.Headers.CacheControl = "public, max-age=60"; // example cache policy
    return Ok(response);
}

public record WantListItem(int Id, string Title);

private static Task<IList<WantListItem>> LoadWantListAsync(string u, int p, int pp)
{
    IList<WantListItem> data = new List<WantListItem>
    {
        new(1,"Item A"), new(2,"Item B"), new(3,"Item C")
    };
    return Task.FromResult(data);
}
```


## 2) PUT vs PATCH (when and how)
**Answer:**  
- **PUT** = replace the **entire** resource. **Idempotent** by definition.  
- **PATCH** = **partial** update (only some fields). May be non-idempotent depending on semantics.  
- In ASP.NET Core, PATCH commonly uses **JSON Patch (RFC 6902)**.

**Examples:**
```csharp
// PUT: replace the whole entity
[HttpPut("users/{id}")]
public async Task<IActionResult> PutUser(Guid id, [FromBody] UserDto dto)
{
    var entity = await _db.Users.FindAsync(id);
    if (entity is null) return NotFound();

    entity.Name = dto.Name;
    entity.Email = dto.Email;
    entity.IsActive = dto.IsActive;

    await _db.SaveChangesAsync();
    return NoContent();
}

// PATCH: partial update via JSON Patch
using Microsoft.AspNetCore.JsonPatch;

[HttpPatch("users/{id}")]
public async Task<IActionResult> PatchUser(Guid id, [FromBody] JsonPatchDocument<UserDto> patch)
{
    var entity = await _db.Users.FindAsync(id);
    if (entity is null) return NotFound();

    var dto = new UserDto { Name = entity.Name, Email = entity.Email, IsActive = entity.IsActive };
    patch.ApplyTo(dto, ModelState);
    if (!ModelState.IsValid) return ValidationProblem(ModelState);

    entity.Name = dto.Name;
    entity.Email = dto.Email;
    entity.IsActive = dto.IsActive;

    await _db.SaveChangesAsync();
    return NoContent();
}

public sealed class UserDto
{
    public string Name { get; set; } = "";
    public string Email { get; set; } = "";
    public bool IsActive { get; set; }
}
```

---

## 3) Ten mid-level REST interview Q&A (concise + precise)
1. **Idempotency:** `PUT`/`DELETE` are idempotent; repeat calls produce the same final state. `POST` usually isn’t.  
2. **Safe methods:** `GET`, `HEAD`, `OPTIONS` — no state changes.  
3. **Caching:** Use `Cache-Control`, `ETag`/`If-None-Match`, and `Last-Modified`/`If-Modified-Since`.  
4. **Pagination:** Offset (`page`, `perPage`) vs Cursor (scales better, stable ordering).  
5. **Filtering/Sorting:** Query params (`?status=active&sort=-createdAt`).  
6. **Error format:** `application/problem+json` with trace IDs; accurate status codes.  
7. **Versioning:** Path (`/v1/...`) or media-type (`Accept: ...;v=2`).  
8. **Security:** TLS everywhere; OAuth2/OIDC; scopes; rate limiting; input validation.  
9. **Consistency:** Embrace eventual consistency across services; document behavior and retries.  
10. **IDs:** Prefer opaque IDs (GUID/ULID) over semantic IDs to avoid leaking internal structure.

------

## 4) OAuth2: access token vs refresh token; how to refresh; session notes
**Answer:**  
- **Access token**: short-lived; used on API calls (`Authorization: Bearer ...`).  
- **Refresh token**: long-lived; used **only** to obtain a new access token. Keep it secure.  
- **Session**: In ASP.NET Core, session is server-side (backed by store), identified by a cookie. Swagger can use it if the cookie flows.

**Refresh flow example:**
```csharp
var form = new Dictionary<string, string>
{
    ["client_id"]     = cfg.ClientId,
    ["client_secret"] = cfg.ClientSecret, // if required
    ["grant_type"]    = "refresh_token",
    ["refresh_token"] = refreshToken
};

using var resp = await http.PostAsync("https://oauth2.example.com/token",
    new FormUrlEncodedContent(form));

resp.EnsureSuccessStatusCode();

var json = await resp.Content.ReadAsStringAsync();
var tokens = JsonSerializer.Deserialize<TokenResponse>(json);

string newAccessToken = tokens!.access_token;
string? newRefreshToken = tokens.refresh_token; // sometimes rotated
```

**Saving tokens in session:**
```csharp
private void SaveTokens(TokenResponse t)
{
    HttpContext.Session.SetString("access_token", t.access_token);
    if (!string.IsNullOrEmpty(t.refresh_token))
        HttpContext.Session.SetString("refresh_token", t.refresh_token);

    var expiresAt = DateTimeOffset.UtcNow.AddSeconds(t.expires_in);
    HttpContext.Session.SetString("access_token_expires_at", expiresAt.ToUnixTimeSeconds().ToString());
}

public sealed class TokenResponse
{
    public string access_token { get; set; } = "";
    public string? refresh_token { get; set; }
    public int expires_in { get; set; }
    // ... other fields as returned by your provider
}
```

---

---

## 5) OAuth2: access token vs refresh token; how to refresh; session notes
**Answer:**  
- **Access token**: short-lived; used on API calls (`Authorization: Bearer ...`).  
- **Refresh token**: long-lived; used **only** to obtain a new access token. Keep it secure.  
- **Session**: In ASP.NET Core, session is server-side (backed by store), identified by a cookie. Swagger can use it if the cookie flows.

**Refresh flow example:**
```csharp
var form = new Dictionary<string, string>
{
    ["client_id"]     = cfg.ClientId,
    ["client_secret"] = cfg.ClientSecret, // if required
    ["grant_type"]    = "refresh_token",
    ["refresh_token"] = refreshToken
};

using var resp = await http.PostAsync("https://oauth2.example.com/token",
    new FormUrlEncodedContent(form));

resp.EnsureSuccessStatusCode();

var json = await resp.Content.ReadAsStringAsync();
var tokens = JsonSerializer.Deserialize<TokenResponse>(json);

string newAccessToken = tokens!.access_token;
string? newRefreshToken = tokens.refresh_token; // sometimes rotated
```

**Saving tokens in session:**
```csharp
private void SaveTokens(TokenResponse t)
{
    HttpContext.Session.SetString("access_token", t.access_token);
    if (!string.IsNullOrEmpty(t.refresh_token))
        HttpContext.Session.SetString("refresh_token", t.refresh_token);

    var expiresAt = DateTimeOffset.UtcNow.AddSeconds(t.expires_in);
    HttpContext.Session.SetString("access_token_expires_at", expiresAt.ToUnixTimeSeconds().ToString());
}

public sealed class TokenResponse
{
    public string access_token { get; set; } = "";
    public string? refresh_token { get; set; }
    public int expires_in { get; set; }
    // ... other fields as returned by your provider
}
```

---

---

// See caching a little bit



---
