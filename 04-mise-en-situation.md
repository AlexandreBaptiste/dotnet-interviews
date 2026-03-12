# Entretien .NET Senior/Expert — Mises en situation

> **Versions ciblées :** .NET 8 / .NET 9 / .NET 10, C# 12 / C# 13 / C# 14
>
> **Niveaux de difficulté :** 🟢 Basique · 🟡 Intermédiaire · 🔴 Senior/Expert
>
> **Distribution senior :** ~25 % basique, ~25 % intermédiaire, ~50 % senior/expert

---

## 🟡 Scénario 1 — `await Task.Delay` vs `Thread.Sleep`

> *"Quelle est la différence entre `await Task.Delay(1000)` et `Thread.Sleep(1000)` dans une méthode async ?"*

### Comparaison

| | `Thread.Sleep(1000)` | `await Task.Delay(1000)` |
|---|---|---|
| Thread bloqué | ✅ Oui | ❌ Non |
| ThreadPool impacté | ✅ Oui | ❌ Non |
| Annulable | ❌ Non | ✅ Oui (`CancellationToken`) |
| Utilisable dans async | ⚠️ Antipattern | ✅ Recommandé |
| Mécanisme | Suspend le thread OS | Timer OS + callback |

### Impact à l'échelle

```
100 requêtes simultanées avec Thread.Sleep(1000) :
  → 100 threads bloqués
  → ThreadPool injecte de nouveaux threads (1/seconde)
  → Latence en cascade sur toutes les requêtes suivantes
  → Consommation mémoire (~1 MB de stack par thread)

100 requêtes simultanées avec await Task.Delay(1000) :
  → 0 thread bloqué pendant le délai
  → Les threads du pool traitent d'autres requêtes
  → Les 100 continuations reprennent après 1 seconde
```

---

## 🟡 Scénario 2 — `lock(this)`

> *"On vous montre ce code : `lock(this) { ... }`. Quel est le problème potentiel ?"*

### Le problème

```csharp
public class MyService
{
    public void DoWork()
    {
        lock(this) { /* section critique */ } // PROBLÈME !
    }
}
```

`this` est une référence **accessible depuis l'extérieur** de la classe. N'importe quel code externe peut locker sur cette même instance :

```csharp
var svc = new MyService();

// Thread 1 (code externe)
lock(svc)
{
    Thread.Sleep(5000); // détient le verrou sur svc pendant 5 secondes
}

// Thread 2, pendant ce temps
svc.DoWork(); // bloqué ! lock(this) == lock(svc) — même objet !
```

**Conséquences :**
- **Contention non intentionnelle** : du code externe ralentit en contenant le lock
- **Deadlock** si le code externe acquiert d'autres locks dans un ordre différent
- **Violation d'encapsulation** : le mécanisme de synchronisation interne est exposé

### La solution

Toujours locker sur un **objet privé dédié**, non accessible de l'extérieur :

```csharp
public class MyService
{
    private readonly object _lock = new object(); // objet privé

    public void DoWork()
    {
        lock(_lock) { /* section critique sûre */ }
    }
}

// .NET 9+ : utiliser System.Threading.Lock
public class MyService
{
    private readonly Lock _lock = new();

    public void DoWork()
    {
        lock(_lock) { /* section critique typée et optimisée */ }
    }
}
```

> **Variante à mentionner :** `lock(typeof(MyService))` est encore pire — le verrou est partagé entre **toutes les instances** de la classe, et `Type` est accessible globalement.

---

## 🔴 Scénario 3 — API lente sous charge

> *"Vous avez une API qui répond en 2 secondes sous faible charge, mais commence à dépasser les 10 secondes à partir de 100 requêtes simultanées. Par où commencez-vous pour investiguer ?"*

### Approche méthodique

**Étape 1 — Métriques runtime en temps réel**

```bash
dotnet-counters monitor --process-id <pid> \
  --counters System.Runtime,Microsoft.AspNetCore.Hosting
```

