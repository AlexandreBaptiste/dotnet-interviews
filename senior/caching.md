# Entretien .NET Senior/Expert — Caching

> **Versions ciblées :** .NET 8 / .NET 9 / .NET 10, C# 12 / C# 13 / C# 14
>
> **Niveaux de difficulté :** 🟢 Basique · 🟡 Intermédiaire · 🔴 Senior/Expert

---

## Fondamentaux

### 🟢 Quelle est la différence entre `IMemoryCache` et `IDistributedCache` ?

`IMemoryCache` est un cache **in-process** : les données sont stockées dans la mémoire du processus de l'application. Il est rapide (accès direct), mais limité à une seule instance et perdu au redémarrage.

`IDistributedCache` est une abstraction pour un cache **partagé entre plusieurs instances** (Redis, SQL Server, NCache…). Idéal pour les déploiements multi-instances (scale-out).

| | `IMemoryCache` | `IDistributedCache` |
|---|---|---|
| Stockage | Mémoire du process | Externe (Redis, SQL…) |
| Multi-instances | ❌ Non | ✅ Oui |
| Sérialisation | Non requise | Requise (`byte[]`) |
| Vitesse | ~ns (accès mémoire) | ~ms (réseau) |
| TTL sliding + absolute | ✅ | ✅ (`DistributedCacheEntryOptions`) |
| Éviction sous pression mémoire | ✅ | Géré par le serveur cache |

```csharp
// Enregistrement
builder.Services.AddMemoryCache();
builder.Services.AddStackExchangeRedisCache(opt =>
    opt.Configuration = builder.Configuration.GetConnectionString("Redis"));

// IMemoryCache : stocker/récupérer des objets typés directement
public class ProductService(IMemoryCache cache, IProductRepository repo)
{
    public async Task<Product?> GetAsync(int id, CancellationToken ct = default)
    {
        if (cache.TryGetValue($"product:{id}", out Product? cached))
            return cached;

        var product = await repo.GetByIdAsync(id, ct);
        if (product is not null)
            cache.Set($"product:{id}", product, TimeSpan.FromMinutes(5));
        return product;
    }
}

// IDistributedCache : sérialisation manuelle (System.Text.Json)
public class ProductDistributedService(IDistributedCache cache, IProductRepository repo)
{
    public async Task<Product?> GetAsync(int id, CancellationToken ct = default)
    {
        var bytes = await cache.GetAsync($"product:{id}", ct);
        if (bytes is not null)
            return JsonSerializer.Deserialize<Product>(bytes);

        var product = await repo.GetByIdAsync(id, ct);
        if (product is not null)
        {
            var serialized = JsonSerializer.SerializeToUtf8Bytes(product);
            await cache.SetAsync($"product:{id}", serialized,
                new DistributedCacheEntryOptions
                {
                    AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5)
                }, ct);
        }
        return product;
    }
}
```

---

### 🟢 Qu'est-ce que le pattern Cache-Aside ? Comment l'implémenter en .NET ?

Le **Cache-Aside** (ou *Lazy Loading*) est le pattern le plus courant : l'application gère elle-même le cache. Elle cherche d'abord dans le cache ; en cas de *miss*, elle interroge la source de données puis stocke le résultat.

```
Requête → Cache HIT ? → retourner valeur
                ↓ MISS
          Lire depuis la DB
          Écrire en cache
          Retourner valeur
```

```csharp
public class OrderService(IMemoryCache cache, IOrderRepository repo)
{
    private static string Key(Guid id) => $"order:{id}";

    public async Task<Order?> GetAsync(Guid id, CancellationToken ct = default)
    {
        // 1. Try cache
        if (cache.TryGetValue(Key(id), out Order? cached))
            return cached;

        // 2. Cache miss → DB
        var order = await repo.GetByIdAsync(id, ct);

        // 3. Populate cache
        if (order is not null)
        {
            var options = new MemoryCacheEntryOptions()
                .SetAbsoluteExpiration(TimeSpan.FromMinutes(10))
                .SetSlidingExpiration(TimeSpan.FromMinutes(2))
                .SetSize(1); // requis si SizeLimit configuré sur le cache
            cache.Set(Key(id), order, options);
        }
        return order;
    }

    // Invalider à la modification pour éviter des données stale
    public async Task UpdateAsync(Order order, CancellationToken ct = default)
    {
        await repo.UpdateAsync(order, ct);
        cache.Remove(Key(order.Id));
    }
}
```

