# Entretien .NET Senior/Expert — API Design

> **Versions ciblées :** .NET 8 / .NET 9 / .NET 10, C# 12 / C# 13 / C# 14
>
> **Niveaux de difficulté :** 🟢 Basique · 🟡 Intermédiaire · 🔴 Senior/Expert

---

## Fondamentaux

### 🟢 Minimal APIs vs Controllers MVC — différences et quand choisir ?

**Controllers** (modèle classique) : classes héritant de `ControllerBase` avec validation automatique, filtres d'action, model binding riche, `[ApiController]` et formatters.

**Minimal APIs** (ASP.NET Core 6+) : définition des endpoints directement via des méthodes d'extension, avec overhead minimal — idéales pour les microservices légers.

| | Controllers | Minimal APIs |
|---|---|---|
| Overhead pipeline | Plus élevé | Minimal |
| Organisation | Classes structurées | Lambdas / handlers groupés |
| Validation automatique | ✅ Via `[ApiController]` | Manuelle ou `IEndpointFilter` |
| Filtres d'action | ✅ `IActionFilter` | `IEndpointFilter` (.NET 7+) |
| Versioning | Bien supporté | Supporté (`Asp.Versioning`) |
| OpenAPI | `[ProducesResponseType]` | `.Produces<T>()`, `WithOpenApi()` |
| Tests d'intégration | `WebApplicationFactory` | `WebApplicationFactory` |
| Recommandation Microsoft | Valide | Recommandé pour nouveaux projets |

```csharp
// Minimal API — organisation par feature group (approche recommandée)
public static class OrdersEndpoints
{
    public static RouteGroupBuilder MapOrders(this IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/api/orders")
            .WithTags("Orders")
            .RequireAuthorization();

        group.MapGet("/",          GetAll);
        group.MapGet("/{id:guid}", GetById);
        group.MapPost("/",         Create);
        group.MapDelete("/{id:guid}", Delete);

        return group;
    }

    // TypedResults : typage fort des retours, OpenAPI auto-documenté
    private static async Task<Results<Ok<OrderDto>, NotFound>> GetById(
        Guid id, IOrderService svc, CancellationToken ct)
    {
        var order = await svc.GetAsync(id, ct);
        return order is not null
            ? TypedResults.Ok(order.ToDto())
            : TypedResults.NotFound();
    }
}

// Program.cs
app.MapOrders();
```

> **Recommandation :** Utiliser Minimal APIs pour les nouveaux projets (.NET 8+) avec une organisation par feature group. Garder les Controllers pour les grandes applications existantes qui exploitent intensivement les filtres d'action.

---

### 🟢 Comment versionner une API REST en ASP.NET Core ?

Trois stratégies principales, souvent combinées :

```csharp
// dotnet add package Asp.Versioning.Http
builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion                  = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ReportApiVersions                  = true; // retourne api-supported-versions en header

    options.ApiVersionReader = ApiVersionReader.Combine(
        new UrlSegmentApiVersionReader(),           // /api/v1/orders
        new HeaderApiVersionReader("X-API-Version"), // header personnalisé
        new QueryStringApiVersionReader("api-version") // ?api-version=1.0
    );
});

// Déclaration des versions sur un groupe Minimal API
var versionSet = app.NewApiVersionSet()
    .HasApiVersion(new ApiVersion(1, 0))
    .HasApiVersion(new ApiVersion(2, 0))
    .HasDeprecatedApiVersion(new ApiVersion(0, 9))
    .ReportApiVersions()
    .Build();

app.MapGet("/api/v{version:apiVersion}/orders", GetAllOrders)
   .WithApiVersionSet(versionSet)
   .MapToApiVersion(1, 0);

app.MapGet("/api/v{version:apiVersion}/orders", GetAllOrdersV2)
   .WithApiVersionSet(versionSet)
   .MapToApiVersion(2, 0);
```

**Synthèse des stratégies :**

| Stratégie | Exemple | Avantages | Inconvénients |
|---|---|---|---|
| **URL segment** | `/v1/orders` | Visible, bookmarkable | Cassant à chaque version |
| **Header** | `X-API-Version: 1` | URL propre | Invisible dans le navigateur |
| **Query string** | `?api-version=1.0` | Facile à tester | Peu esthétique |
| **Media type** | `Accept: application/json;v=1` | REST pur | Complexe à implémenter |