Observer : CPU, mémoire, threads actifs, queue length ThreadPool, nb de requêtes en vol.

**Étape 2 — ThreadPool starvation ?**

Si le nombre de threads actifs est élevé et que la queue s'allonge, suspect n°1 = **code bloquant** :
```csharp
// Antipatterns à rechercher en code review
var result = GetDataAsync().Result;  // bloque un thread ThreadPool
var result = GetDataAsync().GetAwaiter().GetResult();
Task.Run(() => ...).Wait();
Thread.Sleep(...); // dans du code async
```

**Étape 3 — Contention base de données**

- Nb de connexions actives vs pool size (`Max Pool Size` dans la connection string)
- Slow queries (SQL Server : sys.dm_exec_query_stats, Query Store)
- Deadlocks DB (SQL Server Profiler, Extended Events)
- N+1 queries avec EF Core → activer le logging SQL :
```csharp
options.LogTo(Console.WriteLine, LogLevel.Information)
       .EnableSensitiveDataLogging();
```

**Étape 4 — GC pressure**

```bash
dotnet-counters monitor --counters System.Runtime[gc-heap-size,gen-0-gc-count,gen-2-gc-count]
```

Gen 2 collections fréquentes ou LOH fragmentation → profiling mémoire avec dotMemory ou PerfView.

**Étape 5 — Contention applicative**

- `lock` contentés : rechercher les locks tenus longtemps
- `SemaphoreSlim` trop restrictif
- Ressources partagées non scalables (ex: singleton avec état mutable)

**Étape 6 — Profiling CPU**

dotTrace (call tree) ou `dotnet-trace` pour identifier les hot spots CPU réels.

**Étape 7 — Infrastructure**

- Limites réseau, DNS, load balancer
- Timeout TCP / retries en cascade
- Saturation du pool de connexions HTTP (`IHttpClientFactory`)

### Résumé décisionnel

```
Threads saturés ?      → ThreadPool starvation (code bloquant)
DB lente ?             → Slow queries, pool connexions, N+1
Mémoire en hausse ?    → GC pressure, fuite mémoire
CPU élevé ?            → Algo inefficace, trop d'allocations, sérialisation
Tout semble normal ?   → Contention externe (réseau, DNS, service tiers)
```

---

## 🔴 Scénario 4 — Traitement de 1 million de lignes

> *"Comment structureriez-vous un traitement de 1 million de lignes venant d'une base de données en minimisant l'empreinte mémoire ?"*

### Approche naïve à éviter

```csharp
// Charge 1 million d'objets en RAM d'un coup
var allRows = await dbContext.Data.ToListAsync();
foreach (var row in allRows)
    await ProcessAsync(row);
```

### Solution 1 — Streaming simple avec `IAsyncEnumerable`

```csharp
await foreach (var row in dbContext.Data
    .AsNoTracking()       // pas de change tracking EF Core → moins de mémoire
    .AsAsyncEnumerable()) // streaming ligne par ligne
{
    await ProcessAsync(row);
}
```

Empreinte mémoire : **constante** (une ligne à la fois).

### Solution 2 — Traitement par batch

```csharp
const int BatchSize = 1000;

await foreach (var batch in dbContext.Data
    .AsNoTracking()
    .AsAsyncEnumerable()
    .Chunk(BatchSize)) // batch de 1000 éléments
{
    await BulkProcessAsync(batch);
}
```

### Solution 3 — Pipeline producteur/consommateurs avec `Channel<T>`

Pour maximiser le débit avec traitement parallèle sans saturer la mémoire :

