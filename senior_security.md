# Entretien .NET Senior/Expert — Sécurité ASP.NET Core

> **Versions ciblées :** .NET 8 / .NET 9 / .NET 10, C# 12 / C# 13 / C# 14
>
> **Niveaux de difficulté :** 🟢 Basique · 🟡 Intermédiaire · 🔴 Senior/Expert

---

## Authentification & Autorisation

### 🟢 Quelle est la différence entre Authentification et Autorisation en ASP.NET Core ?

| | Authentification | Autorisation |
|---|---|---|
| Question | *Qui es-tu ?* | *As-tu le droit ?* |
| Résultat | Identité (`ClaimsPrincipal`) | Accès accordé ou refusé |
| Middleware | `UseAuthentication()` | `UseAuthorization()` |
| Attribut | `[AllowAnonymous]` | `[Authorize]`, `[Authorize(Policy="...")]` |

```csharp
// Ordre obligatoire dans le pipeline
app.UseAuthentication(); // établit l'identité
app.UseAuthorization();  // vérifie les droits

// Endpoint sécurisé
app.MapGet("/api/orders", GetOrders)
    .RequireAuthorization("AdminOnly");

// Endpoint public
app.MapGet("/health", () => "OK")
    .AllowAnonymous();
```

---

### 🟢 Comment configurer l'authentification JWT Bearer dans ASP.NET Core ?

```csharp
// appsettings.json
// "Jwt": { "Key": "...", "Issuer": "https://auth.example.com", "Audience": "OrderApi" }

builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(opt =>
    {
        opt.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidIssuer = builder.Configuration["Jwt:Issuer"],
            ValidateAudience = true,
            ValidAudience = builder.Configuration["Jwt:Audience"],
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(builder.Configuration["Jwt:Key"]!)),
            ClockSkew = TimeSpan.FromSeconds(30) // tolérance d'horloge
        };

        // Événements : logging, enrichissement de contexte
        opt.Events = new JwtBearerEvents
        {
            OnAuthenticationFailed = ctx =>
            {
                logger.LogWarning("JWT auth failed: {Error}", ctx.Exception.Message);
                return Task.CompletedTask;
            }
        };
    });
```

---

### 🟡 Comment implémenter une autorisation basée sur des politiques (Policy-based) ?

```csharp
// Définition des politiques
builder.Services.AddAuthorization(opt =>
{
    // Politique simple : claim requis
    opt.AddPolicy("AdminOnly", p => p.RequireClaim("role", "admin"));

    // Politique combinée
    opt.AddPolicy("SeniorEmployee", p => p
        .RequireAuthenticatedUser()
        .RequireClaim("department", "Engineering")
        .Requirements.Add(new MinimumSeniorityRequirement(years: 3)));

    // Politique par défaut (appliquée si [Authorize] sans argument)
    opt.DefaultPolicy = new AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser()
        .Build();
});

// Requirement custom
public class MinimumSeniorityRequirement(int years) : IAuthorizationRequirement
{
    public int MinimumYears { get; } = years;
}

// Handler du requirement
public class SeniorityHandler : AuthorizationHandler<MinimumSeniorityRequirement>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext ctx, MinimumSeniorityRequirement req)
    {
        var hiredDateClaim = ctx.User.FindFirst("hired_date");
        if (hiredDateClaim is null)
        {
            ctx.Fail();
            return Task.CompletedTask;
        }

        var years = (DateTime.UtcNow - DateTime.Parse(hiredDateClaim.Value)).TotalDays / 365;
        if (years >= req.MinimumYears)
            ctx.Succeed(req);

        return Task.CompletedTask;
    }
}

// Enregistrement du handler
builder.Services.AddSingleton<IAuthorizationHandler, SeniorityHandler>();
```

---

### 🟡 Qu'est-ce que la transformation de claims (Claims Transformation) ? Quand l'utiliser ?

La transformation de claims permet d'**enrichir l'identité** de l'utilisateur après l'authentification (ajout de claims depuis une DB, cache de rôles, etc.).

```csharp
// IClaimsTransformation : appelée après chaque authentification réussie
public class UserEnrichmentTransformation(IUserRepository userRepo)
    : IClaimsTransformation
{
    public async Task<ClaimsPrincipal> TransformAsync(ClaimsPrincipal principal)
    {
        // Éviter de re-transformer si déjà enrichi
        if (principal.HasClaim(c => c.Type == "app:enriched"))
            return principal;

        var userId = principal.FindFirst(ClaimTypes.NameIdentifier)?.Value;
        if (userId is null) return principal;

        // Charger les rôles depuis la DB (mettre en cache en production !)
        var roles = await userRepo.GetRolesAsync(userId);

        var identity = new ClaimsIdentity();
        identity.AddClaims(roles.Select(r => new Claim(ClaimTypes.Role, r)));
        identity.AddClaim(new Claim("app:enriched", "true"));

        principal.AddIdentity(identity);
        return principal;
    }
}

// Enregistrement
builder.Services.AddSingleton<IClaimsTransformation, UserEnrichmentTransformation>();
```