**Avantages :** simple, charge le cache à la demande, tolérant aux pannes du cache.
**Inconvénients :** premier accès toujours lent (*cold start*), risque de *cache stampede* (voir question dédiée).

---

### 🟡 Comment utiliser Redis avec `StackExchange.Redis` directement ?

`IDistributedCache` est pratique mais limité : pas de types riches, pas de commandes Redis avancées. Pour des fonctionnalités comme les atomic operations, pub/sub ou Lua scripts, utilisez `IConnectionMultiplexer` directement.

```csharp
// dotnet add package StackExchange.Redis
builder.Services.AddSingleton<IConnectionMultiplexer>(
    ConnectionMultiplexer.Connect(builder.Configuration.GetConnectionString("Redis")!));

public class RedisCatalogCache(IConnectionMultiplexer redis)
{
    private readonly IDatabase _db = redis.GetDatabase();

    public async Task<Product?> GetAsync(int id)
    {
        var value = await _db.StringGetAsync($"catalog:product:{id}");
        return value.HasValue ? JsonSerializer.Deserialize<Product>(value!) : null;
    }

    public async Task SetAsync(Product product)
    {
        var json = JsonSerializer.Serialize(product);
        await _db.StringSetAsync(
            $"catalog:product:{product.Id}",
            json,
            expiry: TimeSpan.FromMinutes(30));
    }

    // Atomic increment : pas de race condition possible
    public Task<long> IncrementViewCountAsync(int productId)
        => _db.StringIncrementAsync($"catalog:views:{productId}");

    // Invalidation par pattern (utiliser SCAN, jamais KEYS en production !)
    public async Task InvalidateByCategoryAsync(string category)
    {
        var server = redis.GetServer(redis.GetEndPoints().First());
        await foreach (var key in server.KeysAsync(pattern: $"catalog:*:cat:{category}"))
            await _db.KeyDeleteAsync(key);
    }
}
```

> **Bonne pratique :** Ne jamais utiliser `KEYS *` en production (bloque le thread Redis). Préférer `SCAN` (itération non-bloquante) ou concevoir des clés invalidables par tag avec `HybridCache`.

---

### 🟡 Qu'est-ce que l'Output Cache (.NET 7+) ? En quoi diffère-t-il du Response Cache ?

| | Response Caching | Output Caching |
|---|---|---|
| Niveau | Middleware HTTP (headers `Cache-Control`) | Serveur uniquement (transparent pour le client) |
| Contrôle client | Le client peut bypass (`no-cache`) | Toujours servi depuis le cache serveur |
| Invalidation | Impossible (TTL uniquement) | ✅ Programmable par tag |
| Stockage | In-memory ou `IDistributedCache` | In-memory par défaut, extensible |
| Support Minimal APIs | Partiel | ✅ Natif (.NET 8) |

```csharp
// Registration
builder.Services.AddOutputCache(options =>
{
    options.AddBasePolicy(b => b.Expire(TimeSpan.FromSeconds(60)));
    options.AddPolicy("Products", b =>
        b.Expire(TimeSpan.FromMinutes(5))
         .Tag("products")
         .SetVaryByQuery("category", "page"));
});

app.UseOutputCache();

// Minimal API : appliquer la policy
app.MapGet("/api/products", async (IProductRepository repo) =>
    await repo.GetAllAsync())
   .CacheOutput("Products");

// Invalider par tag après mutation
app.MapPost("/api/products", async (
    CreateProductRequest req,
    IProductRepository repo,
    IOutputCacheStore cacheStore,
    CancellationToken ct) =>
{
    var product = await repo.CreateAsync(req, ct);
    await cacheStore.EvictByTagAsync("products", ct); // invalide tous les endpoints taggés
    return Results.Created($"/api/products/{product.Id}", product);
});
```

---

### 🟡 Comment configurer `MemoryCacheEntryOptions` correctement ?