> **Bonne pratique :** Déprécier avant de supprimer. Maintenir N et N-1 minimum. Toujours documenter les breaking changes dans le changelog.

---

### 🟡 Qu'est-ce que ProblemDetails (RFC 9457) ? Comment l'implémenter en .NET 8+ ?

`ProblemDetails` est un format JSON standardisé (RFC 9457, anciennement RFC 7807) pour les réponses d'erreur API. ASP.NET Core le supporte nativement depuis .NET 7.

```csharp
// .NET 8+ : activation + personnalisation
builder.Services.AddProblemDetails(options =>
{
    options.CustomizeProblemDetails = ctx =>
    {
        ctx.ProblemDetails.Extensions["requestId"] = ctx.HttpContext.TraceIdentifier;
        ctx.ProblemDetails.Extensions["timestamp"]  = DateTimeOffset.UtcNow;
    };
});

// IExceptionHandler (.NET 8) : gestion globale des exceptions → ProblemDetails
public class GlobalExceptionHandler(IProblemDetailsService problemDetailsService)
    : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(
        HttpContext ctx, Exception ex, CancellationToken ct)
    {
        var (statusCode, title) = ex switch
        {
            NotFoundException          => (StatusCodes.Status404NotFound,            "Resource not found"),
            ValidationException        => (StatusCodes.Status400BadRequest,          "Validation failed"),
            UnauthorizedAccessException => (StatusCodes.Status403Forbidden,          "Access denied"),
            _                          => (StatusCodes.Status500InternalServerError, "An unexpected error occurred")
        };

        ctx.Response.StatusCode = statusCode;

        return await problemDetailsService.TryWriteAsync(new ProblemDetailsContext
        {
            HttpContext     = ctx,
            Exception       = ex,
            ProblemDetails  =
            {
                Title  = title,
                Status = statusCode,
                Detail = ex.Message,
                Type   = $"https://myapi.com/errors/{statusCode}"
            }
        });
    }
}

// Enregistrement
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
app.UseExceptionHandler();
```

**Format JSON produit :**
```json
{
  "type":      "https://myapi.com/errors/404",
  "title":     "Resource not found",
  "status":    404,
  "detail":    "Order with id 'abc' was not found.",
  "requestId": "0HMXXXXXX",
  "timestamp": "2026-03-19T10:00:00Z"
}
```

---

### 🟡 Comment configurer OpenAPI avec .NET 9 (Microsoft.AspNetCore.OpenApi + Scalar) ?

.NET 9 introduit le package officiel `Microsoft.AspNetCore.OpenApi` (remplace Swashbuckle) et recommande **Scalar** comme UI de documentation.

```csharp
// dotnet add package Microsoft.AspNetCore.OpenApi
// dotnet add package Scalar.AspNetCore

builder.Services.AddOpenApi(options =>
{
    options.AddDocumentTransformer<BearerSecuritySchemeTransformer>();
    options.AddOperationTransformer((operation, context, ct) =>
    {
        operation.Deprecated = context.Description.ActionDescriptor
            .EndpointMetadata.OfType<ObsoleteAttribute>().Any();
        return Task.CompletedTask;
    });
});

app.MapOpenApi(); // expose /openapi/v1.json

if (app.Environment.IsDevelopment())
{
    app.MapScalarApiReference(options =>
    {
        options.Title = "My API";
        options.Theme = ScalarTheme.DeepSpace;
    }); // expose /scalar/v1
}

// Documenter les endpoints Minimal API
app.MapGet("/api/orders/{id:guid}", async (Guid id, IOrderService svc) => { })
   .WithName("GetOrderById")
   .WithSummary("Get a specific order")
   .WithDescription("Returns a single order by its unique identifier.")
   .Produces<OrderDto>(StatusCodes.Status200OK)
   .Produces<ProblemDetails>(StatusCodes.Status404NotFound)
   .WithTags("Orders");
```

