# Entretien .NET Senior/Expert — Performances

> **Versions ciblées :** .NET 8 / .NET 9 / .NET 10, C# 12 / C# 13 / C# 14
>
> **Niveaux de difficulté :** 🟢 Basique · 🟡 Intermédiaire · 🔴 Senior/Expert
>
> **Distribution senior :** ~25 % basique, ~25 % intermédiaire, ~50 % senior/expert

---

## Mémoire & Allocations

### 🟢 Expliquez la différence entre la stack et la heap. Comment minimiser les allocations sur la heap ?

- **Stack** : allocation/désallocation en O(1), taille limitée (~1 MB par thread), durée de vie liée au scope
- **Heap** : allocation dynamique, gérée par le GC, cause des pauses GC

**Techniques pour réduire les allocations heap :**

```csharp
// 1. Utiliser struct au lieu de class pour les petits types

// 2. Span<T> / stackalloc pour les buffers temporaires
Span<byte> buffer = stackalloc byte[256]; // sur la stack !

// 3. Object pooling (ArrayPool, ObjectPool)
var array = ArrayPool<byte>.Shared.Rent(1024);
try { /* usage */ }
finally { ArrayPool<byte>.Shared.Return(array); }

// 4. StringBuilder au lieu de string concat en boucle
// 5. Éviter les closures inutiles (capturent des variables → allocation)
// 6. record struct au lieu de record class pour les DTOs temporaires
```

---

### 🟢 Qu'est-ce que le boxing et unboxing ? Quel est leur impact sur les performances ?

**Boxing** : conversion d'un value type → `object` (allocation heap).
**Unboxing** : cast explicite de `object` → value type.

```csharp
int x = 42;
object boxed = x;    // boxing : allocation d'un objet sur la heap
int y = (int)boxed;  // unboxing

// Piège classique : ArrayList (pré-génériques)
var list = new ArrayList();
list.Add(42); // boxing à chaque Add !

// Solution : List<int> — aucun boxing
var typed = new List<int>();
typed.Add(42); // pas de boxing

// Autre piège : interface sur struct
interface IFoo { void Bar(); }
struct MyStruct : IFoo { public void Bar() { } }

IFoo foo = new MyStruct(); // boxing ! La struct est copiée sur la heap
```

> **Impact :** allocation inutile + pression GC + indirection mémoire. Critique dans les hot paths.

---

### 🟡 Qu'est-ce que `Span<T>` et `Memory<T>` ? Dans quel cas les utilisez-vous ?

**`Span<T>`** : vue sur une séquence contiguë de mémoire (array, stack, unmanaged) **sans allocation**. Contraint à la stack (`ref struct`).

**`Memory<T>`** : même concept mais peut vivre sur la heap, utilisable dans les contextes async.

```csharp
// Découper un tableau sans allocation
int[] data = { 1, 2, 3, 4, 5 };
Span<int> slice = data.AsSpan(1, 3); // [2, 3, 4] — zéro allocation

// Parser une string sans substring (pas d'allocation)
ReadOnlySpan<char> text = "Hello, World!".AsSpan();
var hello = text.Slice(0, 5); // "Hello" sans nouvelle string

// Memory<T> pour l'async
async Task ProcessAsync(Memory<byte> buffer) {
    await SomeIoAsync(buffer);
}
```

**Quand les utiliser :** parsing haute performance, manipulation de buffers, remplacement des `byte[]` copiés.

---

### 🟡 Qu'est-ce que `ArrayPool<T>` ? Comment l'utiliser et pourquoi ?

Pool d'arrays réutilisables pour éviter les allocations répétées de grands tableaux.

```csharp
// Sans pool : nouvelle allocation à chaque appel
byte[] buffer = new byte[4096]; // → heap, GC pressure

// Avec ArrayPool
var pool = ArrayPool<byte>.Shared;
byte[] rented = pool.Rent(4096); // taille minimale demandée (peut être plus grande !)
try
{
    int actualLength = await stream.ReadAsync(rented.AsMemory(0, 4096));
    Process(rented.AsSpan(0, actualLength));
}
finally
{
    pool.Return(rented, clearArray: false); // TOUJOURS retourner !
}
```

**Points importants :**
- `Rent(n)` peut retourner un array **plus grand** que demandé → toujours travailler avec la longueur logique
- `Return()` doit être appelé même en cas d'exception → `try/finally`
- `clearArray: true` si le buffer contient des données sensibles