> **Performance :** `IClaimsTransformation` est appelée à **chaque requête**. Mettre les données chargées en cache (`IMemoryCache` avec sliding expiration) pour éviter une requête DB à chaque hit.

---

### 🔴 Comment implémenter une autorisation basée sur les ressources (resource-based authorization) ?

```csharp
// Requirement : l'utilisateur ne peut modifier que ses propres commandes
public class SameOwnerRequirement : IAuthorizationRequirement { }

public class OrderOwnerHandler(IOrderRepository repo)
    : AuthorizationHandler<SameOwnerRequirement, Order>
{
    protected override async Task HandleRequirementAsync(
        AuthorizationHandlerContext ctx,
        SameOwnerRequirement req,
        Order resource)
    {
        var userId = ctx.User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
        if (userId is null) return;

        if (resource.OwnerId == userId || ctx.User.IsInRole("admin"))
            ctx.Succeed(req);
    }
}

// Utilisation dans un endpoint
public class OrdersController(IAuthorizationService authz, IOrderRepository repo)
    : ControllerBase
{
    [HttpPut("{id}")]
    public async Task<IActionResult> Update(Guid id, UpdateOrderRequest req)
    {
        var order = await repo.GetByIdAsync(id);
        if (order is null) return NotFound();

        var authResult = await authz.AuthorizeAsync(User, order, new SameOwnerRequirement());
        if (!authResult.Succeeded)
            return Forbid();

        // Mise à jour...
        return Ok();
    }
}
```

---

### 🔴 Qu'est-ce que l'API Data Protection ? Comment l'utiliser ?

L'API Data Protection de .NET Core chiffre des données de façon sécurisée (cookies, tokens anti-CSRF, tokens de reset de mot de passe).

```csharp
// Configuration : gestion des clés
builder.Services.AddDataProtection()
    .PersistKeysToAzureBlobStorage(
        new Uri("https://mystorage.blob.core.windows.net/keys/keys.xml"),
        new DefaultAzureCredential())
    .ProtectKeysWithAzureKeyVault(
        new Uri("https://myvault.vault.azure.net/keys/mykey"),
        new DefaultAzureCredential())
    .SetApplicationName("OrderService") // nécessaire si plusieurs apps partagent le storage
    .SetDefaultKeyLifetime(TimeSpan.FromDays(90));

// Utilisation : chiffrement / déchiffrement
public class TokenService(IDataProtectionProvider provider)
{
    private readonly IDataProtector _protector =
        provider.CreateProtector("TokenService.v1");

    public string Protect(string value) => _protector.Protect(value);
    public string Unprotect(string token)
    {
        try { return _protector.Unprotect(token); }
        catch (CryptographicException) { throw new InvalidTokenException(); }
    }

    // Token avec durée de vie
    public string ProtectWithExpiry(string value, TimeSpan lifetime)
    {
        var timeLimited = _protector.ToTimeLimitedDataProtector();
        return timeLimited.Protect(value, lifetime);
    }
}
```

---

### 🔴 Quelles sont les principales vulnérabilités à prévenir dans une API ASP.NET Core ?

```csharp
// 1. Injection SQL : toujours paramétrer via EF Core ou ADO.NET
// ❌ Jamais :
var sql = $"SELECT * FROM Users WHERE Name = '{userInput}'";
// ✅ EF Core :
var users = await db.Users.Where(u => u.Name == userInput).ToListAsync();
// ✅ ADO.NET :
cmd.Parameters.AddWithValue("@name", userInput);

// 2. XSS : les APIs JSON ne sont pas vulnérables au XSS,
//    mais valider/sanitiser les inputs côté MVC / Razor

// 3. CORS : configurer strictement
builder.Services.AddCors(opt => opt.AddPolicy("ApiPolicy", p => p
    .WithOrigins("https://app.example.com") // pas de wildcard en production
    .WithMethods("GET", "POST", "PUT", "DELETE")
    .AllowCredentials()));

// 4. Exposition de stack traces
builder.Services.AddProblemDetails(); // masque les détails en production
// OU
if (!env.IsDevelopment())
    app.UseExceptionHandler(); // supprime les stacktraces dans les réponses

// 5. HTTPS forcé
app.UseHttpsRedirection();
app.UseHsts(); // HTTP Strict Transport Security

// 6. Headers de sécurité
app.Use(async (ctx, next) =>
{
    ctx.Response.Headers.XContentTypeOptions = "nosniff";
    ctx.Response.Headers.XFrameOptions = "DENY";
    ctx.Response.Headers["X-Permitted-Cross-Domain-Policies"] = "none";
    await next(ctx);
});

// 7. Rate limiting (voir senior_middleware_pipeline.md)

// 8. Validation des inputs : FluentValidation ou DataAnnotations sur tous les DTOs
```