**Transformer Bearer JWT :**
```csharp
internal sealed class BearerSecuritySchemeTransformer(
    IAuthenticationSchemeProvider authenticationSchemeProvider) : IOpenApiDocumentTransformer
{
    public async Task TransformAsync(
        OpenApiDocument doc,
        OpenApiDocumentTransformerContext ctx,
        CancellationToken ct)
    {
        var schemes = await authenticationSchemeProvider.GetAllSchemesAsync();
        if (!schemes.Any(s => s.Name == JwtBearerDefaults.AuthenticationScheme)) return;

        doc.Components ??= new();
        doc.Components.SecuritySchemes ??= new Dictionary<string, OpenApiSecurityScheme>();
        doc.Components.SecuritySchemes["Bearer"] = new OpenApiSecurityScheme
        {
            Type          = SecuritySchemeType.Http,
            Scheme        = "bearer",
            BearerFormat  = "JWT",
        };

        foreach (var operation in doc.Paths.Values.SelectMany(p => p.Operations.Values))
        {
            operation.Security.Add(new OpenApiSecurityRequirement
            {
                [new OpenApiSecurityScheme
                {
                    Reference = new() { Id = "Bearer", Type = ReferenceType.SecurityScheme }
                }] = []
            });
        }
    }
}
```

---

### 🟡 Comment implémenter le Rate Limiting natif (.NET 7+) ?

ASP.NET Core 7 introduit `System.Threading.RateLimiting` nativement, sans dépendance externe.

```csharp
builder.Services.AddRateLimiter(options =>
{
    options.RejectionStatusCode = StatusCodes.Status429TooManyRequests;
    options.OnRejected = async (ctx, ct) =>
    {
        ctx.HttpContext.Response.Headers.RetryAfter = "60";
        await ctx.HttpContext.Response.WriteAsync("Too many requests. Please retry later.", ct);
    };

    // Fixed Window : 10 requêtes / minute par IP
    options.AddPolicy("fixed", httpCtx =>
        RateLimitPartition.GetFixedWindowLimiter(
            partitionKey: httpCtx.Connection.RemoteIpAddress?.ToString() ?? "anon",
            factory: _ => new FixedWindowRateLimiterOptions
            {
                PermitLimit            = 10,
                Window                 = TimeSpan.FromMinutes(1),
                QueueProcessingOrder   = QueueProcessingOrder.OldestFirst,
                QueueLimit             = 2
            }));

    // Sliding Window : distribution plus lisse
    options.AddPolicy("sliding", httpCtx =>
        RateLimitPartition.GetSlidingWindowLimiter(
            partitionKey: httpCtx.User.Identity?.Name
                          ?? httpCtx.Connection.RemoteIpAddress?.ToString()
                          ?? "anon",
            factory: _ => new SlidingWindowRateLimiterOptions
            {
                PermitLimit       = 100,
                Window            = TimeSpan.FromMinutes(1),
                SegmentsPerWindow = 6  // window divisée en 6 segments de 10s
            }));

    // Token Bucket global : burst contrôlé
    options.GlobalLimiter = PartitionedRateLimiter.Create<HttpContext, string>(ctx =>
        RateLimitPartition.GetTokenBucketLimiter(
            partitionKey: "global",
            factory: _ => new TokenBucketRateLimiterOptions
            {
                TokenLimit          = 1000,
                ReplenishmentPeriod = TimeSpan.FromSeconds(1),
                TokensPerPeriod     = 100,
                AutoReplenishment   = true
            }));
});

app.UseRateLimiter();

// Appliquer par endpoint
app.MapGet("/api/orders",       GetOrders).RequireRateLimiting("fixed");
app.MapPost("/api/auth/login",  Login).RequireRateLimiting("sliding");
```

**4 algorithmes disponibles :**

| Algorithme | Caractéristique | Usage typique |
|---|---|---|
| Fixed Window | Simple, pics possibles en bordure de fenêtre | Limites générales |
| Sliding Window | Distribution uniforme, évite les pics | APIs publiques |
| Token Bucket | Autorise les bursts, lisse sur la durée | APIs avec pics occasionnels |
| Concurrency | Limite le nombre de requêtes parallèles | Ressources coûteuses (DB, fichiers) |

---

### 🔴 gRPC vs REST vs GraphQL en .NET — quand utiliser quoi ?

