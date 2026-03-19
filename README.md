# .NET Interview — Questions Senior/Expert

Banque de questions d'entretien pour développeurs **senior et expert .NET**, centrée sur les performances, le multi-threading et la programmation asynchrone.

Chaque fichier contient des questions avec réponses détaillées et exemples de code, tagguées par niveau de difficulté (🟢 Basique · 🟡 Intermédiaire · 🔴 Senior/Expert), ainsi qu'un **lexique** et des **ressources** en fin de document.

> Versions ciblées : **.NET 8 / 9 / 10** · **C# 12 / 13 / 14**

---

## Contenu

| Fichier | Sujet |
|---|---|
| [senior/performances.md](senior/performances.md) | Mémoire, allocations, GC, JIT/AOT, Span, profiling, I/O |
| [senior/multithreading.md](senior/multithreading.md) | ThreadPool, synchronisation, deadlocks, collections thread-safe |
| [senior/async.md](senior/async.md) | async/await, ValueTask, Channel, IAsyncEnumerable, anti-patterns |
| [senior/dependency_injection.md](senior/dependency_injection.md) | Lifetimes, Keyed Services (.NET 8), Decorator, ValidateOnBuild, Primary Constructors |
| [senior/middleware_pipeline.md](senior/middleware_pipeline.md) | Pipeline, Use/Run/Map, Minimal APIs, Endpoint Filters (.NET 7+), Rate Limiting |
| [senior/resilience.md](senior/resilience.md) | Polly v8, Circuit Breaker, Hedging, AddStandardResilienceHandler (.NET 8) |
| [senior/observability.md](senior/observability.md) | OpenTelemetry, Activity, System.Diagnostics.Metrics (.NET 8), ILogger, dotnet-monitor |
| [senior/testing.md](senior/testing.md) | WebApplicationFactory, Testcontainers, TimeProvider (.NET 8), NSubstitute, Respawn |
| [senior/background_services.md](senior/background_services.md) | BackgroundService, PeriodicTimer, Channel&lt;T&gt;, graceful shutdown, health checks |
| [senior/security.md](senior/security.md) | JWT Bearer, Policy-based auth, Data Protection API, Claims Transformation |
| [senior/patterns.md](senior/patterns.md) | SOLID, Repository/UoW, Strategy, Decorator, CQRS/MediatR, Clean Architecture, Outbox, DDD Value Objects |
| [senior/scenarios.md](senior/scenarios.md) | Scénarios réels : diagnostic API lente, fuite mémoire, race condition, optimisation endpoint |

---

## Agent Copilot

Ce dépôt inclut un agent GitHub Copilot (`.github/agents/dotnet-interviewer.agent.md`) capable de :

- Générer de nouvelles questions par catégorie et niveau
- Réviser et améliorer le contenu existant
- Mettre à jour les fichiers lors de nouvelles versions .NET/C#
