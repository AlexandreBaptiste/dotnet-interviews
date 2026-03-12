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
| [senior_scenarios.md](senior_scenarios.md) | Scénarios réels : diagnostic API lente, fuite mémoire, race condition, optimisation endpoint |

---

## Agent Copilot

Ce dépôt inclut un agent GitHub Copilot (`.github/agents/dotnet-interviewer.agent.md`) capable de :

- Générer de nouvelles questions par catégorie et niveau
- Réviser et améliorer le contenu existant
- Mettre à jour les fichiers lors de nouvelles versions .NET/C#
