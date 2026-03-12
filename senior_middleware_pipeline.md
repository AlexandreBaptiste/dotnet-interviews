# Entretien .NET Senior/Expert — Middleware & Pipeline ASP.NET Core

> **Versions ciblées :** .NET 8 / .NET 9 / .NET 10, C# 12 / C# 13 / C# 14
>
> **Niveaux de difficulté :** 🟢 Basique · 🟡 Intermédiaire · 🔴 Senior/Expert

---

## Pipeline de requêtes

### 🟢 Qu'est-ce qu'un middleware en ASP.NET Core ? Comment fonctionne le pipeline de requêtes ?

Le middleware est un composant qui traite les requêtes et réponses HTTP. Les middlewares sont chaînés en pipeline : chacun peut exécuter du code **avant** et/ou **après** le suivant via un délégué `next`.

```csharp
app.Use(async (context, next) =>
{
    // Code AVANT le middleware suivant
    Console.WriteLine($"→ {context.Request.Method} {context.Request.Path}");

    await next(context); // délègue au suivant

    // Code APRÈS le retour du pipeline
    Console.WriteLine($"← {context.Response.StatusCode}");
});
```

**Flux :**
```
Client → MW1 → MW2 → MW3 → Endpoint (terminal)
Client ← MW1 ← MW2 ← MW3 ←
```

---

### 🟢 Quelle est la différence entre `app.Use()`, `app.Run()` et `app.Map()` ?

| Méthode | Court-circuite | Usage |
|---|---|---|
| `Use` | Non — appelle `next` | Middleware intermédiaire |
| `Run` | Oui — terminal, pas de `next` | Dernier middleware de la branche |
| `Map` | Oui — branche selon le path | Routing conditionnel par préfixe |
| `MapWhen` | Oui — branche selon prédicat | Branching sur condition arbitraire |

```csharp
// Use : intermédiaire
app.Use(async (ctx, next) =>
{
    ctx.Response.Headers.Append("X-Trace-Id", Guid.NewGuid().ToString());
    await next(ctx);
});

// Run : terminal — aucun middleware suivant n'est appelé
app.Run(async ctx => await ctx.Response.WriteAsync("Hello"));

// Map : branche sur /api
app.Map("/api", api =>
{
    api.Run(async ctx => await ctx.Response.WriteAsync("API"));
});

// MapWhen : branche conditionnelle
app.MapWhen(ctx => ctx.Request.Headers.ContainsKey("X-Debug"), debug =>
{
    debug.Run(async ctx => await ctx.Response.WriteAsync("Debug mode"));
});
```

---

### 🟡 Comment créer un middleware personnalisé avec une classe dédiée ? Convention-based vs `IMiddleware` ?

```csharp
// Convention-based : instanciation unique (Singleton-like)
// Les dépendances du constructeur doivent être Singleton
public class RequestTimingMiddleware(
    RequestDelegate next,
    ILogger<RequestTimingMiddleware> logger)
{
    public async Task InvokeAsync(HttpContext context)
    {
        var sw = Stopwatch.StartNew();
        try
        {
            await next(context);
        }
        finally
        {
            sw.Stop();
            logger.LogInformation("{Method} {Path} → {Status} in {Ms}ms",
                context.Request.Method, context.Request.Path,
                context.Response.StatusCode, sw.ElapsedMilliseconds);
        }
    }
}

// IMiddleware : résolu par DI à chaque requête → peut injecter des services Scoped
public class AuditMiddleware(IAuditService audit) : IMiddleware
{
    public async Task InvokeAsync(HttpContext context, RequestDelegate next)
    {
        await audit.TrackRequestAsync(context); // service Scoped possible ici
        await next(context);
    }
}

// IMiddleware doit être enregistré dans le DI
builder.Services.AddTransient<AuditMiddleware>();

// Extension methods pour l'enregistrement
public static class MiddlewareExtensions
{
    public static IApplicationBuilder UseRequestTiming(this IApplicationBuilder app)
        => app.UseMiddleware<RequestTimingMiddleware>();

    public static IApplicationBuilder UseAudit(this IApplicationBuilder app)
        => app.UseMiddleware<AuditMiddleware>();
}
```

> **Règle :** convention-based si les dépendances sont Singleton. `IMiddleware` si besoin de services Scoped (ex: DbContext, repositories).

---

### 🟡 Minimal APIs vs Controllers : différences et quand choisir ?

