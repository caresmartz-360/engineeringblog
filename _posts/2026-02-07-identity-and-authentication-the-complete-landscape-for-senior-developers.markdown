---
layout: default
title: "Identity and Authentication: The Complete Landscape for Senior Developers"
date: 2026-02-07 10:00:00 +0530
categories: engineering security authentication identity
---

If you've ever gotten confused between OAuth, OIDC, SSO, and all the other auth acronyms, you're not alone. Even senior developers mix these up because they overlap in practice but have distinct purposes.

This post breaks down the full identity and authentication landscape. No fluff. Just the core concepts you need to understand to build secure, modern applications.

## Why This Matters

Here's the problem: developers often implement auth without understanding the layers. They copy code from Stack Overflow, use a library, and hope it works. Until something breaks or a security audit happens.

Understanding the complete picture helps you:
- Debug auth issues faster
- Choose the right flow for your architecture
- Communicate clearly with security teams
- Build distributed systems correctly

Let's break it down layer by layer.

## 1. Core Protocol Layer: The Standards

These are the actual protocols and specifications. Everything else builds on these.

### OAuth 2.0

**What it is:** An authorization framework.

**What it does:** Issues access tokens that grant permission to access resources.

**The question it answers:** "Can this app call this API?"

**Key point:** OAuth 2.0 is NOT for authentication. It's for authorization. It doesn't tell you who the user is—only what they can access.

**Example scenario:** You build a mobile app that needs to access user photos stored in your API. OAuth 2.0 lets the app get a token to call that API without exposing the user's password.

```csharp
// The app gets a token
string accessToken = "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...";

// Uses it to call your API
var request = new HttpRequestMessage(HttpMethod.Get, "https://api.example.com/photos");
request.Headers.Authorization = new AuthenticationHeaderValue("Bearer", accessToken);
```

### OpenID Connect (OIDC)

**What it is:** A protocol built on top of OAuth 2.0.

**What it adds:** Identity information.

**What it does:** Issues an ID token that tells you who the user is.

**The question it answers:** "Who is this user?"

**Key point:** OIDC extends OAuth 2.0 by adding a standardized way to get user information. This is what you actually use for authentication.

**Example scenario:** After the user logs in, you get an ID token with their name, email, and user ID. You can trust this information because it's signed by the identity provider.

```csharp
// You get an ID token that contains user claims
{
  "sub": "248289761001",
  "name": "John Doe",
  "email": "john@example.com",
  "email_verified": true
}
```

**The relationship:** OAuth 2.0 handles the "what can you do" part. OIDC handles the "who are you" part. OIDC uses OAuth 2.0's flows but adds identity information on top.

## 2. Authentication & Session Concepts

This is where the confusion starts. SSO is not a protocol—it's a user experience.

### SSO (Single Sign-On)

**What it is:** A behavior, not a protocol.

**What it means:** Login once, access multiple applications without re-entering credentials.

**How it's achieved:** Using protocols like OIDC, SAML, or sometimes just cookies.

**Key point:** When you log in to Google and then access Gmail, YouTube, and Google Drive without logging in again—that's SSO. The protocol doing the work underneath is likely OIDC.

**Example scenario:** Your company has 10 internal apps. Users log in once to your corporate identity provider (like Entra ID), and they can access all 10 apps without logging in again.

**Behind the scenes:**
```csharp
// User logs in once to IdP
// Each app redirects to the IdP
// IdP checks: "Is this user already authenticated?"
// If yes, issues tokens immediately without asking for credentials again
```

**Common misconception:** SSO is not the same as OAuth or OIDC. SSO is what the user experiences. OAuth/OIDC is how you implement it.

## 3. Identity Provider (IdP): The Authority

**What it is:** The system that authenticates users and issues tokens.

**Examples:**
- Microsoft Entra ID (formerly Azure AD)
- Auth0
- Keycloak
- Okta
- AWS Cognito

**Key point:** If you're using Entra ID for login, Entra ID is your IdP. Your application trusts the IdP to authenticate users correctly.

**What it does:**
1. Authenticates users (checks username/password, MFA, etc.)
2. Issues tokens (ID tokens, access tokens)
3. Manages user directories
4. Enforces security policies

**Example flow:**
```
User → Your App → Redirects to IdP → User logs in → IdP sends tokens → Your App
```

**In code, you configure your app to trust a specific IdP:**
```csharp
services.AddAuthentication(OpenIdConnectDefaults.AuthenticationScheme)
    .AddOpenIdConnect(options =>
    {
        options.Authority = "https://login.microsoftonline.com/{tenant-id}";
        options.ClientId = "your-client-id";
        options.ClientSecret = "your-client-secret";
    });
```

## 4. Token Types: The Missing Piece

This is where most developers get lost. Not all tokens are the same.

