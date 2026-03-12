# Entretien .NET Senior/Expert — Multi-threading & Concurrence

> **Versions ciblées :** .NET 8 / .NET 9 / .NET 10, C# 12 / C# 13 / C# 14
>
> **Niveaux de difficulté :** 🟢 Basique · 🟡 Intermédiaire · 🔴 Senior/Expert
>
> **Distribution senior :** ~25 % basique, ~25 % intermédiaire, ~50 % senior/expert

---

## Fondamentaux

### 🟢 Quelle est la différence entre un process, un thread et une task en .NET ?

- **Process** : unité d'isolation OS. Mémoire isolée, ressources propres. Coûteux à créer.
- **Thread** : unité d'exécution planifiée par l'OS. Partage la mémoire du process. ~1 MB de stack. Context switch coûteux.
- **Task** : abstraction .NET représentant une **opération asynchrone**. Peut s'exécuter sur un thread du ThreadPool, ou ne pas utiliser de thread du tout (pour I/O async).

```
Process → plusieurs Threads → plusieurs Tasks par Thread
Task ≠ Thread (une Task I/O-bound ne consomme pas de thread pendant l'attente)
```

---

### 🟢 Qu'est-ce que le ThreadPool ? Comment .NET décide-t-il du nombre de threads disponibles ?

Le ThreadPool est un pool de threads worker réutilisables géré par le runtime .NET, évitant le coût de création/destruction des threads.

**Algorithme de scaling :**
- Commence avec un nombre minimal de threads (= nombre de CPU cores)
- Si les threads sont saturés, en injecte de nouveaux (1 par seconde par défaut — *hill climbing*)
- Se réduit si les threads sont oisifs

```csharp
ThreadPool.GetMinThreads(out int workerMin, out int ioMin);
ThreadPool.GetMaxThreads(out int workerMax, out int ioMax);

// Pour les serveurs haute perf : éviter le slow-start
ThreadPool.SetMinThreads(200, 200);
```

> **Problème classique :** blocage synchrone de threads ThreadPool (`.Result`, `.Wait()`) → starvation → latences en cascade.

---

### 🟡 Expliquez la différence entre `Task.Run()`, `Task.Factory.StartNew()` et `new Thread()`

| | `Task.Run()` | `Task.Factory.StartNew()` | `new Thread()` |
|---|---|---|---|
| ThreadPool | ✅ | ✅ (par défaut) | ❌ (thread dédié) |
| Options avancées | ❌ | ✅ (`TaskCreationOptions`) | ✅ |
| Retour de Task | ✅ (`Task`/`Task<T>`) | ✅ | ❌ |
| Usage recommandé | CPU-bound simple | Options spéciales | Jamais (sauf long-running) |

```csharp
// Task.Run : préféré pour du CPU-bound
var result = await Task.Run(() => HeavyComputation());

// Task.Factory.StartNew : nécessaire pour LongRunning (thread dédié hors ThreadPool)
var t = Task.Factory.StartNew(() => InfiniteLoop(),
    TaskCreationOptions.LongRunning);

// new Thread : rare, pour contrôle fin (priorité, apartment state COM)
var thread = new Thread(() => Work()) { IsBackground = true };
thread.Start();
```

---

## Synchronisation

### 🟡 Quelles sont les différentes primitives de synchronisation en .NET ? Leurs différences ?

| Primitive | Scope | Async | Cas d'usage |
|---|---|---|---|
| `lock` / `Monitor` | In-process | ❌ | Section critique simple |
| `System.Threading.Lock` (.NET 9+) | In-process | ❌ | Section critique typée (replace `object`) |
| `Mutex` | Cross-process | ❌ | Protection entre processus |
| `SemaphoreSlim` | In-process | ✅ | Limiter la concurrence (N accès max) |
| `Semaphore` | Cross-process | ❌ | Idem, cross-process |
| `ReaderWriterLockSlim` | In-process | ❌ | Multiple readers, exclusive writers |
| `ManualResetEventSlim` | In-process | ❌ | Signaler un événement one-to-many |
| `AutoResetEvent` | In-process | ❌ | Signaler one-to-one |
| `CountdownEvent` | In-process | ❌ | Attendre N opérations |
| `Barrier` | In-process | ❌ | Synchroniser N threads par phases |

