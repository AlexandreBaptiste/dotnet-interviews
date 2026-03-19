# Entretien .NET Senior/Expert — Résilience & Gestion des erreurs

> **Versions ciblées :** .NET 8 / .NET 9 / .NET 10, C# 12 / C# 13 / C# 14
>
> **Niveaux de difficulté :** 🟢 Basique · 🟡 Intermédiaire · 🔴 Senior/Expert

---

## Fondamentaux

### 🟢 Qu'est-ce que la résilience dans les systèmes distribués ? Quels sont les patterns fondamentaux ?

La résilience est la capacité d'une application à **se remettre des défaillances** de ses dépendances (réseau, base de données, services tiers) et à rester opérationnelle malgré celles-ci.

**Patterns principaux :**

| Pattern | Objectif |
|---|---|
| **Retry** | Réessayer automatiquement une opération après un échec transitoire |
| **Circuit Breaker** | Couper les appels vers un service défaillant pour éviter la cascade |
| **Timeout** | Limiter le temps d'attente d'une opération pour libérer les ressources |
| **Bulkhead** | Isoler les ressources entre composants pour limiter la surface d'impact |
| **Hedging** | Envoyer plusieurs requêtes parallèles, utiliser la première réponse réussie |
| **Fallback** | Retourner une valeur de secours si toutes les tentatives échouent |

---

### 🟢 Qu'est-ce que Polly v8 ? Quelle est la différence avec les versions précédentes ?

Polly est la bibliothèque de résilience de référence en .NET. La version 8 introduit une API unifiée via les `ResiliencePipeline` (remplaçant les `Policy` isolées).

```csharp
// Polly v7 : policies chaînées
var policy = Policy
    .Handle<HttpRequestException>()
    .WaitAndRetryAsync(3, attempt => TimeSpan.FromSeconds(Math.Pow(2, attempt)));

// Polly v8 : ResiliencePipeline unifié
var pipeline = new ResiliencePipelineBuilder()
    .AddRetry(new RetryStrategyOptions
    {
        MaxRetryAttempts = 3,
        Delay = TimeSpan.FromMilliseconds(200),
        BackoffType = DelayBackoffType.Exponential,
        UseJitter = true,
        ShouldHandle = new PredicateBuilder().Handle<HttpRequestException>()
    })
    .AddTimeout(TimeSpan.FromSeconds(10))
    .Build();

// Utilisation
var result = await pipeline.ExecuteAsync(
    async ct => await httpClient.GetStringAsync(url, ct),
    cancellationToken);
```

**Nouveautés Polly v8 :**
- `ResiliencePipeline` composé (une seule API pour tout)
- Intégration native avec `Microsoft.Extensions.Resilience`
- Métriques OpenTelemetry automatiques
- `CancellationToken` propagé nativement dans toutes les stratégies

---

### 🟡 Qu'est-ce que `AddStandardResilienceHandler` (.NET 8+) ? Que configure-t-il ?

`Microsoft.Extensions.Http.Resilience` fournit un handler HTTP préconfiguré selon les meilleures pratiques Microsoft.

```csharp
// dotnet add package Microsoft.Extensions.Http.Resilience

// Configuration par défaut — recommandée pour la plupart des cas
builder.Services.AddHttpClient("MyApi", c =>
    c.BaseAddress = new Uri("https://api.example.com"))
    .AddStandardResilienceHandler();

// Configuration personnalisée
builder.Services.AddHttpClient("PaymentApi")
    .AddResilienceHandler("payment", b =>
    {
        b.AddRetry(new HttpRetryStrategyOptions
        {
            MaxRetryAttempts = 3,
            Delay = TimeSpan.FromMilliseconds(500),
            BackoffType = DelayBackoffType.Exponential,
            UseJitter = true,
            ShouldHandle = args => args.Outcome switch
            {
                { Exception: HttpRequestException } => PredicateResult.True(),
                { Result.StatusCode: >= HttpStatusCode.InternalServerError } => PredicateResult.True(),
                { Result.StatusCode: HttpStatusCode.TooManyRequests } => PredicateResult.True(),
                _ => PredicateResult.False()
            }
        });
        b.AddCircuitBreaker(new HttpCircuitBreakerStrategyOptions
        {
            FailureRatio = 0.5,
            SamplingDuration = TimeSpan.FromSeconds(30),
            MinimumThroughput = 10,
            BreakDuration = TimeSpan.FromSeconds(15)
        });
        b.AddTimeout(TimeSpan.FromSeconds(10));
    });
```

