# Construire des applications Agentic AI de niveau production

> Cours complet reconstruit à partir d'un crash course (Krish Naik Academy, formateurs : Divesh, Yash, Chirantan, Paul) couvrant les quatre piliers indispensables pour construire des agents IA fiables et déployables en production : **Guardrails**, **Evals**, **Mémoire agentique**, **Agent Ops**.

---

## Table des matières

- Partie 0 — Introduction et enjeux
- Partie 1 — LLM Guardrails (sécuriser les LLM)
- Partie 2 — LLM Evals (évaluer rigoureusement une application LLM)
- Partie 3 — Techniques de mémoire agentique (13 techniques, du buffer au forgetting)
- Partie 4 — Agent Ops (déploiement, scaling, observabilité en production)
- Glossaire
- Auto-évaluation

---

## Partie 0 — Introduction et enjeux

La plupart des personnes qui publient du contenu sur les LLM se concentrent sur l'implémentation d'un cas d'usage : assembler des appels LLM, brancher LangChain, Pydantic, etc. Très peu abordent ce qui différencie réellement une application robuste d'un prototype : **les couches de sécurité, d'évaluation, de mémoire et d'exploitation (ops)**. Ce sont ces quatre piliers qui déterminent si une application LLM peut réellement tenir en entreprise, à l'échelle, dans la durée.

Ce cours suit la même progression que le crash course original :
1. **Guardrails** — empêcher le système de sortir du cadre prévu et de fuir des informations.
2. **Evals** — mesurer objectivement la qualité des réponses avant et après mise en production.
3. **Mémoire agentique** — donner à l'agent une continuité dans le temps sans faire exploser les coûts ni le contexte.
4. **Agent Ops** — déployer, faire monter en charge, observer et sécuriser tout cela en environnement réel (Kubernetes, CI/CD, tracing).

---

## Partie 1 — LLM Guardrails

### 1.1 Pourquoi la sécurité des LLM est un sujet négligé