```csharp
var channel = Channel.CreateBounded<DataRow>(capacity: 1000); // backpressure

// Producteur : lit depuis la DB en streaming
var producer = Task.Run(async () =>
{
    try
    {
        await foreach (var row in dbContext.Data.AsNoTracking().AsAsyncEnumerable())
            await channel.Writer.WriteAsync(row);
    }
    finally
    {
        channel.Writer.Complete();
    }
});

// N consommateurs en parallèle
var consumers = Enumerable.Range(0, 4).Select(_ => Task.Run(async () =>
{
    await foreach (var row in channel.Reader.ReadAllAsync())
        await ProcessAsync(row);
}));

await Task.WhenAll(new[] { producer }.Concat(consumers));
```

### Optimisations supplémentaires

```csharp
// Ne sélectionner que les colonnes nécessaires
var query = dbContext.Data
    .AsNoTracking()
    .Select(r => new { r.Id, r.Name, r.Value }); // projection

// Pour des mises à jour bulk : ExecuteUpdateAsync (EF Core 7+)
await dbContext.Data
    .Where(r => r.Status == "pending")
    .ExecuteUpdateAsync(s => s.SetProperty(r => r.Status, "processed"));
// → UPDATE SQL direct, sans charger les entités en mémoire
```

---

## 🔴 Scénario 5 — Fuite mémoire en production

> *"Votre application ASP.NET Core consomme de plus en plus de mémoire en production (de 200 MB à 2 GB en 24h) sans augmentation de charge. Comment diagnostiquez-vous et résolvez-vous le problème ?"*

### Approche de diagnostic

**Étape 1 — Confirmer la fuite (pas juste la pression GC)**

```bash
# Vérifier la mémoire managée vs commit mémoire
dotnet-counters monitor --process-id <pid> \
  --counters System.Runtime[gc-heap-size,gc-committed,gen-2-gc-count]
```

Si `gc-heap-size` croît et que les Gen 2 sont fréquentes → fuite managée.
Si la mémoire commit croît mais pas le heap → fuite native (handles, mémoire non managée).

**Étape 2 — Capture de dumps mémoire**

```bash
# Deux dumps à intervalle pour comparer
dotnet-dump collect -p <pid> --output dump1.dmp
# Attendre 1h
dotnet-dump collect -p <pid> --output dump2.dmp
```

**Étape 3 — Analyse des dumps**

```bash
dotnet-dump analyze dump2.dmp
> dumpheap -stat                    # quels types consomment le plus ?
> dumpheap -type MyClass            # combien d'instances de ce type ?
> gcroot <address>                  # qui empêche le GC de collecter cet objet ?
```

### Causes fréquentes et solutions

```csharp
// 1. Event handlers non désinscrits
myService.DataChanged += OnDataChanged;
// → les instances s'accumulent car l'event maintient une référence
// Solution : -= dans Dispose() ou utiliser WeakEventManager

// 2. ConcurrentDictionary/cache sans politique d'éviction
private static readonly ConcurrentDictionary<string, Data> _cache = new();
// → grossit indéfiniment
// Solution : IMemoryCache avec SlidingExpiration / SizeLimit

// 3. HttpClient mal géré
using var client = new HttpClient(); // dans une boucle → socket exhaustion + allocation
// Solution : IHttpClientFactory

// 4. Timer non disposé dans un service DI scoped
public class ScopedService
{
    private readonly Timer _timer = new(callback, null, 0, 1000); // fuite !
    // → le timer garde une référence au callback → l'instance n'est jamais collectée
    // Solution : IDisposable + disposer le timer
}

// 5. Closures capturant des objets lourds
var bigData = LoadHugeFile();
var tasks = items.Select(item => Task.Run(() =>
{
    Process(item, bigData); // bigData capturé par la closure → pas collecté
}));
// Solution : passer par paramètre, ou réduire la portée de la capture
```

---

## 🔴 Scénario 6 — Race condition subtile dans un singleton

> *"Ce code est un cache singleton thread-safe. Un bug intermittent cause des données incohérentes. Trouvez le problème."*

