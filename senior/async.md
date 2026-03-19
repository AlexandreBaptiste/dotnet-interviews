# Entretien .NET Senior/Expert — Programmation Asynchrone

> **Versions ciblées :** .NET 8 / .NET 9 / .NET 10, C# 12 / C# 13 / C# 14
>
> **Niveaux de difficulté :** 🟢 Basique · 🟡 Intermédiaire · 🔴 Senior/Expert

---

## Fondamentaux async/await

### 🟢 Qu'est-ce que `async`/`await` ? Comment fonctionne la state machine générée par le compilateur ?

Le compilateur transforme une méthode `async` en une **state machine** implémentant `IAsyncStateMachine`. Chaque `await` crée un point de suspension.

```csharp
// Code écrit
async Task<int> GetValueAsync()
{
    var a = await Step1Async();
    var b = await Step2Async(a);
    return a + b;
}

// Ce que le compilateur génère (simplifié)
struct GetValueAsyncStateMachine : IAsyncStateMachine
{
    public int _state; // -1 = running, 0 = after step1, 1 = after step2
    public AsyncTaskMethodBuilder<int> _builder;
    private int _a, _b;

    public void MoveNext()
    {
        switch (_state)
        {
            case -1: // première entrée
                var awaiter1 = Step1Async().GetAwaiter();
                if (!awaiter1.IsCompleted) {
                    _state = 0;
                    _builder.AwaitUnsafeOnCompleted(ref awaiter1, ref this);
                    return; // libère le thread !
                }
                goto case 0;
            case 0:
                _a = awaiter1.GetResult();
                var awaiter2 = Step2Async(_a).GetAwaiter();
                // ...
        }
    }
}
```

> **Point clé :** si l'awaitable est déjà complété (`IsCompleted = true`), aucune suspension n'a lieu — c'est pourquoi `ValueTask` existe.

---

### 🟢 Quelle est la différence entre `Task.WhenAll()` et `Task.WhenAny()` ? Donnez des cas d'usage.

```csharp
// WhenAll : attend que TOUTES les tasks soient terminées
// Cas d'usage : appels parallèles indépendants
var t1 = FetchUserAsync(userId);
var t2 = FetchOrdersAsync(userId);
var t3 = FetchAddressAsync(userId);
await Task.WhenAll(t1, t2, t3);
// accès aux résultats via t1.Result, t2.Result, t3.Result (déjà complétés)

// WhenAny : retourne dès que LA PREMIÈRE task est terminée
// Cas d'usage 1 : timeout
var dataTask = FetchDataAsync();
var timeoutTask = Task.Delay(TimeSpan.FromSeconds(5));
if (await Task.WhenAny(dataTask, timeoutTask) == timeoutTask)
    throw new TimeoutException();

// Cas d'usage 2 : cache-aside avec fallback rapide
var fastCache = CheckCacheAsync();
var slowDb = QueryDatabaseAsync();
var first = await Task.WhenAny(fastCache, slowDb);
```

> **Attention `WhenAll`** : si une task lève une exception, toutes les exceptions sont agrégées dans `AggregateException`. Avec `await`, seule la première est propagée — utiliser `Task.WhenAll` + try/catch pour toutes les capturer.

---

### 🟡 Qu'est-ce que `ConfigureAwait(false)` et quand doit-on l'utiliser ?

Par défaut, après un `await`, l'exécution reprend sur le `SynchronizationContext` capturé (thread UI dans WPF/WinForms, contexte ASP.NET Classic).

`ConfigureAwait(false)` indique de ne **pas** reprendre sur le contexte capturé.

```csharp
// Librairie : toujours utiliser ConfigureAwait(false)
public async Task<Data> GetDataAsync()
{
    var raw = await httpClient.GetAsync(url).ConfigureAwait(false);
    var content = await raw.Content.ReadAsStringAsync().ConfigureAwait(false);
    return Parse(content);
}

// Application UI : NE PAS utiliser ConfigureAwait(false)
// si on doit accéder à des éléments UI après l'await
private async void Button_Click(...)
{
    var data = await GetDataAsync(); // reprend sur le UI thread ✅
    myLabel.Text = data.ToString();  // accès UI thread-safe
}
```

> **ASP.NET Core** : pas de `SynchronizationContext` → `ConfigureAwait(false)` n'a pas d'effet fonctionnel, mais reste une bonne pratique pour les libs portables.

---

## Patterns async avancés

### 🟡 Qu'est-ce que `CancellationToken` ? Comment l'intégrer proprement dans une chaîne d'appels async ?