---

### 🔴 Connaissez-vous le concept de `ref struct` ? Pourquoi est-il contraint à la stack ?

Un `ref struct` est un value type qui **ne peut exister que sur la stack**. Le compilateur interdit :
- Le boxing
- L'utilisation comme champ d'une classe
- L'utilisation dans des méthodes async ou des lambdas capturantes
- Depuis **C# 13 / .NET 9** : un `ref struct` peut implémenter des interfaces, mais ne peut **jamais être boxé** vers le type interface. Utilisable uniquement via un paramètre générique contraint par `allows ref struct`.

```csharp
ref struct StackOnly
{
    public Span<int> Data;
}

// C# 13 : ref struct implémentant une interface
ref struct MyRefStruct : IDisposable
{
    public void Dispose() { /* nettoyage */ }
}

// Utilisable via un générique contraint
void Process<T>(T value) where T : allows ref struct, IDisposable
{
    value.Dispose(); // pas de boxing
}
```

**Pourquoi cette contrainte ?**
`Span<T>` contient un pointeur vers de la mémoire qui peut être invalide si l'objet est déplacé sur la heap par le GC. En restant sur la stack, la durée de vie est garantie et le GC n'y touche pas.

> **`Span<T>` est lui-même un `ref struct`** — c'est pour ça qu'il ne peut pas être utilisé dans les méthodes `async`.

---

### 🔴 Qu'est-ce que `FrozenDictionary<TKey, TValue>` et `FrozenSet<T>` (.NET 8+) ? Quand les utiliser ?

Collections **immuables et optimisées en lecture** introduites dans `System.Collections.Frozen` (.NET 8). Elles ont un coût de construction élevé mais un temps de lookup considérablement plus rapide que `Dictionary` ou `HashSet`.

```csharp
using System.Collections.Frozen;

// Construction lente (acceptable au démarrage)
var config = new Dictionary<string, string>
{
    ["key1"] = "value1",
    ["key2"] = "value2",
    // ... des milliers d'entrées
}.ToFrozenDictionary();

// Lecture ultra-rapide (optimisation interne selon la taille et le type de clé)
var value = config["key1"]; // plus rapide que Dictionary<TKey, TValue>

// FrozenSet : même principe
var allowedRoles = new[] { "admin", "manager", "editor" }.ToFrozenSet(StringComparer.OrdinalIgnoreCase);
bool isAllowed = allowedRoles.Contains(role);
```

**Cas d'usage idéaux :**
- Données de configuration chargées une seule fois au démarrage
- Tables de lookup statiques (codes pays, mappings, feature flags)
- Caches read-only partagés entre threads

**Pourquoi plus rapide ?** La construction analyse la distribution des clés et choisit la meilleure stratégie de hashing interne, voire utilise du branching direct pour les petites collections.

---

### 🔴 Qu'est-ce que `SearchValues<T>` (.NET 8+) ? Quel problème résout-il ?

`SearchValues<T>` optimise la recherche de caractères ou d'octets dans un `Span<T>` en utilisant les meilleures instructions CPU disponibles (SIMD, vectorisation).

```csharp
using System.Buffers;

// Au lieu de :
private static readonly char[] Separators = [' ', '\t', '\n', '\r', ',', ';'];

// Utiliser SearchValues (calcul des optimisations à la construction) :
private static readonly SearchValues<char> SeparatorValues =
    SearchValues.Create([' ', '\t', '\n', '\r', ',', ';']);

// Lookup vectorisé — beaucoup plus rapide sur de longues strings
int idx = span.IndexOfAny(SeparatorValues);
bool containsAny = span.ContainsAny(SeparatorValues);
```

**Utilisation typique :** parsers, tokenizers, validateurs de chaînes dans les hot paths.

> Sur du texte de plusieurs KB, la différence avec `IndexOfAny(char[])` peut être d'un facteur 2x à 10x grâce à la vectorisation automatique.

---

## Garbage Collector

### 🟡 Expliquez le fonctionnement du Garbage Collector en .NET. Quelles sont les différentes générations ?

Le GC de .NET est un GC **générationnel** basé sur l'hypothèse que les objets jeunes meurent vite.