**Couches de `AddStandardResilienceHandler` (dans l'ordre d'exécution) :**
1. **Rate limiter** (côté client) — protège le service appelé
2. **Total timeout** — timeout global de la requête avec ses retries
3. **Retry** — 3 tentatives, backoff exponentiel avec jitter
4. **Circuit breaker** — ouvre sur 50 % d'échecs sur 30 secondes
5. **Attempt timeout** — timeout par tentative individuelle

---

### 🟡 Expliquez le Circuit Breaker. Quels sont ses états et comment les transitionner ?

```
[Closed] ──→ seuil d'échecs dépassé ──→ [Open]
   ↑                                        │
   └────── succès en half-open ─────── [Half-Open] ←── durée de break écoulée
```

| État | Comportement |
|---|---|
| **Closed** | Appels passent normalement. Comptabilise les échecs. |
| **Open** | Tous les appels échouent immédiatement (`BrokenCircuitException`). |
| **Half-Open** | Laisse passer quelques requêtes tests pour évaluer la récupération. |

```csharp
var pipeline = new ResiliencePipelineBuilder<HttpResponseMessage>()
    .AddCircuitBreaker(new CircuitBreakerStrategyOptions<HttpResponseMessage>
    {
        FailureRatio = 0.5,                          // ouvre si 50% d'échecs
        SamplingDuration = TimeSpan.FromSeconds(30), // sur les 30 dernières secondes
        MinimumThroughput = 10,                      // au moins 10 appels pour évaluer
        BreakDuration = TimeSpan.FromSeconds(15),    // reste ouvert 15 secondes
        OnOpened = args =>
        {
            logger.LogWarning("Circuit ouvert. Durée : {Duration}", args.BreakDuration);
            return ValueTask.CompletedTask;
        },
        OnClosed = _ =>
        {
            logger.LogInformation("Circuit refermé.");
            return ValueTask.CompletedTask;
        }
    })
    .Build();
```

> **Point senior :** mentionner que le circuit breaker protège aussi le service appelé (pas seulement le consommateur) en lui donnant le temps de récupérer sans être inondé de requêtes.

---

### 🔴 Qu'est-ce que la stratégie Hedging dans Polly v8 ? Quand l'utiliser à la place d'un retry ?

Le Hedging envoie **plusieurs requêtes en parallèle décalées** et utilise la **première réponse réussie**, abandonnant les autres. C'est une optimisation de latence, pas de fiabilité.

```csharp
var pipeline = new ResiliencePipelineBuilder<string>()
    .AddHedging(new HedgingStrategyOptions<string>
    {
        MaxHedgedAttempts = 2,                        // 2 hedges = 3 tentatives au total
        Delay = TimeSpan.FromMilliseconds(200),       // lance un hedge si pas de réponse en 200ms
        ShouldHandle = args => args.Outcome switch
        {
            { Exception: HttpRequestException } => PredicateResult.True(),
            _ => PredicateResult.False()
        },
        // Optionnel : envoyer vers un endpoint alternatif
        ActionGenerator = args => () =>
            ValueTask.FromResult(Outcome.FromResult(
                await fallbackClient.GetStringAsync(url, args.ActionContext.CancellationToken)))
    })
    .Build();
```

**Hedging vs Retry :**

