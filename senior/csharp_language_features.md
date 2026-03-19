# Entretien .NET Senior/Expert — Fonctionnalités du langage C#

> **Versions ciblées :** .NET 8 / .NET 9 / .NET 10, C# 12 / C# 13 / C# 14
>
> **Niveaux de difficulté :** 🟢 Basique · 🟡 Intermédiaire · 🔴 Senior/Expert

---

## Fondamentaux

### 🟢 Quelle est la différence entre `record`, `record struct` et `class` ?

`record` (C# 9) est un type de référence conçu pour les **données immuables** avec sémantique de valeur :
- Égalité structurelle automatique (compare les propriétés, pas la référence)
- `ToString()` lisible généré automatiquement
- Déconstruction automatique
- Expression `with` pour la copie non-destructive

```csharp
// record (type référence, C# 9) : sémantique de valeur
public record OrderDto(Guid Id, string CustomerName, decimal Amount);

var o1 = new OrderDto(Guid.NewGuid(), "Alice", 99m);
var o2 = o1 with { CustomerName = "Bob" }; // copie avec mutation d'une propriété
// o1 != o2 (CustomerName différent)

// record struct (type valeur, C# 10) : sur la stack, zéro allocations heap
public record struct Point(int X, int Y);

// class : sémantique de référence (égalité = même adresse mémoire par défaut)
public class OrderClass { public Guid Id { get; set; } }
var c1 = new OrderClass { Id = Guid.NewGuid() };
var c2 = new OrderClass { Id = c1.Id };
Console.WriteLine(c1 == c2); // false (deux objets différents en mémoire)

// record : égalité par valeur
var r1 = new OrderDto(Guid.NewGuid(), "Alice", 99m);
var r2 = new OrderDto(r1.Id, "Alice", 99m);
Console.WriteLine(r1 == r2); // true (mêmes valeurs de propriétés)
```

**Quand utiliser quoi :**

| Type | Usage idéal |
|---|---|
| `record` | DTOs, value objects de domaine, résultats de requêtes |
| `record struct` | Petits value objects fréquemment alloués (réduire pression GC) |
| `class` | Entités mutables avec identité (services, repositories, entités EF) |
| `struct` | Types très petits, hot paths performance, interop |

---

### 🟢 Comment fonctionne le pattern matching en C# ? Quelles formes existent ?

Le pattern matching (C# 7 → C# 13) permet d'inspecter la forme et la valeur d'une expression de façon déclarative dans `switch expressions`, `is`, et les clauses `when`.

```csharp
// 1. Type pattern
object obj = new Order(Guid.NewGuid(), "Alice", 99m);
if (obj is Order order) Console.WriteLine(order.CustomerName);

// 2. Switch expression (C# 8) — retourne une valeur
string Describe(Shape shape) => shape switch
{
    Circle c when c.Radius > 10 => "Large circle",
    Circle c                    => "Small circle",
    Rectangle { Width: var w, Height: var h } when w == h => "Square",
    Rectangle r                 => $"Rectangle {r.Width}x{r.Height}",
    _                           => "Unknown"
};

// 3. Property pattern (C# 8)
bool IsExpiredOrder(Order o)
    => o is { Status: OrderStatus.Pending, CreatedAt: var d }
       && d < DateTime.UtcNow.AddDays(-30);

// 4. Positional pattern (déconstruction)
bool IsOrigin(Point p) => p is (0, 0);

// 5. List patterns (C# 11)
bool StartsWithOneTwo(int[] arr) => arr is [1, 2, ..];
bool HasExactlyTwo(int[] arr)    => arr is [_, _];
int FirstElement(int[] arr)      => arr is [var first, ..] ? first : -1;
int[] TailElements(int[] arr)    => arr is [_, .. var rest] ? rest : [];

// 6. Logical patterns (C# 9)
bool IsValidAge(int age)    => age is >= 0 and <= 120;
bool IsWeekend(DayOfWeek d) => d is DayOfWeek.Saturday or DayOfWeek.Sunday;

// 7. Nested patterns
bool IsHighValuePendingOrder(object obj)
    => obj is Order { Status: OrderStatus.Pending, Amount: > 1000m, Customer.IsVip: true };
```

---

### 🟡 Que sont les Nullable Reference Types (NRT) ? Comment les utiliser efficacement ?

Les **NRT** (C# 8, activés via `<Nullable>enable</Nullable>`) permettent au compilateur de distinguer les types référence nullables (`string?`) des non-nullables (`string`), éliminant les `NullReferenceException` à la compilation.

```csharp
// Avec NRT activé
string nonNullable = "Alice";        // Garantie non-null par le compilateur
string? nullable   = null;           // Explicitement nullable

// Opérateurs null-safe
int len   = nullable?.Length ?? 0;   // null-conditional + null-coalescing
string up = nullable!.ToUpper();     // null-forgiving operator (! = "je garantis non-null")

// Flow analysis : le compilateur suit l'état de nullabilité
public void Process(string? input)
{
    if (input is null) throw new ArgumentNullException(nameof(input));
    Console.WriteLine(input.Length); // input est non-null ici (flow analysis)
}

// Attributs d'annotation pour les cas complexes
public bool TryFind(int id, [NotNullWhen(true)] out Order? order)
{
    order = _orders.TryGetValue(id, out var o) ? o : null;
    return order is not null;
}

// [return: MaybeNull] : retourner null même si le type est non-nullable (ex : default(T))
[return: MaybeNull]
public T GetOrDefault<T>(T defaultValue = default!) => defaultValue;

// [NotNull] : garantir qu'une valeur nullable ne sera pas null après l'appel
public static void ThrowIfNull([NotNull] object? value, string paramName)
    => ArgumentNullException.ThrowIfNull(value, paramName);
```

**Migration progressive :**
```xml
<!-- Activer par projet dans .csproj -->
<Nullable>enable</Nullable>

<!-- Traiter les avertissements de nullabilité comme erreurs -->
<WarningsAsErrors>nullable</WarningsAsErrors>
```

> Ne pas abuser du `!` (null-forgiving) : chaque `!` est une dette technique qui masque un potentiel `NullReferenceException`. Préférer la vérification explicite ou le pattern matching.

---

### 🟡 Que sont les `required` members et les init-only setters (C# 11 / C# 9) ?

**Init-only setters** (C# 9) : propriétés assignables uniquement lors de la construction (constructeur ou initialiseur d'objet) — après quoi elles sont immuables.

```csharp
public class CustomerDto
{
    public Guid   Id   { get; init; }           // init-only
    public string Name { get; init; } = "";     // valeur par défaut
}

var dto = new CustomerDto { Id = Guid.NewGuid(), Name = "Alice" };
// dto.Name = "Bob"; ❌ CS8852 : init-only property
```

**Required members** (C# 11) : force l'appelant à fournir la valeur, sans passer par un constructeur paramétré.

```csharp
public class CreateOrderRequest
{
    public required string  CustomerName { get; init; } // DOIT être fourni
    public required decimal Amount       { get; init; }
    public string?          Notes        { get; init; } // optionnel
}

// ✅ OK
var req = new CreateOrderRequest { CustomerName = "Alice", Amount = 99m };
// ❌ CS9035 : Required member 'CustomerName' must be set
var invalid = new CreateOrderRequest { Amount = 99m };

// [SetsRequiredMembers] : indiquer qu'un constructeur satisfait tous les required
public class CreateOrderRequest
{
    [SetsRequiredMembers]
    public CreateOrderRequest(string customerName, decimal amount)
    {
        CustomerName = customerName;
        Amount = amount;
    }

    public required string  CustomerName { get; init; }
    public required decimal Amount       { get; init; }
}
```

**Avantage vs constructeur :** Plus lisible pour les objets avec de nombreuses propriétés optionnelles, et compatible avec les sérialiseurs et ORMs nécessitant un constructeur sans paramètre.

---

### 🟡 Que sont les collection expressions (C# 12) ?

Les **collection expressions** (C# 12) sont une syntaxe unifiée `[...]` pour créer des collections — applicable à `List<T>`, `T[]`, `ImmutableArray<T>`, `Span<T>`, etc. Le compilateur choisit la représentation optimale selon le type cible.

```csharp
// Syntaxe unifiée pour tous les types collection
int[]               array     = [1, 2, 3];
List<string>        list      = ["Alice", "Bob", "Charlie"];
ImmutableArray<int> immutable = [10, 20, 30];
Span<byte>          span      = [0x00, 0xFF, 0x7F];  // stack-allocated
HashSet<int>        set       = [1, 2, 3, 2, 1];     // dédupliqué automatiquement

// Spread operator (..) : fusionner des collections
int[] first    = [1, 2, 3];
int[] second   = [4, 5, 6];
int[] combined = [..first, ..second, 7, 8]; // [1, 2, 3, 4, 5, 6, 7, 8]

// Collection vide (sans ambiguïté de type)
List<int> empty = [];

// Dans les arguments de méthode
void Process(IEnumerable<string> names) { /* ... */ }
Process(["Alice", "Bob"]);

// Le compilateur optimise selon le type cible :
// T[]             → tableau direct, zéro overhead
// List<T>         → CollectionsMarshal fast path
// Span<T>         → stack-alloc si taille connue à la compilation
// ImmutableArray  → builder optimisé
```

---

### 🔴 Que sont les primary constructors pour les classes et structs (C# 12) ?

Les **primary constructors** (C# 12) permettent de déclarer les paramètres du constructeur directement dans la en-tête de classe, éliminant le boilerplate de DI. Ils existaient déjà pour les `record` depuis C# 9.

```csharp
// Avant C# 12 : boilerplate d'injection de dépendances
public class OrderService
{
    private readonly IOrderRepository  _repo;
    private readonly ILogger<OrderService> _logger;

    public OrderService(IOrderRepository repo, ILogger<OrderService> logger)
    {
        _repo   = repo;
        _logger = logger;
    }
}

// C# 12 : primary constructor — les paramètres sont capturés automatiquement
public class OrderService(IOrderRepository repo, ILogger<OrderService> logger)
{
    public async Task<Order?> GetAsync(Guid id, CancellationToken ct = default)
    {
        logger.LogInformation("Getting order {Id}", id); // capture directe
        return await repo.GetByIdAsync(id, ct);
    }
}

// Valider et assigner si un traitement explicite est nécessaire
public class PricingService(IOptions<PricingOptions> options)
{
    private readonly PricingOptions _options = options.Value
        ?? throw new ArgumentNullException(nameof(options));

    public decimal Calculate(decimal price) => price * _options.TaxRate;
}
```

**Subtilités importantes :**

```csharp
// ⚠️ CS9124 : le paramètre 'logger' est aussi utilisé pour initialiser un champ
// → deux captures du même paramètre = WARNING
public class MyService(ILogger<MyService> logger)
{
    // Redéclarer le paramètre en champ crée une double capture
    private readonly ILogger<MyService> _logger = logger; // ← inutile ET potentiellement confus

    public void DoWork() => logger.LogInformation("Working"); // ET capture ici
}

// ✅ Choisir une approche : utiliser directement le paramètre du primary ctor
public class MyService(ILogger<MyService> logger)
{
    public void DoWork() => logger.LogInformation("Working");
}

// OU re-déclarer explicitement mais ne pas utiliser le paramètre directement ailleurs
public class MyService(ILogger<MyService> logger)
{
    private readonly ILogger<MyService> _logger = logger;
    public void DoWork() => _logger.LogInformation("Working");
}
```

> Pour les `record`, les paramètres du primary constructor deviennent des **propriétés publiques** — comportement différent des classes où ce sont des champs capturés privés.

---

### 🔴 Que sont les static abstract interface members et la Generic Math (C# 11+) ?

C# 11 (.NET 7) introduit les **static abstract members** dans les interfaces, permettant de définir des contrats sur des opérations statiques — notamment des opérateurs. Cela rend possible la **Generic Math** : écrire des algorithmes génériques sur des types numériques.

```csharp
// Interface avec member statique abstrait (C# 11)
public interface IAddable<T> where T : IAddable<T>
{
    static abstract T operator +(T left, T right);
    static abstract T Zero { get; }
}

// Algorithme générique qui fonctionne avec n'importe quel type implémentant IAddable
public static T Sum<T>(IEnumerable<T> source) where T : IAddable<T>
{
    var result = T.Zero;
    foreach (var item in source)
        result = result + item;
    return result;
}

// Interfaces standard de System.Numerics (.NET 7+)
// INumber<T>, IAdditionOperators<T,T,T>, IMultiplyOperators<T,T,T>, etc.
public static T DotProduct<T>(T[] a, T[] b) where T : INumber<T>
{
    T result = T.Zero;
    for (int i = 0; i < a.Length; i++)
        result += a[i] * b[i];
    return result;
}

// Fonctionne avec int, double, decimal, float, Complex, etc.
double d = DotProduct([1.0, 2.0], [3.0, 4.0]); // 11.0
int    n = DotProduct([1, 2],     [3, 4]);      // 11

// IParsable<T> — parser générique
public static T ParseConfig<T>(string value) where T : IParsable<T>
    => T.Parse(value, CultureInfo.InvariantCulture);

int    parsed1 = ParseConfig<int>("42");
double parsed2 = ParseConfig<double>("3.14159");
```

**Cas d'usage réels :**
- Bibliothèques mathématiques génériques (ML.NET, SIMD, numerics)
- Parseurs et formateurs génériques
- Agrégations génériques (somme, min, max) sur des types personnalisés

---

### 🔴 Quelles sont les nouveautés majeures de C# 13 (.NET 9) ?

**1. `params` collections — plus seulement des tableaux**
```csharp
// Avant C# 13 : uniquement params T[]
void Log(string template, params object[] args) { }

// C# 13 : params sur tout type collection
void Log(string template, params IEnumerable<object> args)              { }
void Log(string template, params ReadOnlySpan<object> args)             { } // zéro alloc
void Process(params List<string> items)                                 { }
```

**2. `ref struct` implémentant des interfaces**
```csharp
// C# 13 : les ref struct peuvent implémenter des interfaces
// (pas de boxing : ne peut pas être utilisé via la référence d'interface — compile error)
public ref struct SpanReader<T>(ReadOnlySpan<T> span) : IDisposable
{
    private int _position = 0;

    public bool TryRead(out T value)
    {
        if (_position >= span.Length) { value = default!; return false; }
        value = span[_position++];
        return true;
    }

    public void Dispose() { /* cleanup */ }
}
```

**3. `allows ref struct` — nouvelle contrainte générique**
```csharp
// Permet à un paramètre de type générique d'accepter des ref struct
public static void Process<T>(T value) where T : allows ref struct
{
    // value peut être Span<byte>, ReadOnlySpan<char>, etc.
}

Process(new Span<byte>(new byte[10])); // ✅
```

**4. Nouveau type `System.Threading.Lock`**
```csharp
// C# 13 : Lock est un type dédié, plus performant que lock(object)
private readonly System.Threading.Lock _lock = new();

void DoWork()
{
    using (_lock.EnterScope()) // RAII, libération garantie même en cas d'exception
    {
        // section critique
    }
}
```

**5. `\e` — escape sequence pour ESC (0x1B)**
```csharp
// Simplifie les séquences ANSI d'échappement terminal (avant : \x1b ou \u001b)
Console.WriteLine($"\e[32mGreen text\e[0m");
```

---

### 🔴 Quelles sont les nouveautés de C# 14 (.NET 10) ?

**1. `field` keyword — semi-auto properties**

Permet d'accéder au backing field implicite d'une propriété auto sans le déclarer explicitement.

```csharp
// Avant C# 14 : champ backing à déclarer manuellement pour ajouter une validation
private string _name = "";
public string Name
{
    get => _name;
    set => _name = value ?? throw new ArgumentNullException(nameof(value));
}

// C# 14 : field keyword élimine la déclaration du champ
public string Name
{
    get;
    set => field = value ?? throw new ArgumentNullException(nameof(value));
}

// Autre cas d'usage : lazy initialization
public IReadOnlyList<string> Tags
{
    get => field ??= LoadTags();
}
```

**2. Extension members — nouvelle syntaxe unifiée**

C# 14 introduit un nouveau mécanisme d'extension supportant propriétés, indexeurs et méthodes statiques — impossible avec les méthodes d'extension classiques.

```csharp
// Avant C# 14 : uniquement des méthodes d'extension statiques
public static class StringExtensions
{
    public static bool IsEmail(this string s) => s.Contains('@') && s.Contains('.');
}

// C# 14 : extension block — propriétés ET méthodes
public static class StringExtensions
{
    extension(string s)
    {
        // Propriété d'extension (impossible avant C# 14)
        public bool IsEmail => s.Contains('@') && s.Contains('.');
        public bool IsNullOrEmpty => string.IsNullOrEmpty(s);

        // Méthode d'extension (équivalent à l'ancienne syntaxe)
        public string Truncate(int maxLength)
            => s.Length <= maxLength ? s : s[..maxLength] + "…";
    }
}

// Usage identique
bool valid  = "alice@example.com".IsEmail; // true
string text = "Hello World".Truncate(5);   // "Hello…"
```

**3. Paramètres anonymes / discard `_` multiples**
```csharp
// C# 14 : plusieurs _ sans conflit de nom dans le même scope
var (_, _, third) = (1, 2, 3);

// Lambda avec paramètres ignorés (déjà possible en C# 9+ mais clarifié)
Action<int, int> handler = (_, _) => Console.WriteLine("fired");

// Méthode
void OnEvent(object _, EventArgs _) => Console.WriteLine("event received");
```

**4. Partial properties (C# 13)**
```csharp
// Source generators peuvent désormais générer des partial properties
public partial class Order
{
    public partial decimal TaxedAmount { get; }
}

// Dans le fichier généré
public partial class Order
{
    public partial decimal TaxedAmount => Amount * 1.20m;
}
```

---

### 🔴 Que sont les interceptors (C# 12+) ?

Les **interceptors** (C# 12 expérimental, stabilisé en C# 13) permettent à un **source generator** de remplacer un appel de méthode spécifique par une autre implémentation à la compilation, sans modifier le code source.

```csharp
// L'attribut [InterceptsLocation] est utilisé dans le code GÉNÉRÉ (jamais écrit manuellement)
// Il identifie l'appel à intercepter par fichier + ligne + colonne

// Code utilisateur (inchangé)
// Program.cs, ligne 5 :
app.MapGet("/api/orders", GetOrders);

// Code généré par le source generator :
namespace System.Runtime.CompilerServices
{
    [AttributeUsage(AttributeTargets.Method, AllowMultiple = true)]
    file sealed class InterceptsLocationAttribute(int version, string data) : Attribute { }
}

file static class GeneratedInterceptors
{
    [InterceptsLocation(1, "encoded_location_data")] // remplace l'appel en Program.cs:5
    internal static RouteHandlerBuilder MapGet_Intercepted(
        this IEndpointRouteBuilder app,
        string pattern,
        Delegate handler)
    {
        // Implémentation optimisée sans réflexion, générée à la compilation
        return app.MapGet(pattern, (HttpContext ctx) =>
        {
            // Résolution des paramètres connue à la compilation → pas de réflexion
            var ordersService = ctx.RequestServices.GetRequiredService<IOrderService>();
            return GetOrders(ordersService);
        });
    }
}
```

**Cas d'usage réels :**
- **Minimal APIs source generator** (`Microsoft.AspNetCore.Http.RequestDelegateGenerator`) — élimine la réflexion du pipeline HTTP
- **EF Core** — requêtes compilées automatiques
- **Logging source generator** (`LoggerMessage`) — délégués de log sans boxing

> Ne jamais écrire des interceptors dans le code applicatif ordinaire — ils sont réservés aux auteurs de bibliothèques et de générateurs de source.

---

## Lexique

| Terme | Définition |
|---|---|
| **allows ref struct** | Contrainte générique C# 13 permettant aux paramètres de type d'accepter des `ref struct`. |
| **Collection expression** | Syntaxe `[...]` unifiée C# 12 pour créer des collections de tout type. |
| **Extension member** | Propriété ou méthode définie à l'extérieur d'un type via la nouvelle syntaxe C# 14 `extension(T)`. |
| **field keyword** | Mot-clé C# 14 référençant le backing field implicite d'une propriété auto. |
| **Generic math** | Algorithmes génériques sur types numériques via `INumber<T>` et static abstract members (C# 11). |
| **Init-only setter** | Propriété assignable uniquement pendant l'initialisation de l'objet (C# 9). |
| **Interceptor** | Mécanisme C# 12+ permettant à un source generator de substituer un appel de méthode à la compilation. |
| **List pattern** | Pattern matching (C# 11) sur la structure d'une collection : `[1, 2, ..]`. |
| **NRT (Nullable Reference Types)** | Système C# 8+ faisant distinguer `string?` (nullable) de `string` (non-null) au compilateur. |
| **params collections** | Extension C# 13 du mot-clé `params` à tout type collection (plus seulement `T[]`). |
| **Partial property** | Propriété déclarée en deux parties (déclaration + implémentation) pour les source generators (C# 13+). |
| **Primary constructor** | Syntaxe C# 12 déclarant les paramètres du constructeur dans l'en-tête de la classe. |
| **Record** | Type C# 9 avec sémantique de valeur, immutabilité par convention et égalité structurelle. |
| **Required member** | Propriété C# 11 devant obligatoirement être assignée à la construction (`required`). |
| **Static abstract member** | Membre statique défini dans une interface (C# 11), fondement de la Generic Math. |
| **Switch expression** | Expression C# 8 retournant une valeur selon un pattern, sans `break` ni `return`. |

---

## Ressources

### Documentation officielle Microsoft
- [What's new in C# 12](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-12)
- [What's new in C# 13](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-13)
- [What's new in C# 14](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-14)
- [Pattern matching (C# guide)](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/functional/pattern-matching)
- [Generic math in .NET](https://learn.microsoft.com/en-us/dotnet/standard/generics/math)
- [Nullable reference types](https://learn.microsoft.com/en-us/dotnet/csharp/nullable-references)
- [Interceptors (C# spec)](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-12#interceptors)

### Blogs & Articles
- [Mads Torgersen — C# Language Design Blog](https://devblogs.microsoft.com/dotnet/tag/csharp/)
- [Andrew Lock — Primary constructors deep dive](https://andrewlock.net/primary-constructors-in-csharp-12/)
- [Nick Chapsas — C# 12/13/14 features (YouTube)](https://www.youtube.com/@nickchapsas)
- [Kathleen Dollard — C# 14 extension members](https://devblogs.microsoft.com/dotnet/)