```csharp
// Controllers : attribute-based, adapté aux APIs complexes
[ApiController]
[Route("api/[controller]")]
public class OrdersController(IOrderService service) : ControllerBase
{
    [HttpGet("{id:guid}")]
    public async Task<ActionResult<OrderDto>> GetById(Guid id, CancellationToken ct)
        => await service.GetByIdAsync(id, ct) is { } order
            ? Ok(order)
            : NotFound();

    [HttpPost]
    [ProducesResponseType<OrderDto>(StatusCodes.Status201Created)]
    public async Task<IActionResult> Create(CreateOrderRequest req, CancellationToken ct)
    {
        var id = await service.CreateAsync(req, ct);
        return CreatedAtAction(nameof(GetById), new { id }, new OrderDto(id));
    }
}

// Minimal APIs (.NET 6+) : code-first, moins de boilerplate, plus performant
var orders = app.MapGroup("/api/orders")
    .RequireAuthorization()
    .WithTags("Orders");

orders.MapGet("/{id:guid}", async (Guid id, IOrderService svc, CancellationToken ct) =>
    await svc.GetByIdAsync(id, ct) is { } order ? Results.Ok(order) : Results.NotFound())
    .WithName("GetOrder");

orders.MapPost("/", async (CreateOrderRequest req, IOrderService svc, CancellationToken ct) =>
{
    var id = await svc.CreateAsync(req, ct);
    return Results.CreatedAtRoute("GetOrder", new { id }, new OrderDto(id));
});
```