| Critère | REST | gRPC | GraphQL |
|---|---|---|---|
| Protocole | HTTP/1.1 ou HTTP/2 | HTTP/2 obligatoire | HTTP/1.1 ou HTTP/2 |
| Format | JSON / XML | Protobuf (binaire) | JSON |
| Contrat | OpenAPI (optionnel) | `.proto` (obligatoire) | Schéma SDL |
| Performance | Moyen | Excellent (5-10× REST) | Moyen |
| Streaming | SSE / WebSocket | Natif bi-directionnel | Subscriptions |
| Browsers | ✅ Direct | ❌ grpc-web requis | ✅ Direct |
| Type safety | Via OpenAPI codegen | Natif (protobuf) | Via codegen |
| Usage idéal | API publiques, mobile | Microservices internes | BFF, clients riches |

```csharp
// gRPC server en .NET
// dotnet add package Grpc.AspNetCore

// Fichier orders.proto :
// service OrderService {
//   rpc GetOrder (GetOrderRequest) returns (OrderResponse);
//   rpc StreamOrders (Empty) returns (stream OrderResponse);
// }

public class OrderGrpcService(IOrderRepository repo) : OrderServiceBase
{
    public override async Task<OrderResponse> GetOrder(
        GetOrderRequest request, ServerCallContext context)
    {
        var order = await repo.GetByIdAsync(
            Guid.Parse(request.Id), context.CancellationToken)
            ?? throw new RpcException(
                new Status(StatusCode.NotFound, $"Order {request.Id} not found"));

        return new OrderResponse
        {
            Id           = order.Id.ToString(),
            CustomerName = order.CustomerName,
            Amount       = (double)order.Amount
        };
    }

    // Server streaming : envoyer les orders en temps réel
    public override async Task StreamOrders(
        Empty request,
        IServerStreamWriter<OrderResponse> responseStream,
        ServerCallContext context)
    {
        await foreach (var order in repo.StreamAllAsync(context.CancellationToken))
        {
            await responseStream.WriteAsync(new OrderResponse
            {
                Id           = order.Id.ToString(),
                CustomerName = order.CustomerName
            });
        }
    }
}

// Registration
builder.Services.AddGrpc();
app.MapGrpcService<OrderGrpcService>();
```

