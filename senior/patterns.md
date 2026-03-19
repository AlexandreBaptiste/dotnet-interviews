# Entretien .NET Senior/Expert — Patterns & Bonnes Pratiques

> **Versions ciblées :** .NET 8 / .NET 9 / .NET 10, C# 12 / C# 13 / C# 14
>
> **Niveaux de difficulté :** 🟢 Basique · 🟡 Intermédiaire · 🔴 Senior/Expert

---

## SOLID & Fondamentaux

### 🟢 Expliquer les principes SOLID avec des exemples C# concrets

**S — Single Responsibility Principle (SRP)**  
Une classe ne doit avoir qu'**une seule raison de changer**.

```csharp
// ❌ Viole SRP : gère la logique métier ET la persistance ET la notification
public class OrderService
{
    public void PlaceOrder(Order order)
    {
        // Validation
        if (order.Items.Count == 0) throw new InvalidOperationException();
        // Persistance
        db.Orders.Add(order);
        db.SaveChanges();
        // Notification
        emailService.Send(order.CustomerEmail, "Commande confirmée");
    }
}

// ✅ SRP : chaque classe a une responsabilité unique
public class OrderValidator { public void Validate(Order o) { ... } }
public class OrderRepository  { public Task SaveAsync(Order o) { ... } }
public class OrderNotifier    { public Task NotifyAsync(Order o) { ... } }
```

**O — Open/Closed Principle (OCP)**  
Ouvert à l'extension, fermé à la modification.

```csharp
// ✅ Ajouter un nouveau type de remise sans modifier DiscountCalculator
public interface IDiscountStrategy { decimal Apply(decimal price); }

public class PercentageDiscount(decimal percent) : IDiscountStrategy
    { public decimal Apply(decimal price) => price * (1 - percent / 100); }

public class FixedDiscount(decimal amount) : IDiscountStrategy
    { public decimal Apply(decimal price) => price - amount; }

public class DiscountCalculator(IDiscountStrategy strategy)
    { public decimal Calculate(decimal price) => strategy.Apply(price); }
```

**L — Liskov Substitution Principle (LSP)**  
Un sous-type doit pouvoir remplacer son type de base sans altérer le comportement.

```csharp
// ❌ Viole LSP : Square hérite de Rectangle mais casse les invariants
public class Rectangle { public virtual int Width { get; set; } public virtual int Height { get; set; } }
public class Square : Rectangle
{
    public override int Width  { set { base.Width = base.Height = value; } }
    public override int Height { set { base.Width = base.Height = value; } }
}

// ✅ Utiliser la composition ou une abstraction commune
public interface IShape { int Area(); }
public record RectangleShape(int Width, int Height) : IShape { public int Area() => Width * Height; }
public record SquareShape(int Side) : IShape { public int Area() => Side * Side; }
```

**I — Interface Segregation Principle (ISP)**  
Les clients ne doivent pas dépendre d'interfaces qu'ils n'utilisent pas.

```csharp
// ❌ Interface trop large
public interface IRepository<T>
{
    T? GetById(int id); IEnumerable<T> GetAll();
    void Add(T entity); void Update(T entity); void Delete(int id);
    IQueryable<T> Query(); Task<int> CountAsync();
}

// ✅ Interfaces ciblées
public interface IReadRepository<T>  { T? GetById(int id); IEnumerable<T> GetAll(); }
public interface IWriteRepository<T> { void Add(T entity); void Delete(int id); }
```

**D — Dependency Inversion Principle (DIP)**  
Dépendre des abstractions, pas des implémentations concrètes.

```csharp
// ❌ Couplage fort à l'implémentation
public class ReportService { private readonly SqlOrderRepository _repo = new(); }

// ✅ Injection de l'abstraction
public class ReportService(IOrderRepository repo) { ... }
```

---

### 🟢 Qu'est-ce que le pattern Repository et le pattern Unit of Work ? Comment les implémenter ?

