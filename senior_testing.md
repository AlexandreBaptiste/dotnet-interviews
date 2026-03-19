# Entretien .NET Senior/Expert — Testing

> **Versions ciblées :** .NET 8 / .NET 9 / .NET 10, C# 12 / C# 13 / C# 14
>
> **Niveaux de difficulté :** 🟢 Basique · 🟡 Intermédiaire · 🔴 Senior/Expert

---

## Fondamentaux

### 🟢 Quels sont les grands types de tests (Solitary, Sociable, E2E) ? À quoi servent-ils et quand les utiliser ?

La taxonomie la plus répandue dans l'écosystème .NET moderne distingue trois grandes familles de tests, popularisées par Martin Fowler et le mouvement « Growing Object-Oriented Software » :

#### 1. Tests Solitaires (*Solitary Unit Tests*)

L'unité testée est **complètement isolée** de tous ses collaborateurs. Chaque dépendance externe (repository, service, horloge…) est remplacée par un double de test (mock/stub/fake).

- **Pourquoi ?** Vitesse maximale (millisecondes), feedback immédiat, déterminisme total. Ils localisent précisément l'origine d'un bug.
- **Quand ?** Sur la logique métier pure (calculs, règles de domaine, algorithmes) où l'on veut vérifier le comportement d'une seule classe indépendamment du reste.
- **Limites :** N'exercent pas les contrats entre composants. Un test solitaire vert ne garantit pas que l'intégration fonctionne.

```csharp
// Solitaire : OrderService totalement isolé — le repository est mocké
public class OrderServiceSolitaryTests
{
    private readonly IOrderRepository _repo = Substitute.For<IOrderRepository>();
    private readonly OrderService _sut;

    public OrderServiceSolitaryTests()
        => _sut = new OrderService(_repo, NullLogger<OrderService>.Instance);

    [Fact]
    public async Task PlaceOrder_ValidRequest_CallsSaveOnce()
    {
        var request = new PlaceOrderRequest("Alice", 99.99m);

        await _sut.PlaceOrderAsync(request);

        await _repo.Received(1).SaveAsync(Arg.Any<Order>(), Arg.Any<CancellationToken>());
    }
}
```

---

#### 2. Tests Sociables (*Sociable Unit Tests*)

L'unité testée collabore avec de **vraies implémentations** de certains voisins (entités de domaine, value objects, services purement fonctionnels), tout en remplaçant les dépendances avec effets de bord (I/O, réseau, temps…) par des fakes.

- **Pourquoi ?** Meilleure fidélité aux comportements réels sans le coût des tests d'intégration. Ils exercent les interactions internes du domaine.
- **Quand ?** Pour tester un agrégat ou un service applicatif aux côtés de ses vraies entités de domaine (ex. `Order` + `OrderLine`), en mockant uniquement les ports d'infrastructure.
- **Limites :** Plus difficiles à maintenir si de nombreuses classes changent en même temps ; peuvent cacher des problèmes d'isolation.

```csharp
// Sociable : OrderService utilise de vrai objets Order/OrderLine,
// mais le repository reste un fake
public class OrderServiceSociableTests
{
    private readonly FakeOrderRepository _repo = new();
    private readonly OrderService _sut;

    public OrderServiceSociableTests()
        => _sut = new OrderService(_repo, NullLogger<OrderService>.Instance);

    [Fact]
    public async Task PlaceOrder_MultipleLines_ComputesTotalCorrectly()
    {
        var request = new PlaceOrderRequest("Alice",
            Lines: [new("Widget", 2, 10m), new("Gadget", 1, 50m)]);

        var order = await _sut.PlaceOrderAsync(request);

        // On vérifie le comportement réel du domaine Order + OrderLine
        Assert.Equal(70m, order.Total);
    }
}

// Fake repository en mémoire (implémentation réelle légère, pas un mock)
public class FakeOrderRepository : IOrderRepository
{
    private readonly List<Order> _store = [];

    public Task SaveAsync(Order order, CancellationToken ct = default)
    {
        _store.Add(order);
        return Task.CompletedTask;
    }

    public Task<Order?> GetByIdAsync(Guid id, CancellationToken ct = default)
        => Task.FromResult(_store.FirstOrDefault(o => o.Id == id));
}
```