- **Gen 0** : objets fraîchement alloués. Collecte très fréquente et rapide.
- **Gen 1** : objets ayant survécu à une collecte Gen 0. Tampon entre Gen 0 et Gen 2.
- **Gen 2** : objets long-lived (singletons, caches...). Collecte coûteuse (Full GC), suspend tous les threads managés (*stop-the-world*).
- **LOH (Large Object Heap)** : objets ≥ 85 000 bytes. Traités comme Gen 2, non compactés par défaut.
- **POH (Pinned Object Heap)** : .NET 5+, pour les objets épinglés (évite de fragmenter les autres heaps).

**Modes :** Workstation GC (single-thread, interactif) vs Server GC (multi-thread, un heap par CPU core, débit maximal).

> **Indicateur d'un senior :** mentionner que `GC.Collect()` est à éviter, parler de la pression sur le GC via les allocations évitables, connaître `GCSettings.LargeObjectHeapCompactionMode`, et savoir que le Server GC Gen 2 a un mode concurrent (background GC).

---

### 🔴 Qu'est-ce que le GC Regions et Dynamic PGO ? Impact sur les performances ?

**GC Regions** (défaut depuis .NET 8) :
Remplace l'ancien modèle de segments par des **régions mémoire** plus petites (~4 MB). Avantages :
- Meilleure utilisation mémoire (moins de fragmentation)
- Temps de pause GC réduits (collecte par régions)
- Scaling amélioré sur les machines avec beaucoup de RAM

**Dynamic PGO (Profile-Guided Optimization)** (défaut depuis .NET 9) :
Le JIT collecte des données d'exécution en runtime pour **recompiler les méthodes** chaudes avec des optimisations basées sur le profil réel.

```
Sans PGO : le JIT optimise "en aveugle" au premier passage
Avec Dynamic PGO : le Tier-0 collecte un profil, puis le Tier-1 recompile avec :
  - Dévirtualisation spéculative (inline des méthodes virtuelles)
  - Branch layout optimisé (branches chaudes en premier)
  - Inlining plus agressif
  - Guard specialization
```

Configuration :
```xml
<!-- .csproj ou runtimeconfig.json -->
<PropertyGroup>
  <TieredPGO>true</TieredPGO>  <!-- Défaut depuis .NET 9 -->
</PropertyGroup>
```

> **Impact mesuré :** 10-20% d'amélioration des performances sur les workloads typiques ASP.NET Core (TechEmpower benchmarks).

---

## Compilation & JIT

### 🟡 Qu'est-ce que le JIT ? Connaissez-vous la différence entre JIT et AOT (NativeAOT) ?

**JIT (Just-In-Time) :**
- Compilation IL → code natif **au moment de l'exécution**
- Peut optimiser selon le CPU cible réel
- Warm-up nécessaire (cold start)
- Tiered compilation : code rapide d'abord, puis recompilé avec optimisations complètes

**AOT (Ahead-Of-Time) / NativeAOT :**
- Compilation complète **avant le déploiement**
- Démarrage quasi-instantané (serverless, CLI tools)
- Pas de JIT runtime → pas de tiered compilation
- Contraintes : pas de reflection dynamique, trimming requis, taille binaire fixe

```
JIT       → flexibilité, optimisations runtime, warm-up lent
NativeAOT → démarrage rapide, binaire autonome, contraintes reflection
R2R       → compromis (pré-compilation partielle, JIT encore présent)
```

---

### 🔴 Connaissez-vous les Source Generators ? Quel problème de performance résolvent-ils ?

Les Source Generators s'exécutent **à la compilation** et génèrent du code C# supplémentaire, remplaçant les usages coûteux de Reflection à l'exécution.

**Problèmes résolus :**
- Sérialisation JSON sans reflection (`System.Text.Json` source gen)
- Regex compilées (`[GeneratedRegex(...)]`)
- Logging structuré sans boxing (`[LoggerMessage]`)
- DI auto-registration, mappers, validators...

```csharp
// Avant (reflection à l'exécution)
JsonSerializer.Serialize(myObj); // découverte des propriétés à l'exécution

// Après (source generator)
[JsonSerializable(typeof(MyClass))]
partial class MyJsonContext : JsonSerializerContext { }

JsonSerializer.Serialize(myObj, MyJsonContext.Default.MyClass); // code généré à la compilation
```

**Avantages :** zéro reflection runtime, compatible NativeAOT, meilleures performances, erreurs détectées à la compilation.

---