| | Retry | Hedging |
|---|---|---|
| Déclenchement | Après un échec | Après un délai sans réponse |
| Latence ajoutée | Élevée (séquentiel) | Faible (parallèle) |
| Charge sur le serveur | 1 requête à la fois | Plusieurs simultanées |
| Usage | Erreurs transitoires | P99 latence critique |

> **Cas typique :** 99 % des requêtes répondent en 50 ms, mais 1 % prend > 2 secondes (slow outliers). Hedging après 100 ms garantit la latence sans attendre l'échec complet.

---

### 🔴 Comment configurer la résilience EF Core pour les bases de données SQL Server ?

```csharp
// EF Core : stratégie de retry intégrée pour SQL Server (transient fault detection)
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString, sqlOptions =>
    {
        sqlOptions.EnableRetryOnFailure(
            maxRetryCount: 5,
            maxRetryDelay: TimeSpan.FromSeconds(10),
            errorNumbersToAdd: null); // null = liste SQL Server par défaut (timeouts, deadlocks...)
    }));

// ⚠️ Transactions + retry : utiliser ExecutionStrategy pour les opérations non-idempotentes
public async Task ProcessOrderAsync(Order order)
{
    var strategy = _context.Database.CreateExecutionStrategy();

    await strategy.ExecuteAsync(async () =>
    {
        await using var tx = await _context.Database.BeginTransactionAsync();
        try
        {
            _context.Orders.Add(order);
            await _context.SaveChangesAsync();

            // L'appel externe doit être idempotent si la stratégie rejoue
            await _externalService.NotifyAsync(order.Id);

            await tx.CommitAsync();
        }
        catch
        {
            await tx.RollbackAsync();
            throw;
        }
    });
}
```