```csharp
// Source : crée et contrôle le token
using var cts = new CancellationTokenSource();
cts.CancelAfter(TimeSpan.FromSeconds(30)); // timeout automatique

// Propagation dans une chaîne async
public async Task<Result> ProcessAsync(CancellationToken ct = default)
{
    ct.ThrowIfCancellationRequested(); // vérification synchrone en début de méthode

    var data = await FetchDataAsync(ct);            // propager à chaque appel !
    var processed = await TransformAsync(data, ct);
    return await SaveAsync(processed, ct);
}

// Combiner plusieurs tokens (ex: timeout + annulation utilisateur)
using var linkedCts = CancellationTokenSource.CreateLinkedTokenSource(
    userToken, timeoutToken);
```

**Bonnes pratiques :**
- Toujours propager le token à tous les appels async
- `ThrowIfCancellationRequested()` dans les boucles CPU-bound longues
- Attraper `OperationCanceledException`, pas `TaskCanceledException`
- Ne jamais ignorer le token dans les méthodes async publiques

---

### 🔴 Qu'est-ce que `ValueTask<T>` ? Dans quel cas l'utiliser plutôt que `Task<T>` ?

`Task<T>` alloue toujours un objet sur la heap. `ValueTask<T>` évite cette allocation quand le résultat est **déjà disponible synchronement** (cas fréquent avec caches).

```csharp
// Task<T> : alloue toujours, même si la valeur est en cache
async Task<int> GetWithCache_Task()
{
    if (_cache.TryGetValue(key, out var v)) return v; // alloue quand même un Task !
    return await FetchFromDbAsync();
}

// ValueTask<T> : zéro allocation si synchrone (cache hit)
async ValueTask<int> GetWithCache_ValueTask()
{
    if (_cache.TryGetValue(key, out var v)) return v; // pas d'allocation !
    return await FetchFromDbAsync();
}
```

**Quand utiliser `ValueTask` :**
- Méthodes souvent complétées synchronement (cache hit, fast path)
- Hot paths avec des millions d'appels

**Quand ne PAS l'utiliser :**
- Méthodes presque toujours async (overhead inutile)
- Si la Task est attendue plusieurs fois (`ValueTask` ne supporte pas le multi-await)
- Si la Task est stockée pour être réutilisée

> **Piège d'implémentation :** un `ValueTask` ne doit être **awaité qu'une seule fois** et ne doit pas être stocké. Pour les cas où on a besoin de réutiliser un résultat, utiliser `.AsTask()` d'abord.

---

### 🔴 Connaissez-vous `IAsyncEnumerable<T>` ? Quel est son avantage par rapport à `Task<IEnumerable<T>>` ?

Permet de **consommer un flux de données de façon asynchrone**, item par item, sans charger tout en mémoire.

```csharp
// Task<IEnumerable<T>> : attend que TOUT soit chargé avant de retourner
async Task<IEnumerable<User>> GetAllUsers()
{
    return await dbContext.Users.ToListAsync(); // charge 1 million de lignes en RAM !
}

// IAsyncEnumerable<T> : lit et yield au fur et à mesure
async IAsyncEnumerable<User> StreamUsersAsync(
    [EnumeratorCancellation] CancellationToken ct = default)
{
    await foreach (var user in dbContext.Users.AsAsyncEnumerable().WithCancellation(ct))
        yield return user;
}

// Consommation
await foreach (var user in StreamUsersAsync(ct))
    await ProcessUserAsync(user);
```

**Avantages :** empreinte mémoire constante, premiers résultats disponibles immédiatement, annulation à la volée.

> **ASP.NET Core :** les controller actions peuvent retourner `IAsyncEnumerable<T>` — Kestrel streamera le JSON au client au fur et à mesure.

---

### 🔴 Connaissez-vous les `Channel<T>` ? Quel problème résolvent-ils ?

Un Channel est une file thread-safe **producteur/consommateur** avec backpressure native, remplaçant avantageusement `BlockingCollection<T>` dans les contextes async.

```csharp
// Bounded : backpressure — le producteur attend si la file est pleine
var channel = Channel.CreateBounded<WorkItem>(new BoundedChannelOptions(100)
{
    FullMode = BoundedChannelFullMode.Wait
});

// Producteur
await channel.Writer.WriteAsync(new WorkItem(...), ct);
channel.Writer.Complete(); // signale la fin de la production

// Consommateur
await foreach (var item in channel.Reader.ReadAllAsync(ct))
{
    await ProcessAsync(item);
}

// Multi-consommateurs : la répartition est automatique
var consumers = Enumerable.Range(0, 4).Select(_ => Task.Run(async () =>
{
    await foreach (var item in channel.Reader.ReadAllAsync(ct))
        await ProcessAsync(item);
}));
await Task.WhenAll(consumers);
```