### 🔴 Quand et comment utilisez-vous `[AggressiveInlining]` ou `[SkipLocalsInit]` ?

**`[MethodImpl(MethodImplOptions.AggressiveInlining)]`** : demande au JIT d'inliner la méthode à l'endroit de l'appel (évite le coût du call stack). Utile pour les micro-méthodes dans les hot paths.

```csharp
[MethodImpl(MethodImplOptions.AggressiveInlining)]
static int Add(int a, int b) => a + b; // sera inliné par le JIT
```

**`[SkipLocalsInit]`** : par défaut, le runtime initialise les variables locales à zéro (requis par la spec CIL). Cet attribut saute cette initialisation — utile avec `stackalloc`.

```csharp
[SkipLocalsInit]
void FastMethod()
{
    Span<byte> buffer = stackalloc byte[256]; // pas de zero-init → légèrement plus rapide
    // ATTENTION : le buffer contient des données indéterminées !
    buffer.Clear(); // si nécessaire, initialiser manuellement
}
```

> Ces attributs sont des **micro-optimisations de dernier recours**, mesurables uniquement avec BenchmarkDotNet dans un contexte identifié.

---

## Strings & Collections

### 🟢 Qu'est-ce que `StringBuilder` et dans quel cas l'utiliser plutôt que la concaténation de `string` ?

Les `string` sont **immuables** en C#. Chaque `+` crée une nouvelle instance.

```csharp
// Mauvais : O(n²) en temps et mémoire
string result = "";
for (int i = 0; i < 10000; i++)
    result += i.ToString(); // nouvelle string à chaque itération !

// Bon : O(n)
var sb = new StringBuilder(capacity: 10000 * 4);
for (int i = 0; i < 10000; i++)
    sb.Append(i);
string result = sb.ToString();
```

> **Exception :** pour 2-3 concaténations simples hors boucle, `string.Concat()` ou interpolation est optimisé par le compilateur — `StringBuilder` est superflu.
>
> **C# 10+ :** `string.Create()` et les interpolated string handlers (`DefaultInterpolatedStringHandler`) permettent des interpolations sans allocation intermédiaire.

---

### 🟡 Quand utiliseriez-vous `HashSet<T>` plutôt que `List<T>` ?

| Opération | `List<T>` | `HashSet<T>` |
|---|---|---|
| Contient (Contains) | O(n) | O(1) |
| Ajout | O(1) amorti | O(1) amorti |
| Suppression | O(n) | O(1) |
| Ordre | Préservé | Non garanti |
| Doublons | Autorisés | Interdits |

**Utiliser `HashSet<T>` quand :**
- Vérification fréquente de présence (`Contains`)
- Dédoublonnage d'une collection
- Opérations ensemblistes : `UnionWith`, `IntersectWith`, `ExceptWith`

```csharp
// Mauvais : O(n²) pour 10 000 éléments
var list = new List<int>(data);
foreach (var x in otherData)
    if (list.Contains(x)) // O(n) à chaque appel !
        Process(x);

// Bon : O(n)
var set = new HashSet<int>(data);
foreach (var x in otherData)
    if (set.Contains(x)) // O(1)
        Process(x);
```

---

### 🟡 Quelle est la différence entre `IQueryable<T>` et `IEnumerable<T>` dans un contexte EF Core ?

| | `IEnumerable<T>` | `IQueryable<T>` |
|---|---|---|
| Exécution | En mémoire (client-side) | Traduit en SQL (server-side) |
| Provider | LINQ to Objects | LINQ to Entities (EF Core) |
| Expression tree | Non | Oui |

```csharp
// IQueryable : le WHERE est traduit en SQL → filtre côté base
var q = dbContext.Users
    .Where(u => u.Age > 18) // SQL: WHERE Age > 18
    .ToList();

// IEnumerable : charge TOUT en mémoire, filtre en C#
IEnumerable<User> all = dbContext.Users.AsEnumerable();
var filtered = all.Where(u => u.Age > 18); // filtre en mémoire !
```

> **Piège classique :** appeler `.AsEnumerable()` ou `.ToList()` trop tôt dans la chaîne LINQ force le chargement complet en mémoire avant le filtre.

---

## Profiling & Benchmarking

### 🟡 Quels outils utilisez-vous pour profiler une application .NET ?