```csharp
var options = new MemoryCacheEntryOptions
{
    // Expiration absolue : expire à une date fixe, quoi qu'il arrive
    AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10),

    // Expiration sliding : repart à 0 à chaque accès
    // DOIT être < AbsoluteExpiration pour éviter un cache immortel
    SlidingExpiration = TimeSpan.FromMinutes(2),

    // Priorité lors de l'éviction sous pression mémoire
    Priority = CacheItemPriority.High, // Normal par défaut

    // Taille relative (requis si SizeLimit est configuré sur MemoryCache)
    Size = 1,
};

// Callback d'éviction (logging, cleanup de ressources)
options.RegisterPostEvictionCallback((key, value, reason, state) =>
{
    var logger = (ILogger)state!;
    logger.LogInformation("Cache key {Key} evicted: {Reason}", key, reason);
}, logger);

// Token d'annulation manuel : permet d'invalider une entrée depuis l'extérieur
var cts = new CancellationTokenSource();
options.AddExpirationToken(new CancellationChangeToken(cts.Token));
cache.Set("my-key", myValue, options);

// Invalidation manuelle via le token
cts.Cancel();
```

> **Règle d'or :** Toujours définir une `AbsoluteExpiration` même en combinaison avec `SlidingExpiration`. Sans expiration absolue, une entrée fortement consultée ne sera jamais expirée — fuite mémoire progressive.

---

### 🔴 Qu'est-ce qu'un cache stampede (dog-piling) ? Comment le prévenir en .NET ?

Un **cache stampede** survient quand une entrée expire et que des centaines de requêtes simultanées constatent un *miss* et envoient toutes une requête vers la base de données en même temps, la surchargeant.

**Solution 1 : `SemaphoreSlim` + double-check locking**

```csharp
public class StampedeSafeCache(IMemoryCache cache, IProductRepository repo)
{
    // ⚠️ SemaphoreSlim global = goulot d'étranglement pour des clés différentes
    // → Préférer un SemaphoreSlim par clé via ConcurrentDictionary
    private readonly ConcurrentDictionary<string, SemaphoreSlim> _locks = new();

    public async Task<Product?> GetAsync(int id, CancellationToken ct = default)
    {
        var key = $"p:{id}";
        if (cache.TryGetValue(key, out Product? hit)) return hit;

        var semaphore = _locks.GetOrAdd(key, _ => new SemaphoreSlim(1, 1));
        await semaphore.WaitAsync(ct);
        try
        {
            // Double-check après acquisition du lock
            if (cache.TryGetValue(key, out hit)) return hit;

            var product = await repo.GetByIdAsync(id, ct);
            if (product is not null)
                cache.Set(key, product, TimeSpan.FromMinutes(5));
            return product;
        }
        finally
        {
            semaphore.Release();
            _locks.TryRemove(key, out _);
        }
    }
}
```

**Solution 2 : `Lazy<Task<T>>` + `ConcurrentDictionary` (déduplique les requêtes in-flight)**

```csharp
private readonly ConcurrentDictionary<string, Lazy<Task<Product?>>> _inFlight = new();

public Task<Product?> GetAsync(int id, CancellationToken ct)
{
    var key = $"p:{id}";
    if (cache.TryGetValue(key, out Product? hit)) return Task.FromResult(hit);

    return _inFlight.GetOrAdd(key, _ => new Lazy<Task<Product?>>(async () =>
    {
        var product = await repo.GetByIdAsync(id, ct);
        if (product is not null) cache.Set(key, product, TimeSpan.FromMinutes(5));
        _inFlight.TryRemove(key, out _);
        return product;
    })).Value;
}
```

**Solution 3 : `HybridCache.GetOrCreateAsync` (.NET 9) — protection intégrée**
`HybridCache` déduplique automatiquement les appels concurrents pour la même clé sans code supplémentaire (voir question dédiée).

---

### 🔴 Quelles sont les stratégies d'invalidation de cache ? Laquelle choisir ?

L'invalidation de cache est l'un des problèmes les plus difficiles en informatique (*"There are only two hard things in Computer Science: cache invalidation and naming things"* — Phil Karlton).