```csharp
// SemaphoreSlim : limiter à 10 appels HTTP parallèles
var semaphore = new SemaphoreSlim(10);
var tasks = urls.Select(async url => {
    await semaphore.WaitAsync();
    try { return await client.GetStringAsync(url); }
    finally { semaphore.Release(); }
});
await Task.WhenAll(tasks);
```

---

### 🔴 Qu'est-ce que le type `Lock` introduit dans .NET 9 ? En quoi est-il meilleur que `lock(object)` ?

.NET 9 introduit `System.Threading.Lock`, un type dédié au verrouillage qui remplace l'utilisation de `lock(new object())`.

```csharp
// Ancien pattern : object non typé, sujet aux erreurs
private readonly object _lock = new();
void DoWork()
{
    lock (_lock) { /* section critique */ }
}

// .NET 9+ : Lock typé
private readonly Lock _lock = new();
void DoWork()
{
    lock (_lock) { /* section critique — le compilateur utilise Lock.EnterScope() */ }
}

// Avantage : API explicite, pas de boxing accidentel
// Le compilateur génère du code optimisé via Lock.EnterScope()
```

**Avantages de `Lock` vs `object` :**
- **Type dédié** : impossible de locker accidentellement sur un objet non prévu (string, Type, this)
- **API enrichie** : `Lock.EnterScope()` retourne un `Lock.Scope` (ref struct) — pas d'allocation
- **Lisibilité** : intention claire dans le code
- **Perf** : le compilateur émet un code plus optimal que `Monitor.Enter/Exit`

> **Attention :** si l'objet `Lock` est assigné à une variable de type `object`, le compilateur retombe sur le pattern `Monitor.Enter/Exit` classique — le bénéfice est perdu.

---

### 🟡 Expliquez ce qu'est un deadlock. Comment le détecter et le prévenir ?

Un deadlock survient quand deux threads s'attendent mutuellement indéfiniment.

```csharp
// Deadlock classique avec deux locks dans un ordre inversé
private static readonly object LockA = new();
private static readonly object LockB = new();

// Thread 1            // Thread 2
lock(LockA)            lock(LockB)
{                      {
    lock(LockB) { }        lock(LockA) { } // deadlock !
}                      }

// Prévention : toujours acquérir les locks dans le même ordre
// Ou utiliser Monitor.TryEnter avec timeout
bool acquired = Monitor.TryEnter(LockA, TimeSpan.FromSeconds(1));

// Deadlock async classique (ASP.NET Classic / UI)
public string GetData() =>
    GetDataAsync().Result; // bloque le thread UI/context

public async Task<string> GetDataAsync()
{
    await Task.Delay(100); // essaie de reprendre sur le context bloqué → deadlock !
    return "data";
}
// Solution : ConfigureAwait(false) dans GetDataAsync
// OU utiliser async tout le long (async all the way down)
```

**Détection :** WinDbg + SOS extension, Visual Studio Diagnostic, `dotnet-dump`.

---

### 🟡 Qu'est-ce qu'une race condition ? Donnez un exemple et montrez comment la corriger.

```csharp
// Problème : _count++ n'est pas atomique (read-modify-write)
private int _count = 0;

void Increment() => _count++; // race condition si appelé concurremment !

// Thread 1 lit _count = 5
// Thread 2 lit _count = 5
// Thread 1 écrit _count = 6
// Thread 2 écrit _count = 6 → une incrémentation est perdue !

// Solution 1 : lock
private readonly object _lock = new();
void Increment() { lock(_lock) { _count++; } }

// Solution 2 : Interlocked (meilleure perf, atomique)
void Increment() => Interlocked.Increment(ref _count);

// Solution 3 : type thread-safe
private ConcurrentDictionary<string, int> _dict = new();
```

---

### 🔴 Qu'est-ce que le mot-clé `volatile` ? Est-ce suffisant pour garantir la thread-safety ?

`volatile` garantit deux choses :
1. Pas de **réordonnancement** des instructions par le compilateur/JIT/CPU autour de la variable
2. Lecture toujours depuis la mémoire principale (pas du cache CPU local)