| Token Type | Purpose | Where Used | Can Be Decoded? |
|------------|---------|------------|-----------------|
| **ID Token** | Contains user identity info | Authentication | Yes (JWT) |
| **Access Token** | Authorizes API calls | Authorization | Maybe (can be opaque) |
| **Refresh Token** | Gets new access tokens | Token renewal | No (opaque) |
| **Authorization Code** | Temporary code exchanged for tokens | Initial flow | No (short-lived code) |

### ID Token

**Purpose:** Tells you who the user is.

**Format:** Always a JWT (JSON Web Token).

**Contains:** User claims like name, email, sub (subject/user ID).

**Where you use it:** In your app to identify the logged-in user.

```csharp
// Decode and read claims from ID token
var handler = new JwtSecurityTokenHandler();
var token = handler.ReadJwtToken(idToken);
var userId = token.Claims.First(c => c.Type == "sub").Value;
var email = token.Claims.First(c => c.Type == "email").Value;
```

### Access Token

**Purpose:** Grants permission to call APIs.

**Format:** Can be JWT or opaque (random string).

**Contains:** Scopes, permissions, expiration.

**Where you use it:** Sent to APIs in the Authorization header.

```csharp
// API validates the access token
[Authorize]
[HttpGet("photos")]
public IActionResult GetPhotos()
{
    // This endpoint requires a valid access token
    var userId = User.FindFirst("sub")?.Value;
    return Ok(photoService.GetUserPhotos(userId));
}
```

### Refresh Token

**Purpose:** Gets new access tokens without requiring the user to log in again.

**Format:** Opaque string (you never decode it).

**Lifetime:** Long (days, weeks, months).

**Why it exists:** Access tokens expire quickly (minutes to hours). Refresh tokens let you get new ones securely.

```csharp
// When access token expires, use refresh token
var tokenResponse = await httpClient.PostAsync("https://idp.example.com/token", new FormUrlEncodedContent(new[]
{
    new KeyValuePair<string, string>("grant_type", "refresh_token"),
    new KeyValuePair<string, string>("refresh_token", refreshToken),
    new KeyValuePair<string, string>("client_id", clientId),
    new KeyValuePair<string, string>("client_secret", clientSecret)
}));
```

### Authorization Code

**Purpose:** Temporary code exchanged for tokens during login.

**Lifetime:** Very short (30-60 seconds).

**Why it exists:** Security. Instead of sending tokens directly to your browser, the IdP sends a code. Your backend exchanges the code for tokens securely.

**The flow:**
```
1. User clicks "Login"
2. Redirected to IdP
3. User logs in
4. IdP redirects back with authorization code
5. Your backend exchanges code for tokens
6. Tokens stored securely
```

**Key point:** Never store authorization codes. They're single-use and short-lived.

## 5. Grant Types: Critical for Distributed Systems

OAuth 2.0 defines different "flows" called grant types. Each solves a specific use case.

### Authorization Code Flow

**Use case:** Web applications with a backend.

**How it works:**
1. User clicks login
2. Redirected to IdP
3. IdP sends back an authorization code
4. Backend exchanges code for tokens
5. Tokens stored securely (never exposed to browser)

**Why this is secure:** Tokens never pass through the browser. Only the authorization code does, and it's useless without your client secret.

```csharp
// This happens in your backend after receiving the code
var tokenResponse = await httpClient.PostAsync("https://idp.example.com/token", new FormUrlEncodedContent(new[]
{
    new KeyValuePair<string, string>("grant_type", "authorization_code"),
    new KeyValuePair<string, string>("code", authorizationCode),
    new KeyValuePair<string, string>("redirect_uri", redirectUri),
    new KeyValuePair<string, string>("client_id", clientId),
    new KeyValuePair<string, string>("client_secret", clientSecret)
}));
```

**When to use:** Almost always for web apps where users log in.

### Client Credentials Flow

**Use case:** Machine-to-machine communication (no user involved).

**How it works:**
1. Service A needs to call Service B
2. Service A authenticates with IdP using client ID and secret
3. Gets access token
4. Calls Service B with token

**Key point:** This is critical for microservices and distributed systems. No user is involved—it's purely service-to-service.

```csharp
// Service A gets a token to call Service B
var tokenResponse = await httpClient.PostAsync("https://idp.example.com/token", new FormUrlEncodedContent(new[]
{
    new KeyValuePair<string, string>("grant_type", "client_credentials"),
    new KeyValuePair<string, string>("client_id", serviceAClientId),
    new KeyValuePair<string, string>("client_secret", serviceAClientSecret),
    new KeyValuePair<string, string>("scope", "api://serviceB/.default")
}));

var accessToken = tokenResponse.AccessToken;

// Use token to call Service B
var request = new HttpRequestMessage(HttpMethod.Get, "https://serviceb.example.com/api/data");
request.Headers.Authorization = new AuthenticationHeaderValue("Bearer", accessToken);
```

