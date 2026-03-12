# Entretien .NET Senior/Expert — Background Services

> **Versions ciblées :** .NET 8 / .NET 9 / .NET 10, C# 12 / C# 13 / C# 14
>
> **Niveaux de difficulté :** 🟢 Basique · 🟡 Intermédiaire · 🔴 Senior/Expert

---

## Fondamentaux

### 🟢 Quelle est la différence entre `IHostedService` et `BackgroundService` ?

`IHostedService` est l'interface de bas niveau. `BackgroundService` est une classe abstraite au-dessus qui simplifie l'implémentation en gérant le threading et l'annulation.

```csharp
// IHostedService : contrôle total, utile pour des tâches ponctuelles (warmup, etc.)
public class DataWarmupService(IDataCache cache) : IHostedService
{
    public async Task StartAsync(CancellationToken ct)
    {
        // Exécuté au démarrage du host — bloque jusqu'à la fin
        await cache.PreloadAsync(ct);
    }

    public Task StopAsync(CancellationToken ct)
    {
        // Nettoyage synchrone
        return Task.CompletedTask;
    }
}

// BackgroundService : pour les tâches longues en boucle
public class DataSyncService(ILogger<DataSyncService> logger) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        // ExecuteAsync s'exécute en arrière-plan en parallèle du reste de l'app
        while (!stoppingToken.IsCancellationRequested)
        {
            logger.LogInformation("Syncing data...");
            await DoSyncAsync(stoppingToken);
            await Task.Delay(TimeSpan.FromMinutes(5), stoppingToken);
        }
    }
}
```

> **Important :** si `ExecuteAsync` lève une exception non gérée, le `BackgroundService` s'arrête silencieusement par défaut. Il faut encapsuler dans un `try/catch` ou configurer `HostOptions.BackgroundServiceExceptionBehavior`.

---

### 🟢 Comment créer un Worker Service en .NET Core ? Quelle est sa structure ?

Un Worker Service est un projet de type `Worker Service` (`dotnet new worker`) — un host générique sans HTTP, idéal pour les consommateurs de messages, jobs planifiés, etc.

```csharp
// Program.cs (Worker Service)
var builder = Host.CreateApplicationBuilder(args);

builder.Services.AddHostedService<DataSyncService>();
builder.Services.AddHostedService<MessageConsumerService>();
builder.Services.AddDbContext<AppDbContext>(o =>
    o.UseSqlServer(builder.Configuration.GetConnectionString("Default")));

var host = builder.Build();
await host.RunAsync();

// Worker Service peut être déployé comme :
// - Service Windows   : builder.Services.UseWindowsService()
// - Service systemd   : builder.Services.UseSystemd()
// - Container Docker  : directement (process unique)
```

---

### 🟡 Comment gérer le graceful shutdown d'un `BackgroundService` ?

```csharp
public class MessageConsumerService(
    IMessageBroker broker,
    ILogger<MessageConsumerService> logger) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        // stoppingToken est signalé quand l'host s'arrête (SIGTERM, Ctrl+C)
        await broker.SubscribeAsync(async (message, ct) =>
        {
            await ProcessMessageAsync(message, ct);
        }, stoppingToken);
    }

    public override async Task StopAsync(CancellationToken cancellationToken)
    {
        logger.LogInformation("Stopping consumer — completing in-flight messages...");

        // Laisser jusqu'à 30 secondes pour finir les messages en cours
        await base.StopAsync(cancellationToken);

        await broker.FlushAsync();
        logger.LogInformation("Consumer stopped gracefully.");
    }
}

// Configuration du timeout de shutdown (défaut : 5 secondes)
builder.Services.Configure<HostOptions>(opt =>
{
    opt.ShutdownTimeout = TimeSpan.FromSeconds(30);
    // .NET 8 : comportement sur exception dans un BackgroundService
    opt.BackgroundServiceExceptionBehavior = BackgroundServiceExceptionBehavior.StopHost;
});
```

---

### 🟡 Comment utiliser des services Scoped dans un `BackgroundService` (Singleton) ?

Un `BackgroundService` est par nature un Singleton (une seule instance). Injecter directement un service Scoped (ex: `DbContext`) est une **captive dependency**. Il faut créer un scope par unité de travail.