```csharp
public class CacheService
{
    private readonly Dictionary<string, CacheEntry> _cache = new();
    private readonly SemaphoreSlim _semaphore = new(1, 1);

    public async Task<string> GetOrAddAsync(string key, Func<Task<string>> factory)
    {
        if (_cache.TryGetValue(key, out var entry) && !entry.IsExpired)
            return entry.Value; // ⚠️ lecture sans lock

        await _semaphore.WaitAsync();
        try
        {
            // Double-check
            if (_cache.TryGetValue(key, out entry) && !entry.IsExpired)
                return entry.Value;

            var value = await factory();
            _cache[key] = new CacheEntry(value, DateTimeOffset.UtcNow.AddMinutes(5));
            return value;
        }
        finally
        {
            _semaphore.Release();
        }
    }
}
```

### Le problème

La **lecture initiale** (avant le lock) accède à un `Dictionary<>` **non thread-safe** pendant qu'un autre thread peut être en train d'écrire à l'intérieur du lock → **comportement indéfini** (corruption de structure interne, `NullReferenceException`, boucle infinie lors du resize).

### La solution

```csharp
public class CacheService
{
    private readonly ConcurrentDictionary<string, Lazy<Task<CacheEntry>>> _cache = new();

    public async Task<string> GetOrAddAsync(string key, Func<Task<string>> factory)
    {
        var lazy = _cache.GetOrAdd(key, k =>
            new Lazy<Task<CacheEntry>>(async () =>
            {
                var value = await factory();
                return new CacheEntry(value, DateTimeOffset.UtcNow.AddMinutes(5));
            }));

        var entry = await lazy.Value;
        if (entry.IsExpired)
        {
            // Renouveler : remplacer atomiquement
            var newLazy = new Lazy<Task<CacheEntry>>(async () =>
            {
                var value = await factory();
                return new CacheEntry(value, DateTimeOffset.UtcNow.AddMinutes(5));
            });
            _cache.TryUpdate(key, newLazy, lazy);
            entry = await newLazy.Value;
        }

        return entry.Value;
    }
}
```

> **Ou plus simplement :** utiliser `IMemoryCache` / `HybridCache` (.NET 9+) qui gère tout cela nativement.

---

## 🔴 Scénario 7 — Optimiser un endpoint critique

> *"Ce endpoint est appelé 10 000 fois/seconde. Le profiling montre beaucoup d'allocations Gen 0. Comment l'optimiser ?"*

```csharp
// Code initial
[HttpGet("status/{id}")]
public async Task<IActionResult> GetStatus(string id)
{
    var user = await _dbContext.Users
        .Include(u => u.Permissions)
        .FirstOrDefaultAsync(u => u.Id == id);

    if (user == null) return NotFound();

    var response = new StatusResponse
    {
        Name = $"{user.FirstName} {user.LastName}",
        Role = user.Permissions.Select(p => p.Name).ToList(),
        LastSeen = user.LastLogin.ToString("yyyy-MM-dd HH:mm:ss")
    };

    return Ok(response);
}
```

### Optimisations

```csharp
// 1. Projection SQL au lieu de charger l'entité + relations
// 2. Cache pour les données chaudes
// 3. Réduction des allocations string

[HttpGet("status/{id}")]
public async Task<IActionResult> GetStatus(string id)
{
    // IMemoryCache ou HybridCache (.NET 9) avec sliding expiration
    var response = await _cache.GetOrCreateAsync($"status:{id}", async entry =>
    {
        entry.SlidingExpiration = TimeSpan.FromSeconds(30);

        // Projection : SQL ciblé, pas de load d'entités
        return await _dbContext.Users
            .Where(u => u.Id == id)
            .Select(u => new StatusResponse
            {
                Name = string.Concat(u.FirstName, " ", u.LastName),
                Role = u.Permissions.Select(p => p.Name).ToArray(),
                LastSeen = u.LastLogin
            })
            .FirstOrDefaultAsync();
    });

    return response is null ? NotFound() : Ok(response);
}

// StatusResponse optimisé
public sealed class StatusResponse // sealed → devirtualisation JIT
{
    public required string Name { get; init; }
    public required string[] Role { get; init; } // array au lieu de List
    public required DateTime LastSeen { get; init; } // garder DateTime, formater au JSON
}
```