Le **Repository** abstrait l'accès aux données derrière une interface orientée domaine.  
Le **Unit of Work** coordonne plusieurs repositories dans une seule transaction.

```csharp
// Repository générique
public interface IRepository<T> where T : class
{
    Task<T?> GetByIdAsync(int id, CancellationToken ct = default);
    Task<IReadOnlyList<T>> GetAllAsync(CancellationToken ct = default);
    void Add(T entity);
    void Remove(T entity);
}

// Repository spécialisé
public interface IOrderRepository : IRepository<Order>
{
    Task<IReadOnlyList<Order>> GetByCustomerAsync(int customerId, CancellationToken ct = default);
}

// Implémentation EF Core
public class OrderRepository(AppDbContext db) : IOrderRepository
{
    public async Task<Order?> GetByIdAsync(int id, CancellationToken ct)
        => await db.Orders.Include(o => o.Items).FirstOrDefaultAsync(o => o.Id == id, ct);

    public async Task<IReadOnlyList<Order>> GetByCustomerAsync(int customerId, CancellationToken ct)
        => await db.Orders.Where(o => o.CustomerId == customerId).ToListAsync(ct);

    public async Task<IReadOnlyList<Order>> GetAllAsync(CancellationToken ct)
        => await db.Orders.ToListAsync(ct);

    public void Add(Order order)    => db.Orders.Add(order);
    public void Remove(Order order) => db.Orders.Remove(order);
}

// Unit of Work
public interface IUnitOfWork : IAsyncDisposable
{
    IOrderRepository Orders { get; }
    IProductRepository Products { get; }
    Task<int> SaveChangesAsync(CancellationToken ct = default);
}

public class UnitOfWork(AppDbContext db) : IUnitOfWork
{
    public IOrderRepository Orders { get; } = new OrderRepository(db);
    public IProductRepository Products { get; } = new ProductRepository(db);

    public Task<int> SaveChangesAsync(CancellationToken ct) => db.SaveChangesAsync(ct);
    public ValueTask DisposeAsync() => db.DisposeAsync();
}

// Utilisation dans un service
public class OrderService(IUnitOfWork uow)
{
    public async Task PlaceOrderAsync(CreateOrderRequest request, CancellationToken ct)
    {
        var order = Order.Create(request);
        uow.Orders.Add(order);
        await uow.SaveChangesAsync(ct); // une seule transaction
    }
}
```

> **En pratique avec EF Core :** `DbContext` est lui-même un Unit of Work. Encapsuler les repositories dans une couche supplémentaire est optionnel, mais améliore la testabilité.

---

## Design Patterns

### 🟡 Expliquer le pattern Strategy et le pattern Factory — différences et usages

**Strategy** : encapsuler un algorithme interchangeable dans une famille d'implémentations.

```csharp
// Tri différent selon le contexte
public interface ISortStrategy<T> { IEnumerable<T> Sort(IEnumerable<T> items); }

public class AlphabeticalSort : ISortStrategy<Product>
    { public IEnumerable<Product> Sort(IEnumerable<Product> items) => items.OrderBy(p => p.Name); }

public class PriceAscSort : ISortStrategy<Product>
    { public IEnumerable<Product> Sort(IEnumerable<Product> items) => items.OrderBy(p => p.Price); }

// Injection de la stratégie via DI (Keyed Services .NET 8)
builder.Services.AddKeyedSingleton<ISortStrategy<Product>, AlphabeticalSort>("alpha");
builder.Services.AddKeyedSingleton<ISortStrategy<Product>, PriceAscSort>("price");

public class CatalogService([FromKeyedServices("alpha")] ISortStrategy<Product> sort) { ... }
```

**Factory Method** : déléguer la création d'objets à une sous-classe ou méthode dédiée.

