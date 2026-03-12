# .NET Interview — Questions Senior/Expert

Banque de questions d'entretien pour développeurs **senior et expert .NET**, centrée sur les performances, le multi-threading et la programmation asynchrone.

Chaque fichier contient des questions avec réponses détaillées et exemples de code, tagguées par niveau de difficulté (🟢 Basique · 🟡 Intermédiaire · 🔴 Senior/Expert), ainsi qu'un **lexique** et des **ressources** en fin de document.

> Versions ciblées : **.NET 8 / 9 / 10** · **C# 12 / 13 / 14**

---

## Contenu

| Fichier | Sujet |
|---|---|
| [senior_performances.md](senior_performances.md) | Mémoire, allocations, GC, JIT/AOT, Span, profiling, I/O |
| [senior_multithreading.md](senior_multithreading.md) | ThreadPool, synchronisation, deadlocks, collections thread-safe |
| [senior_async.md](senior_async.md) | async/await, ValueTask, Channel, IAsyncEnumerable, anti-patterns |
| [senior_dependency_injection.md](senior_dependency_injection.md) | Lifetimes, Keyed Services (.NET 8), Decorator, ValidateOnBuild, Primary Constructors |
| [senior_middleware_pipeline.md](senior_middleware_pipeline.md) | Pipeline, Use/Run/Map, Minimal APIs, Endpoint Filters (.NET 7+), Rate Limiting |
| [senior_resilience.md](senior_resilience.md) | Polly v8, Circuit Breaker, Hedging, AddStandardResilienceHandler (.NET 8) |
| [senior_observability.md](senior_observability.md) | OpenTelemetry, Activity, System.Diagnostics.Metrics (.NET 8), ILogger, dotnet-monitor |
| [senior_testing.md](senior_testing.md) | WebApplicationFactory, Testcontainers, TimeProvider (.NET 8), NSubstitute, Respawn |
| [senior_background_services.md](senior_background_services.md) | BackgroundService, PeriodicTimer, Channel&lt;T&gt;, graceful shutdown, health checks |
| [senior_security.md](senior_security.md) | JWT Bearer, Policy-based auth, Data Protection API, Claims Transformation |
| [senior_scenarios.md](senior_scenarios.md) | Scénarios réels : diagnostic API lente, fuite mémoire, race condition, optimisation endpoint |

---

## Agent Copilot

Ce dépôt inclut un agent GitHub Copilot (`.github/agents/dotnet-interviewer.agent.md`) capable de :

- Générer de nouvelles questions par catégorie et niveau
- Réviser et améliorer le contenu existant
- Mettre à jour les fichiers lors de nouvelles versions .NET/C#