---

### 🔴 Comment gérer plusieurs schémas d'authentification simultanément ?

```csharp
// Scénario : API consommée par des clients internes (ApiKey) et externes (JWT)
builder.Services
    .AddAuthentication()
    .AddJwtBearer("External", opt => { /* ... */ })
    .AddScheme<ApiKeyAuthOptions, ApiKeyAuthHandler>("ApiKey", _ => { });

// Handler API Key
public class ApiKeyAuthHandler(
    IOptionsMonitor<ApiKeyAuthOptions> options,
    IApiKeyValidator validator,
    ILoggerFactory logger, UrlEncoder encoder)
    : AuthenticationHandler<ApiKeyAuthOptions>(options, logger, encoder)
{
    protected override async Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        if (!Request.Headers.TryGetValue("X-Api-Key", out var key))
            return AuthenticateResult.NoResult();

        var clientId = await validator.ValidateAsync(key.ToString());
        if (clientId is null)
            return AuthenticateResult.Fail("Invalid API key");

        var claims = new[] { new Claim(ClaimTypes.NameIdentifier, clientId) };
        var identity = new ClaimsIdentity(claims, Scheme.Name);
        return AuthenticateResult.Success(new AuthenticationTicket(
            new ClaimsPrincipal(identity), Scheme.Name));
    }
}

// Politique multi-schéma
builder.Services.AddAuthorization(opt =>
{
    opt.AddPolicy("AnyAuthScheme", p => p
        .AddAuthenticationSchemes("External", "ApiKey")
        .RequireAuthenticatedUser());
});
```

---

## Lexique

| Terme | Définition |
|---|---|
| **Authentification** | Processus établissant l'identité d'un utilisateur (qui es-tu ?). |
| **Autorisation** | Processus vérifiant les droits d'accès d'une identité authentifiée (as-tu le droit ?). |
| **Claim** | Paire clé/valeur décrivant un attribut d'une identité (rôle, email, identifiant). |
| **ClaimsPrincipal** | Conteneur des identités (`ClaimsIdentity`) d'un utilisateur, accessible via `HttpContext.User`. |
| **CORS** | Cross-Origin Resource Sharing : mécanisme autorisant les requêtes cross-domain selon une politique. |
| **Data Protection API** | API .NET Core de chiffrement de données (cookies, tokens) avec rotation de clés automatique. |
| **HSTS** | HTTP Strict Transport Security : instruit le navigateur d'utiliser HTTPS uniquement. |
| **IAuthorizationHandler** | Interface implémentant la logique d'un requirement d'autorisation. |
| **IClaimsTransformation** | Interface enrichissant le `ClaimsPrincipal` après authentification. |
| **JWT** | JSON Web Token : token signé (et optionnellement chiffré) encodant des claims. |
| **Policy** | Ensemble de requirements d'autorisation nommé et enregistré dans `AddAuthorization`. |
| **Requirement** | Condition d'autorisation évaluée par un `IAuthorizationHandler`. |
| **Resource-based authorization** | Autorisation tenant compte de la ressource spécifique accédée en plus de l'identité. |
| **Scheme** | Schéma d'authentification nommé (JWT Bearer, Cookies, ApiKey...) gérant un mécanisme d'auth. |

---

## Ressources

### Documentation officielle Microsoft
- [Authentication in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/security/authentication/)
- [Authorization in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/security/authorization/introduction)
- [Policy-based authorization](https://learn.microsoft.com/en-us/aspnet/core/security/authorization/policies)
- [Resource-based authorization](https://learn.microsoft.com/en-us/aspnet/core/security/authorization/resourcebased)
- [Data Protection in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/security/data-protection/introduction)
- [JWT Bearer authentication](https://learn.microsoft.com/en-us/aspnet/core/security/authentication/jwt-authn)

### Livres
- *ASP.NET Core Security* — Christian Wenz
- *ASP.NET Core in Action* — Andrew Lock

### Blogs & Articles
- [Andrew Lock — Security in ASP.NET Core](https://andrewlock.net/)
- [OWASP Top 10 for .NET](https://owasp.org/www-project-top-ten/)