```csharp
// Factory statique (simple)
public static class NotificationFactory
{
    public static INotification Create(NotificationType type) => type switch
    {
        NotificationType.Email => new EmailNotification(),
        NotificationType.Sms   => new SmsNotification(),
        NotificationType.Push  => new PushNotification(),
        _ => throw new ArgumentOutOfRangeException(nameof(type))
    };
}

// Abstract Factory (famille d'objets liés)
public interface IUiFactory { IButton CreateButton(); ICheckbox CreateCheckbox(); }
public class WindowsUiFactory : IUiFactory { ... }
public class MacUiFactory     : IUiFactory { ... }
```

| | Strategy | Factory |
|---|---|---|
| Intent | **Comportement** variable | **Création** d'objets |
| Variabilité | Algorithme | Type concret |
| Injection DI | Interface injectée | Résolution via DI / pattern |

---

### 🟡 Comment implémenter le pattern Decorator en C# avec l'injection de dépendances ?

Le Decorator ajoute un comportement **sans modifier** la classe décorée — idéal pour le logging, la validation, le caching.

```csharp
public interface IProductRepository
{
    Task<Product?> GetByIdAsync(Guid id, CancellationToken ct);
}

// Implémentation de base
public class ProductRepository(AppDbContext db) : IProductRepository
{
    public async Task<Product?> GetByIdAsync(Guid id, CancellationToken ct)
        => await db.Products.FindAsync([id], ct);
}

// Decorator : ajoute du caching transparent
public class CachingProductRepository(IProductRepository inner, IMemoryCache cache)
    : IProductRepository
{
    public async Task<Product?> GetByIdAsync(Guid id, CancellationToken ct)
    {
        var key = $"product:{id}";
        if (cache.TryGetValue(key, out Product? cached)) return cached;
        var product = await inner.GetByIdAsync(id, ct);
        if (product is not null) cache.Set(key, product, TimeSpan.FromMinutes(5));
        return product;
    }
}

// Decorator : ajoute du logging
public class LoggingProductRepository(IProductRepository inner, ILogger<LoggingProductRepository> logger)
    : IProductRepository
{
    public async Task<Product?> GetByIdAsync(Guid id, CancellationToken ct)
    {
        logger.LogInformation("Fetching product {Id}", id);
        var result = await inner.GetByIdAsync(id, ct);
        logger.LogInformation("Product {Id} {Found}", id, result is null ? "not found" : "found");
        return result;
    }
}

// Chaînage manuel dans DI (sans Scrutor)
builder.Services.AddScoped<ProductRepository>();
builder.Services.AddScoped<IProductRepository>(sp =>
    new LoggingProductRepository(
        new CachingProductRepository(
            sp.GetRequiredService<ProductRepository>(),
            sp.GetRequiredService<IMemoryCache>()),
        sp.GetRequiredService<ILogger<LoggingProductRepository>>()));

// ✅ Avec Scrutor (plus lisible) — dotnet add package Scrutor
builder.Services.AddScoped<IProductRepository, ProductRepository>();
builder.Services.Decorate<IProductRepository, CachingProductRepository>();
builder.Services.Decorate<IProductRepository, LoggingProductRepository>();
```

---

## Architecture

### 🔴 Qu'est-ce que CQRS et le pattern Mediator ? Comment les implémenter avec MediatR ?

**CQRS** (Command Query Responsibility Segregation) sépare les opérations de **lecture** (Query) des opérations d'**écriture** (Command).

**Mediator** découple l'expéditeur d'une requête de son handler, via un objet intermédiaire.