**Bounded vs Unbounded :**

| | `CreateBounded<T>(n)` | `CreateUnbounded<T>()` |
|---|---|---|
| Mémoire | Bornée | Illimitée (OOM possible) |
| Backpressure | Oui (producteur bloqué/drop) | Non |
| `FullMode` | `Wait`, `DropNewest`, `DropOldest`, `DropWrite` | N/A |
| Usage | Production (contrôle mémoire) | Quand le consommateur est toujours plus rapide |

**Résout :** découplage producteur/consommateur, contrôle du débit (backpressure), pas de polling, entièrement async.

---

### 🔴 Qu'est-ce qu'un `SynchronizationContext` ? Pourquoi est-il important ?

Un `SynchronizationContext` capture l'environnement d'exécution pour que les continuations async reprennent sur le bon thread/contexte.

| Contexte | SynchronizationContext | Comportement |
|---|---|---|
| WPF / WinForms | `DispatcherSynchronizationContext` | Reprend sur le UI thread |
| ASP.NET Core | `null` | Reprend sur un thread du ThreadPool |

```csharp
// WPF : après un await, on est de retour sur le UI thread
private async void Button_Click(object sender, RoutedEventArgs e)
{
    var data = await LoadDataAsync(); // suspend, libère le UI thread
    // ici, on est revenu sur le UI thread grâce au SynchronizationContext
    myTextBox.Text = data; // accès UI safe ✅
}

// Dans une librairie : ConfigureAwait(false) pour ne pas capturer le contexte
public async Task<string> GetAsync()
{
    return await httpClient.GetStringAsync(url).ConfigureAwait(false);
}
```

> **Deadlock classique :** une méthode async capturant le contexte, appelée depuis ce même contexte avec `.Result` ou `.Wait()` → le contexte est bloqué et ne peut pas reprendre → deadlock.

---

### 🔴 Qu'est-ce que `PeriodicTimer` ? Pourquoi le préférer à `Timer` ou `Task.Delay` en boucle ?

`PeriodicTimer` (.NET 6+) fournit un timer async déterministe, non-réentrant et disposable, idéal pour les tâches périodiques dans les `BackgroundService`.

```csharp
// Pattern ancien : Task.Delay en boucle (problèmes de drift et réentrance)
while (!ct.IsCancellationRequested)
{
    await DoWorkAsync(ct);
    await Task.Delay(TimeSpan.FromSeconds(30), ct); // drift : le temps de DoWork s'ajoute
}

// Pattern ancien : System.Threading.Timer (callback-based, réentrant !)
var timer = new Timer(callback, null, TimeSpan.Zero, TimeSpan.FromSeconds(30));
// Problème : si callback prend plus de 30s, deux exécutions se chevauchent

// PeriodicTimer (.NET 6+) : async, non-réentrant, propre
public class MyBackgroundService : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        using var timer = new PeriodicTimer(TimeSpan.FromSeconds(30));

        while (await timer.WaitForNextTickAsync(ct))
        {
            // Non-réentrant : le prochain tick n'arrive jamais avant
            // que ce bloc ne soit terminé
            await DoWorkAsync(ct);
        }
    }
}
```

**Avantages :**
- **Non-réentrant** : impossible que deux exécutions se chevauchent
- **Async natif** : pas de callback, pas de thread bloqué
- **Disposable** : nettoyage propre
- **CancellationToken** : annulation propre intégrée

---

## Anti-patterns et bonnes pratiques

### 🟢 Quelle est la différence entre `await Task.Delay` et `Thread.Sleep` dans une méthode async ?

| | `Thread.Sleep(1000)` | `await Task.Delay(1000)` |
|---|---|---|
| Thread bloqué | ✅ Oui | ❌ Non |
| ThreadPool impacté | ✅ Oui | ❌ Non |
| Annulable | ❌ Non | ✅ Oui (`CancellationToken`) |
| Utilisable dans async | ⚠️ Antipattern | ✅ Recommandé |
| Mécanisme | Suspend le thread OS | Timer OS + callback |