> **Piège senior :** les appels externes (ex: envoi d'email, notification) à l'intérieur d'une `ExecutionStrategy` peuvent être rejoués. S'assurer qu'ils sont idempotents (vérification de deduplication côté serveur, ou déplacer hors de la stratégie).

---

### 🔴 Qu'est-ce que le Bulkhead pattern ? Comment l'implémenter en .NET Core ?

Le Bulkhead (**cloison étanche**) isole les ressources consommées par service/utilisateur pour qu'une saturation locale ne consomme pas toutes les ressources disponibles.

```csharp
// Polly v8 : ConcurrencyLimiter (équivalent du bulkhead)
var pipeline = new ResiliencePipelineBuilder()
    .AddConcurrencyLimiter(new ConcurrencyLimiterOptions
    {
        PermitLimit = 10,  // max 10 appels simultanés vers ce service
        QueueLimit = 5     // 5 en file, le reste rejeté immédiatement
    })
    .Build();

// Pattern plus granulaire : SemaphoreSlim par service
public class IsolatedPaymentClient(HttpClient http)
{
    private readonly SemaphoreSlim _semaphore = new(10, 10);

    public async Task<PaymentResult> ChargeAsync(decimal amount, CancellationToken ct)
    {
        if (!await _semaphore.WaitAsync(TimeSpan.FromSeconds(2), ct))
            throw new BulkheadRejectedException("Payment bulkhead capacity exceeded");
        try
        {
            return await http.PostAsJsonAsync<PaymentResult>("/charge", amount, ct)!;
        }
        finally
        {
            _semaphore.Release();
        }
    }
}
```

**Exemple d'architecture :**
```
API Gateway
├─ Bulkhead[PaymentService]    → max 20 concurrent
├─ Bulkhead[RecommendationSvc] → max 50 concurrent
└─ Bulkhead[NotificationSvc]   → max 10 concurrent
```
Si `RecommendationSvc` sature, les 20 slots du `PaymentService` restent disponibles.

---

### 🔴 Comment configurer un pipeline de résilience complet injectable via `ResiliencePipelineProvider` ?

```csharp
// Enregistrement centralisé
builder.Services.AddResiliencePipeline("payment", builder =>
{
    builder
        .AddTimeout(TimeSpan.FromSeconds(15))
        .AddRetry(new RetryStrategyOptions
        {
            MaxRetryAttempts = 3,
            Delay = TimeSpan.FromMilliseconds(300),
            BackoffType = DelayBackoffType.Exponential,
            UseJitter = true,
            OnRetry = args =>
            {
                logger.LogWarning("Retry {Attempt}", args.AttemptNumber);
                return ValueTask.CompletedTask;
            }
        })
        .AddCircuitBreaker(new CircuitBreakerStrategyOptions
        {
            FailureRatio = 0.5,
            MinimumThroughput = 5,
            SamplingDuration = TimeSpan.FromSeconds(30),
            BreakDuration = TimeSpan.FromSeconds(20)
        });
});

// Injection et utilisation dans un service
public class PaymentService(ResiliencePipelineProvider<string> pipelines)
{
    private readonly ResiliencePipeline _pipeline = pipelines.GetPipeline("payment");

    public async Task<PaymentResult> ProcessAsync(PaymentRequest req, CancellationToken ct)
        => await _pipeline.ExecuteAsync(
            async ct => await CallPaymentApiAsync(req, ct), ct);
}
```

> Polly v8 émet automatiquement des **métriques OpenTelemetry** pour chaque stratégie : tentatives de retry, ouvertures de circuit, durées de timeout — sans configuration supplémentaire.

---

## Lexique

| Terme | Définition |
|---|---|
| **Bulkhead** | Pattern isolant les ressources par service pour éviter qu'une saturation partielle n'épuise tout. |
| **BrokenCircuitException** | Exception levée par Polly quand le circuit est ouvert et les appels rejetés immédiatement. |
| **Circuit Breaker** | Pattern coupant les appels vers un service défaillant pour éviter la cascade d'erreurs. |
| **Exponential Backoff** | Stratégie de retry doublant le délai entre chaque tentative. |
| **Failure Ratio** | Taux d'échecs (0-1) sur la durée d'échantillonnage déclenchant l'ouverture du circuit. |
| **Hedging** | Pattern envoyant plusieurs requêtes parallèles décalées et retenant la première réponse réussie. |
| **Idempotence** | Propriété d'une opération pouvant être répétée sans effet supplémentaire. |
| **Jitter** | Variation aléatoire ajoutée au délai de retry pour éviter les tempêtes de reconnexion synchronisées. |
| **Polly** | Bibliothèque .NET de résilience (retry, circuit breaker, timeout, hedging, bulkhead). |
| **ResiliencePipeline** | Abstraction Polly v8 composant plusieurs stratégies en une seule chaîne. |
| **Thundering herd** | Phénomène où de nombreux clients relancent simultanément des requêtes lors du rétablissement d'un service. |
| **Transient fault** | Erreur temporaire (timeout, réseau) susceptible de disparaître à la tentative suivante. |

---

## Ressources

### Documentation officielle Microsoft
- [Resilience with Polly in .NET](https://learn.microsoft.com/en-us/dotnet/core/resilience/)
- [HTTP resilience — AddStandardResilienceHandler](https://learn.microsoft.com/en-us/dotnet/core/resilience/http-resilience)
- [EF Core — Connection resiliency](https://learn.microsoft.com/en-us/ef/core/miscellaneous/connection-resiliency)

### Packages NuGet
- [Polly v8](https://github.com/App-vNext/Polly)
- [Microsoft.Extensions.Http.Resilience](https://www.nuget.org/packages/Microsoft.Extensions.Http.Resilience)

### Blogs & Articles
- [Polly documentation](https://pollydocs.org/)
- [Steve Gordon — Resilience patterns in .NET](https://www.stevejgordon.co.uk/)
- [Bryan Hogan — Polly v8 migration guide](https://www.bryanhogan.net/)