```csharp
private volatile bool _running = true;

// Thread 1
while (_running) { DoWork(); }

// Thread 2
_running = false; // sans volatile, Thread 1 pourrait ne jamais voir ce changement
```

**Est-ce suffisant pour la thread-safety ? NON.**

`volatile` ne rend pas les opérations composées atomiques :
```csharp
private volatile int _x = 0;

void Increment() => _x++; // toujours une race condition !
                           // (read + increment + write sont 3 opérations)
```

> Utiliser `Interlocked` pour les opérations atomiques, `lock` pour les sections critiques.

---

### 🟡 Expliquez `Interlocked.Increment()`. Quand l'utiliser plutôt qu'un `lock` ?

`Interlocked` utilise des instructions CPU atomiques (CAS - Compare-And-Swap) **sans verrou OS**.

```csharp
// lock : acquire/release mutex → coûteux (~25-100 ns), kernel mode possible
private int _count;
private readonly object _lock = new();
void Inc() { lock(_lock) { _count++; } }

// Interlocked : instruction CPU native → très rapide (~1-5 ns)
private int _count;
void Inc() => Interlocked.Increment(ref _count);

// Autres opérations Interlocked :
Interlocked.Add(ref _sum, value);
Interlocked.CompareExchange(ref _val, newVal, expected); // CAS pattern
Interlocked.Exchange(ref _flag, 1);
```

**Utiliser `Interlocked`** pour compteurs, flags, opérations atomiques simples sur une seule variable.
**Utiliser `lock`** pour sections critiques complexes (plusieurs variables à modifier ensemble de façon cohérente).

---

## Patterns avancés

### 🔴 Qu'est-ce que `Parallel.ForEach()` vs `Parallel.ForEachAsync()` ? Quand utiliser l'un ou l'autre ?

```csharp
// Parallel.ForEach : CPU-bound pur, synchrone — excellent
Parallel.ForEach(data, new ParallelOptions { MaxDegreeOfParallelism = 4 },
    item => ComputeHeavy(item));

// MAUVAIS pour de l'I/O async : bloque des threads ThreadPool !
Parallel.ForEach(urls, url => {
    var result = client.GetStringAsync(url).Result; // thread starvation
});

// Parallel.ForEachAsync (.NET 6+) : I/O-bound async — la bonne approche
await Parallel.ForEachAsync(urls,
    new ParallelOptions { MaxDegreeOfParallelism = 10 },
    async (url, ct) =>
    {
        var result = await client.GetStringAsync(url, ct);
        await ProcessAsync(result, ct);
    });
```

**Comparaison :**

| | `Parallel.ForEach` | `Parallel.ForEachAsync` |
|---|---|---|
| Synchrone/Async | Synchrone | Async (awaitable) |
| Adapté pour | CPU-bound | I/O-bound et CPU-bound async |
| CancellationToken | Via `ParallelOptions` | Via `ParallelOptions` et par item |
| Thread bloqué | Oui (normal pour CPU-bound) | Non (vrai async) |
| Disponible depuis | .NET Framework | .NET 6+ |

> **Règle :** `Parallel.ForEach` → CPU-bound pur. `Parallel.ForEachAsync` → I/O-bound ou mixed. L'ancien pattern `SemaphoreSlim` + `Task.WhenAll` n'est plus nécessaire dans la plupart des cas.

---

### 🔴 Connaissez-vous `TaskCompletionSource<T>` ? Quand et comment l'utiliser ?

`TaskCompletionSource<T>` permet de créer manuellement une `Task<T>` et de contrôler quand elle se complète. C'est le pont entre des APIs callback/événementielles et le monde async/await.