| Stratégie | Description | Avantages | Inconvénients |
|---|---|---|---|
| **TTL (Time-To-Live)** | Expiration automatique après un délai | Simple, aucun couplage | Données potentiellement stale jusqu'à expiration |
| **Write-Through** | Mise à jour cache + DB simultanément | Toujours à jour | Latence d'écriture, couplage fort |
| **Write-Behind (Write-Back)** | Écriture en cache d'abord, DB en asynchrone | Écriture ultra-rapide | Risque de perte de données |
| **Event-driven** | Invalidation via message (MassTransit, Redis pub/sub) | Faible latence de propagation | Complexité, at-least-once delivery |
| **Tag-based** | Invalider un groupe d'entrées par tag | Flexible, batch | Nécessite support du cache (HybridCache, OutputCache) |
| **Cache-busting key** | Inclure une version dans la clé (`product:v3:42`) | Simple, sans invalidation | Accumulation de clés obsolètes |

```csharp
// Event-driven : invalider via Redis pub/sub
public class CacheInvalidationHandler(
    IConnectionMultiplexer redis,
    IMemoryCache localCache) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        var sub = redis.GetSubscriber();
        await sub.SubscribeAsync(
            RedisChannel.Literal("cache:invalidate"),
            (_, message) => localCache.Remove(message.ToString()));

        await Task.Delay(Timeout.Infinite, ct);
    }
}

// Publier depuis le service d'écriture
public async Task UpdateProductAsync(Product product, CancellationToken ct)
{
    await repo.UpdateAsync(product, ct);
    var pub = redis.GetSubscriber();
    await pub.PublishAsync(
        RedisChannel.Literal("cache:invalidate"),
        $"product:{product.Id}");
}
```

**Recommandation :** Pour la majorité des cas, TTL + invalidation explicite à l'écriture (*Cache-Aside + remove*) est le meilleur compromis. Passer à l'event-driven uniquement si plusieurs services partagent le même cache et exigent une cohérence quasi-immédiate.

---

### 🔴 Qu'est-ce que `HybridCache` (.NET 9+) ? Quels problèmes résout-il ?

`HybridCache` (`Microsoft.Extensions.Caching.Hybrid`) est la nouvelle abstraction de cache .NET 9 combinant :
- **L1** : cache in-process (`IMemoryCache`) pour l'accès ultra-rapide
- **L2** : cache distribué (`IDistributedCache` / Redis) pour la cohérence multi-instances
- **Protection stampede intégrée** : déduplication automatique des requêtes concurrentes sur la même clé
- **Invalidation par tag** cross L1/L2
- **Sérialisation automatique** via `IHybridCacheSerializer<T>`

```csharp
// dotnet add package Microsoft.Extensions.Caching.Hybrid
builder.Services.AddHybridCache(options =>
{
    options.MaximumPayloadBytes = 1024 * 1024; // 1 MB max par entrée
    options.DefaultEntryOptions = new HybridCacheEntryOptions
    {
        Expiration = TimeSpan.FromMinutes(5),          // TTL L2 (distribué)
        LocalCacheExpiration = TimeSpan.FromMinutes(1) // TTL L1 (in-process)
    };
});

// Optionnel : brancher Redis comme L2
builder.Services.AddStackExchangeRedisCache(opt =>
    opt.Configuration = builder.Configuration.GetConnectionString("Redis"));

// Usage
public class ProductService(HybridCache cache, IProductRepository repo)
{
    public async Task<Product?> GetAsync(int id, CancellationToken ct = default)
        => await cache.GetOrCreateAsync(
            $"product:{id}",
            async cancel => await repo.GetByIdAsync(id, cancel),
            new HybridCacheEntryOptions { Expiration = TimeSpan.FromMinutes(10) },
            tags: [$"product", $"product:{id}"],
            cancellationToken: ct);

    public async Task UpdateAsync(Product product, CancellationToken ct = default)
    {
        await repo.UpdateAsync(product, ct);
        // Invalide L1 + L2 pour toutes les entrées taguées
        await cache.RemoveByTagAsync($"product:{product.Id}", ct);
    }

    public async Task ClearAllProductsAsync(CancellationToken ct = default)
        => await cache.RemoveByTagAsync("product", ct);
}
```

**`HybridCache` vs approche manuelle L1+L2 :**

| | Manuelle (IMemoryCache + IDistributedCache) | HybridCache |
|---|---|---|
| Sérialisation | Code à écrire | Automatique |
| Stampede protection | Code à écrire | Intégrée |
| Tags | Non supportés | ✅ Cross L1/L2 |
| Complexité | Élevée | Faible |
| .NET version | Toutes | .NET 9+ |