```csharp
// dotnet add package MediatR

// --- QUERY ---
public record GetOrderQuery(Guid OrderId) : IRequest<OrderDto?>;

public class GetOrderQueryHandler(IOrderRepository repo, IMapper mapper)
    : IRequestHandler<GetOrderQuery, OrderDto?>
{
    public async Task<OrderDto?> Handle(GetOrderQuery request, CancellationToken ct)
    {
        var order = await repo.GetByIdAsync(request.OrderId, ct);
        return order is null ? null : mapper.Map<OrderDto>(order);
    }
}

// --- COMMAND ---
public record PlaceOrderCommand(Guid CustomerId, List<OrderItemDto> Items) : IRequest<Guid>;

public class PlaceOrderCommandHandler(IUnitOfWork uow, IPublisher publisher)
    : IRequestHandler<PlaceOrderCommand, Guid>
{
    public async Task<Guid> Handle(PlaceOrderCommand request, CancellationToken ct)
    {
        var order = Order.Create(request.CustomerId, request.Items);
        uow.Orders.Add(order);
        await uow.SaveChangesAsync(ct);

        // Notification via MediatR Notification (event interne)
        await publisher.Publish(new OrderPlacedEvent(order.Id), ct);
        return order.Id;
    }
}

// --- NOTIFICATION ---
public record OrderPlacedEvent(Guid OrderId) : INotification;

public class SendConfirmationEmailHandler(IEmailService email)
    : INotificationHandler<OrderPlacedEvent>
{
    public async Task Handle(OrderPlacedEvent notification, CancellationToken ct)
        => await email.SendOrderConfirmationAsync(notification.OrderId, ct);
}

// --- PIPELINE BEHAVIOR (cross-cutting concerns) ---
public class ValidationBehavior<TRequest, TResponse>(IEnumerable<IValidator<TRequest>> validators)
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : notnull
{
    public async Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        var context = new ValidationContext<TRequest>(request);
        var failures = validators
            .Select(v => v.Validate(context))
            .SelectMany(r => r.Errors)
            .Where(f => f is not null)
            .ToList();

        if (failures.Count != 0)
            throw new ValidationException(failures);

        return await next();
    }
}

// Enregistrement
builder.Services.AddMediatR(cfg =>
{
    cfg.RegisterServicesFromAssembly(typeof(Program).Assembly);
    cfg.AddBehavior(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
    cfg.AddBehavior(typeof(IPipelineBehavior<,>), typeof(LoggingBehavior<,>));
});

// Utilisation dans un controller / endpoint
app.MapPost("/orders", async (PlaceOrderCommand cmd, IMediator mediator, CancellationToken ct)
    => Results.Ok(await mediator.Send(cmd, ct)));
```

---

### 🔴 Qu'est-ce que la Clean Architecture ? Comment l'organiser dans un projet .NET ?

La Clean Architecture organise le code en **couches concentriques** avec une règle fondamentale : les dépendances ne pointent que **vers l'intérieur** (vers le domaine).

```
┌─────────────────────────────────────┐
│  Infrastructure (EF Core, Redis...) │
│  ┌───────────────────────────────┐  │
│  │  Application (CQRS, Services) │  │
│  │  ┌─────────────────────────┐  │  │
│  │  │  Domain (Entities, VO)  │  │  │
│  │  └─────────────────────────┘  │  │
│  └───────────────────────────────┘  │
└─────────────────────────────────────┘
         ↑ Web / Presentation (API)
```

```
MyApp/
├── MyApp.Domain/               # Couche intérieure - zéro dépendance externe
│   ├── Entities/               # Entités riches (Order, Product)
│   ├── ValueObjects/           # Immutables (Money, Address)
│   ├── Aggregates/             # Racines d'agrégats
│   ├── Events/                 # Domain events
│   └── Interfaces/             # IOrderRepository (abstraction définie ici)
│
├── MyApp.Application/          # Use cases — dépend uniquement de Domain
│   ├── Orders/
│   │   ├── Commands/           # PlaceOrderCommand + Handler
│   │   └── Queries/            # GetOrderQuery + Handler
│   ├── Behaviors/              # MediatR pipeline behaviors
│   └── Interfaces/             # IEmailService (port secondaire)
│
├── MyApp.Infrastructure/       # Adaptateurs — implémente les interfaces Domain/Application
│   ├── Persistence/
│   │   ├── AppDbContext.cs
│   │   └── OrderRepository.cs  # implémente IOrderRepository
│   ├── Messaging/
│   └── ExternalServices/
│
└── MyApp.WebApi/               # Point d'entrée — orchestre la composition
    ├── Controllers/ ou Endpoints/
    ├── Program.cs              # DI, middleware pipeline
    └── Mappers/                # DTO ↔ Commandes/Queries
```