```csharp
// Wrapper d'une API callback-based en async
public Task<string> ReadLineAsync(StreamReader reader)
{
    var tcs = new TaskCompletionSource<string>();

    reader.BaseStream.BeginRead(buffer, 0, buffer.Length, ar =>
    {
        try
        {
            var result = reader.BaseStream.EndRead(ar);
            tcs.SetResult(Encoding.UTF8.GetString(buffer, 0, result));
        }
        catch (Exception ex)
        {
            tcs.SetException(ex);
        }
    }, null);

    return tcs.Task;
}

// Pattern : timeout avec annulation
public static async Task<T> WithTimeout<T>(Task<T> task, TimeSpan timeout)
{
    using var cts = new CancellationTokenSource();
    var delayTask = Task.Delay(timeout, cts.Token);

    var completed = await Task.WhenAny(task, delayTask);
    if (completed == delayTask)
        throw new TimeoutException();

    cts.Cancel(); // annule le timer
    return await task;
}

// RunContinuationsAsynchronously : éviter les deadlocks
var tcs = new TaskCompletionSource<int>(TaskCreationOptions.RunContinuationsAsynchronously);
```

**Points clés :**
- `RunContinuationsAsynchronously` évite que la continuation s'exécute inline sur le thread qui appelle `SetResult`
- Ne jamais appeler `SetResult` dans un context locké — utiliser `RunContinuationsAsynchronously`
- Utile pour wrapper des APIs legacy (événements, callbacks, completion ports)

---

### 🔴 Expliquez le concept de `Lazy<T>` et sa thread-safety. Quand l'utiliser ?

`Lazy<T>` fournit une initialisation **différée** (lazy) avec thread-safety configurable.

```csharp
// Thread-safe par défaut (LazyThreadSafetyMode.ExecutionAndPublication)
private readonly Lazy<ExpensiveService> _service = new(() => new ExpensiveService());

public ExpensiveService Service => _service.Value; // initialisé au premier accès

// Modes disponibles :
// ExecutionAndPublication : un seul thread initialise, les autres attendent (défaut)
// PublicationOnly : plusieurs threads peuvent initialiser, mais un seul résultat est publié
// None : pas de thread-safety (perf maximale si single-threaded)

var lazy = new Lazy<Data>(
    () => LoadExpensiveData(),
    LazyThreadSafetyMode.ExecutionAndPublication);
```

**Cas d'usage :**
- Initialisation coûteuse différée (connexions, caches lourds)
- Singletons thread-safe dans l'injection de dépendances
- Double-checked locking sans écrire le pattern manuellement

**Piège :** si le factory lève une exception, toutes les tentatives futures retournent la même exception (en mode `ExecutionAndPublication`). Utiliser `PublicationOnly` si une nouvelle tentative est souhaitée.

---

### 🔴 Quelles sont les collections thread-safe en .NET ? Quand les utiliser ?

| Collection | Équivalent non-safe | Pattern |
|---|---|---|
| `ConcurrentDictionary<TKey, TValue>` | `Dictionary<TKey, TValue>` | Lock-free reads, fine-grained locking writes |
| `ConcurrentQueue<T>` | `Queue<T>` | Lock-free (CAS) |
| `ConcurrentStack<T>` | `Stack<T>` | Lock-free (CAS) |
| `ConcurrentBag<T>` | `List<T>` (unordered) | Per-thread partitioning |
| `BlockingCollection<T>` | — | Producer/consumer avec blocking |
| `ImmutableDictionary<TKey, TValue>` | — | Structural sharing, snapshot-safe |
| `FrozenDictionary<TKey, TValue>` (.NET 8) | — | Read-only optimisé |

```csharp
// ConcurrentDictionary : pattern GetOrAdd atomique
var cache = new ConcurrentDictionary<string, Data>();
var value = cache.GetOrAdd("key", key => LoadData(key));

// ATTENTION : le factory de GetOrAdd peut être appelé plusieurs fois !
// Pour éviter les initialisations dupliquées :
var lazyCache = new ConcurrentDictionary<string, Lazy<Data>>();
var value = lazyCache.GetOrAdd("key",
    key => new Lazy<Data>(() => LoadData(key))).Value;

// ConcurrentQueue : pattern producer/consumer simple
var queue = new ConcurrentQueue<WorkItem>();
queue.Enqueue(new WorkItem(...));
if (queue.TryDequeue(out var item))
    Process(item);
```

> **Règle :** ne pas utiliser `lock` + `Dictionary` quand `ConcurrentDictionary` suffit. Mais `ConcurrentDictionary` n'est pas toujours le bon choix — pour un cache read-heavy, `FrozenDictionary` ou `ImmutableDictionary` peut être plus adapté.
