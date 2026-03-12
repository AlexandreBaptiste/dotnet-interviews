# Entretien .NET Senior/Expert — Dependency Injection

> **Versions ciblées :** .NET 8 / .NET 9 / .NET 10, C# 12 / C# 13 / C# 14
>
> **Niveaux de difficulté :** 🟢 Basique · 🟡 Intermédiaire · 🔴 Senior/Expert

---

## Fondamentaux

### 🟢 Qu'est-ce que l'injection de dépendances (DI) en .NET Core ? Quels sont les 3 lifetimes ?

L'injection de dépendances est un pattern où les dépendances d'un service sont **fournies par le runtime** plutôt qu'instanciées par l'objet lui-même. Le conteneur DI de .NET Core est intégré au framework via `IServiceCollection`.

**Les 3 lifetimes :**

| Lifetime | Durée de vie | Cas d'usage |
|---|---|---|
| **Singleton** | Durée de l'application | Caches, configuration, clients HTTP partagés |
| **Scoped** | Durée d'un scope (une requête HTTP) | `DbContext` EF Core, unit of work |
| **Transient** | Durée d'une résolution | Services légers et sans état |

```csharp
// Program.cs
builder.Services.AddSingleton<IMyCache, InMemoryCache>();
builder.Services.AddScoped<IOrderRepository, OrderRepository>();
builder.Services.AddTransient<IEmailSender, SmtpEmailSender>();
```

---

### 🟢 Pourquoi `DbContext` EF Core doit-il être Scoped et non Singleton ou Transient ?

```csharp
// ✅ Correct : Scoped par défaut via AddDbContext
builder.Services.AddDbContext<AppDbContext>(o =>
    o.UseSqlServer(connectionString));

// ❌ Singleton : le DbContext n'est pas thread-safe, les connexions seront partagées
builder.Services.AddSingleton<AppDbContext>();

// ❌ Transient : un nouveau DbContext à chaque résolution = pas de unit-of-work,
//    overhead de connexion, les Changes ne sont pas consolidés
builder.Services.AddTransient<AppDbContext>();
```

**Pourquoi Scoped :**
- `DbContext` maintient un graphe de changements (change tracker) — une instance par requête garantit que toutes les opérations d'un flux participent au même `SaveChangesAsync()`
- Les connexions ADO.NET sont poolées : un Scoped ouvre une connexion par requête, pas par lecture

---

### 🟡 Qu'est-ce qu'une "captive dependency" ? Comment la détecter et l'éviter ?

Une captive dependency survient quand un service de **lifetime long** (Singleton) détient une référence à un service de **lifetime plus court** (Scoped ou Transient). Le service Scoped est alors capturé et ne sera jamais recréé.

```csharp
// ❌ Captive dependency : Singleton qui injecte un Scoped
public class MySingleton
{
    private readonly IScopedRepository _repo; // ⚠️ l'instance ne change jamais !

    public MySingleton(IScopedRepository repo) => _repo = repo;
}

// ✅ Solution : injecter IServiceScopeFactory et créer un scope à la demande
public class MySingleton
{
    private readonly IServiceScopeFactory _scopeFactory;

    public MySingleton(IServiceScopeFactory factory) => _scopeFactory = factory;

    public async Task DoWorkAsync()
    {
        await using var scope = _scopeFactory.CreateAsyncScope();
        var repo = scope.ServiceProvider.GetRequiredService<IScopedRepository>();
        await repo.DoAsync();
    } // scope disposé ici → IScopedRepository également
}
```

**Détection :**
- En développement, `ValidateScopes = true` (activé par défaut via `CreateDefaultBuilder`) lève une `InvalidOperationException` à la résolution
- `ValidateOnBuild = true` détecte encore plus tôt, au `builder.Build()`

---

### 🟡 Qu'est-ce que les Keyed Services introduits dans .NET 8 ?

Les Keyed Services permettent d'enregistrer **plusieurs implémentations d'une même interface** sous des clés distinctes et de les résoudre par clé, sans pattern factory maison.

```csharp
// Enregistrement
builder.Services.AddKeyedSingleton<IPaymentProcessor, StripeProcessor>("stripe");
builder.Services.AddKeyedSingleton<IPaymentProcessor, PayPalProcessor>("paypal");

// Résolution via attribut dans le constructeur
public class CheckoutService(
    [FromKeyedServices("stripe")] IPaymentProcessor stripe,
    [FromKeyedServices("paypal")] IPaymentProcessor paypal)
{
    public async Task PayAsync(string method, decimal amount)
    {
        var processor = method == "stripe" ? stripe : paypal;
        await processor.ChargeAsync(amount);
    }
}

// Résolution manuelle
var processor = sp.GetRequiredKeyedService<IPaymentProcessor>("stripe");
```