```csharp
public class OrderProcessorService(
    IServiceScopeFactory scopeFactory,
    ILogger<OrderProcessorService> logger) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        using var timer = new PeriodicTimer(TimeSpan.FromSeconds(10));

        while (await timer.WaitForNextTickAsync(ct))
        {
            // Créer un nouveau scope par cycle → DbContext Scoped correctement géré
            await using var scope = scopeFactory.CreateAsyncScope();
            var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
            var processor = scope.ServiceProvider.GetRequiredService<IOrderProcessor>();

            var pendingOrders = await db.Orders
                .Where(o => o.Status == OrderStatus.Pending)
                .Take(50)
                .ToListAsync(ct);

            foreach (var order in pendingOrders)
                await processor.ProcessAsync(order, ct);

            await db.SaveChangesAsync(ct);
        } // scope disposé ici → DbContext fermé proprement
    }
}
```

---

### 🔴 Pourquoi `PeriodicTimer` est-il préférable à `Timer` ou `Task.Delay` en boucle ?

```csharp
// ❌ Task.Delay en boucle : drift — le temps de DoWork s'ajoute à l'intervalle
while (!ct.IsCancellationRequested)
{
    await DoWorkAsync(ct);
    await Task.Delay(TimeSpan.FromSeconds(30), ct); // 30s + durée de DoWork
}

// ❌ System.Threading.Timer : callback-based, réentrant
// Si le callback prend > 30s, deux exécutions se chevauchent
var timer = new Timer(_ => DoWork(), null, 0, 30_000);

// ✅ PeriodicTimer (.NET 6+) : async, non-réentrant, propre
public class CleanupService(ICleanupRepository repo) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        using var timer = new PeriodicTimer(TimeSpan.FromMinutes(1));

        while (await timer.WaitForNextTickAsync(ct))
        {
            // Non-réentrant par nature : le prochain tick attend que ce bloc termine
            await repo.DeleteExpiredAsync(ct);
        }
        // timer.Dispose() automatique via using
    }
}
```

**Avantages de `PeriodicTimer` :**
- **Non-réentrant** — pas de chevauchement possible
- **Async natif** — pas de thread bloqué entre les ticks
- **Annulable** — `WaitForNextTickAsync(ct)` retourne `false` sur annulation
- **Proche de l'intervalle** — ne dérive pas comme `Task.Delay`

---

### 🔴 Comment implémenter un pipeline producteur/consommateur avec `Channel<T>` dans un `BackgroundService` ?

```csharp
// Architecture : un producteur lit depuis RabbitMQ, N consommateurs traitent en parallèle
public class PipelineService(IMessageBroker broker, IOrderProcessor processor)
    : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        var channel = Channel.CreateBounded<Order>(new BoundedChannelOptions(500)
        {
            FullMode = BoundedChannelFullMode.Wait, // backpressure : le producteur attend
            SingleWriter = true,
            SingleReader = false
        });

        // Producteur : lit messages du broker et alimente le channel
        var producer = Task.Run(async () =>
        {
            try
            {
                await foreach (var msg in broker.ReadAllAsync(ct))
                    await channel.Writer.WriteAsync(msg, ct);
            }
            finally
            {
                channel.Writer.Complete();
            }
        }, ct);

        // 4 consommateurs en parallèle
        var consumers = Enumerable.Range(0, 4).Select(_ => Task.Run(async () =>
        {
            await foreach (var order in channel.Reader.ReadAllAsync(ct))
            {
                await using var scope = scopeFactory.CreateAsyncScope();
                var proc = scope.ServiceProvider.GetRequiredService<IOrderProcessor>();
                await proc.ProcessAsync(order, ct);
            }
        }, ct));

        await Task.WhenAll(consumers.Prepend(producer));
    }
}
```

---

### 🔴 Comment implémenter une stratégie de redémarrage et de robustesse dans un `BackgroundService` ?

```csharp
public class ResilientWorker(ILogger<ResilientWorker> logger) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            try
            {
                await DoWorkAsync(ct);
            }
            catch (OperationCanceledException) when (ct.IsCancellationRequested)
            {
                // Arrêt propre demandé : ne pas loguer comme erreur
                break;
            }
            catch (Exception ex)
            {
                logger.LogError(ex, "Worker error, restarting in 10s...");
                try
                {
                    await Task.Delay(TimeSpan.FromSeconds(10), ct);
                }
                catch (OperationCanceledException)
                {
                    break;
                }
            }
        }
    }
}

// Configuration du comportement global en cas d'exception non gérée
builder.Services.Configure<HostOptions>(o =>
{
    // StopHost : arrête l'application si un BackgroundService crashe (plus visible)
    o.BackgroundServiceExceptionBehavior = BackgroundServiceExceptionBehavior.StopHost;
});
```