**Gains ciblés :**
- **Projection SQL** : réduit I/O DB et allocations EF Core (pas de change tracking)
- **Cache** : 0 query DB pour les données chaudes
- **`sealed`** : devirtualisation JIT possible
- **`string.Concat`** : évite l'allocation de `FormattableString` de l'interpolation
- **`DateTime` au lieu de `string`** : le serializer JSON formate sans allocation intermédiaire

---

## Lexique

| Terme | Définition |
|---|---|
| **AsNoTracking** | Option EF Core désactivant le change tracking, réduisant la mémoire et les allocations. |
| **Backpressure** | Mécanisme de contrôle de flux où le consommateur signale au producteur de ralentir. |
| **Change tracking** | Système EF Core surveillant les modifications des entités chargées pour générer des UPDATE SQL. |
| **Channel\<T\>** | File async producteur/consommateur avec backpressure native. |
| **dotnet-counters** | Outil CLI pour observer les métriques runtime .NET en temps réel (CPU, GC, threads). |
| **dotnet-dump** | Outil CLI pour capturer et analyser des dumps mémoire de processus .NET. |
| **ExecuteUpdateAsync** | Méthode EF Core 7+ exécutant un UPDATE SQL sans charger les entités en mémoire. |
| **GC root** | Référence (variable locale, champ statique, handle) empêchant le GC de collecter un objet. |
| **HybridCache** | API de cache unifiée (.NET 9+) combinant cache L1 in-memory et L2 distribué. |
| **IAsyncEnumerable\<T\>** | Interface pour consommer un flux de données async, item par item, sans tout charger en RAM. |
| **IMemoryCache** | Interface de cache in-memory avec expiration et éviction automatique. |
| **N+1 queries** | Anti-pattern ORM : 1 requête pour les entités parentes + N requêtes pour les enfants. |
| **Projection** | Sélection d'un sous-ensemble de colonnes dans la requête SQL (via `.Select()`), évitant le chargement complet. |
| **Socket exhaustion** | Épuisement des ports réseau causé par la création répétée de `HttpClient`. |
| **ThreadPool starvation** | Saturation du pool de threads par des opérations bloquantes, empêchant les nouvelles requêtes d'être traitées. |
| **TIME_WAIT** | État TCP où un socket reste ouvert ~4 min après fermeture, contribuant au socket exhaustion. |

---

## Ressources

### Documentation officielle Microsoft
- [ASP.NET Core performance best practices](https://learn.microsoft.com/en-us/aspnet/core/performance/performance-best-practices)
- [.NET diagnostics tools](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/)
- [IHttpClientFactory guidelines](https://learn.microsoft.com/en-us/dotnet/core/extensions/httpclient-factory)
- [EF Core performance tips](https://learn.microsoft.com/en-us/ef/core/performance/)
- [HybridCache (.NET 9)](https://learn.microsoft.com/en-us/aspnet/core/performance/caching/hybrid)

### Outils de diagnostic
- [dotnet-counters](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/dotnet-counters) — métriques runtime
- [dotnet-dump](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/dotnet-dump) — analyse de dumps
- [dotnet-trace](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/dotnet-trace) — traces cross-platform
- [PerfView](https://github.com/microsoft/perfview) — GC analysis, CPU sampling

### Blogs & Articles
- [David Fowler — ASP.NET Core diagnostic scenarios](https://github.com/davidfowl/AspNetCoreDiagnosticScenarios)
- [Steve Gordon — HttpClient best practices](https://www.stevejgordon.co.uk/)
- [Andrew Lock — ASP.NET Core in Action](https://andrewlock.net/)