Deux types de cas d'usage LLM dominent aujourd'hui : les usages **agentiques** et les usages **RAG** (Retrieval-Augmented Generation, où le LLM interroge une base de documents propre à l'entreprise). Dans les deux cas, un système non sécurisé peut être détourné de son usage prévu.

Deux motivations poussent à investir dans la sécurité :
- **La sécurité proprement dite** : empêcher les utilisateurs de faire dire au modèle des choses qu'il ne devrait pas dire, ou de lui faire révéler des informations sensibles (identité du fournisseur LLM utilisé en interne, coordonnées personnelles, etc.).
- **La réduction des coûts** : chaque token consommé sur une requête hors-sujet est un token facturé inutilement. Bloquer les questions hors périmètre en amont réduit la facture.

### 1.2 Démonstration : un chatbot RAG « marketing »

Exemple concret : un chatbot entraîné sur les données propres d'un site (formations, projets, webinaires) répond très bien aux questions pertinentes (« suggère-moi du contenu NLP », « suggère-moi un projet sur Azure »). Mais sans garde-fou :
- Si on lui demande « comment faire un café, je m'ennuie », un bot bien conçu doit refuser et rediriger vers son domaine (comme un service client dédié refuserait une question hors sujet).
- Si on lui demande le numéro de téléphone personnel du fondateur, il doit refuser de fournir une donnée personnelle.

Ce comportement de refus/redirection est la première fonction visible des guardrails.

### 1.3 Le schéma de sécurité : guardrails en entrée ET en sortie

Sans garde-fou : `Utilisateur → LLM → Réponse`, directement.

Avec garde-fou :
`Utilisateur → Guardrails (vérification de la question) → LLM → Guardrails (vérification de la réponse) → Utilisateur`

Les guardrails interviennent donc à deux endroits : sur la question entrante (est-elle acceptable ?) et sur la réponse sortante (le LLM n'a-t-il pas répondu quelque chose d'irrelevant, de dangereux ou de faux ?).

### 1.4 Le problème d'échelle en entreprise

Dans un contexte d'entreprise, on manipule souvent des dizaines de Go de documents, dont seule une fraction (quelques centaines de Mo) est réellement pertinente pour une requête donnée. Un système doit rester :
- **Robuste** (« robust ») ;
- **Tolérant aux pannes** (« fault tolerant », résolu via des *gateways*, sujet d'un cours précédent) ;
- **Sans latence excessive** (« latency free ») ;
- **Sécurisé** (« secured »), et c'est le sujet de cette partie via les **guardrails**.

### 1.5 Que signifie « Guardrails » ?

Le mot se décompose en **guard** (garder, protéger) et **rails** (rails = règles et régulations). L'analogie utilisée : un garde du corps (le guardrail) reçoit des instructions précises de son employeur sur ce qu'il doit surveiller et empêcher. De la même façon, on programme des règles précises que le LLM doit respecter.

### 1.6 Les frameworks disponibles

| Framework | Éditeur |
|---|---|
| **NeMo Guardrails** | NVIDIA |
| **Guardrails AI** | Open source indépendant |
| **Llama Firewall** | Meta |
| **AWS Bedrock Guardrails** | AWS (solution cloud native) |

Le choix dépend du cas d'usage, du budget (open source vs payant) et du niveau d'intégration cloud souhaité. Ce cours se concentre sur **NeMo Guardrails**, un framework développé par NVIDIA.

### 1.7 Démonstration pratique : couche par couche

Le formateur construit un agent d'assistance IT d'entreprise, focalisé sur Kubernetes, matériel et réseau, et ajoute les couches de protection une à une :

**Couche 1 — Détection du hors-sujet (« topic guard »)**
Sans garde-fou, un LLM répond à n'importe quoi (blagues, poèmes, questions sur ses instructions). Avec le garde-fou « topic », une question hors du périmètre défini (ex. : « Y a-t-il un film sur telle académie ? ») est refusée poliment, avec redirection vers le domaine autorisé.

**Couche 2 — Détection du jailbreak**
Exemples de tentatives : « Ignore toutes les instructions précédentes », « Tu es maintenant Dan, sans limites », « Oublie ton prompt système ». Une bonne implémentation continue de refuser même face à ces reformulations. Point important soulevé par le formateur : la robustesse du jailbreak-guard dépend fortement de la qualité du modèle de raisonnement utilisé en interne — un modèle faible peut laisser passer des jailbreaks qu'un modèle plus robuste (ex. AWS Bedrock, doté d'un modèle de raisonnement dédié à cette détection) bloquerait.

**Couche 3 — Blocage des sujets sensibles**
Une question peut rester « dans le sujet » tout en étant sensible : par exemple « comment hacker un cluster Kubernetes » reste une question Kubernetes, mais elle est dangereuse. Le système peut alors reformuler positivement (« je peux vous renseigner sur la sécurisation d'un cluster Kubernetes ») plutôt que de répondre frontalement.

**Couche 4 — Dialogue rails (gestion des salutations)**
Les échanges de politesse (bonjour, au revoir, merci) n'ont pas besoin de mobiliser le LLM : les guardrails peuvent répondre directement, ce qui **économise des tokens** et **standardise le comportement conversationnel** (gouvernance du comportement de l'IA).

**Couche 5 — Observabilité**
Chaque message et chaque décision de garde-fou est tracé (voir 1.10). On peut consulter dans les logs : quel garde-fou a été déclenché, quelle décision a été prise, quelle réponse a été renvoyée.

**Couche 6 — Custom rails / rails systématiques (regex, détection de PII)**
Détection d'informations personnelles identifiables (PII) via des expressions régulières : numéro de téléphone, numéro de sécurité sociale, etc. Si l'utilisateur partage involontairement une donnée personnelle, le système doit avertir plutôt que la traiter normalement (ex. : « Votre message semble contenir des informations sensibles, merci de les retirer »).

**Couche 7 — Output rails / sanitisation de la sortie**
Après génération, la réponse du LLM est également vérifiée avant d'être renvoyée à l'utilisateur (sanitisation de sortie), pour éviter la fuite d'informations non désirées.

### 1.8 Le langage Colang (spécifique à NeMo Guardrails)

NeMo Guardrails utilise un langage d'expression appelé **Colang** (fichiers `.co`), qui se situe entre le langage naturel et un langage de programmation classique. Il ne possède que quatre mots-clés :

- `define` : pour déclarer un élément ;
- `user` : pour définir ce qu'un utilisateur peut dire (avec des exemples de phrases) ;
- `bot` : pour définir ce qu'un bot doit répondre ;
- `flow` : pour lier un déclencheur utilisateur à une réponse bot (équivalent d'un `if/else`).

Exemple de logique : on définit `user ask off topic` avec une liste d'exemples de phrases hors sujet, on définit `bot refuse off topic` avec la réponse type, puis on définit un `flow` nommé qui dit : si `user off topic` est détecté, alors `bot refuse off topic`.

**Mécanisme interne de détection** : chaque nouvelle requête utilisateur est convertie en vecteur (embedding), de même que les exemples déclarés dans les rails. La requête est comparée par **similarité vectorielle** (et non par égalité stricte) aux exemples déclarés ; si la similarité dépasse un seuil, le rail correspondant est déclenché. Le moteur de similarité utilisé par défaut est **FastEmbed**, téléchargé automatiquement à l'installation de NeMo Guardrails.

**Limite importante** : si une question de l'utilisateur ne ressemble à aucun des exemples déclarés dans les rails, le score de similarité peut être trop faible pour déclencher la détection — c'est la principale faiblesse de NeMo Guardrails en mode autonome (sans LLM interne pour juger). C'est pourquoi, en usage standalone (hors intégration LangChain), NeMo Guardrails n'utilise **pas nécessairement de LLM** pour l'analyse d'intention, ce qui peut faire échouer certaines détections de jailbreak plus subtiles. **Pour un usage réellement production-grade, AWS Bedrock Guardrails est recommandé** : il embarque un modèle de raisonnement spécifiquement entraîné à détecter les tentatives de contournement, ce qui le rend beaucoup plus difficile à contourner.

### 1.9 Types de rails (récapitulatif théorique)

1. **Input rails** — appliqués sur ce que l'utilisateur envoie.
2. **Output rails** — appliqués sur ce que le LLM renvoie.
3. **Custom rails (ou rails systématiques)** — logique Python personnalisée, notamment les expressions régulières pour détecter des schémas précis (PII, urgence, etc.).

### 1.10 Observabilité et écosystème Pydantic

Le cours utilise **Pydantic Logfire** pour tracer chaque interaction. Différence clé avec LangSmith : Logfire couvre l'observabilité générale de l'exécution applicative (pas seulement les appels LLM), alors que LangSmith est strictement dédié à l'observabilité des LLM.

Écosystème Pydantic (à mettre en miroir avec l'écosystème LangChain : LangChain / LangGraph / LangSmith) :
- **Pydantic Validation** — couche de validation de données (utilisée historiquement par de nombreux frameworks : LangChain, AutoGen, CrewAI, FastAPI).
- **Pydantic AI** — framework pour construire des workflows agentiques, né du constat que tout le monde utilisait déjà Pydantic pour la validation.
- **Pydantic Logfire** — couche d'observabilité (équivalent de LangSmith côté Pydantic).

Vocabulaire de la tracabilité : un **span** est une entrée de log individuelle (un appel), une **trace** est l'ensemble des spans liés à une même interaction, et la représentation visuelle en cascade de ces spans s'appelle un **waterfall**.

### 1.11 Mise en pratique (BYOK — Bring Your Own Key)

La démonstration invite à créer ses propres clés API : une clé **Groq** (fournisseur LLM) et une clé **Pydantic Logfire** (observabilité), à renseigner dans une interface pour tester soi-même le pipeline de guardrails (détection hors-sujet, jailbreak, sujets sensibles, PII) et visualiser les traces générées en temps réel.

---

## Partie 2 — LLM Evals (évaluation des applications LLM)

### 2.1 Point de départ : une application RAG « brute »

Démonstration d'une application Streamlit simple : on téléverse un PDF (exemple utilisé : le papier *Attention Is All You Need*), le document est découpé en chunks, vectorisé, et on peut poser des questions dessus. Cette version basique manque de plusieurs éléments identifiés collectivement : **reranking**, **recherche hybride**, et surtout — le sujet de ce module — **l'évaluation**.

**Pourquoi les evals sont indispensables** : sans elles, impossible de savoir si l'application hallucine, si elle répond correctement à toutes les catégories de questions, ou si elle est prête pour la production.

### 2.2 Analogie pédagogique : le recrutement

Aucune entreprise sérieuse n'embauche sans entretien. Un candidat est jugé sur deux types d'information :
1. **Des scores prédéfinis** (notes, diplômes, CGPA) — l'équivalent des **benchmarks** pour un LLM (comparaisons publiques entre modèles lors de leur sortie).
2. **Un entretien en face-à-face**, évalué au regard d'un poste précis — l'équivalent de l'**évaluation applicative personnalisée** (comment ce LLM performe-t-il *dans mon application spécifique* ?).

Autrement dit, il existe deux niveaux d'évaluation d'un LLM :
- **Évaluation du modèle** (benchmarks, leaderboards) — relève surtout de la recherche, pas du travail quotidien du développeur.
- **Évaluation de l'application** (l'architecture personnalisée que vous avez conçue pour votre cas d'usage) — c'est ici que 90 % du travail du développeur se concentre, et c'est le cœur de ce module.

### 2.3 Construire un pipeline d'évaluation personnalisé

Trois briques sont nécessaires :
1. **Un jeu de données « goldens »** (vérité de référence).
2. **Des métriques spécifiques à la tâche**.
3. **Un LLM-juge** (« LLM as a judge »).

### 2.4 Les « Goldens »

Un *golden* est l'unité de base sur laquelle on évalue l'application. Il contient généralement :
- **La requête (query)** — une question représentative posée par un utilisateur final ;
- **La vérité attendue (« truth » / « reference »)** — la réponse idéale à cette question ;
- **Le contexte attendu (chunks / sources)** — les extraits de documents qui devraient étayer la réponse.

**Qui rédige les goldens ?** Idéalement des experts métier du domaine concerné (ex. : professionnels de santé pour une application médicale), car ils connaissent la vérité en profondeur. Problème : leur temps est rare et coûteux, il faut donc l'utiliser avec parcimonie plutôt que de leur faire relire chaque interaction.

Cette logique est très proche de la séparation **données d'entraînement / données de validation** en Machine Learning classique : les goldens servent à valider que le système répond correctement, pas à l'entraîner.

### 2.5 Les deux phases de l'évaluation

**Phase 1 — Exécution du pipeline (obtenir les résultats réels)**
On fait tourner le pipeline RAG réel sur chaque golden pour obtenir : la réponse réelle générée (« actual answer ») et le contexte réellement récupéré (« retrieved context »). Analogie : comme un examinateur qui doit d'abord recueillir les copies des étudiants avant de pouvoir les comparer au corrigé.

**Phase 2 — Évaluation (comparer réel vs attendu)**
On compare la réponse réelle à la réponse attendue, et le contexte récupéré au contexte attendu. Faire cela manuellement pour 5 goldens est faisable ; pour 100 ou plus, cela devient ingérable. D'où le besoin d'automatiser via un **LLM-juge**.

### 2.6 Pourquoi un LLM comme juge ?

Une simple comparaison d'embeddings (similarité sémantique) est rapide et peu coûteuse, mais insuffisante pour des requêtes complexes. Un LLM « état de l'art » (SOTA — *state of the art*, ex. GPT, Claude Opus) donne une évaluation bien plus fine, au prix d'un coût plus élevé — mais ce coût reste **inférieur à celui d'un expert humain**.

**Point clé de conception** : on ne peut pas simplement demander au LLM-juge « est-ce que c'est bon ? » de façon libre — il faut lui fournir une **structure précise** (une métrique définie avec sa méthode de calcul), sinon l'évaluation manque de rigueur et de reproductibilité. C'est le rôle des frameworks d'évaluation comme **Ragas** ou **DeepEval**.

### 2.7 Les métriques Ragas expliquées en détail

**1. Faithfulness (fidélité)**
Vérifie si la réponse générée est *fondée* sur le contexte récupéré (détection d'hallucination). Méthode : le LLM-juge décompose la réponse en **affirmations atomiques** (« atomic claims »), puis vérifie pour chacune si elle peut être déduite du contexte récupéré. Score = (nombre d'affirmations « ancrées » dans le contexte) / (nombre total d'affirmations). Exemple concret du cours : 4 affirmations, 3 ancrées, 1 non ancrée (hallucinée) → score de 0,75.

**2. Answer relevancy (pertinence de la réponse)**
Vérifie si la réponse est pertinente par rapport à la question posée. Méthode inversée : le LLM-juge lit la réponse générée et invente des questions auxquelles elle répondrait ; on calcule ensuite la similarité moyenne entre ces questions générées et la question originale de l'utilisateur (recherche dense par embeddings, pas une recherche par mot-clé). Un score faible peut signaler une réponse hors-sujet, remplie de remplissage (« padded »), ou incomplète (ne couvrant pas tous les aspects de la question).

**3. Context precision (précision du contexte)**
Vérifie si les chunks récupérés sont **bien classés** (les plus pertinents en premier). Le LLM-juge attribue une pertinence (pertinent/non pertinent) à chaque chunk selon son rang, avec une pénalité plus forte si un chunk non pertinent apparaît avant un chunk pertinent. Le reranking améliore directement cette métrique.

**4. Context recall (rappel du contexte)**
Vérifie si le contexte récupéré couvre **l'intégralité** des affirmations présentes dans la réponse de référence (le « golden »). Le LLM-juge extrait les affirmations atomiques de la réponse de référence, puis vérifie si chacune peut être attribuée à un chunk récupéré. Un score bas indique un **problème côté récupération** (retrieval), pas côté génération : chunk manquant, paramètre *K* (nombre de chunks récupérés) trop petit, ou écart d'embedding.

**5. Answer correctness (exactitude de la réponse)**
Combine deux composantes pondérées :
- Une composante **factuelle** (F1) : le LLM-juge compare les affirmations de la réponse réelle à celles de la réponse de référence, en classant chaque affirmation comme correcte, hallucinée ou manquante.
- Une composante de **similarité sémantique** : similarité cosinus entre les embeddings de la réponse réelle et de la réponse de référence.
- Formule : `score = W1 × F1_factuel + W2 × similarité_sémantique`, où W1 et W2 sont des poids (hyperparamètres) ajustables selon ce que l'on privilégie.

### 2.8 Contraintes pratiques du LLM-juge

**Explosion du nombre d'appels** : pour 5 goldens × 5 métriques = **25 appels LLM**. Envoyer tous ces appels simultanément à un fournisseur (ex. Groq en offre gratuite) déclenche des limites de débit (*rate limits*). Solution : introduire des **temporisations** (« cooldowns ») — un temps de pause après chaque métrique (ex. 25 secondes) et après chaque golden (ex. 35 secondes), configurables.

**Pourquoi calculer une métrique par appel plutôt que tout en un seul appel ?** Les LLM, comme les humains, performent moins bien quand on leur donne trop de contexte à traiter en même temps. Règle empirique évoquée : un LLM donne ses meilleurs résultats lorsque sa fenêtre de contexte est utilisée à environ 20 % de sa capacité. Comme le LLM-juge doit évaluer l'application, on veut éviter à tout prix qu'il « hallucine » son propre jugement (analogie : un juge de tribunal biaisé ou halluciné rendrait un jugement injuste).

**Éviter le biais de fournisseur** : il est déconseillé d'utiliser le même fournisseur pour le LLM générateur et le LLM-juge (risque de biais systématique) — d'où l'usage de deux clés API différentes dans la démonstration (ex. Groq pour la génération, Gemini pour le jugement).

### 2.9 Résultats et tableau de bord

L'application de démonstration propose :
- Un **score par golden** (par métrique) ;
- Un **résultat détaillé par golden** (réponse réelle, chunks récupérés, réponse de référence) ;
- Un **export JSON** des résultats ;
- La possibilité de tester soi-même en fournissant sa propre clé Groq et Gemini.

---

## Partie 3 — Techniques de mémoire agentique

### 3.1 Cadrage général

La mémoire agentique est un sujet vaste, souvent mal compris : beaucoup de personnes utilisent des termes comme « mémoire sémantique », « mémoire épisodique », « mémoire procédurale » sans comprendre pourquoi ces techniques existent ni ce qu'elles résolvent réellement. Ce module retrace la **lignée** (« lineage ») de la mémoire agentique, c'est-à-dire l'évolution historique des techniques, chacune ayant été inventée pour corriger le défaut de la précédente.

**Lectures de référence citées dans le cours** :
- Un article de recherche (National University of Singapore, début 2026) sur la taxonomie de la mémoire : ce qui « porte » la mémoire (mémoire au niveau des tokens, mémoire plate, planaire, hiérarchique, paramétrique, latente).
- **Magma** — architecture de mémoire agentique multi-graphe.
- **Graffiti** (Neo4j) — mémoire basée sur les graphes de connaissances.
- **« Small Language Models are the future of Agentic AI »** (NVIDIA) — position selon laquelle les petits modèles de langage (SLM) sont suffisamment puissants, plus adaptés opérationnellement et plus économiques pour la plupart des invocations agentiques.
- **Reflexion** — cadre de renforcement par feedback verbal (base de la mémoire auto-réflexive).
- **Self-Refine** — cadre itératif d'auto-amélioration par auto-évaluation.
- La **courbe de l'oubli d'Ebbinghaus** — fondement mathématique du « forgetting and decay ».

**Principe fondamental à retenir** : les LLM sont **sans état (stateless) par nature** — ils ne se souviennent de rien entre deux appels API. Ce n'est pas un bug, c'est une caractéristique voulue (isolation). Toute mémoire doit donc être **construite explicitement autour du modèle**.

**Concept de « hot path » vs « cold path »** : certaines mises à jour de mémoire se font en temps réel, dans le fil de la conversation (« hot path »), au prix d'une latence supplémentaire (ex. mémoire d'entités) ; d'autres se font en arrière-plan, de façon asynchrone, sans bloquer l'utilisateur (« cold path », ex. mémoire épisodique générée en fin de session).

### 3.2 Technique 1 — Conversation Buffer Memory

**Principe** : une simple **liste** (« buffer ») à laquelle on ajoute (« append ») chaque message (système, utilisateur, assistant) au fil de la conversation.

**Problème** : la liste grandit à chaque tour (« turn » = un échange question/réponse), donc le nombre de tokens envoyés à l'API croît de façon quasi linéaire, voire explosive avec l'ajout de pièces jointes. Exemple chiffré du cours : une conversation « coach financier » de 5 tours consomme déjà 18,3 % du budget de tokens total, avec une croissance de ×6,6 entre le premier et le dixième tour.

**Conclusion** : aucun système en production n'utilise le buffer brut comme unique couche de mémoire — il reste néanmoins très utilisé pour la **session active courante**, en complément d'une mémoire à long terme.

### 3.3 Technique 2 — Sliding Window Memory (mémoire à fenêtre glissante)

**Principe** : ne conserver que les *K* derniers tours de conversation (K = hyperparamètre). Quand un nouveau tour arrive et que la fenêtre est pleine, le tour le plus ancien est **évincé** (pas supprimé — voir techniques hybrides ci-après).

**Problème** : résout le problème de coût, mais introduit un **problème d'oubli**. Exemple du cours : un utilisateur donne son salaire au tour 1 ; au tour 6, la fenêtre (taille 4 ou 5) a évincé cette information, et l'agent redemande le salaire, frustrant l'utilisateur — analogie avec la limite des réseaux de neurones récurrents (RNN) que les architectures Transformer ont justement cherché à dépasser.

**Nuance importante** : la fenêtre est définie en **tours**, pas en tokens — un tour peut être très court ou très long, ce qui rend le budget de tokens seulement approximatif avec cette méthode (contrairement au *token buffer memory*, technique 5).

**Verdict pratique** : quasiment toujours présente dans les systèmes en production, mais **jamais seule** — toujours associée à une couche de récupération à long terme (vector store, graphe de connaissances).

### 3.4 Technique 3 — Summary Memory (mémoire par résumé)

**Principe** : au lieu de jeter l'historique ancien, on le **compresse**. Un LLM secondaire condense périodiquement les tours les plus anciens en un résumé textuel continu, qui remplace les tours bruts dans la fenêtre de contexte. C'est une **compression abstractive** (le modèle génère un nouveau texte qui capture l'essence, il ne se contente pas d'extraire des phrases existantes).

**Déclenchement** : par seuil, pas à chaque tour — soit par nombre de tours (« turn-based »), soit par volume de tokens (« token-based »).

**Risque fondamental : la compression avec perte (« lossy compression »)**. Le modèle décide seul de ce qui est important, et peut se tromper. Un fait rare mais critique (ex. « le client est allergique aux produits actions ») peut survivre à un premier résumé, puis disparaître lors d'un troisième cycle de compression — un risque documenté par la recherche récente, particulièrement critique dans les domaines médical et financier.

**Résumé progressif / hiérarchique** : quand le résumé lui-même devient trop long, on le re-résume à un niveau plus condensé. Hiérarchie typique :
- Niveau 0 : tours bruts, mot pour mot (verbatim) ;
- Niveau 1 : résumé glissant (« rolling summary ») ;
- Niveau 2 : résumé de session (un paragraphe par session) ;
- Niveau 3 : profil utilisateur (faits clés uniquement, le plus condensé).

### 3.5 Technique 4 — Summary Buffer Memory (mémoire hybride buffer + résumé)

**Principe** : combine le buffer de conversation et la mémoire par résumé. Les messages **récents** sont conservés **mot pour mot** (verbatim), tandis que l'historique **ancien** est progressivement résumé. Analogie : se souvenir textuellement des dernières phrases d'un appel téléphonique, mais seulement de l'essentiel de ce qui a été dit vingt minutes plus tôt.

**Défi d'ingénierie principal** : régler le **seuil de transition** (le point où un message passe du buffer brut au résumé) et la répartition du budget de tokens entre les deux zones.

**Cas d'usage** : agents de support client, chatbots de coaching long terme — tout usage nécessitant à la fois un contexte récent précis et une conscience historique globale.

### 3.6 Technique 5 — Token Buffer Memory

**Principe** : proche de la fenêtre glissante, mais le seuil est défini en **nombre exact de tokens**, pas en nombre de tours. Formule : `tokens en entrée = tokens du prompt système + min(tokens de l'historique, tokens maximum du buffer)`.

**Avantages** : mémoire à court terme la plus simple et la plus prévisible — pas d'appel de compression, pas de pic de latence, pas de dépendance externe, budget de tokens exact (contrairement à la fenêtre glissante, seulement approximative).

**Implémentation citée** : la classe `ConversationTokenBufferMemory` de LangChain.

### 3.7 Technique 6 — Vector Store Memory (mémoire vectorielle, début du long terme)

**Rupture par rapport aux techniques précédentes** : c'est la première technique qui **persiste au-delà de la session** (les précédentes vivent en RAM et se réinitialisent à la fermeture de la session). Chaque échange est converti en vecteur (embedding), stocké dans une base persistante, et à chaque nouveau tour on récupère les messages passés **sémantiquement les plus pertinents** — la récupération se fait par **pertinence sémantique**, pas par position ni par récence.

**Vector store vs vector database** : nuance technique soulevée dans le cours — un vector store n'a pas nécessairement d'opérations CRUD complètes, d'authentification ou de gestion de la concurrence, alors qu'une vraie base de données vectorielle (ex. **Chroma DB**) les propose.

**Structure de stockage** : chaque document est découpé en chunks ; chaque chunk est lui-même un objet contenant `page_content` (le texte, qui sera vectorisé) et `metadata`. Une fois stocké dans la base vectorielle, chaque entrée contient un identifiant unique (ID), le chunk, l'embedding, et les métadonnées associées.

**Importance stratégique** : le vector store est le cœur des applications RAG — environ 80 à 90 % des projets IA d'entreprise actuels reposent sur une architecture RAG.

**Combinaisons hybrides possibles** discutées : vector store + conversation buffer, vector store + fenêtre glissante, vector store + token buffer — chaque combinaison a ses avantages et inconvénients (le buffer complet noie le signal pertinent dans du bruit ; la fenêtre glissante a plus de sens à combiner avec un vector store car les messages évincés peuvent y être stockés plutôt que perdus).

### 3.8 Technique 7 — Entity Memory (mémoire d'entités)

**Fondement NLP** : s'appuie sur la **reconnaissance d'entités nommées** (Named Entity Recognition, NER) — la capacité à identifier et catégoriser dans un texte des entités comme une personne, une organisation, un lieu, une date, une devise (ex. distinguer « Tesla » l'entreprise automobile de « Nikola Tesla » le scientifique, ou « Amazon » l'entreprise de « Amazon » la forêt).

**Principe** : au lieu de stocker des messages bruts, on extrait des **faits structurés** sur des entités nommées, stockés typiquement comme un simple JSON/dictionnaire (ex. `{"nom": "Chirantan", "salaire": 120000}`). Analogie : une fiche contact CRM mise à jour à chaque nouvelle information (« Sarah a été promue » → la fiche de Sarah est mise à jour).

**Mode d'exécution** : typiquement en **« hot path »** — la mémoire se met à jour en temps réel dans la conversation, ce qui introduit une légère latence supplémentaire mais garantit une information toujours à jour.

**Avantages** : information toujours actuelle (mise à jour en place, pas de faits obsolètes), récupération par **lookup direct** (pas besoin de recherche par similarité cosinus, juste un filtrage sur les métadonnées).

**Limite** : l'extraction peut parfois mal classer ou halluciner une entité. Surtout : la mémoire d'entités **ne capture pas les relations entre entités** — pour cela, il faut se tourner vers une mémoire de type graphe (ex. Graffiti/Neo4j), hors du périmètre détaillé de ce module.

### 3.9 Technique 8 — Episodic Memory (mémoire épisodique)

**Fondement théorique** : s'appuie sur la distinction proposée par le psychologue **Endel Tulving en 1972** entre mémoire épisodique (événements spécifiques situés dans le temps) et mémoire sémantique (connaissances générales, indépendantes du moment où elles ont été apprises — voir technique 9).

**Principe** : chaque session est stockée comme un **épisode complet, horodaté** (timestamped), avec ce qui a été discuté, quelles décisions ont été prises, quels conseils ont été donnés, et le contexte émotionnel. Analogie : on ne se souvient pas de sa vie comme une liste plate de faits, mais comme des épisodes ancrés dans un temps et un lieu.

**Frontière d'épisode (« episode boundary »)** : la ligne qui sépare deux épisodes distincts. Elle peut être définie par session (une session = un épisode) ou par changement de sujet (chaque nouveau sujet démarre un nouvel épisode).

**Génération** : les épisodes sont générés **en fin de session** (pas tour par tour), via une résumé structurée exécutée **de façon asynchrone**, sans faire attendre l'utilisateur.

**Stratégies de récupération** : hybrides — combinant similarité sémantique, filtrage temporel et filtrage par type d'épisode.

**Avantages** : historique **immutable** (utile pour la conformité et les audits), capacité de raisonnement temporel (« que s'est-il passé en avril ? »), raisonnement par cas (« case-based reasoning », s'appuyer sur des épisodes passés similaires).

**Inconvénients** : ajoute de la latence après la session, qualité dépendante du prompt de génération du résumé, plus coûteux que le stockage brut, ne remplace ni la mémoire d'entités ni la mémoire vectorielle.

### 3.10 Technique 9 — Semantic Memory (mémoire sémantique)

**Principe** : distille, à partir des enregistrements épisodiques bruts, des **faits généraux, réutilisables et durables**, ainsi que des **schémas de comportement**, indépendamment du moment où ils ont été appris. Exemple : savoir que Paris est la capitale de la France, sans se souvenir du moment précis où on l'a appris — ou, dans le contexte agentique, savoir qu'un utilisateur panique systématiquement en période de volatilité des marchés et a besoin d'être rassuré avant de décider, en synthétisant plusieurs sessions passées.

**Cas d'usage typique** : construire un profil utilisateur croissant au fil du temps (langage de programmation préféré, taille d'équipe, cible de déploiement mentionnés au fil de sessions successives) — sans mémoire sémantique, chaque session repart de zéro.

### 3.11 Technique 10 — Procedural Memory (mémoire procédurale)

**Principe** : au lieu de stocker des faits sur l'utilisateur, la mémoire procédurale stocke des **règles réutilisables sur le comportement de l'agent lui-même** — des workflows étape par étape, des règles de décision. Concrètement, elle **vit dans le prompt système** : elle est injectée comme des **directives/instructions**, et non comme du contexte d'arrière-plan (contrairement à la mémoire épisodique ou sémantique, injectées comme blocs de contexte ou faits utilisateur).

Analogie humaine : savoir faire du vélo sans avoir besoin de se rappeler de la session où on l'a appris — le comportement est encodé, pas le souvenir de l'apprentissage.

**Point clé** : les procédures sont **apprises à partir des résultats obtenus**, pas seulement déclarées à l'avance — c'est ce qui en fait un pont vers les **agents auto-améliorants**.

**Risque** : de mauvaises procédures peuvent se renforcer et provoquer des erreurs systématiques (une forme de « confusion de contexte » ou « empoisonnement du contexte », sujet plus large de l'ingénierie du contexte). Des procédures peuvent aussi entrer en conflit entre elles.

### 3.12 Technique 11 — Self-Reflection Memory (mémoire d'auto-réflexion)

**Principe** : après chaque tâche ou session, l'agent **analyse sa propre performance**, identifie ce qu'il a bien fait, ce qu'il a mal fait, ce qu'il aurait dû faire différemment, et rédige des notes d'amélioration structurées, réinjectées lors des sessions futures.

Analogie : un médecin qui, après une consultation difficile, prend cinq minutes pour se demander s'il a posé les bonnes questions, manqué des symptômes, communiqué clairement — et consigne ces réflexions dans un journal personnel consulté avant la consultation suivante.

**Fondement académique** : le papier **Reflexion** montre que des agents capables de réfléchir verbalement sur leurs tentatives échouées et d'intégrer ces réflexions dans leurs tentatives suivantes surpassent significativement les agents entraînés sans réflexion — et ce, **sans mettre à jour les poids du réseau de neurones**, uniquement via un retour linguistique/verbal. Le papier **Self-Refine** propose un cadre similaire d'auto-amélioration itérative par auto-évaluation.

**Lien avec les autres techniques** : la mémoire d'auto-réflexion utilise souvent la **mémoire épisodique** en sous-couche pour stocker le texte réflexif, et recouvre en grande partie le même rôle que la **mémoire procédurale** (mettre à jour le comportement futur de l'agent), avec une implémentation différente. C'est une technique **gourmande en tokens**.

### 3.13 Technique 12 — Memory Routing (routage de la mémoire)

**Principe** : chaque message entrant subit une **classification d'intention**, et un routeur dirige la requête vers le magasin de mémoire adapté :
- Fait structuré sur l'utilisateur (« quel est mon salaire actuel ? ») → **entity store** ;
- Question historique (« qu'avons-nous décidé en avril ? ») → **episodic store** ;
- Question de connaissance générale (« comment fonctionne un SIP/plan d'investissement ? ») → **vector store** ;
- Mise à jour d'un fait (« j'ai changé de travail ») → mise à jour de l'**entity store**, potentiellement en cascade dans plusieurs magasins à la fois (« fan-out ») : profil d'entité, mémoire procédurale, buffer, vector store.
- Contrainte comportementale exprimée par l'utilisateur (« ne me suggère plus jamais de cryptomonnaies ») → mise à jour de la **mémoire procédurale** (contrainte système) et du buffer.

Cette approche est très utilisée dans les systèmes d'entreprise à grande échelle, où l'on ne peut pas se permettre de dépendre d'une seule couche de mémoire pour des milliers de requêtes concurrentes très diverses.

### 3.14 Technique 13 — Forgetting and Decay (oubli et décroissance de la mémoire)

**Principe directeur** : conserver indéfiniment toute la mémoire n'est **pas toujours souhaitable**. Un agent qui se souvient de tout devient plus lent à interroger, accumule du bruit, et perd la capacité à distinguer ce qui compte aujourd'hui de ce qui comptait il y a deux ans.

**Fondement mathématique — courbe de l'oubli d'Ebbinghaus** : `R(t) = e^(-t/S)`, où R(t) est la rétention de la mémoire au temps t, et S la « stabilité » de cette mémoire. Cette formule sert de base à toute la gestion de la mémoire par décroissance.

**Concept de demi-vie (« half-life »)** : le temps nécessaire pour que la force d'un souvenir tombe à la moitié de sa valeur actuelle. Un score de demi-vie qui tend vers zéro indique que le moment est venu d'évincer ce souvenir. Exemple : une demi-vie de 24 heures signifie qu'un souvenir non renforcé perd la moitié de sa force chaque jour.

**Quatre stratégies d'oubli** :

1. **TTL — Time To Live** : chaque souvenir a une date d'expiration fixe, après laquelle il est supprimé ou archivé. Avantage : simple, déterministe, borne de stockage prévisible. Inconvénient : des faits rares mais critiques (allergies, contraintes strictes) ne devraient pas s'effacer selon un calendrier fixe. Idéal pour des données à forte rotation (actualités de marché, agents boursiers).

2. **LRU — Least Recently Used** : quand le stockage est plein, on évince les souvenirs les moins récemment consultés (chaque accès réinitialise l'horloge). Avantage : conserve ce qui est réellement utilisé, imite le fonctionnement du cache d'un système d'exploitation. Inconvénient : le même que le TTL — un fait rare mais critique peut être élagué simplement parce qu'il n'a pas été consulté récemment.

3. **Importance-weighted eviction (éviction pondérée par l'importance)** : chaque souvenir reçoit un score d'importance, fonction de la récence, du nombre d'accès, de la pertinence sémantique et d'un poids de catégorie. C'est une approche plus fine que les précédentes, apparentée à l'élagage contrôlé (« control pruning ») plutôt qu'à une suppression heuristique brute.

4. **Budget-constrained pruning (élagage sous contrainte de budget)** : élagage motivé par une contrainte globale de stockage/tokens plutôt que par l'âge ou la fréquence d'un souvenir individuel.

---

## Partie 4 — Agent Ops (déploiement, scaling et observabilité en production)

### 4.1 Prototype vs réalité de production

Les notebooks Jupyter sont excellents pour le prototypage : clés API en dur, exécution locale, aucune concurrence, aucun mode de défaillance géré. La réalité de la production impose : la **mise à l'échelle** (des milliers d'utilisateurs concurrents), la gestion de **sorties non déterministes à grande échelle**, la possibilité de **rollback** (retour à un état antérieur, notamment via Kubernetes), et une véritable **orchestration de déploiement**.

### 4.2 Exemple d'architecture réelle n°1 : agent de recherche clinique (santé)

Un projet de recherche s'appuyant sur deux bases de données publiques américaines : **ClinicalTrials.gov** (environ 500 000 études cliniques) et **PubMed** (articles de recherche). Architecture :
- **Airflow** pour l'ingestion des données ;
- **Google Cloud Storage** pour les données brutes/traitées, et **Cloud SQL** utilisé à la fois comme base relationnelle et comme base vectorielle ;
- Une couche de traitement (chunking, embedding) ;
- **LangMem** pour la mémoire, combinant mémoire épisodique (sessions passées de recherche), procédurale et sémantique selon les besoins ;
- Une architecture **multi-agents** : six agents travaillant en parallèle plus un septième agent orchestrateur ;
- Un **human-in-the-loop** piloté par des seuils de confiance (l'intervention humaine n'est déclenchée que si le niveau de confiance de l'agent est insuffisant), complété par une **boucle d'apprentissage** qui réinjecte les retours vers les agents ;
- Une couche d'observabilité utilisant un gestionnaire de secrets, la journalisation Cloud (GCP Cloud Logging) et **LangSmith**.

### 4.3 Exemple d'architecture réelle n°2 : moteur RAG scientifique déployé sur Kubernetes (AWS EKS)

**Composants de la stack** :
- **Apache Airflow** : orchestration de l'ingestion (DAG comportant les étapes : configuration de l'environnement, recherche des papiers sur arXiv, indexation, génération d'un rapport quotidien, nettoyage des fichiers temporaires) ;
- **Neon** : base de données Postgres serverless, utilisée pour stocker les métadonnées des documents ingérés et le suivi des exécutions Airflow ;
- **OpenSearch** : base de données vectorielle, avec un tableau de bord natif pour la gestion des index et des métadonnées associées à chaque chunk (résumé, ID arXiv, auteurs, catégories) ;
- **FastAPI** : application exposant les routes de l'API (santé du service, recherche simple, question/réponse, streaming, route agentique) ;
- **Redis (via Upstash)** : cache serverless, avec une durée de vie (TTL) configurable (ex. 6 heures), permettant de répondre instantanément à une question déjà posée ;
- **LangFuse** : observabilité/traçage détaillé de chaque requête agentique (latence totale, étapes traversées, score des guardrails) ;
- **Docker Compose** : orchestration locale de l'ensemble des services (réseaux Docker, health checks, volumes persistants, mapping de ports).

**Pipeline RAG complet** : requête utilisateur → vérification du cache Redis → si absent du cache, recherche hybride dans la base vectorielle (combinaison de **BM25**, recherche par mots-clés/dérivée de TF-IDF, et de recherche **dense** par embeddings, fusionnées via **Reciprocal Rank Fusion**, RRF) → assemblage du contexte (construction du prompt) → appel au LLM → réponse. Le passage par le cache ramène le temps de réponse de 20-30 secondes à 200-300 millisecondes.

**Parsing des documents** : utilisation de **Docling** pour l'extraction, avec un découpage par section plutôt qu'un découpage naïf — le cours souligne qu'une mauvaise étape de parsing condamne d'avance la qualité du chunking.

**Guardrails en production (AWS Bedrock)** : filtres de contenu (haine, insultes, violence, comportement répréhensible, contenu sexuel, tentatives d'attaque de prompt), **refus thématique** (« topic denial », bloquant toute question hors du domaine autorisé — ex. refuser une question de recette de cuisine sur un système dédié à la recherche scientifique), **détection et rédaction (redaction) de PII** — email, téléphone, numéro de carte bancaire — avec une nuance importante : contrairement au blocage total, la donnée personnelle est **masquée (« redacted »)** plutôt que le message entier rejeté, et des seuils de **grounding** (les chunks récupérés sont-ils pertinents ?) et de **relevance** (la réponse générée est-elle pertinente ?) complètent le dispositif.

**Human-in-the-loop pour l'évaluation** : une route API dédiée permet de soumettre un score et un commentaire de feedback sur une trace donnée, **de façon asynchrone**, sans alourdir le temps de réponse initial — le feedback vient nourrir un tableau de bord d'évaluation manuelle dans LangFuse.

**Graphe LangGraph de l'agent** : `start → guardrails → vérification hors-sujet → (si hors-sujet) reformulation de la requête → retrieve → sélection d'outil (ex. l'outil de recherche OpenSearch) → génération de la réponse → end`.

**Conversion en serveur MCP (Model Context Protocol)** : l'ensemble de l'application FastAPI est exposé comme un **serveur MCP**, ce qui permet à n'importe quel client compatible (inspecteur MCP, assistants de codage, applications de chat) de découvrir et d'invoquer les mêmes routes comme des « outils », sans réimplémenter la logique métier. C'est une pratique de plus en plus répandue en entreprise : exposer une application existante en MCP plutôt que de construire une interface dédiée pour chaque nouveau client.

**Intégration Telegram** : exemple d'un canal supplémentaire branché sur la même API, illustrant qu'une même logique métier peut alimenter plusieurs canaux (API REST, bot de messagerie, serveur MCP) sans duplication.

### 4.4 Déploiement Kubernetes (AWS EKS) et mise à l'échelle

**Structure du cluster** : créé via `eksctl` (alternative possible : Terraform, CDK, Pulumi), avec un espace de noms (« namespace ») dédié à la production, séparé du namespace de supervision (monitoring/Grafana).

**Mise à l'échelle horizontale (Horizontal Pod Autoscaling, HPA)** : de nouveaux pods (répliques de l'application) sont automatiquement créés lorsque l'utilisation mémoire/CPU dépasse un seuil cible (exemple du cours : seuil à 70 %, minimum 2 pods, maximum 6 pods).

**Mise à l'échelle verticale** : chaque pod dispose d'une limite mémoire minimale et maximale (exemple : 6 Go minimum, 8 Go maximum) ; contrairement au scaling horizontal (multiplier les pods), le scaling vertical augmente les ressources allouées à un même pod.

**Test de charge (Locust)** : démonstration en direct où l'on augmente progressivement le nombre d'utilisateurs concurrents simulés. Observations clés :
- À faible charge (10 requêtes concurrentes), le taux d'échec reste proche de zéro avec seulement 2 pods actifs.
- En augmentant la charge (20, puis 50 requêtes concurrentes), l'utilisation CPU grimpe, de nouveaux pods passent en état « pending » puis démarrent, et le taux d'échec augmente transitoirement pendant la montée en charge, avant de se stabiliser une fois les nouveaux pods opérationnels.
- **Les limites ne viennent pas seulement de Kubernetes** : elles proviennent aussi des services tiers en aval — dans l'exemple du cours, AWS Bedrock plafonnait autour de 20 requêtes concurrentes, l'API Gemini autour de 100, et le pool de connexions SQLAlchemy vers Neon autour de 40-50. **Concevoir une architecture pour 10 000 utilisateurs concurrents nécessite de repenser chacune de ces dépendances**, éventuellement en auto-hébergeant certains composants plutôt qu'en dépendant de services managés à quota limité.

### 4.5 CI/CD et gouvernance de production

- **CI/CD via GitHub Actions** : construction des images Docker, exécution de tests d'intégration (incluant potentiellement le jeu de données « golden » du module Evals), publication des images, déploiement sur EKS.
- **Rollback** facilité par Kubernetes (retour à un état antérieur du déploiement).
- **Observabilité double** : traçage applicatif général (logs Kubernetes/Grafana) et traçage spécifique aux appels LLM (LangFuse).
- **Les trois piliers qui distinguent un projet « production-grade » d'un projet jouet**, selon le formateur : **l'infrastructure, la sécurité et la gouvernance**. Un projet qui néglige l'un de ces trois piliers reste, dans les faits, une application à échelle moyenne, pas un système de production.

---

## Glossaire

| Terme | Définition courte |
|---|---|
| Guardrails | Couches de règles appliquées en entrée et/ou en sortie d'un LLM pour encadrer son comportement. |
| Jailbreak | Tentative de contourner les instructions système d'un modèle pour lui faire produire un contenu interdit. |
| Colang | Langage d'expression (ni naturel, ni de programmation classique) utilisé par NeMo Guardrails, basé sur `define`, `user`, `bot`, `flow`. |
| Golden (golden dataset) | Jeu de données de référence (requête + réponse attendue + contexte attendu) servant de vérité pour l'évaluation. |
| LLM as a judge | Utilisation d'un LLM pour évaluer automatiquement la qualité d'une réponse selon des métriques structurées. |
| Faithfulness | Métrique mesurant si une réponse est fondée sur le contexte récupéré (absence d'hallucination). |
| Context precision / recall | Mesure de la qualité du classement (precision) et de l'exhaustivité (recall) des chunks récupérés. |
| Hot path / Cold path | Mise à jour de mémoire en temps réel dans la conversation (hot) vs en arrière-plan asynchrone (cold). |
| Entity memory | Mémoire structurée stockant des faits sur des entités nommées (personnes, organisations, etc.). |
| Episodic memory | Mémoire stockant des sessions complètes horodatées comme des épisodes distincts. |
| Semantic memory | Mémoire stockant des faits généraux et durables, indépendants du moment de leur apprentissage. |
| Procedural memory | Mémoire encodant des règles de comportement de l'agent, injectées comme instructions système. |
| Self-reflection memory | Mémoire où l'agent analyse ses propres performances passées pour s'améliorer. |
| Memory routing | Classification de l'intention d'un message pour le diriger vers le bon type de mémoire. |
| Half-life (demi-vie) | Temps nécessaire pour qu'un souvenir perde la moitié de sa force selon la courbe d'oubli d'Ebbinghaus. |
| HPA (Horizontal Pod Autoscaling) | Mécanisme Kubernetes ajoutant/retirant automatiquement des répliques selon la charge. |
| MCP (Model Context Protocol) | Protocole permettant d'exposer une application comme un ensemble d'outils invocables par un client IA. |

---

## Auto-évaluation

1. Pourquoi les guardrails doivent-ils s'appliquer à la fois en entrée et en sortie du LLM ?
2. Expliquez comment NeMo Guardrails détecte techniquement qu'une question est hors-sujet.
3. Pourquoi AWS Bedrock Guardrails est-il présenté comme plus robuste que NeMo Guardrails en mode autonome ?
4. Quelle est la différence entre un benchmark de modèle et une évaluation applicative personnalisée ?
5. Décrivez le calcul de la métrique *faithfulness* à l'aide des « atomic claims ».
6. Pourquoi faut-il introduire des temporisations (cooldowns) lors de l'évaluation avec un LLM-juge ?
7. Citez trois techniques de mémoire à court terme et expliquez leur principal inconvénient commun.
8. À partir de quelle technique la mémoire devient-elle réellement « à long terme » ? Pourquoi ?
9. Quelle est la différence entre la mémoire épisodique et la mémoire sémantique ?
10. Expliquez le concept de demi-vie (half-life) dans la stratégie d'oubli et de décroissance de la mémoire.
11. Pourquoi la conception d'une architecture pour 10 000 utilisateurs concurrents dépend-elle autant des services tiers que de Kubernetes lui-même ?
12. Quels sont les trois piliers qui, selon le formateur, distinguent un projet « production-grade » d'un projet à échelle moyenne ?

---

*Fin du cours. Je peux approfondir n'importe quel module (par exemple : implémentation détaillée de Ragas, architecture LangGraph complète, ou configuration précise de l'autoscaling Kubernetes) si tu le souhaites.*