**When to use:** Background jobs, scheduled tasks, service-to-service calls, daemon applications.

### Device Code Flow

**Use case:** Devices without browsers (smart TVs, IoT devices, CLI tools).

**How it works:**
1. Device shows a code and URL
2. User visits URL on another device (phone, laptop)
3. User enters code and logs in
4. Device gets tokens

**Example:** Netflix on your smart TV shows a code. You go to netflix.com/activate on your phone and enter the code.

**When to use:** CLI tools, smart TVs, IoT devices.

### Refresh Token Flow

**Use case:** Renewing expired access tokens.

**How it works:**
1. Access token expires
2. App uses refresh token to get new access token
3. No user interaction needed

**When to use:** Any app that needs long-running access without asking users to log in repeatedly.

## Common Architecture Patterns

### Frontend + Backend API

**Pattern:**
- Frontend (SPA, mobile app): Gets ID token for user identity
- Backend API: Validates access token from frontend

```csharp
// Frontend stores both tokens
localStorage.setItem('id_token', idToken);        // For showing user info
localStorage.setItem('access_token', accessToken); // For API calls

// Backend validates access token
[Authorize]
public class MyController : ControllerBase
{
    // Token validation happens automatically via middleware
}
```

### Microservices

**Pattern:**
- API Gateway: Validates user token
- Internal services: Use client credentials for service-to-service calls

```csharp
// API Gateway validates user token
[Authorize]
public async Task<IActionResult> GetUserData()
{
    // Get token for calling internal service
    var serviceToken = await GetServiceToken();
    
    // Call internal service
    var response = await internalServiceClient.GetDataAsync(serviceToken);
    return Ok(response);
}

private async Task<string> GetServiceToken()
{
    // Client credentials flow
    // Returns service-to-service token
}
```

### Legacy Application + Modern IdP

**Pattern:**
- Reverse proxy handles OIDC
- Passes user info to legacy app via headers

```
User → Reverse Proxy (OIDC) → Sets headers → Legacy App
                                 X-User-Id: 12345
                                 X-User-Email: user@example.com
```

## What Most Developers Get Wrong

### Mistake 1: Using ID Tokens for API Authorization

**Wrong:**
```csharp
// Never do this
request.Headers.Authorization = new AuthenticationHeaderValue("Bearer", idToken);
```

**Right:**
```csharp
// Use access token for APIs
request.Headers.Authorization = new AuthenticationHeaderValue("Bearer", accessToken);
```

**Why:** ID tokens are for identity. Access tokens are for authorization. Mixing them up creates security issues.

### Mistake 2: Not Understanding Token Expiration

**Problem:** Access tokens expire (usually 1 hour). Apps break when tokens expire.

**Solution:** Implement token refresh logic.

```csharp
public async Task<string> GetValidAccessToken()
{
    if (IsAccessTokenExpired())
    {
        // Use refresh token to get new access token
        await RefreshAccessToken();
    }
    return accessToken;
}
```

### Mistake 3: Storing Tokens in localStorage Without Consideration

**Risk:** XSS attacks can steal tokens from localStorage.

**Better approach for web apps:**
- Store refresh token in HTTP-only cookie
- Keep access token in memory
- Use short-lived access tokens

### Mistake 4: Using Password Grant Flow

**Problem:** This flow is deprecated in OAuth 2.1. It requires apps to handle user passwords directly.

**Solution:** Use Authorization Code Flow with PKCE instead.

## Quick Decision Tree

**"How should I implement auth?"**

1. **User-facing web app?** → Authorization Code Flow + OIDC
2. **Mobile app?** → Authorization Code Flow with PKCE + OIDC
3. **SPA (React, Angular, Vue)?** → Authorization Code Flow with PKCE + OIDC
4. **Service-to-service?** → Client Credentials Flow
5. **CLI tool?** → Device Code Flow
6. **Smart TV / IoT?** → Device Code Flow

## Final Thoughts

The identity and auth landscape isn't as complicated as it seems once you understand the layers:

**Protocols:** OAuth 2.0 (authorization) + OIDC (identity)

**Behavior:** SSO (login once, access many)

**Authority:** IdP (issues tokens, authenticates users)

**Token types:** ID token (identity), Access token (authorization), Refresh token (renewal)

**Flows:** Authorization Code (users), Client Credentials (services), Device Code (devices)

Once you map your architecture to these components, implementation becomes straightforward.

The key is understanding what each piece does and when to use it. Don't just copy code—understand the flow. That's what separates senior developers from those who struggle when auth breaks.

If you're building distributed systems, master Client Credentials flow. If you're building user-facing apps, master Authorization Code flow. Everything else follows from there.