```csharp
// Thread.Sleep : thread du ThreadPool bloqué 1 seconde
async Task BadDelayAsync()
{
    Thread.Sleep(1000); // ANTIPATTERN dans un contexte async !
    await DoWorkAsync();
}

// Task.Delay : le thread est rendu au ThreadPool, repris via un timer OS
async Task GoodDelayAsync(CancellationToken ct = default)
{
    await Task.Delay(1000, ct); // thread libre pendant 1 seconde
    await DoWorkAsync();
}
```

> **`Thread.Sleep` dans du code async = antipattern majeur** — à relever immédiatement en code review.

---

### 🟡 Quels sont les principaux anti-patterns async en .NET ?

```csharp
// 1. Sync-over-async : bloquer un thread sur une Task
var data = GetDataAsync().Result;       // ❌ deadlock potentiel
var data = GetDataAsync().GetAwaiter().GetResult(); // ❌ même problème

// 2. Async void : pas d'exception handling, pas d'await
async void BadMethod() { await DoAsync(); } // ❌ exceptions non observées
async Task GoodMethod() { await DoAsync(); } // ✅

// 3. Thread.Sleep dans du code async
async Task Bad() { Thread.Sleep(1000); }   // ❌ thread bloqué
async Task Good() { await Task.Delay(1000); } // ✅

// 4. Ignorer le CancellationToken
async Task Bad(CancellationToken ct) {
    await httpClient.GetAsync(url); // ❌ token pas propagé !
}
async Task Good(CancellationToken ct) {
    await httpClient.GetAsync(url, ct); // ✅
}

// 5. Capturer le contexte inutilement dans les librairies
// (voir ConfigureAwait(false))

// 6. Fire-and-forget sans gestion d'erreur
_ = DoWorkAsync(); // ❌ exception silencieuse
_ = Task.Run(async () => {
    try { await DoWorkAsync(); }
    catch (Exception ex) { logger.LogError(ex, "..."); } // ✅
});

// 7. Utiliser Task.Run pour wrapper du code async (async-over-sync illusion)
Task<string> Fake() => Task.Run(async () => await GetDataAsync()); // ❌ inutile
Task<string> Good() => GetDataAsync(); // ✅
```

---

### 🔴 Comment implémenter un retry avec backoff exponentiel en async ?

```csharp
// Implémentation manuelle
public static async Task<T> RetryWithBackoffAsync<T>(
    Func<CancellationToken, Task<T>> operation,
    int maxRetries = 3,
    CancellationToken ct = default)
{
    var delay = TimeSpan.FromMilliseconds(200);

    for (int attempt = 0; ; attempt++)
    {
        try
        {
            return await operation(ct);
        }
        catch (HttpRequestException) when (attempt < maxRetries)
        {
            // Backoff exponentiel avec jitter pour éviter le thundering herd
            var jitter = TimeSpan.FromMilliseconds(Random.Shared.Next(0, 100));
            await Task.Delay(delay + jitter, ct);
            delay *= 2; // double le délai à chaque tentative
        }
    }
}

// Utilisation
var result = await RetryWithBackoffAsync(
    ct => httpClient.GetStringAsync(url, ct),
    maxRetries: 3, ct);

// En production : utiliser Microsoft.Extensions.Http.Resilience (.NET 8+)
// basé sur Polly v8
builder.Services.AddHttpClient("MyApi")
    .AddStandardResilienceHandler(); // retry, circuit-breaker, timeout intégrés
```

> **En production**, préférer la pipeline de résilience intégrée dans `Microsoft.Extensions.Http.Resilience` (.NET 8+) plutôt qu'un retry maison. Elle fournit retry, circuit-breaker, hedging et timeout out-of-the-box, basée sur `Polly v8`.

---

### 🔴 Comment gérer proprement les exceptions dans `Task.WhenAll` ?

```csharp
// Par défaut, await ne propage que la PREMIÈRE exception
try
{
    await Task.WhenAll(task1, task2, task3);
}
catch (Exception ex)
{
    // ex = seulement la première exception !
}

// Pour capturer TOUTES les exceptions :
var allTasks = Task.WhenAll(task1, task2, task3);
try
{
    await allTasks;
}
catch
{
    // allTasks.Exception contient l'AggregateException avec TOUTES les erreurs
    foreach (var ex in allTasks.Exception!.InnerExceptions)
    {
        logger.LogError(ex, "Task failed");
    }
}

// Pattern robuste avec résultats partiels
var tasks = urls.Select(async url =>
{
    try
    {
        return (Url: url, Data: await FetchAsync(url), Error: (Exception?)null);
    }
    catch (Exception ex)
    {
        return (Url: url, Data: (string?)null, Error: ex);
    }
}).ToArray();

var results = await Task.WhenAll(tasks);
var successes = results.Where(r => r.Error is null);
var failures = results.Where(r => r.Error is not null);
```

