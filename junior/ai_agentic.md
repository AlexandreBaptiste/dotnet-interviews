# Entretien Junior — IA & Programmation Agentique

> **Niveaux de difficulté :** 🟢 Basique · 🟡 Intermédiaire · 🔴 Senior/Expert

---

## Qu'est-ce qu'un modèle de langage ?

### 🟢 Qu'est-ce qu'un LLM (Large Language Model) ? Donnez trois exemples courants.

Un **LLM** (Large Language Model) est un modèle d'intelligence artificielle entraîné sur d'immenses quantités de texte pour comprendre et générer du langage naturel. Il fonctionne en prédisant le token le plus probable à chaque étape.

**Exemples courants :**

| Modèle | Éditeur | Accès typique |
|--------|---------|--------------|
| GPT-4o / GPT-4.1 | OpenAI / Microsoft | API REST, Azure OpenAI, SDKs |
| Claude 3.7 Sonnet | Anthropic | API REST, Amazon Bedrock |
| Gemini 2.0 Flash | Google | API REST, Google AI Studio |

**Analogie simple :** un LLM est comme un moteur de complétion automatique extrêmement avancé — il prédit la suite du texte mot par mot (token par token) en tenant compte du contexte entier.

> **À retenir :** un LLM ne « comprend » pas vraiment — il calcule des probabilités sur son vocabulaire à partir de patterns appris lors de l'entraînement.

---

### 🟢 Qu'est-ce qu'un **token** ? Pourquoi est-ce important pour un développeur qui intègre un LLM ?

Un **token** est la plus petite unité de traitement d'un LLM. Le tokeniseur convertit le texte en suite d'entiers.

```
Exemple avec GPT-4 (tokeniseur cl100k_base) :

  "Bonjour développeur"  →  ["Bon", "jour", " dé", "vel", "opp", "eur"]  →  6 tokens
  "Hello developer"      →  ["Hello", " developer"]                       →  2 tokens
```

**Pourquoi c'est important :**

1. **Facturation** — les API LLM facturent à la consommation de tokens (input + output).
2. **Limite de contexte** — chaque modèle a une fenêtre maximale (ex. GPT-4o = 128 000 tokens).
   Si vous la dépassez, la requête échoue ou le texte est tronqué.
3. **Latence** — plus de tokens = réponse plus longue à générer.

```
Estimation rapide (côté client) :
  nombre_de_tokens ≈ nombre_de_caractères / 4  (approximation pour l'anglais)
  → Si count > seuil : réduire le prompt ou tronquer l'historique
```

> **Règle empirique :** 1 token ≈ 0,75 mot en anglais. Les langues non-latines (arabe, japonais, chinois) consomment significativement plus de tokens pour un même contenu.

---

### 🟢 Qu'est-ce qu'un **prompt** ? Quelle est la différence entre un `system message` et un `user message` ?

Un **prompt** est le texte envoyé au LLM pour lui demander quelque chose. L'API Chat Completions distingue trois rôles :

```
system    → Définit qui est l'IA, ses règles, ses limites.
              Injecté UNE FOIS au début. Invisible pour l'utilisateur.

user      → Message de la personne qui utilise l'application.

assistant → Réponse précédente du modèle.
              Sert à maintenir l'historique dans une conversation multi-tours.
```

**Exemple de structure de conversation :**

```
SYSTEM    : "Tu es un assistant informatique. Tu réponds uniquement sur des sujets techniques."

USER      : "Comment lire un fichier texte ?"
ASSISTANT : "Ouvre le fichier avec un reader, lis son contenu ligne par ligne..."

USER      : "Et pour les très gros fichiers ?"
ASSISTANT : "Pour les gros fichiers, utilise un stream pour éviter de tout charger en mémoire..."
```

> **Bonne pratique :** le system message doit être court, précis et stable. Évitez d'y mettre des informations dynamiques (date, données utilisateur) — utilisez le user message pour ça.

---

### 🟢 Qu'est-ce que GitHub Copilot ? Comment aide-t-il un développeur au quotidien ?

**GitHub Copilot** est un assistant de programmation basé sur l'IA, intégré directement dans les IDEs (VS Code, JetBrains, Visual Studio…). Il est alimenté par des modèles LLM de Microsoft/OpenAI.

**Ce qu'il peut faire :**

| Fonctionnalité | Exemple |
|---------------|---------|
| Complétion de code | Complète une méthode à partir de son nom et de son commentaire |
| Chat en ligne | Pose des questions sur le code sélectionné |
| Génération de tests | Génère automatiquement des tests unitaires |
| Explication de code | Explique un bloc de code complexe en langage naturel |
| Fix rapide | Propose des corrections sur les erreurs de compilation ou de lint |

