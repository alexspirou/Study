# ASP.NET Security — Q&A (66–94)

## 66) How does token-based authentication work?
It’s a two-step process:  
1) The client sends a username/email and password to the server and receives a token.  
2) For later requests, the client sends **only the token**; the server validates the token and returns the resource. No credentials are re-sent.

## 67) Why is it called a JWT token?
Because it uses **JSON Web Token (JWT)**—a lightweight JSON format (name/value pairs in `{}`) for representing the token data.

## 68) Explain the 3 sections of a JWT.
A JWT is `header.payload.signature` (three Base64URL-encoded parts separated by dots):  
- **Header:** algorithm/metadata (e.g., HMAC, RSA).  
- **Payload:** identity and related claims (e.g., username, flags like `isAdmin`).  
- **Signature:** created from header + payload + secret key; the server verifies this to detect tampering.

## 69) What are Identity and claims?
- **Identity:** the data that uniquely identifies the user (e.g., email/username).  
- **Claims:** statements about the user (permissions/attributes), such as “is admin,” “can view reports,” or other subject info (country, age).

## 70) Authentication vs Authorization?
- **Authentication:** verifying **who** the user is (e.g., “this is Shiv”).  
- **Authorization:** determining **what** the user can do (roles/permissions).

## 71) Claims vs Roles?
- **Roles:** classify user types and attach permissions (e.g., Admin, ReadOnly).  
- **Claims:** can include roles **and** other subject-specific details (e.g., “gold customer,” “country=India”). Claims are per-user/context; roles are user-type centric.

## 72) Principal vs Identity
- **Identity:** represents the user and holds their roles/claims.  
- **Principal (ClaimsPrincipal):** encapsulates the Identity (and its roles/claims) and can be assigned to an execution context/thread to represent the current security context.

## 73) Can we put critical information in a JWT?
No. The header and payload are **encoded, not encrypted** (easily decoded as Base64). Do **not** place sensitive data (like passwords) in the payload.

## 74) How do you create a JWT in ASP.NET Core MVC?
- Add the NuGet package: `Microsoft.AspNetCore.Authentication.JwtBearer`.  
- Choose an algorithm (e.g., HMAC-SHA256).  
- Build a **claims collection** (subject, email, flags).  
- Use a **secret key** with the algorithm to generate the token.  
- Only generate the token after validating credentials (DB check).

## 75) What HTTP status code for unauthorized access?
**401 Unauthorized** (not 500 or 404).

## 76) Where is the token checked in ASP.NET Core MVC?
In the **authentication middleware**, configured via `services.AddAuthentication().AddJwtBearer(...)` using the **same secret key** that signed the token.

## 77) What is the use of the `[Authorize]` attribute?
It protects controllers/actions by enforcing authentication/authorization (token validation). Only decorated endpoints are checked.

## 78) How did you implement JWT token security?
- **Create** the token (package, algorithm, claims, secret).  
- **Validate** the token in **middleware** (`AddJwtBearer` with the same key).  
- **Protect** endpoints with `[Authorize]`.

## 79) How do we send tokens from the client side?
In the HTTP **Authorization** header as:  
`Authorization: Bearer <token>`

## 80) From JavaScript/jQuery/Angular, how is the token passed?
Add the header in each request, e.g.:  
- Fetch/JS: `headers: { Authorization: 'Bearer <token>' }`  
- jQuery: `beforeSend: xhr => xhr.setRequestHeader('Authorization','Bearer <token>')`  
- Angular: set `Authorization: Bearer <token>` in `HttpHeaders` (often via an interceptor).

## 81) Improve mobile UX to avoid re-login?
Use **refresh tokens** instead of making access tokens last for months.

## 82) What are refresh tokens?
Separate, **long-lived** tokens used to obtain **new** JWT access tokens without forcing the user to log in again.

## 83) How does a refresh token work?
- Server issues **access token + refresh token** after login.  
- Client uses the access token until it expires.  
- On expiry, client sends the **refresh token** to get a **new access token (and new refresh token)**, continuing seamlessly.

## 84) Access tokens vs Refresh tokens?
- **Access token (JWT):** short-lived; used to access APIs.  
- **Refresh token:** longer-lived; used only to obtain new access tokens.

## 85) Whose expiry is longer—access or refresh?
**Refresh tokens** have a longer lifetime; access tokens should be short-lived.

## 86) Explain revocation of a refresh token.
Invalidate the current refresh token (and related access token) and issue a **new** refresh token (and access token), e.g., upon abnormal behavior.

## 87) How to extract a Principal from a token?
Use `JwtSecurityTokenHandler.ValidateToken(...)` with the token and validation parameters; it returns a **ClaimsPrincipal** (the principal) and the validated token.

## 88) Best practice to store tokens on the client?
- **Prefer In-Memory** variables (most secure; gone on app close).  
- If persistence is needed, use a **cookie** and mark it **HttpOnly**.  
- Avoid `localStorage`, `sessionStorage`, and IndexedDB due to XSS risk.  
- Some apps place tokens only in **headers** per request.

## 89) If we store JWT in a cookie, how do we mitigate XSS?
Make the cookie **HttpOnly** (so JS can’t read it). *(Your source text stresses HttpOnly specifically.)*

## 90) What are OAuth and OpenID (and OpenID Connect)?
They are **protocols**:  
- **OpenID:** authentication (who the user is).  
- **OAuth:** authorization (what access the client/app has).  
- **OpenID Connect (OIDC):** adds identity on top of OAuth—**both** who and what.  
*JWT is just the token format used to carry this info.*

## 91) When should we use what?
- **OpenID:** when you only need authentication (user identity).  
- **OAuth:** when you only need authorization (scoped rights).  
- **OpenID Connect:** when you need **both** identity and authorization (common in real apps).

## 92) What is IdentityServer?
An **OpenID Connect** (and OAuth) framework you can deploy as your own **identity provider** (IdP). It implements these protocols (certified by the OpenID Foundation) so your apps can authenticate and authorize against a central server.

## 93) How to achieve single sign-on (SSO)?
Use a **central identity/authorization server** (e.g., IdentityServer). Each site redirects to it for login; once authenticated, the user can access the group of sites without re-logging in.

## 94) What are scopes in IdentityServer?
Scopes represent **authorization scopes/roles** (e.g., read, write). The client requests specific scopes; tokens are issued accordingly and enforced by the resource server.