**Règles clés :**
- `Domain` ne connaît ni EF Core, ni ASP.NET Core, ni aucune infrastructure
- `Application` définit des **ports** (interfaces), `Infrastructure` fournit les **adaptateurs**
- Les tests unitaires ciblent `Domain` et `Application` sans dépendance d'infrastructure

---

### 🔴 Qu'est-ce que l'Outbox Pattern ? Pourquoi est-il essentiel dans une architecture event-driven ?

L'Outbox Pattern garantit qu'un événement domaine est **publié une et une seule fois**, même en cas de crash entre la sauvegarde et la publication sur un broker (Kafka, RabbitMQ...).

```csharp
// Problème : crash possible entre SaveChanges et PublishAsync
await db.SaveChangesAsync();   // ✅ transaction commit
await bus.PublishAsync(evt);   // 💥 crash ici → événement perdu

// ✅ Solution Outbox : sauvegarder l'événement dans la même transaction
public class OutboxMessage
{
    public Guid Id          { get; init; } = Guid.NewGuid();
    public string Type      { get; init; } = default!;
    public string Payload   { get; init; } = default!;
    public DateTime OccurredAt { get; init; } = DateTime.UtcNow;
    public DateTime? ProcessedAt { get; set; }
}

// Dans le handler de commande
public async Task<Guid> Handle(PlaceOrderCommand cmd, CancellationToken ct)
{
    var order = Order.Create(cmd.CustomerId, cmd.Items);
    db.Orders.Add(order);

    // Même transaction : si SaveChanges échoue, l'outbox n'est pas sauvegardé non plus
    db.OutboxMessages.Add(new OutboxMessage
    {
        Type    = nameof(OrderPlacedEvent),
        Payload = JsonSerializer.Serialize(new OrderPlacedEvent(order.Id))
    });

    await db.SaveChangesAsync(ct); // atomique
    return order.Id;
}

// Background worker qui traite l'outbox
public class OutboxProcessor(AppDbContext db, IMessageBus bus, ILogger<OutboxProcessor> log)
    : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        using var timer = new PeriodicTimer(TimeSpan.FromSeconds(5));
        while (await timer.WaitForNextTickAsync(stoppingToken))
        {
            var messages = await db.OutboxMessages
                .Where(m => m.ProcessedAt == null)
                .OrderBy(m => m.OccurredAt)
                .Take(50)
                .ToListAsync(stoppingToken);

            foreach (var msg in messages)
            {
                try
                {
                    await bus.PublishAsync(msg.Type, msg.Payload, stoppingToken);
                    msg.ProcessedAt = DateTime.UtcNow;
                }
                catch (Exception ex)
                {
                    log.LogError(ex, "Failed to publish outbox message {Id}", msg.Id);
                }
            }

            await db.SaveChangesAsync(stoppingToken);
        }
    }
}
```

**Librairies prêtes à l'emploi :** [MassTransit Outbox](https://masstransit.io/documentation/patterns/transactional-outbox), [Wolverine](https://wolverinefx.net/)

---

### 🔴 Qu'est-ce que les Value Objects dans DDD ? Comment les implémenter en C# 12+ ?

Un **Value Object** (VO) est un objet **immuable** défini par sa valeur, sans identité propre. Deux VO avec les mêmes valeurs sont égaux.