**Stratégie recommandée :**
- Toujours catch dans `ExecuteAsync` et loguer
- Différencier `OperationCanceledException` (arrêt propre) des erreurs réelles
- Retry avec délai exponentiel en cas d'erreur de connectivité
- Utiliser Polly dans les opérations I/O du worker (voir fichier `senior_resilience.md`)

---

### 🔴 Comment implémenter des health checks pour un `BackgroundService` ?

```csharp
// Enregistrement des health checks
builder.Services.AddHealthChecks()
    .AddCheck<WorkerHealthCheck>("data-sync-worker");

app.MapHealthChecks("/healthz/ready",
    new HealthCheckOptions { Predicate = h => h.Tags.Contains("ready") });
app.MapHealthChecks("/healthz/live");

// Implémentation : le BackgroundService expose son état via un HealthCheck
public class DataSyncService : BackgroundService
{
    private DateTimeOffset _lastSuccessfulRun = DateTimeOffset.MinValue;
    private bool _isHealthy = true;

    public DateTimeOffset LastRun => _lastSuccessfulRun;
    public bool IsHealthy => _isHealthy;

    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        using var timer = new PeriodicTimer(TimeSpan.FromMinutes(5));
        while (await timer.WaitForNextTickAsync(ct))
        {
            try
            {
                await SyncAsync(ct);
                _lastSuccessfulRun = DateTimeOffset.UtcNow;
                _isHealthy = true;
            }
            catch (Exception ex) when (!ct.IsCancellationRequested)
            {
                _isHealthy = false;
                logger.LogError(ex, "Sync failed");
            }
        }
    }
}

public class WorkerHealthCheck(DataSyncService worker) : IHealthCheck
{
    public Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext ctx, CancellationToken ct)
    {
        if (!worker.IsHealthy)
            return Task.FromResult(HealthCheckResult.Unhealthy("Last sync failed"));

        var staleDuration = DateTimeOffset.UtcNow - worker.LastRun;
        if (staleDuration > TimeSpan.FromMinutes(15))
            return Task.FromResult(HealthCheckResult.Degraded(
                $"No successful run in {staleDuration.TotalMinutes:F0} minutes"));

        return Task.FromResult(HealthCheckResult.Healthy());
    }
}
```

---

## Lexique

| Terme | Définition |
|---|---|
| **BackgroundService** | Classe abstraite .NET Core simplifiant l'implémentation de services s'exécutant en arrière-plan. |
| **BackgroundServiceExceptionBehavior** | Option configurant si une exception non gérée dans un `BackgroundService` arrête le host. |
| **BoundedChannel** | Channel avec capacité maximale : le producteur attend si la file est pleine (backpressure). |
| **Graceful shutdown** | Arrêt propre permettant aux opérations en cours de se terminer avant la fin du processus. |
| **IHostedService** | Interface de bas niveau pour les services démarrés et arrêtés avec le host .NET. |
| **IServiceScopeFactory** | Factory permettant de créer des scopes DI dans des Singletons (ex: BackgroundService). |
| **PeriodicTimer** | Timer async non-réentrant (.NET 6+) idéal pour les tâches périodiques dans les BackgroundService. |
| **ShutdownTimeout** | Durée maximale accordée aux services pour s'arrêter proprement avant forçage. |
| **StoppingToken** | `CancellationToken` signalé quand le host commence son arrêt, passé à `ExecuteAsync`. |
| **Worker Service** | Template de projet .NET Core sans HTTP, basé sur le host générique, pour les tâches de fond. |

---

## Ressources

### Documentation officielle Microsoft
- [Background tasks with hosted services in ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/host/hosted-services)
- [Worker Services in .NET](https://learn.microsoft.com/en-us/dotnet/core/extensions/workers)
- [PeriodicTimer](https://learn.microsoft.com/en-us/dotnet/api/system.threading.periodictimer)
- [.NET Generic Host](https://learn.microsoft.com/en-us/dotnet/core/extensions/generic-host)

### Blogs & Articles
- [Andrew Lock — BackgroundService pitfalls](https://andrewlock.net/)
- [Steve Gordon — Worker Services in depth](https://www.stevejgordon.co.uk/)
- [David Fowler — ASP.NET Core diagnostic scenarios (background services)](https://github.com/davidfowl/AspNetCoreDiagnosticScenarios)