| | Controllers | Minimal APIs |
|---|---|---|
| Boilerplate | Plus verbeux | Minimal |
| Performances | Légèrement plus lent | Plus rapide (moins d'abstraction) |
| Filtres | `ActionFilter`, `ExceptionFilter` | `IEndpointFilter` |
| Tests | `WebApplicationFactory` | `WebApplicationFactory` |
| Idéal pour | APIs complexes, équipes larges | Microservices, APIs simples |

---

### 🔴 Quel est l'ordre correct des middlewares dans ASP.NET Core ? Pourquoi est-il critique ?

```csharp
// Ordre recommandé par Microsoft
app.UseExceptionHandler("/error");   // 1. En premier : capture toutes les exceptions
app.UseHsts();                       // 2. HSTS header
app.UseHttpsRedirection();           // 3. Redirect HTTP → HTTPS
app.UseStaticFiles();                // 4. Court-circuite avant auth (perf + sécurité)
app.UseRouting();                    // 5. Résout l'endpoint (requis avant CORS/Auth)
app.UseCors();                       // 6. Après routing : politiques CORS par endpoint
app.UseAuthentication();             // 7. Qui est l'utilisateur ?
app.UseAuthorization();              // 8. A-t-il le droit ?
app.UseRateLimiter();                // 9. Limite le débit (.NET 7+)
app.UseOutputCache();                // 10. Cache la réponse (.NET 7+)
app.MapControllers();                // 11. Terminal : exécute l'endpoint
```

**Pourquoi l'ordre est critique :**
- `UseExceptionHandler` en premier : doit wrapper tout le pipeline pour catcher les erreurs
- `UseAuthentication` **avant** `UseAuthorization` : l'identité doit être établie avant d'autoriser
- `UseRouting` **avant** `UseCors` : CORS a besoin de connaître l'endpoint pour les politiques fine-grained
- `UseStaticFiles` tôt : court-circuite sans passer par l'authentification (fichiers publics)

---

### 🔴 Qu'est-ce que les Endpoint Filters (.NET 7+) ? Comment diffèrent-ils des ActionFilters ?

```csharp
// Endpoint Filter : s'applique aux Minimal APIs
public class ValidationFilter<T> : IEndpointFilter where T : class
{
    public async ValueTask<object?> InvokeAsync(
        EndpointFilterInvocationContext ctx, EndpointFilterDelegate next)
    {
        var validator = ctx.HttpContext.RequestServices.GetService<IValidator<T>>();
        if (validator is not null)
        {
            var arg = ctx.Arguments.OfType<T>().FirstOrDefault();
            if (arg is not null)
            {
                var result = await validator.ValidateAsync(arg);
                if (!result.IsValid)
                    return Results.ValidationProblem(result.ToDictionary());
            }
        }
        return await next(ctx);
    }
}

// Application sur un groupe
orders.AddEndpointFilter<ValidationFilter<CreateOrderRequest>>();

// Ou inline sur un endpoint
orders.MapPost("/", Handler)
    .AddEndpointFilter(async (ctx, next) =>
    {
        // logique avant
        var result = await next(ctx);
        // logique après
        return result;
    });
```

| | `ActionFilter` | `IEndpointFilter` |
|---|---|---|
| Portée | Controllers uniquement | Minimal APIs uniquement |
| Contexte | `ActionExecutingContext` | `EndpointFilterInvocationContext` |
| DI | Via attributs / constructeur | Via `IServiceProvider` du contexte |
| Ordre | Attributs + conventions | Chaîne explicite |

---

### 🔴 Comment implémenter une gestion d'exceptions globale avec `IExceptionHandler` (.NET 8+) ?

```csharp
// IExceptionHandler : remplace les ExceptionHandlerMiddleware personnalisés
public class GlobalExceptionHandler(ILogger<GlobalExceptionHandler> logger)
    : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(
        HttpContext context, Exception exception, CancellationToken ct)
    {
        var (status, title) = exception switch
        {
            NotFoundException      => (404, "Resource not found"),
            ValidationException    => (400, "Validation failed"),
            UnauthorizedAccessException => (403, "Forbidden"),
            _                      => (500, "An unexpected error occurred")
        };

        logger.LogError(exception, "Unhandled exception: {Title}", title);

        context.Response.StatusCode = status;
        await context.Response.WriteAsJsonAsync(new ProblemDetails
        {
            Status = status,
            Title = title,
            Detail = exception.Message,
            Instance = context.Request.Path
        }, ct);

        return true; // true = exception traitée ; false = passe au handler suivant
    }
}

// Enregistrement (plusieurs handlers possibles, exécutés dans l'ordre)
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
builder.Services.AddProblemDetails();
app.UseExceptionHandler();
```

**Avantages vs `app.UseExceptionHandler(path)` :**
- Logique typée par exception dans `TryHandleAsync`
- Plusieurs handlers chaînés (retourner `false` délègue au suivant)
- DI natif, testable unitairement

---

### 🔴 Qu'est-ce que le Rate Limiting intégré dans .NET 7+ ? Quels algorithmes sont disponibles ?

```csharp
builder.Services.AddRateLimiter(opt =>
{
    // Politique nommée : sliding window
    opt.AddSlidingWindowLimiter("api", o =>
    {
        o.Window = TimeSpan.FromMinutes(1);
        o.PermitLimit = 60;
        o.SegmentsPerWindow = 6;
        o.QueueLimit = 5;
        o.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
    });

    // Politique globale : partitionnée par utilisateur/IP
    opt.GlobalLimiter = PartitionedRateLimiter.Create<HttpContext, string>(ctx =>
        RateLimitPartition.GetFixedWindowLimiter(
            partitionKey: ctx.User.Identity?.Name ?? ctx.Connection.RemoteIpAddress?.ToString() ?? "anon",
            factory: _ => new FixedWindowRateLimiterOptions
            {
                Window = TimeSpan.FromSeconds(10),
                PermitLimit = 100
            }));

    opt.OnRejected = async (ctx, ct) =>
    {
        ctx.HttpContext.Response.StatusCode = StatusCodes.Status429TooManyRequests;
        if (ctx.Lease.TryGetMetadata(MetadataName.RetryAfter, out var retryAfter))
            ctx.HttpContext.Response.Headers.RetryAfter = retryAfter.TotalSeconds.ToString();
        await ctx.HttpContext.Response.WriteAsync("Too many requests.", ct);
    };
});

app.UseRateLimiter();

// Application sur un endpoint ou groupe
app.MapGet("/api/data", Handler).RequireRateLimiting("api");
```

| Algorithme | Comportement |
|---|---|
| **Fixed Window** | N requêtes max par fenêtre fixe (ex: 100/minute) |
| **Sliding Window** | N requêtes sur une fenêtre glissante découpée en segments |
| **Token Bucket** | N tokens rechargés à taux fixe, consommés par requête |
| **Concurrency** | N requêtes simultanées maximum |

---

## Lexique

| Terme | Définition |
|---|---|
| **ActionFilter** | Filtre s'exécutant avant/après une action de Controller. |
| **Endpoint** | Composant terminal associant un pattern de route à un handler. |
| **IEndpointFilter** | Interface de filtre pour Minimal APIs, s'exécutant autour de l'endpoint. |
| **IExceptionHandler** | Interface .NET 8+ pour gérer globalement les exceptions non traitées. |
| **IMiddleware** | Interface DI-friendly pour les middlewares nécessitant des services Scoped. |
| **MapGroup** | Méthode Minimal APIs regroupant des endpoints sous un préfixe commun. |
| **Middleware** | Composant du pipeline HTTP traitant les requêtes et pouvant déléguer via `next`. |
| **Minimal APIs** | Style d'API .NET 6+ sans Controllers, avec routing déclaratif. |
| **Pipeline** | Chaîne ordonnée de middlewares traitant chaque requête HTTP. |
| **ProblemDetails** | Format standardisé RFC 7807 pour les réponses d'erreur HTTP en JSON. |
| **Rate Limiting** | Limitation du nombre de requêtes sur une période pour protéger l'API. |
| **RequestDelegate** | Délégué `async Task(HttpContext)` représentant le prochain middleware. |
| **Short-circuiting** | Interruption du pipeline sans appeler `next`, stoppant le traitement. |

---

## Ressources

### Documentation officielle Microsoft
- [ASP.NET Core Middleware](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/middleware/)
- [Minimal APIs overview](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/minimal-apis/overview)
- [Endpoint filters in Minimal APIs](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/minimal-apis/min-api-filters)
- [Rate limiting in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/performance/rate-limit)
- [Error handling in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/error-handling)

### Livres
- *ASP.NET Core in Action* — Andrew Lock
- *Pro ASP.NET Core 8* — Adam Freeman

### Blogs & Articles
- [Andrew Lock — Middleware internals](https://andrewlock.net/)
- [Damian Edwards / David Fowler — .NET blog](https://devblogs.microsoft.com/dotnet/)