> En .NET 9+, `HybridCache` est la solution recommandée pour remplacer `IMemoryCache` + `IDistributedCache` dans les nouveaux projets.

---

### 🔴 Quand NE PAS mettre en cache ? Les pièges courants

**1. Données hautement personnalisées**
Cacher des données spécifiques à chaque utilisateur (panier, notifications) gonfle le cache sans bénéfice de partage. Envisager un TTL très court avec une clé incluant l'userId, ou éviter complètement le cache.

**2. Données en mutation très fréquente**
Si les données changent plus vite que le TTL, le cache ajoute de la complexité sans gain de performance.

**3. Données sensibles dans un cache partagé**
Ne jamais stocker des tokens, permissions ou données personnelles dans un cache distribué non chiffré.

**4. Incohérence entre couches de cache**
```csharp
// ❌ Dangereux : cache une liste ET chaque élément individuellement
// Mettre à jour un élément invalide son entrée individuelle,
// mais la liste reste stale
cache.Set("products:all", products);
foreach (var p in products) cache.Set($"product:{p.Id}", p);

// ✅ Utiliser les tags HybridCache pour invalider les deux niveaux atomiquement
```

**5. Sérialisation coûteuse supérieure au gain**
Si sérialiser un objet complexe prend 50 ms et que la DB répond en 5 ms, le cache distribué est contre-productif.

**6. Tests et environnements de développement**
Toujours permettre de désactiver le cache en dev (env variable, `IOptions<CacheOptions>`) :

```csharp
public class ConditionalCache(IMemoryCache inner, IOptions<CacheSettings> opts)
{
    public bool TryGetValue<T>(string key, out T? value)
    {
        if (!opts.Value.Enabled) { value = default; return false; }
        return inner.TryGetValue(key, out value);
    }
}
```

---

## Lexique

| Terme | Définition |
|---|---|
| **Cache-Aside** | Pattern où l'application gère elle-même le cache : lecture cache → miss → DB → écriture cache. |
| **Cache stampede** | Phénomène où l'expiration simultanée d'une entrée provoque un afflux de requêtes vers la DB. |
| **Dog-piling** | Synonyme de cache stampede. |
| **Eviction** | Suppression d'une entrée du cache (TTL expiré, pression mémoire, invalidation manuelle). |
| **HybridCache** | Abstraction .NET 9 combinant cache L1 (in-process) et L2 (distribué) avec stampede protection intégrée. |
| **IDistributedCache** | Abstraction .NET pour les caches distribués (Redis, SQL Server, NCache). |
| **IMemoryCache** | Abstraction .NET pour le cache in-process (mémoire du process). |
| **Output Cache** | Cache .NET 8+ côté serveur des réponses HTTP, invalidable par tag, transparent pour le client. |
| **Sliding expiration** | TTL se réinitialisant à chaque accès à l'entrée, dans la limite de l'expiration absolue. |
| **Stampede protection** | Mécanisme évitant que plusieurs threads rechargent simultanément la même entrée expirée. |
| **Tag-based invalidation** | Invalidation groupée d'entrées partageant un tag commun. |
| **TTL (Time-To-Live)** | Durée de vie maximale d'une entrée en cache avant expiration automatique. |
| **Write-through** | Stratégie écrivant simultanément en cache et en base de données à chaque mutation. |

---

## Ressources

### Documentation officielle Microsoft
- [IMemoryCache in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/performance/caching/memory)
- [Distributed caching in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/performance/caching/distributed)
- [Output caching middleware (.NET 8+)](https://learn.microsoft.com/en-us/aspnet/core/performance/caching/output)
- [HybridCache in ASP.NET Core (.NET 9)](https://learn.microsoft.com/en-us/aspnet/core/performance/caching/hybrid)
- [StackExchange.Redis documentation](https://stackexchange.github.io/StackExchange.Redis/)

### Blogs & Articles
- [Andrew Lock — Output caching in .NET 8](https://andrewlock.net/adding-output-caching-to-a-net-8-minimal-api/)
- [Nick Chapsas — HybridCache in .NET 9 (YouTube)](https://www.youtube.com/@nickchapsas)
- [Microsoft DevBlogs — Caching guidance](https://devblogs.microsoft.com/dotnet/)