---

## Lexique

| Terme | Définition |
|---|---|
| **AggregateException** | Exception conteneur regroupant plusieurs exceptions, typiquement levée par `Task.WhenAll`. |
| **async void** | Méthode async sans Task de retour : les exceptions ne sont pas observables. Réservé aux event handlers UI. |
| **Awaitable** | Tout objet exposant `GetAwaiter()` avec `IsCompleted`, `GetResult()` et `OnCompleted()`. |
| **Awaiter** | Objet retourné par `GetAwaiter()` qui gère la suspension et la reprise d'une opération async. |
| **Backpressure** | Mécanisme où le consommateur signale au producteur de ralentir quand il ne peut pas suivre le rythme. |
| **Bounded Channel** | Channel avec une capacité maximale : le producteur attend/drop si la file est pleine. |
| **CancellationToken** | Jeton coopératif permettant de demander l'annulation d'une opération async. |
| **CancellationTokenSource** | Objet créant et contrôlant un `CancellationToken`, avec support du timeout. |
| **Channel\<T\>** | File async producteur/consommateur thread-safe avec backpressure native (`System.Threading.Channels`). |
| **ConfigureAwait** | Configure si la continuation async reprend sur le `SynchronizationContext` capturé ou non. |
| **Continuation** | Code exécuté après la completion d'un `await`, planifié par l'infrastructure async. |
| **Fire-and-forget** | Pattern où une Task est lancée sans être awaitée — risque de perte d'exceptions silencieuse. |
| **IAsyncEnumerable\<T\>** | Interface permettant de consommer un flux async item par item avec `await foreach`. |
| **IOCP** | I/O Completion Ports : mécanisme Windows pour gérer des I/O async sans bloquer de threads. |
| **OperationCanceledException** | Exception levée quand un `CancellationToken` est activé, signalant une annulation coopérative. |
| **PeriodicTimer** | Timer async non-réentrant (.NET 6+) conçu pour les tâches périodiques dans les `BackgroundService`. |
| **Polly** | Bibliothèque de résilience pour .NET : retry, circuit-breaker, timeout, hedging. |
| **State machine** | Structure générée par le compilateur pour chaque méthode `async`, gérant les points de suspension. |
| **SynchronizationContext** | Abstraction capturant l'environnement d'exécution (UI thread, HTTP context) pour les continuations. |
| **Sync-over-async** | Anti-pattern consistant à bloquer un thread sur une Task (`.Result`, `.Wait()`). |
| **TaskCompletionSource** | Permet de créer manuellement une `Task` et de contrôler quand elle se complète. |
| **Thundering herd** | Problème où de nombreux clients retentent simultanément une opération, surchargeant le serveur. |
| **ValueTask\<T\>** | Struct awaitable évitant l'allocation heap quand le résultat est déjà disponible. |

---

## Ressources

### Documentation officielle Microsoft
- [Asynchronous programming with async and await](https://learn.microsoft.com/en-us/dotnet/csharp/asynchronous-programming/)
- [Task-based asynchronous pattern (TAP)](https://learn.microsoft.com/en-us/dotnet/standard/asynchronous-programming-patterns/task-based-asynchronous-pattern-tap)
- [Channels — System.Threading.Channels](https://learn.microsoft.com/en-us/dotnet/core/extensions/channels)
- [IAsyncEnumerable\<T\>](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.iasyncenumerable-1)
- [Microsoft.Extensions.Http.Resilience](https://learn.microsoft.com/en-us/dotnet/core/resilience/http-resilience)

### Livres
- *Concurrency in C# Cookbook* (2nd ed.) — Stephen Cleary
- *Async in C# 5.0* — Alex Davies

### Blogs & Articles
- [Stephen Cleary — async best practices](https://blog.stephencleary.com/)
- [David Fowler — ASP.NET Core diagnostic scenarios](https://github.com/davidfowl/AspNetCoreDiagnosticScenarios)
- [Stephen Toub — How async/await really works](https://devblogs.microsoft.com/dotnet/how-async-await-really-works/)
- [Stephen Toub — ConfigureAwait FAQ](https://devblogs.microsoft.com/dotnet/configureawait-faq/)
