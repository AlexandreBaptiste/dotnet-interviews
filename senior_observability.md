# Entretien .NET Senior/Expert — Observabilité

> **Versions ciblées :** .NET 8 / .NET 9 / .NET 10, C# 12 / C# 13 / C# 14
>
> **Niveaux de difficulté :** 🟢 Basique · 🟡 Intermédiaire · 🔴 Senior/Expert

---

## Les 3 piliers

### 🟢 Qu'est-ce que l'observabilité ? Quels sont ses 3 piliers ?

L'observabilité est la capacité à comprendre l'état interne d'un système en observant ses sorties externes, sans modifier le code pour chaque nouvelle question.

**Les 3 piliers :**

| Pilier | Quoi | Outil .NET |
|---|---|---|
| **Logs** | Événements textuels discrets horodatés | `ILogger<T>`, Serilog, NLog |
| **Métriques** | Mesures numériques agrégées dans le temps | `System.Diagnostics.Metrics`, Prometheus |
| **Traces** | Suivis de bout en bout d'une requête distribuée | `Activity` / `ActivitySource`, OpenTelemetry |

> Un système observable répond à la question *"Pourquoi le service est-il lent ?"* avec des données déjà collectées, sans nécessiter de redéploiement.

---

### 🟢 Qu'est-ce que `ILogger<T>` ? Comment structurer les logs en .NET Core ?

```csharp
// Injection via DI (automatiquement enregistré par CreateDefaultBuilder)
public class OrderService(ILogger<OrderService> logger)
{
    public async Task<Order?> GetByIdAsync(Guid id, CancellationToken ct)
    {
        // Structured logging : les {} sont des propriétés, pas des strings
        logger.LogInformation("Fetching order {OrderId}", id);

        var order = await _repo.GetByIdAsync(id, ct);

        if (order is null)
            logger.LogWarning("Order {OrderId} not found", id);

        return order;
    }
}

// Levels : Trace < Debug < Information < Warning < Error < Critical
// Ne jamais interpoler : logger.LogInformation($"Order {id}"); ← pas structuré, allocation string

// High-performance : [LoggerMessage] source generator (zéro allocation)
public static partial class Log
{
    [LoggerMessage(Level = LogLevel.Information, Message = "Processing order {OrderId}")]
    public static partial void ProcessingOrder(ILogger logger, Guid orderId);

    [LoggerMessage(Level = LogLevel.Warning, Message = "Order {OrderId} not found")]
    public static partial void OrderNotFound(ILogger logger, Guid orderId);
}

// Utilisation
Log.ProcessingOrder(logger, orderId);
```

---

### 🟡 Qu'est-ce qu'OpenTelemetry ? Comment l'intégrer dans une application ASP.NET Core ?

OpenTelemetry est un standard open source pour la collecte de télémétrie (logs, métriques, traces). Il fournit des APIs et des SDKs indépendants du backend (Jaeger, Prometheus, Azure Monitor...).

```csharp
// dotnet add package OpenTelemetry.Extensions.Hosting
// dotnet add package OpenTelemetry.Instrumentation.AspNetCore
// dotnet add package OpenTelemetry.Instrumentation.HttpClient
// dotnet add package OpenTelemetry.Exporter.Otlp

builder.Services.AddOpenTelemetry()
    .ConfigureResource(r => r.AddService(
        serviceName: "OrderService",
        serviceVersion: "1.0.0"))
    .WithTracing(tracing => tracing
        .AddAspNetCoreInstrumentation()   // traces HTTP entrantes
        .AddHttpClientInstrumentation()   // traces HTTP sortantes
        .AddEntityFrameworkCoreInstrumentation()
        .AddSource("OrderService.*")      // traces custom
        .AddOtlpExporter())              // envoi vers Jaeger, Grafana Tempo, etc.
    .WithMetrics(metrics => metrics
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddRuntimeInstrumentation()      // métriques GC, ThreadPool, etc.
        .AddPrometheusExporter())
    .WithLogging(logging => logging
        .AddOtlpExporter());
```

---

### 🟡 Qu'est-ce que `Activity` et `ActivitySource` ? Comment créer des traces distribuées custom ?

`Activity` est l'implémentation .NET de la spec OpenTelemetry **Span**. Elle représente une opération dans une trace distribuée.