> **Avant .NET 8 :** nécessitait un pattern factory (`Func<string, IProcessor>`) ou des packages tiers comme Scrutor.

---

### 🟡 Quelle est la différence entre `GetService<T>()` et `GetRequiredService<T>()` ?

```csharp
// GetService<T> : retourne null si le service n'est pas enregistré
var optional = serviceProvider.GetService<IOptionalFeature>(); // null si absent
if (optional is not null)
    optional.Execute();

// GetRequiredService<T> : lève InvalidOperationException si absent
var required = serviceProvider.GetRequiredService<IRequiredService>(); // exception si absent

// Résolution de toutes les implémentations d'une interface
builder.Services.AddSingleton<INotificationChannel, EmailChannel>();
builder.Services.AddSingleton<INotificationChannel, SmsChannel>();
builder.Services.AddSingleton<INotificationChannel, PushChannel>();

public class NotificationService(IEnumerable<INotificationChannel> channels)
{
    public async Task BroadcastAsync(string msg, CancellationToken ct)
        => await Task.WhenAll(channels.Select(c => c.SendAsync(msg, ct)));
}
```

> **Anti-pattern** : injecter `IServiceProvider` dans une classe pour résoudre les dépendances en interne = **Service Locator** pattern. Cela masque les dépendances et rend le code difficile à tester. À éviter sauf dans les factories et les tests.

---

### 🔴 Comment fonctionne l'injection via les Primary Constructors de C# 12 ?

C# 12 introduit les Primary Constructors, qui simplifient considérablement la déclaration des dépendances injectées.

```csharp
// Avant C# 12 : code verbeux
public class OrderService : IOrderService
{
    private readonly IOrderRepository _repo;
    private readonly ILogger<OrderService> _logger;

    public OrderService(IOrderRepository repo, ILogger<OrderService> logger)
    {
        _repo = repo;
        _logger = logger;
    }

    public async Task<Order?> GetByIdAsync(Guid id)
    {
        _logger.LogInformation("Fetching order {OrderId}", id);
        return await _repo.GetByIdAsync(id);
    }
}

// C# 12 : Primary Constructor
public class OrderService(
    IOrderRepository repo,
    ILogger<OrderService> logger) : IOrderService
{
    public async Task<Order?> GetByIdAsync(Guid id)
    {
        logger.LogInformation("Fetching order {OrderId}", id);
        return await repo.GetByIdAsync(id);
    }
}
```

**Points importants :**
- Les paramètres du primary constructor sont des champs synthétiques capturés dans toute la classe
- Ils ne peuvent pas être déclarés `readonly` explicitement — par convention, ne pas les réassigner
- Compatible avec l'héritage : `class Derived(IDep dep) : Base(dep)`
- `record class` utilise le même mécanisme mais les paramètres sont des propriétés `init`

---

### 🔴 Connaissez-vous `ValidateOnBuild` et `ValidateScopes` ? Comment les configurer ?

```csharp
// Configuration explicite (activés automatiquement en Development)
builder.Host.UseDefaultServiceProvider(options =>
{
    options.ValidateScopes = true;  // détecte captive dependencies à la résolution
    options.ValidateOnBuild = true; // vérifie la résolvabilité au Build()
});

// ValidateOnBuild : parcourt tous les services enregistrés et vérifie que chacun
// peut être résolu. Lève une exception AVANT que l'app démarre.
// Exemple : service enregistré mais implémentation non enregistrée
builder.Services.AddScoped<IMyService>(); // ❌ sans implémentation → erreur au build

// ValidateScopes : détecte qu'un Singleton résout un Scoped
// Exemple :
builder.Services.AddSingleton<MySingleton>(); // résout IScopedService → exception
builder.Services.AddScoped<IScopedService, ScopedService>();
```

> **En production :** `ValidateOnBuild` ajoute un coût au démarrage (parcours de toutes les registrations). À désactiver en prod si le démarrage est critique, mais les erreurs DI doivent avoir été détectées en dev.

---

### 🔴 Comment implémenter le pattern Decorator via le conteneur DI .NET Core ?

Le conteneur DI .NET Core ne supporte pas nativement le Decorator — il faut une registration manuelle ou utiliser le package **Scrutor**.