**Comment fonctionne-t-il concrètement :**

1. Vous tapez votre code — Copilot lit le fichier actif et les onglets ouverts.
2. Il construit un prompt (préfixe + suffixe du curseur + contexte des fichiers voisins).
3. Il envoie ce prompt à un modèle LLM et affiche la suggestion en grisé.
4. Vous acceptez ou ignorez la suggestion.

> **Astuce** : nommez bien vos fonctions et variables, et ajoutez des commentaires — Copilot s'appuie fortement sur ces noms et descriptions pour générer le corps correct.

---

### 🟢 Qu'est-ce qu'un **framework d'orchestration IA** ? Donnez des exemples et expliquez pourquoi ils existent.

Un **framework d'orchestration IA** est une bibliothèque qui simplifie l'intégration de LLMs dans des applications. Sans lui, vous gérez manuellement les appels HTTP, la sérialisation JSON des messages, la gestion du contexte, les erreurs d'API...

**Frameworks populaires :**

| Framework | Langage principal | Éditeur |
|-----------|------------------|---------|
| Semantic Kernel | .NET, Python, Java | Microsoft |
| LangChain | Python, JavaScript | LangChain AI |
| LlamaIndex | Python, TypeScript | LlamaIndex |
| Spring AI | Java | Broadcom |

**Ce qu'ils apportent :**

- Abstraction des appels API LLM (OpenAI, Azure, Anthropic, Google…)
- Gestion de l'historique de conversation (multi-turn)
- Intégration des outils/fonctions (Tool Calling)
- Connecteurs vers les bases de données vectorielles
- Composants prêts à l'emploi pour le RAG et les agents

> **Analogie :** un framework d'orchestration IA est à l'IA ce qu'un ORM est aux bases de données — il vous évite de gérer les détails bas niveau tout en vous donnant accès à la puissance sous-jacente.

---

### 🟢 Qu'est-ce qu'un **outil (tool / function)** dans le contexte des agents IA ? Pourquoi sont-ils nécessaires ?

Un **outil** (aussi appelé *tool*, *function* ou *plugin* selon le framework) est une capacité externe que le LLM peut invoquer pour accomplir une tâche qu'il ne peut pas réaliser seul.

**Pourquoi les LLMs ont besoin d'outils :**

- Ils ne connaissent pas les données en temps réel (météo, cours de bourse, heure actuelle)
- Ils ne peuvent pas modifier des systèmes externes (envoyer un email, écrire en base de données)
- Leurs connaissances sont figées à leur date d'entraînement

**Cycle d'un appel d'outil :**

```
USER     : "Quel est le prix actuel de l'action MSFT ?"
    ↓
LLM      → décide d'appeler l'outil get_stock_price("MSFT")
    ↓
OUTIL    → interroge l'API bourse → retourne "432.50 USD"
    ↓
LLM      → génère la réponse : "L'action Microsoft (MSFT) vaut 432,50 USD."
```

**Types d'outils courants :**

| Type | Exemples |
|------|---------|
| Recherche | Web search, recherche documentaire interne |
| Données | Requêtes base de données, appels API REST |
| Calcul | Exécution de code, calculs complexes |
| Actions | Envoi d'email, création de ticket, gestion calendrier |

> **Règle de sécurité :** ne donner à un agent que les outils strictement nécessaires — principe du moindre privilège.

---

### 🟡 Qu'est-ce que le **RAG** (Retrieval-Augmented Generation) ? Expliquez le concept avec une analogie simple et un schéma.

**RAG** résout un problème fondamental des LLMs : leur connaissance est figée à la date de leur entraînement et ils ne connaissent pas VOS données internes.

**Analogie :** Imaginez un expert qui répond à des questions. Sans RAG, il répond uniquement de mémoire. Avec RAG, avant de répondre, il consulte d'abord les documents pertinents dans une bibliothèque, puis formule sa réponse en s'appuyant sur eux.

```
Sans RAG :
  [Question] → [LLM] → Réponse basée sur l'entraînement uniquement

Avec RAG :
  [Question]
      ↓
  [1. Recherche] → Trouve les 3-5 documents les plus pertinents
      ↓
  [2. Augmentation] → Injecte ces documents dans le prompt
      ↓
  [3. LLM] → Génère une réponse ancrée sur vos données réelles
      ↓
  [Réponse fiable et actuelle]
```

**Exemples d'usage :**