| Outil | Type | Usage |
|---|---|---|
| **BenchmarkDotNet** | Micro-benchmark | Comparer des implémentations précisément |
| **dotMemory** (JetBrains) | Mémoire | Allocations, fuites mémoire |
| **dotTrace** (JetBrains) | CPU | Call tree, hot spots |
| **PerfView** (Microsoft) | ETW / bas niveau | GC analysis, CPU sampling |
| **Visual Studio Diagnostic Tools** | Intégré | Premier diagnostic rapide |
| **dotnet-counters** | CLI | Métriques runtime en temps réel |
| **dotnet-trace** | CLI | Traces cross-platform |
| **dotnet-dump** | CLI | Analyse de crash dump |
| **Application Insights / OpenTelemetry** | Production | Profiling par sampling en prod |

> **Réponse senior attendue :** distinguer profiling **CPU** (où le temps est passé) vs profiling **mémoire** (allocations, fuites) et savoir choisir le bon outil selon le symptôme.

---

### 🟡 Qu'est-ce que BenchmarkDotNet et comment l'utilisez-vous ?

```csharp
[MemoryDiagnoser]
[SimpleJob(RuntimeMoniker.Net90)]
public class StringBenchmarks
{
    private const int N = 1000;

    [Benchmark(Baseline = true)]
    public string Concatenation()
    {
        var s = string.Empty;
        for (int i = 0; i < N; i++) s += i;
        return s;
    }

    [Benchmark]
    public string WithStringBuilder()
    {
        var sb = new StringBuilder();
        for (int i = 0; i < N; i++) sb.Append(i);
        return sb.ToString();
    }
}
// dotnet run -c Release
```

**Points clés :**
- Toujours exécuter en mode `Release`
- `[MemoryDiagnoser]` pour voir les allocations (Gen0, Gen1, bytes alloués)
- Éviter les benchmarks triviaux qui se font optimiser/éliminer par le JIT

---

## Optimisations JIT & Compilateur

### 🟡 Que fait le mot-clé `sealed` ? Quel est son impact sur les performances ?

Empêche l'héritage d'une `class` ou la surcharge d'un membre `virtual` dans une classe dérivée.

```csharp
sealed class FinalImpl : BaseClass { } // ne peut plus être héritée

class Base {
    public virtual void Method() { }
}
class Derived : Base {
    public sealed override void Method() { } // ne peut plus être surchargée
}
```

**Impact performance :** le JIT peut **devirtualiser** les appels sur un type `sealed` (pas besoin de vtable lookup). Sur les types fréquemment alloués avec des méthodes virtuelles, `sealed` peut être un gain mesurable.

> Avec Dynamic PGO (.NET 9+), le JIT effectue de la **dévirtualisation spéculative** même sans `sealed`, mais `sealed` reste une garantie statique plus fiable.

---

## I/O & Réseau

### 🟢 Quelle est la différence entre I/O synchrone et asynchrone en termes de threads et de performances ?

**Synchrone :**
- Le thread est **bloqué** pendant toute la durée de l'opération I/O
- Sur 100 requêtes simultanées → 100 threads bloqués → pression sur le ThreadPool → context switches coûteux

**Asynchrone :**
- Le thread est **libéré** pendant l'attente I/O (géré par l'OS via IOCP sur Windows)
- Un seul thread peut gérer des centaines d'opérations I/O en cours
- Le thread reprend à la continuation après le `await`

```csharp
// Synchrone : thread bloqué pendant la lecture réseau
string data = File.ReadAllText("file.txt");

// Asynchrone : thread libéré, reprend à la fin
string data = await File.ReadAllTextAsync("file.txt");
```

> **Règle :** `async/await` apporte un bénéfice sur les opérations **I/O-bound**. Pour du **CPU-bound**, il n'y a pas de gain — utiliser `Task.Run()` pour ne pas bloquer le thread appelant.

---

### 🟡 Qu'est-ce que `IHttpClientFactory` et pourquoi ne faut-il pas instancier `HttpClient` directement ?

**Problèmes de l'instanciation directe de `HttpClient` :**

1. **Socket exhaustion** : `HttpClient` dispose les connexions HTTP, mais les sockets restent en état `TIME_WAIT` pendant ~4 minutes → épuisement des ports disponibles sous charge
2. **DNS stale** : une instance longue durée ne renouvelle pas la résolution DNS → problèmes lors de changements d'IP