```csharp
// Sans Scrutor : résolution manuelle
builder.Services.AddSingleton<OrderRepository>(); // implémentation concrète
builder.Services.AddSingleton<IOrderRepository>(sp =>
    new CachedOrderRepository(
        inner: sp.GetRequiredService<OrderRepository>(),
        cache: sp.GetRequiredService<IMemoryCache>()));

// Avec Scrutor : fluent Decorator
// dotnet add package Scrutor
builder.Services.AddSingleton<IOrderRepository, OrderRepository>();
builder.Services.Decorate<IOrderRepository, CachedOrderRepository>();

// CachedOrderRepository : reçoit IOrderRepository (inner) par DI
public class CachedOrderRepository(
    IOrderRepository inner,
    IMemoryCache cache) : IOrderRepository
{
    public async Task<Order?> GetByIdAsync(Guid id)
    {
        var key = $"order:{id}";
        if (cache.TryGetValue(key, out Order? cached))
            return cached;

        var order = await inner.GetByIdAsync(id);
        if (order is not null)
            cache.Set(key, order, TimeSpan.FromMinutes(5));

        return order;
    }
}
```

---

### 🔴 Qu'est-ce que `IServiceProviderFactory<TContainerBuilder>` ? Comment intégrer un conteneur tiers (Autofac) ?

```csharp
// Intégration Autofac comme conteneur DI alternatif (.NET Core supporte le remplacement)
// dotnet add package Autofac.Extensions.DependencyInjection
builder.Host.UseServiceProviderFactory(new AutofacServiceProviderFactory());

builder.Host.ConfigureContainer<ContainerBuilder>(containerBuilder =>
{
    // API Autofac : registration par assembly scanning, modules, etc.
    containerBuilder.RegisterAssemblyTypes(typeof(Program).Assembly)
        .Where(t => t.Name.EndsWith("Service"))
        .AsImplementedInterfaces()
        .InstancePerLifetimeScope(); // = Scoped

    containerBuilder.RegisterModule<InfrastructureModule>();
});

// Raisons d'utiliser Autofac :
// - Scanning d'assemblies automatique
// - Decorator et interceptors natifs (AOP)
// - Modules réutilisables
// - Résolution par nom sans Keyed Services
```

---

## Lexique

| Terme | Définition |
|---|---|
| **Captive dependency** | Service Scoped ou Transient détenu par un Singleton, capturé pour toute la vie de l'app. |
| **Conteneur DI** | Registre centralisé qui instancie et gère les services selon leur lifetime. |
| **Decorator pattern** | Pattern structurel enveloppant une implémentation avec une autre de même interface pour ajouter du comportement. |
| **IServiceCollection** | Collection de descripteurs de services, configurée au démarrage via `builder.Services`. |
| **IServiceProvider** | Interface du conteneur permettant de résoudre les services enregistrés. |
| **IServiceScope** | Scope délimitant la durée de vie des services Scoped. Créé par requête HTTP ou manuellement. |
| **IServiceScopeFactory** | Factory créant des scopes manuellement, injectée typiquement dans des Singletons. |
| **Keyed Services** | Fonctionnalité .NET 8 permettant d'enregistrer plusieurs implémentations sous des clés distinctes. |
| **Primary Constructor** | Syntaxe C# 12 déclarant les paramètres du constructeur directement sur la classe. |
| **Scrutor** | Package NuGet ajoutant Decorator et assembly scanning au DI .NET Core. |
| **Service Locator** | Anti-pattern résolvant les dépendances via `IServiceProvider` directement dans le code métier. |
| **ValidateOnBuild** | Option vérifiant la résolvabilité de tous les services lors du `Build()`. |
| **ValidateScopes** | Option détectant les captive dependencies à la résolution d'un service. |

---

## Ressources

### Documentation officielle Microsoft
- [Dependency injection in .NET](https://learn.microsoft.com/en-us/dotnet/core/extensions/dependency-injection)
- [Dependency injection guidelines](https://learn.microsoft.com/en-us/dotnet/core/extensions/dependency-injection-guidelines)
- [Keyed services in .NET 8](https://learn.microsoft.com/en-us/dotnet/core/whats-new/dotnet-8/runtime#keyed-di-services)
- [DI in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/dependency-injection)

### Livres
- *Dependency Injection Principles, Practices, and Patterns* — Steven van Deursen & Mark Seemann
- *ASP.NET Core in Action* — Andrew Lock

### Blogs & Articles
- [Scrutor — Decoration and assembly scanning](https://github.com/khellang/Scrutor)
- [Mark Seemann — DI patterns and anti-patterns](https://blog.ploeh.dk/)
- [Andrew Lock — .NET DI deep dives](https://andrewlock.net/)