```csharp
// ✅ C# 9+ : record struct pour VO légers sans allocation heap
public readonly record struct Money(decimal Amount, string Currency)
{
    // Invariants dans le constructeur ou factory
    public static Money Of(decimal amount, string currency)
    {
        if (amount < 0)        throw new ArgumentException("Amount cannot be negative");
        if (string.IsNullOrWhiteSpace(currency)) throw new ArgumentException("Currency required");
        return new Money(amount, currency.ToUpperInvariant());
    }

    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new InvalidOperationException($"Cannot add {Currency} and {other.Currency}");
        return new Money(Amount + other.Amount, Currency);
    }

    public override string ToString() => $"{Amount:F2} {Currency}";
}

// Value Object complexe : adresse
public sealed record Address(string Street, string City, string PostalCode, string Country)
{
    // Validation à la construction
    public Address : this(Street, City, PostalCode, Country)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(Street);
        ArgumentException.ThrowIfNullOrWhiteSpace(City);
    }
}

// Persistance EF Core : owned entity (pas de table séparée)
public class Order
{
    public Guid    Id       { get; private set; }
    public Money   Total    { get; private set; }
    public Address Delivery { get; private set; } = default!;
}

public class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        builder.OwnsOne(o => o.Delivery);          // colonnes inlinées dans la table Orders
        builder.Property(o => o.Total)
               .HasConversion(m => m.Amount, v => Money.Of(v, "EUR"))
               .HasColumnName("TotalAmount");
    }
}
```

**Avantages :** auto-validation, égalité structurelle, expressivité du domaine, immuabilité garantie.

---

## Lexique

| Terme | Définition |
|---|---|
| **Abstract Factory** | Pattern créant des familles d'objets liés sans spécifier leurs classes concrètes. |
| **Aggregate** | Groupe d'entités DDD traité comme une unité, avec une racine (Aggregate Root). |
| **Clean Architecture** | Architecture en couches concentriques où les dépendances pointent vers le domaine. |
| **CQRS** | Command Query Responsibility Segregation : séparation des lectures et des écritures. |
| **Decorator** | Pattern ajoutant un comportement à un objet sans modifier sa classe. |
| **Domain Event** | Événement représentant quelque chose d'important survenu dans le domaine métier. |
| **DDD** | Domain-Driven Design : approche centrée sur le domaine métier pour concevoir des logiciels complexes. |
| **Factory Method** | Pattern délégant la création d'objets à une méthode ou sous-classe. |
| **Idempotency** | Propriété d'une opération : exécutée plusieurs fois, elle produit le même résultat. |
| **Mediator** | Pattern découplant l'émetteur et le récepteur d'une requête via un intermédiaire. |
| **Outbox Pattern** | Stockage des événements dans la même transaction DB avant publication sur un broker. |
| **Repository** | Abstraction orientée domaine pour accéder aux données, masquant la persistance. |
| **SOLID** | Cinq principes de conception : SRP, OCP, LSP, ISP, DIP. |
| **Strategy** | Pattern encapsulant un algorithme interchangeable dans une famille d'implémentations. |
| **Unit of Work** | Pattern coordonnant plusieurs repositories dans une transaction unique. |
| **Value Object** | Objet DDD immuable défini par sa valeur, sans identité propre. |

---

## Ressources

### Documentation officielle Microsoft
- [.NET Application Architecture Guides](https://learn.microsoft.com/en-us/dotnet/architecture/)
- [eShopOnWeb — Clean Architecture reference app](https://github.com/dotnet-architecture/eShopOnWeb)
- [Blazor Architecture Best Practices](https://learn.microsoft.com/en-us/dotnet/architecture/blazor-for-web-forms-developers/)
- [Microservices Architecture with .NET](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/)

### Packages NuGet
- [MediatR](https://www.nuget.org/packages/MediatR)
- [Scrutor](https://www.nuget.org/packages/Scrutor)
- [MassTransit](https://www.nuget.org/packages/MassTransit)
- [FluentValidation](https://www.nuget.org/packages/FluentValidation)

### Blogs & Articles
- [Steve Smith (Ardalis) — Clean Architecture](https://ardalis.com/clean-architecture-asp-net-core/)
- [Vladimir Khorikov — DDD, CQRS and Value Objects](https://enterprisecraftsmanship.com/posts/domain-model-purity-and-lazy-loading/)
- [Milan Jovanović — CQRS and MediatR in .NET](https://www.milanjovanovic.tech/)
- [Andrew Lock — Decorator pattern with Scrutor](https://andrewlock.net/adding-decorated-classes-to-the-asp.net-core-di-container-using-scrutor/)