```csharp
// Déclaration d'une source de traces (static, créée une seule fois)
public static class Telemetry
{
    public static readonly ActivitySource Source = new("OrderService", "1.0.0");
}

// Création d'une span custom
public async Task<ProcessResult> ProcessOrderAsync(Order order, CancellationToken ct)
{
    using var activity = Telemetry.Source.StartActivity("ProcessOrder");

    // Ajout de tags (attributs OpenTelemetry)
    activity?.SetTag("order.id", order.Id);
    activity?.SetTag("order.amount", order.TotalAmount);
    activity?.SetTag("order.currency", order.Currency);

    try
    {
        var result = await _paymentService.ChargeAsync(order, ct);

        activity?.SetTag("payment.status", result.Status);
        activity?.SetStatus(ActivityStatusCode.Ok);
        return result;
    }
    catch (Exception ex)
    {
        // Enregistrer l'exception dans la trace
        activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
        activity?.RecordException(ex);
        throw;
    }
}

// Propagation du contexte entre services (W3C TraceContext standard)
// HttpClient la propage automatiquement via AddHttpClientInstrumentation()
```

**Contexte distribué :**
```
Service A                      Service B
Activity("HandleRequest")  →   Activity("ProcessOrder")  →  Activity("ChargePayment")
TraceId: abc123                TraceId: abc123               TraceId: abc123 (même!)
SpanId:  000001                SpanId:  000002               SpanId:  000003
```

---

### 🔴 Qu'est-ce que `System.Diagnostics.Metrics` (.NET 8+) ? Comment créer des métriques custom ?

`System.Diagnostics.Metrics` est l'API native .NET pour créer des métriques compatibles OpenTelemetry, sans dépendance externe.

```csharp
// Déclaration (static, une seule fois par application)
public static class OrderMetrics
{
    private static readonly Meter _meter = new("OrderService", "1.0.0");

    // Counter : valeur qui monte (nb de commandes créées)
    public static readonly Counter<long> OrdersCreated =
        _meter.CreateCounter<long>("orders.created", "commandes",
            "Nombre de commandes créées");

    // Histogram : distribution de valeurs (temps de traitement)
    public static readonly Histogram<double> OrderProcessingDuration =
        _meter.CreateHistogram<double>("orders.processing.duration", "ms",
            "Durée de traitement d'une commande");

    // Gauge observable : valeur point-in-time (nb de commandes en attente)
    public static readonly ObservableGauge<int> PendingOrders =
        _meter.CreateObservableGauge<int>("orders.pending",
            () => OrderQueue.PendingCount,
            "commandes", "Commandes en attente de traitement");
}

// Utilisation dans le code
public async Task<Order> CreateOrderAsync(CreateOrderRequest req)
{
    var sw = Stopwatch.StartNew();
    try
    {
        var order = await ProcessAsync(req);

        OrderMetrics.OrdersCreated.Add(1,
            new KeyValuePair<string, object?>("currency", req.Currency),
            new KeyValuePair<string, object?>("region", req.Region));

        return order;
    }
    finally
    {
        OrderMetrics.OrderProcessingDuration.Record(sw.Elapsed.TotalMilliseconds,
            new KeyValuePair<string, object?>("status", "success"));
    }
}
```

**Enregistrement pour OpenTelemetry :**
```csharp
.WithMetrics(m => m.AddMeter("OrderService"))
```

---

### 🔴 Comment corréler logs, traces et métriques dans ASP.NET Core ?

```csharp
// La corrélation est automatique via ILogger + OpenTelemetry :
// Chaque log emis pendant une Activity est enrichi du TraceId/SpanId courant

// Middleware de corrélation (intégré via AddOpenTelemetry + AddAspNetCoreInstrumentation)
// Les logs contiendront automatiquement :
// { "TraceId": "abc123", "SpanId": "def456", "Message": "...", ... }

// Configuration Serilog avec enrichissement OpenTelemetry
builder.Host.UseSerilog((ctx, config) =>
{
    config
        .ReadFrom.Configuration(ctx.Configuration)
        .Enrich.WithProperty("ServiceName", "OrderService")
        .Enrich.WithProperty("Environment", ctx.HostingEnvironment.EnvironmentName)
        .WriteTo.Console(new JsonFormatter())
        .WriteTo.OpenTelemetry(); // envoi via OTLP
});

// Enrichissement manuel d'une Activity avec des infos de contexte
using var activity = Telemetry.Source.StartActivity("ProcessOrder");
activity?.SetBaggage("user.id", userId);     // propagé aux services enfants
activity?.SetTag("tenant.id", tenantId);     // visible dans les traces
```