**Recommandations pratiques :**
- **REST** : API publique, intégration tierce, consumers browser — choisir REST par défaut
- **gRPC** : communication inter-services interne, streaming temps-réel, haute performance (sérialisations lourdes)
- **GraphQL** : BFF, client SPA/mobile avec besoins de données flexibles (évite l'over-fetching)

---

### 🔴 Qu'est-ce que YARP ? Comment l'utiliser comme reverse proxy / API Gateway ?

**YARP** (*Yet Another Reverse Proxy*, `Microsoft.ReverseProxy`) est un reverse proxy .NET haute performance, configurable par code ou JSON. Il sert de fondation pour un API Gateway ou un BFF.

```csharp
// dotnet add package Yarp.ReverseProxy
builder.Services.AddReverseProxy()
    .LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"))
    .AddTransforms(ctx =>
    {
        // Injecter un header tenant sur toutes les requêtes proxiées
        ctx.AddRequestTransform(async transform =>
        {
            var tenant = transform.HttpContext.User.FindFirst("tenant_id")?.Value;
            if (tenant is not null)
                transform.ProxyRequest.Headers.TryAddWithoutValidation("X-Tenant-Id", tenant);
        });
    });

app.MapReverseProxy(pipeline =>
{
    pipeline.UseSessionAffinity();
    pipeline.UseLoadBalancing();
    pipeline.UseHealthChecks();
});
```

```json
// appsettings.json
{
  "ReverseProxy": {
    "Routes": {
      "orders-route": {
        "ClusterId": "orders-cluster",
        "Match": { "Path": "/api/orders/{**catch-all}" },
        "Transforms": [{ "PathRemovePrefix": "/api/orders" }]
      },
      "products-route": {
        "ClusterId": "products-cluster",
        "Match": { "Path": "/api/products/{**catch-all}" }
      }
    },
    "Clusters": {
      "orders-cluster": {
        "LoadBalancingPolicy": "RoundRobin",
        "HealthCheck": {
          "Active": { "Enabled": true, "Path": "/health" }
        },
        "Destinations": {
          "orders-1": { "Address": "http://orders-service:8080/" },
          "orders-2": { "Address": "http://orders-service-2:8080/" }
        }
      }
    }
  }
}
```

**Fonctionnalités YARP :**
- Load balancing (RoundRobin, LeastRequests, Random, PowerOfTwoChoices)
- Health checks actifs et passifs avec éviction automatique
- Session affinity (sticky sessions)
- Transforms de requêtes/réponses (headers, path, query string)
- Configuration dynamique sans redémarrage
- Intégration native avec l'authentification ASP.NET Core

---

### 🔴 Qu'est-ce que le pattern Idempotency Key pour les APIs REST ?

Une **clé d'idempotence** permet au client de rejouer une requête plusieurs fois sans effet de bord dupliqué — indispensable pour les opérations financières et les retries réseau.

```csharp
// Middleware d'idempotence
public class IdempotencyMiddleware(IDistributedCache cache, RequestDelegate next)
{
    public async Task InvokeAsync(HttpContext ctx)
    {
        // Uniquement pour les opérations non-idempotentes par nature
        if (ctx.Request.Method is not ("POST" or "PATCH"))
        {
            await next(ctx);
            return;
        }

        if (!ctx.Request.Headers.TryGetValue("Idempotency-Key", out var keyValues)
            || string.IsNullOrWhiteSpace(keyValues))
        {
            ctx.Response.StatusCode = StatusCodes.Status400BadRequest;
            await ctx.Response.WriteAsJsonAsync(
                new { error = "Idempotency-Key header is required." });
            return;
        }

        var cacheKey = $"idempotency:{keyValues}";

        // Résultat déjà en cache → retourner la réponse précédente
        var cached = await cache.GetStringAsync(cacheKey);
        if (cached is not null)
        {
            ctx.Response.StatusCode  = StatusCodes.Status200OK;
            ctx.Response.ContentType = "application/json";
            await ctx.Response.WriteAsync(cached);
            return;
        }

        // Capturer la réponse générée
        var originalBody = ctx.Response.Body;
        using var memStream = new MemoryStream();
        ctx.Response.Body = memStream;

        await next(ctx);

        memStream.Seek(0, SeekOrigin.Begin);
        var responseBody = await new StreamReader(memStream).ReadToEndAsync();

        // Mettre en cache uniquement les succès (2xx)
        if (ctx.Response.StatusCode is >= 200 and < 300)
        {
            await cache.SetStringAsync(cacheKey, responseBody,
                new DistributedCacheEntryOptions
                {
                    AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(24)
                });
        }

        memStream.Seek(0, SeekOrigin.Begin);
        await memStream.CopyToAsync(originalBody);
        ctx.Response.Body = originalBody;
    }
}
```

**Règles d'or :**
1. La clé est générée côté **client** (UUID v4)
2. Stocker clé + corps de réponse dans un cache **distribué** (Redis)
3. TTL suffisant pour couvrir les retry storms (24h typiquement)
4. Retourner exactement la même réponse (même statut, même corps)
5. Documenter l'en-tête dans la spec OpenAPI

---

### 🔴 Comment implémenter le pattern BFF (Backend for Frontend) en .NET ?

Le **BFF** est une API dédiée à un client spécifique (SPA, mobile) qui agrège plusieurs microservices, évitant l'over-fetching et l'under-fetching des clients.

```csharp
// BFF pour SPA React — agrège Orders + Products + Users en un seul appel
public class DashboardBffService(
    IOrdersClient  orders,
    IProductsClient products,
    IUserClient     users)
{
    public async Task<DashboardDto> GetDashboardAsync(
        Guid userId, CancellationToken ct = default)
    {
        // Appels parallèles aux microservices sous-jacents
        var ordersTask    = orders.GetRecentAsync(userId, count: 5, ct);
        var watchlistTask = products.GetWatchlistAsync(userId, ct);
        var profileTask   = users.GetProfileAsync(userId, ct);

        await Task.WhenAll(ordersTask, watchlistTask, profileTask);

        return new DashboardDto
        {
            Profile      = profileTask.Result.ToDto(),
            RecentOrders = ordersTask.Result.Select(o => o.ToSummary()),
            Watchlist    = watchlistTask.Result.Select(p => p.ToCard())
        };
    }
}

// Clients HTTP typés avec résilience Polly v8
builder.Services.AddHttpClient<IOrdersClient, OrdersClient>(client =>
{
    client.BaseAddress = new Uri(builder.Configuration["Services:Orders"]!);
    client.Timeout     = TimeSpan.FromSeconds(5);
}).AddStandardResilienceHandler(); // retry + circuit breaker automatiques

// Endpoint BFF
app.MapGet("/api/dashboard", async (
    Guid userId,
    DashboardBffService bff,
    CancellationToken ct) =>
    await bff.GetDashboardAsync(userId, ct))
   .RequireAuthorization()
   .WithTags("Dashboard BFF");
```

**BFF vs API Gateway :**

| | API Gateway (YARP) | BFF |
|---|---|---|
| Responsabilité | Routage, auth, rate limiting | Agrégation, transformation, logique spécifique au client |
| Couplage clients | Générique (tous les clients) | Dédié par type de client (SPA, mobile, TV) |
| Logique métier | Non | Légère (orchestration uniquement) |
| Exemple .NET | YARP | ASP.NET Core Minimal API |

> Combiner les deux : YARP comme gateway pour les routes 1-to-1 et l'auth, BFF pour les endpoints d'agrégation complexes.

---

## Lexique

| Terme | Définition |
|---|---|
| **BFF (Backend for Frontend)** | API dédiée à un client spécifique qui agrège plusieurs microservices en adaptant les réponses. |
| **Content negotiation** | Mécanisme HTTP où le client indique le format attendu (`Accept` header) et le serveur sélectionne la représentation. |
| **gRPC** | Framework RPC utilisant HTTP/2 et Protobuf, haute performance, idéal pour la communication inter-services. |
| **Idempotency Key** | En-tête client (UUID) permettant de rejouer une requête sans dupliquer son effet de bord. |
| **Minimal APIs** | Style ASP.NET Core de définition d'endpoints sans contrôleurs, avec overhead minimal. |
| **OpenAPI** | Spécification standard pour décrire les APIs REST (anciennement Swagger). |
| **ProblemDetails** | Format JSON standardisé (RFC 9457) pour les réponses d'erreur API. |
| **Rate Limiting** | Mécanisme limitant le nombre de requêtes par client et par fenêtre de temps. |
| **Scalar** | UI open-source pour les spécifications OpenAPI, remplaçant officiel de Swagger UI en .NET 9. |
| **TypedResults** | API Minimal APIs (.NET 7+) retournant des types de résultat fortement typés, auto-documentés dans OpenAPI. |
| **YARP** | *Yet Another Reverse Proxy* — reverse proxy .NET haute performance de Microsoft. |
| **API Versioning** | Stratégie exposant plusieurs versions d'une API (URL segment, header, query string, media type). |

---

## Ressources

### Documentation officielle Microsoft
- [Minimal APIs in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/minimal-apis)
- [Rate limiting middleware in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/performance/rate-limit)
- [OpenAPI support in ASP.NET Core (.NET 9)](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/openapi/aspnetcore-openapi)
- [YARP documentation](https://microsoft.github.io/reverse-proxy/)
- [gRPC in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/grpc/)
- [ProblemDetails / IExceptionHandler (.NET 8)](https://learn.microsoft.com/en-us/aspnet/core/web-api/handle-errors)
- [Asp.Versioning — API versioning for .NET](https://github.com/dotnet/aspnet-api-versioning)

### Blogs & Articles
- [David Fowler — Minimal APIs design decisions](https://devblogs.microsoft.com/dotnet/)
- [Andrew Lock — API versioning in ASP.NET Core](https://andrewlock.net/exploring-the-dotnet-8-preview-asp-net-core-metrics/)
- [Nick Chapsas — YARP and gRPC in .NET (YouTube)](https://www.youtube.com/@nickchapsas)
- [Scalar — Official documentation](https://scalar.com/swagger-ui-alternatives/scalar/)
- [sam-mccall — Pattern BFF (Sam Newman)](https://samnewman.io/patterns/architectural/bff/)