- Chatbot qui répond sur la documentation interne de votre entreprise
- Assistant qui connaît vos procédures RH
- Support technique qui répond en se basant sur vos tickets résolus

**Pseudo-code conceptuel :**

```
top_chunks = vector_db.similarity_search(question, top_k=5)
context    = join(top_chunks)
prompt     = "Réponds en te basant UNIQUEMENT sur : {context}\n\nQuestion : {question}"
response   = llm.complete(prompt)
```

---

### 🟡 Qu'est-ce qu'un **embedding** et pourquoi est-il nécessaire pour faire de la recherche sémantique ?

Un **embedding** est la représentation mathématique d'un texte sous forme d'un tableau de nombres flottants (vecteur). Des textes ayant un sens similaire produisent des vecteurs proches dans cet espace mathématique.

```
"chien"   → [0.12, -0.45, 0.78, ...]   ←── vecteurs proches
"canidé"  → [0.14, -0.43, 0.76, ...]   ←┘

"table"   → [-0.89, 0.23, -0.11, ...]  ←── vecteur éloigné
```

**Pourquoi c'est utile pour la recherche :**

- Recherche classique (SQL `LIKE`) : trouve `chien` si vous cherchez `chien` exactement.
- Recherche sémantique (embeddings) : trouve `chien`, `canidé`, `animal de compagnie` si vous cherchez `chien`.

**Modèles d'embedding courants :**

| Modèle | Dimensions | Éditeur |
|--------|-----------|---------|
| text-embedding-3-small | 1 536 | OpenAI |
| text-embedding-3-large | 3 072 | OpenAI |
| text-embedding-004 | 768 | Google |
| nomic-embed-text | 768 | Nomic AI (open-source) |

```
Processus :
  texte → modèle d'embedding → vecteur float[N]
  similarité = cosine_similarity(vecteur_question, vecteur_document)
  → Score proche de 1 = très similaire sémantiquement
```

> **À retenir :** les embeddings permettent de chercher par *sens* plutôt que par *mot exact*. C'est la brique fondamentale du RAG et de toute recherche IA sur des documents.

---

## Lexique & Ressources

### Lexique

| Terme | Définition simple |
|-------|------------------|
| **LLM** | Modèle d'IA entraîné sur du texte, capable de comprendre et de générer du langage naturel. |
| **Token** | Unité de découpage du texte pour le LLM. ≈ 0,75 mot en anglais. |
| **Prompt** | Le texte qu'on envoie au LLM pour lui demander quelque chose. |
| **System message** | Instructions permanentes définissant le comportement du LLM. |
| **User message** | Message de l'utilisateur dans la conversation. |
| **Context window** | Mémoire maximale du LLM pour une seule conversation (en tokens). |
| **Hallucination** | Quand le LLM invente une réponse fausse avec confiance. |
| **GitHub Copilot** | Assistant IA intégré dans les IDEs pour aider au développement. |
| **Framework d'orchestration IA** | Bibliothèque (Semantic Kernel, LangChain…) facilitant l'intégration de LLMs. |
| **Outil / Tool / Function** | Capacité externe (appel API, requête BDD…) invocable par le LLM. |
| **Embedding** | Représentation numérique (vecteur) d'un texte capturant son sens. |
| **RAG** | Retrieval-Augmented Generation : enrichir le prompt avec des documents pertinents. |
| **Température** | Paramètre [0–2] qui contrôle la créativité du LLM (0 = déterministe). |
| **Vector database** | Base de données spécialisée pour stocker et rechercher des vecteurs (Qdrant, Pinecone, Azure AI Search…). |

### Ressources

- [What are Large Language Models? — Microsoft Learn](https://learn.microsoft.com/en-us/azure/ai-services/openai/concepts/models)
- [Azure OpenAI Service — Documentation](https://learn.microsoft.com/en-us/azure/ai-services/openai/)
- [GitHub Copilot — Documentation](https://docs.github.com/en/copilot)
- [Semantic Kernel — Démarrage rapide](https://learn.microsoft.com/en-us/semantic-kernel/get-started/quick-start-guide)
- [LangChain — Documentation](https://python.langchain.com/docs/introduction/)
- [Microsoft AI Blog — Semantic Kernel updates](https://devblogs.microsoft.com/semantic-kernel/)
- [RAG Pattern — Azure Architecture Center](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/rag/rag-solution-design-and-evaluation-guide)
- [Tiktoken — Tokenizer for OpenAI models](https://github.com/openai/tiktoken)
- [OWASP Top 10 for LLM Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