```csharp
// Mauvais : nouvelle instance à chaque requête
using var client = new HttpClient(); // socket exhaustion !
var result = await client.GetAsync(url);

// IHttpClientFactory : gestion du cycle de vie des HttpClientHandler
// Les handlers sont poolés et recréés périodiquement (2 min par défaut)
public class MyService(IHttpClientFactory factory)
{
    public async Task<string> GetAsync()
    {
        var client = factory.CreateClient("MyApi");
        return await client.GetStringAsync("/endpoint");
    }
}

// Program.cs
builder.Services.AddHttpClient("MyApi", c => {
    c.BaseAddress = new Uri("https://api.example.com");
    c.Timeout = TimeSpan.FromSeconds(30);
});
```

> **Avancé :** typed clients, `Polly` pour le retry/circuit-breaker, `DelegatingHandler` pour la télémétrie.

---

### 🔴 Qu'est-ce que `System.IO.Pipelines` ? Quand l'utiliser plutôt que `Stream` ?

`System.IO.Pipelines` est un framework pour le traitement de données binaires à **très haute performance**, conçu pour résoudre les problèmes classiques des Streams : allocations de buffers, copies inutiles, gestion complexe du backpressure.

```csharp
// Stream classique : alloue un buffer, lit, parse, recommence
var buffer = new byte[4096]; // allocation
int read = await stream.ReadAsync(buffer);
// Problème : et si le message chevauche deux reads ?
// → buffer management complexe, copies manuelles

// System.IO.Pipelines : le runtime gère les buffers
var pipe = new Pipe();

// Writer (producteur)
async Task FillPipeAsync(Stream stream, PipeWriter writer)
{
    while (true)
    {
        Memory<byte> memory = writer.GetMemory(4096); // buffer du pool
        int bytesRead = await stream.ReadAsync(memory);
        if (bytesRead == 0) break;
        writer.Advance(bytesRead);

        FlushResult result = await writer.FlushAsync(); // backpressure
        if (result.IsCompleted) break;
    }
    await writer.CompleteAsync();
}

// Reader (consommateur)
async Task ReadPipeAsync(PipeReader reader)
{
    while (true)
    {
        ReadResult result = await reader.ReadAsync();
        ReadOnlySequence<byte> buffer = result.Buffer;

        // Parser le protocole sans copie
        while (TryParseMessage(ref buffer, out var message))
            ProcessMessage(message);

        reader.AdvanceTo(buffer.Start, buffer.End); // signale ce qui a été consommé
        if (result.IsCompleted) break;
    }
    await reader.CompleteAsync();
}
```

**Avantages vs Stream :**
- Zéro copie (les données restent dans des buffers poolés)
- Backpressure natif (le producteur ralentit si le consommateur est lent)
- Gestion automatique des messages qui chevauchent les limites de buffer
- Utilisé en interne par Kestrel pour les performances record d'ASP.NET Core

---

### 🔴 Qu'est-ce que la Reflection et quel est son coût ? Quelles sont les alternatives performantes ?

La Reflection permet d'**inspecter et manipuler dynamiquement les métadonnées** (types, propriétés, méthodes) à l'exécution via `System.Reflection`.

```csharp
var type = typeof(MyClass);
var props = type.GetProperties();
var method = type.GetMethod("MyMethod");
method.Invoke(instance, new object[] { arg1 }); // lent + boxing des args
```

**Coût :**
- Pas de JIT inline possible
- Allocations à chaque appel (`object[]` pour les arguments, boxing des value types)
- Late-binding → erreurs runtime uniquement
- Incompatible avec NativeAOT/Trimming

**Alternatives performantes :**

| Alternative | Quand |
|---|---|
| Source Generators | Sérialisation, logging, mappers (compile-time) |
| `Expression<T>` compilé | Génération dynamique de delegates, ORM légers |
| `Delegate.CreateDelegate` | Cacher un appel reflection en delegate rapide |
| `UnsafeAccessor` (.NET 8+) | Accès direct à des membres privés sans reflection |
| Interceptors (C# 12) | Remplacement d'appels à la compilation |

```csharp
// UnsafeAccessor (.NET 8) — accès à un champ privé sans reflection ni boxing
[UnsafeAccessor(UnsafeAccessorKind.Field, Name = "_privateField")]
extern static ref int GetPrivateField(MyClass instance);

var value = GetPrivateField(obj); // accès direct, zéro overhead
```