**Stratégie de corrélation en production :**
1. **TraceId** propagé W3C TraceContext dans les headers HTTP
2. **Logs** enrichis automatiquement du TraceId courant via ILogger + OTel
3. **Métriques** taguées avec les dimensions (region, service, status)
4. **Alerting** : métriques → alertes → traces → logs pour investigation

---

### 🔴 Qu'est-ce que `dotnet-monitor` ? Comment l'utiliser en production / Kubernetes ?

`dotnet-monitor` est un outil sidecar exposant une API REST pour collecter de la télémétrie d'un processus .NET sans modifier son code.

```yaml
# docker-compose.yml : pattern sidecar
services:
  api:
    image: myapi:latest
    environment:
      - DOTNET_DiagnosticPorts=/tmp/diag/port

  monitor:
    image: mcr.microsoft.com/dotnet/monitor:8
    command: ["collect", "--no-auth"]
    volumes:
      - /tmp/diag:/tmp/diag
    ports:
      - "52323:52323"   # API diagnostics
      - "52325:52325"   # métriques Prometheus
```

**Endpoints disponibles :**

```bash
# Informations process
GET /info

# Dump mémoire à chaud
GET /dump?type=Full

# Trace CPU (5 secondes)
GET /trace?durationSeconds=5

# Métriques Prometheus
GET /metrics

# Logs en streaming
GET /logs?level=Warning
```

**Triggers automatiques (ruleset) :**
```json
{
  "Triggers": {
    "HighCpu": {
      "Type": "CpuUsage",
      "Settings": {
        "GreaterThan": 70,
        "SlidingWindowDuration": "00:00:05",
        "Actions": ["CollectDump"]
      }
    }
  }
}
```

---

## Lexique

| Terme | Définition |
|---|---|
| **Activity** | Représentation .NET d'une span OpenTelemetry, traçant une opération avec son contexte. |
| **ActivitySource** | Usine d'`Activity`, déclarée statiquement et nommée pour identifier la source des traces. |
| **Baggage** | Paires clé/valeur propagées dans le contexte distribué d'une trace, accessibles par tous les spans enfants. |
| **Counter** | Métrique monotone croissante (nb d'appels, nb d'erreurs). |
| **Gauge** | Métrique point-in-time pouvant monter ou descendre (nb de connexions actives). |
| **Histogram** | Métrique distribuant des valeurs en buckets pour calculer percentiles (p50, p95, p99). |
| **OTLP** | OpenTelemetry Protocol — protocole standard d'export de télémétrie vers les backends. |
| **OpenTelemetry** | Standard open source pour la collecte de logs, métriques et traces, indépendant du backend. |
| **Sampling** | Stratégie ne collectant qu'une fraction des traces pour réduire le volume de données. |
| **Span** | Unité de travail dans une trace : début, fin, tags, événements, statut. |
| **Structured logging** | Logging où les données sont structurées (JSON) avec des propriétés typées, pas de simples strings. |
| **TraceId** | Identifiant unique d'une trace distribuée, propagé entre services pour corréler les spans. |
| **W3C TraceContext** | Standard HTTP pour propager le TraceId/SpanId entre services via les headers `traceparent`. |
| **dotnet-monitor** | Outil sidecar exposant une API REST de diagnostics (.NET runtime) sans modification du code. |

---

## Ressources

### Documentation officielle Microsoft
- [.NET Observability with OpenTelemetry](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/observability-with-otel)
- [System.Diagnostics.Metrics](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/metrics-instrumentation)
- [Distributed tracing in .NET](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/distributed-tracing)
- [dotnet-monitor](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/dotnet-monitor)
- [ILogger structured logging](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/logging/)

### Packages NuGet
- [OpenTelemetry.Extensions.Hosting](https://opentelemetry.io/docs/languages/net/)
- [Serilog.AspNetCore](https://github.com/serilog/serilog-aspnetcore)

### Blogs & Articles
- [Andrew Lock — OpenTelemetry in .NET](https://andrewlock.net/)
- [Jimmy Bogard — Structuring logs and traces](https://jimmybogard.com/)