---

#### 3. Tests End-to-End (E2E)

Ils exercent **l'ensemble de la pile applicative** — de l'interface utilisateur ou du point d'entrée HTTP jusqu'à la base de données réelle et les services externes — en simulant un vrai utilisateur ou client.

- **Pourquoi ?** Seule catégorie qui garantit que tous les composants s'assemblent correctement en production. Ils détectent des régressions invisibles pour les tests unitaires.
- **Quand ?** Sur les scénarios critiques métier (parcours d'achat, authentification, génération de rapport…). Limiter leur nombre : ils sont lents et fragiles.
- **En .NET :** `WebApplicationFactory` + Testcontainers pour les tests E2E d'API ; Playwright ou Selenium pour les tests E2E d'interface Web.

```csharp
// E2E API : WebApplicationFactory + vraie base PostgreSQL via Testcontainers
public class PlaceOrderE2ETests(E2EApiFactory factory) : IClassFixture<E2EApiFactory>
{
    private readonly HttpClient _client = factory.CreateClient();

    [Fact]
    public async Task PlaceOrder_FullStack_Returns201AndPersistsOrder()
    {
        // Act : requête HTTP réelle vers l'app démarrée en mémoire
        var response = await _client.PostAsJsonAsync("/api/orders",
            new { CustomerName = "Alice", Amount = 99.99m });

        // Assert : statut HTTP
        Assert.Equal(HttpStatusCode.Created, response.StatusCode);

        // Assert : persistance réelle en base
        var dto = await response.Content.ReadFromJsonAsync<OrderDto>();
        var check = await _client.GetAsync($"/api/orders/{dto!.Id}");
        check.EnsureSuccessStatusCode();
    }
}

public class E2EApiFactory : WebApplicationFactory<Program>, IAsyncLifetime
{
    private readonly PostgreSqlContainer _db = new PostgreSqlBuilder()
        .WithImage("postgres:16").Build();

    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            var descriptor = services.Single(d => d.ServiceType == typeof(DbContextOptions<AppDbContext>));
            services.Remove(descriptor);
            services.AddDbContext<AppDbContext>(o => o.UseNpgsql(_db.GetConnectionString()));
        });
    }

    public async Task InitializeAsync() => await _db.StartAsync();
    public new async Task DisposeAsync() => await _db.StopAsync();
}
```

---

#### Synthèse et pyramide de tests recommandée

```
        /\
       /E2E\          ← peu nombreux, lents, fragiles, haute confiance
      /______\
     /Sociable \      ← couverture des flux métier, bonne fidélité
    /____________\
   /  Solitaire   \   ← très nombreux, ultra-rapides, très ciblés
  /________________\
```

| Type | Isolement | Vitesse | Nombre idéal | Frameworks .NET |
|---|---|---|---|---|
| Solitaire | Total (tout mocké) | ~1 ms | Beaucoup | xUnit + Moq / NSubstitute |
| Sociable | Partiel (vrais objets domaine) | ~5–50 ms | Moyen | xUnit + Fakes maison |
| E2E | Aucun (stack complète) | ~500 ms – plusieurs s | Peu | xUnit + WebApplicationFactory + Testcontainers + Playwright |

> **Règle pratique :** Écrire beaucoup de tests solitaires pour le domaine, des tests sociables pour les flux applicatifs, et quelques tests E2E pour les parcours critiques. Si un comportement ne peut être couvert qu'en E2E, c'est souvent le symptôme d'un manque d'abstraction dans le code de production.

---

### 🟢 Quelle est la différence entre tests unitaires et tests d'intégration en .NET Core ?

| | Test unitaire | Test d'intégration |
|---|---|---|
| Portée | 1 classe / 1 méthode | Plusieurs couches (HTTP → DB) |
| Dépendances | Mockées / stubbées | Réelles ou in-memory |
| Vitesse | Très rapide (ms) | Plus lent (secondes) |
| Isolation | Maximale | Partielle |
| Frameworks | xUnit, NUnit, MSTest + Moq/NSubstitute | xUnit + WebApplicationFactory + Testcontainers |

```csharp
// Test unitaire : OrderService isolé, repository mocké
public class OrderServiceTests
{
    private readonly Mock<IOrderRepository> _repoMock = new();
    private readonly OrderService _sut;

    public OrderServiceTests()
        => _sut = new OrderService(_repoMock.Object, NullLogger<OrderService>.Instance);

    [Fact]
    public async Task GetById_ExistingOrder_ReturnsOrder()
    {
        var expected = new Order(Guid.NewGuid(), "Alice", 99.99m);
        _repoMock.Setup(r => r.GetByIdAsync(expected.Id, default))
                 .ReturnsAsync(expected);

        var result = await _sut.GetByIdAsync(expected.Id);

        Assert.Equal(expected.Id, result?.Id);
    }
}
```

---

### 🟢 Qu'est-ce que `WebApplicationFactory<TProgram>` ? À quoi sert-elle ?

`WebApplicationFactory<TProgram>` est la classe de test d'intégration officielle d'ASP.NET Core. Elle démarre votre application **en mémoire** avec une stack HTTP complète (pipeline, DI, middleware...) sans port réseau réel.

```csharp
// Classe de base partagée entre les tests d'un projet
public class ApiFactory : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.UseEnvironment("Testing");
        builder.ConfigureServices(services =>
        {
            // Remplacer la vraie DB par une in-memory
            var dbDescriptor = services.SingleOrDefault(
                d => d.ServiceType == typeof(DbContextOptions<AppDbContext>));
            if (dbDescriptor is not null)
                services.Remove(dbDescriptor);

            services.AddDbContext<AppDbContext>(o =>
                o.UseInMemoryDatabase("TestDb"));

            // Remplacer un service externe par un mock
            services.AddSingleton<IEmailSender, FakeEmailSender>();
        });
    }
}

// Test d'intégration
public class OrdersEndpointTests(ApiFactory factory)
    : IClassFixture<ApiFactory>
{
    private readonly HttpClient _client = factory.CreateClient();

    [Fact]
    public async Task CreateOrder_ValidRequest_Returns201()
    {
        var response = await _client.PostAsJsonAsync("/api/orders",
            new CreateOrderRequest("Alice", 99.99m));

        Assert.Equal(HttpStatusCode.Created, response.StatusCode);
        var order = await response.Content.ReadFromJsonAsync<OrderDto>();
        Assert.NotNull(order);
    }
}
```

---

### 🟡 Comment mocker des dépendances avec Moq et NSubstitute ?

```csharp
// Moq
var repoMock = new Mock<IOrderRepository>();

// Setup : retourner une valeur
repoMock.Setup(r => r.GetByIdAsync(It.IsAny<Guid>(), It.IsAny<CancellationToken>()))
        .ReturnsAsync(new Order(Guid.NewGuid(), "Alice", 99.99m));

// Vérification : méthode appelée exactement 1 fois
repoMock.Verify(r => r.SaveAsync(It.IsAny<Order>(), default), Times.Once);

// NSubstitute (API plus fluide)
var repo = Substitute.For<IOrderRepository>();

repo.GetByIdAsync(Arg.Any<Guid>(), Arg.Any<CancellationToken>())
    .Returns(new Order(Guid.NewGuid(), "Alice", 99.99m));

// Vérification
await repo.Received(1).SaveAsync(Arg.Any<Order>(), Arg.Any<CancellationToken>());
```

**Quand ne PAS mocker :**
- Les types `sealed` sans interface (utiliser un wrapper ou `Microsoft.Fakes` dans certains cas)
- Les `static` et `extension methods` → refactorer pour injecter une dépendance
- Les value types (struct) — inutile

> **Silver bullet** : si un test nécessite beaucoup de mocks, c'est souvent un signal que la classe fait trop de choses (SRP violation).

---

### 🟡 Qu'est-ce que Testcontainers ? Quand l'utiliser à la place d'une DB in-memory ?

Testcontainers démarre de **vraies instances Docker** (PostgreSQL, Redis, Kafka...) pour les tests d'intégration, garantissant une fidélité maximale avec la production.

```csharp
// dotnet add package Testcontainers.PostgreSql

public class PostgreSqlFixture : IAsyncLifetime
{
    private readonly PostgreSqlContainer _container = new PostgreSqlBuilder()
        .WithImage("postgres:16")
        .WithDatabase("testdb")
        .WithUsername("test")
        .WithPassword("test")
        .Build();

    public string ConnectionString => _container.GetConnectionString();

    public async Task InitializeAsync() => await _container.StartAsync();
    public async Task DisposeAsync() => await _container.StopAsync();
}

public class OrderRepositoryTests(PostgreSqlFixture db)
    : IClassFixture<PostgreSqlFixture>
{
    [Fact]
    public async Task GetById_WithRealDb_ReturnsOrder()
    {
        await using var ctx = CreateContext(db.ConnectionString);
        await ctx.Database.MigrateAsync();

        var repo = new OrderRepository(ctx);
        var order = new Order(Guid.NewGuid(), "Alice", 99.99m);
        await repo.AddAsync(order);
        await ctx.SaveChangesAsync();

        var found = await repo.GetByIdAsync(order.Id);
        Assert.Equal(order.Id, found?.Id);
    }
}
```

**In-memory vs Testcontainers :**

| | `UseInMemoryDatabase` | Testcontainers |
|---|---|---|
| Fidélité | Faible (pas de SQL réel) | Haute (vraie DB) |
| Vitesse | Très rapide | Plus lent (démarrage Docker) |
| Transactions | Non supportées | Supportées |
| Contraintes FK | Non enforced | Enforced |
| CI/CD | Toujours dispo | Nécessite Docker |

---

### 🔴 Comment tester les Minimal APIs avec `WebApplicationFactory` ?

```csharp
public class MinimalApiTests(ApiFactory factory) : IClassFixture<ApiFactory>
{
    private readonly HttpClient _client = factory.CreateClient();

    [Fact]
    public async Task GetOrder_ValidId_Returns200WithBody()
    {
        // Arrange : seed de donnée via le scope DI de la factory
        await using var scope = factory.Services.CreateAsyncScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        var order = new Order(Guid.NewGuid(), "Alice", 99.99m);
        db.Orders.Add(order);
        await db.SaveChangesAsync();

        // Act
        var response = await _client.GetAsync($"/api/orders/{order.Id}");

        // Assert
        response.EnsureSuccessStatusCode();
        var dto = await response.Content.ReadFromJsonAsync<OrderDto>();
        Assert.Equal(order.Id, dto?.Id);
    }

    [Theory]
    [InlineData("/api/orders", HttpStatusCode.Unauthorized)]     // sans auth
    public async Task ProtectedEndpoints_WithoutAuth_Return401(
        string path, HttpStatusCode expected)
    {
        var response = await _client.GetAsync(path);
        Assert.Equal(expected, response.StatusCode);
    }
}

// Factory avec authentification mockée
public class AuthenticatedApiFactory : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            // Remplacer l'auth par un handler de test
            services.AddAuthentication("Test")
                .AddScheme<AuthenticationSchemeOptions, TestAuthHandler>("Test", _ => { });
        });
    }
}
```

---

### 🔴 Comment tester les `BackgroundService` en isolation ?

```csharp
// Pattern 1 : extraire la logique métier dans un service injectable
public class DataSyncJob(IDataSyncService syncService, ILogger<DataSyncJob> logger)
{
    public async Task ExecuteAsync(CancellationToken ct)
    {
        logger.LogInformation("Starting sync");
        await syncService.SyncAsync(ct);
    }
}

// Tester le job isolément
[Fact]
public async Task ExecuteAsync_CallsSyncService()
{
    var syncMock = Substitute.For<IDataSyncService>();
    var job = new DataSyncJob(syncMock, NullLogger<DataSyncJob>.Instance);

    await job.ExecuteAsync(CancellationToken.None);

    await syncMock.Received(1).SyncAsync(Arg.Any<CancellationToken>());
}

// Pattern 2 : tester le BackgroundService via IHostedService
[Fact]
public async Task BackgroundService_StopsCleanlyOnCancellation()
{
    using var cts = new CancellationTokenSource(timeout: TimeSpan.FromSeconds(2));
    var service = new DataSyncBackgroundService(
        Substitute.For<IDataSyncService>(),
        NullLogger<DataSyncBackgroundService>.Instance);

    // StartAsync + StopAsync ne doivent pas lever d'exception
    await service.StartAsync(cts.Token);
    await service.StopAsync(CancellationToken.None);
}
```

---

### 🔴 Comment utiliser `TimeProvider` (.NET 8+) pour tester le code dépendant du temps ?

`TimeProvider` est une abstraction .NET 8 pour `DateTime.UtcNow`, `Task.Delay` et les timers, permettant de contrôler le temps dans les tests.

```csharp
// Service utilisant TimeProvider (injectable)
public class TokenService(TimeProvider timeProvider)
{
    public bool IsTokenExpired(Token token)
        => token.ExpiresAt < timeProvider.GetUtcNow();

    public async Task WaitForRefreshAsync(CancellationToken ct)
        => await Task.Delay(TimeSpan.FromSeconds(30), timeProvider, ct);
}

// Enregistrement
builder.Services.AddSingleton(TimeProvider.System);

// Test : FakeTimeProvider contrôle le temps
[Fact]
public void IsTokenExpired_ExpiredToken_ReturnsTrue()
{
    var fakeTime = new FakeTimeProvider();
    fakeTime.SetUtcNow(new DateTimeOffset(2026, 1, 1, 12, 0, 0, TimeSpan.Zero));

    var svc = new TokenService(fakeTime);
    var expired = new Token { ExpiresAt = new DateTimeOffset(2026, 1, 1, 11, 0, 0, TimeSpan.Zero) };

    Assert.True(svc.IsTokenExpired(expired));
}

// Test : avancer le temps
[Fact]
public async Task WaitForRefresh_CanBeAccelerated()
{
    var fakeTime = new FakeTimeProvider();
    var svc = new TokenService(fakeTime);

    var waitTask = svc.WaitForRefreshAsync(CancellationToken.None);
    fakeTime.Advance(TimeSpan.FromSeconds(30)); // avance le temps instantanément

    await waitTask; // complète immédiatement
}
```

---

### 🔴 Comment structurer les tests d'intégration pour éviter les effets de bord entre tests ?

```csharp
// Stratégie 1 : transaction rollback par test
public class TransactionalTestBase : IAsyncLifetime
{
    protected AppDbContext Context { get; private set; } = null!;
    private IDbContextTransaction _transaction = null!;

    public async Task InitializeAsync()
    {
        Context = CreateContext();
        _transaction = await Context.Database.BeginTransactionAsync();
    }

    public async Task DisposeAsync()
    {
        await _transaction.RollbackAsync(); // annule les changements du test
        await Context.DisposeAsync();
    }
}

// Stratégie 2 : recréer la DB entre les tests (Testcontainers + Respawn)
// dotnet add package Respawn
public class DatabaseFixture : IAsyncLifetime
{
    private readonly PostgreSqlContainer _container = new PostgreSqlBuilder().Build();
    private Respawner _respawner = null!;

    public async Task InitializeAsync()
    {
        await _container.StartAsync();
        // Migration initiale
        using var conn = new NpgsqlConnection(_container.GetConnectionString());
        await conn.OpenAsync();
        _respawner = await Respawner.CreateAsync(conn,
            new RespawnerOptions { DbAdapter = DbAdapter.Postgres });
    }

    // Appelé entre chaque test
    public async Task ResetAsync()
    {
        using var conn = new NpgsqlConnection(_container.GetConnectionString());
        await conn.OpenAsync();
        await _respawner.ResetAsync(conn); // DELETE ciblé, plus rapide que DROP/CREATE
    }

    public async Task DisposeAsync() => await _container.StopAsync();
}
```

---

## Lexique

| Terme | Définition |
|---|---|
| **AAA (Arrange / Act / Assert)** | Structure d'un test : préparation, exécution, vérification. |
| **Double de test (Test Double)** | Terme générique pour tout objet remplaçant une dépendance réelle dans un test (mock, stub, fake, spy, dummy). |
| **E2E (End-to-End)** | Test exercant la totalité de la pile applicative (HTTP → logique → base de données réelle) pour valider un scénario complet. |
| **Fake** | Implémentation légère mais fonctionnelle d'une dépendance (ex. repository en mémoire), sans framework de mock. |
| **Pyramide de tests** | Modèle recommandant beaucoup de tests unitaires à la base, des tests d'intégration au milieu, et peu de tests E2E au sommet. |
| **Sociable (Sociable Unit Test)** | Test unitaire laissant les vrais objets du domaine collaborer, mais remplaçant les dépendances avec effets de bord (I/O). |
| **Solitaire (Solitary Unit Test)** | Test unitaire isolant complètement l'unité testée de tous ses collaborateurs via des doubles de test. |
| **FakeTimeProvider** | Implémentation de `TimeProvider` pour les tests, permettant de contrôler et avancer le temps. |
| **Fixture** | Contexte partagé entre plusieurs tests (connexion DB, serveur HTTP), instancié une seule fois. |
| **IClassFixture\<T\>** | Interface xUnit partageant une instance entre tous les tests d'une classe. |
| **Mock** | Objet substitut d'une dépendance réelle, contrôlable et vérifiable dans les tests. |
| **NSubstitute** | Bibliothèque de mock .NET avec une API fluide, alternative à Moq. |
| **Respawn** | Bibliothèque réinitialisant une DB entre tests via DELETE ciblés (plus rapide que DROP/CREATE). |
| **Stub** | Implémentation factice d'une dépendance retournant des données fixes, sans vérification. |
| **SUT (System Under Test)** | Classe ou composant testé dans un test. |
| **Testcontainers** | Bibliothèque démarrant des containers Docker pour les tests d'intégration. |
| **TimeProvider** | Abstraction .NET 8+ pour le temps courant, les délais et les timers, injectable et testable. |
| **WebApplicationFactory** | Classe de test ASP.NET Core démarrant l'application en mémoire pour les tests d'intégration HTTP. |
| **xUnit** | Framework de test .NET (préféré par l'équipe ASP.NET Core) avec `[Fact]`, `[Theory]`, `IClassFixture`. |

---

## Ressources

### Documentation officielle Microsoft
- [Integration tests in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/test/integration-tests)
- [Unit testing in .NET](https://learn.microsoft.com/en-us/dotnet/core/testing/)
- [TimeProvider (.NET 8)](https://learn.microsoft.com/en-us/dotnet/api/system.timeprovider)

### Packages NuGet
- [xUnit](https://xunit.net/)
- [Moq](https://github.com/devlooped/moq)
- [NSubstitute](https://nsubstitute.github.io/)
- [Testcontainers for .NET](https://dotnet.testcontainers.org/)
- [Respawn](https://github.com/jbogard/Respawn)

### Blogs & Articles
- [Martin Fowler — Unit Test (Solitary vs Sociable)](https://martinfowler.com/bliki/UnitTest.html)
- [Martin Fowler — Test Pyramid](https://martinfowler.com/bliki/TestPyramid.html)
- [Jimmy Bogard — Integration testing ASP.NET Core](https://jimmybogard.com/)
- [Andrew Lock — Testing Minimal APIs](https://andrewlock.net/)
- [Nick Chapsas — .NET testing best practices (YouTube)](https://www.youtube.com/@nickchapsas)
