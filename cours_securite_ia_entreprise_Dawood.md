# Sécuriser l'IA en entreprise : de l'« AI Security » à la « Security for AI »

> Cours synthétisé à partir d'un épisode du podcast *Ask Developer*, avec pour invité Tarek Dawood, Product Manager chez Microsoft (Identity & Sécurité). Le contenu original était un échange oral en arabe égyptien ; il a été restructuré, reformulé et organisé ici en parcours pédagogique.

---

## Objectifs pédagogiques

À la fin de ce cours, vous serez capable de :

1. Distinguer clairement **« Security for AI »** (sécuriser un système d'IA que l'on construit) et **« AI Security » / sécurisation de l'usage de l'IA en entreprise** (encadrer la manière dont l'IA est utilisée une fois déployée).
2. Identifier les grandes familles de vulnérabilités propres à l'IA générative et aux agents.
3. Comprendre pourquoi les agents IA représentent une amplification du risque « insider » classique.
4. Connaître les contrôles techniques concrets (MFA, device compliance, labellisation, DLP, contrôle réseau, traçabilité des agents) utilisés pour encadrer l'IA en entreprise.
5. Distinguer l'IA responsable (biais, équité) de la sécurité de l'IA.
6. Appliquer quelques principes pratiques quand vous concevez une solution à base d'IA agentique.

---

## Table des matières

1. Introduction : pourquoi ce sujet est confus
2. Module 1 — Sécuriser la construction de l'IA (la couche technique)
3. Module 2 — Sécuriser l'usage direct de l'IA (chatbots, Copilot…)
4. Module 3 — L'IA agentique : le « collègue numérique »
5. Module 4 — Les contrôles de sécurité appliqués aux agents
6. Module 5 — IA responsable (biais et équité)
7. Module 6 — Conformité réglementaire
8. Module 7 — ROI, automatisation de l'emploi et prise de recul
9. Module 8 — Bonnes pratiques pour les architectes et développeurs
10. Glossaire
11. Auto-évaluation

---

## Introduction : pourquoi ce sujet est confus

Le monde de la cybersécurité souffre d'un problème structurel : à chaque conférence, chaque stand promet de résoudre *la* faille critique qui menace votre entreprise. Le résultat : des entreprises qui accumulent des dizaines d'outils redondants, incompatibles entre eux, sans réduire réellement le risque — et parfois en l'aggravant, car empiler des solutions non coordonnées crée de nouveaux angles morts.

Le même phénomène touche aujourd'hui l'IA. Deux discussions bien différentes sont souvent confondues :

- **Security for AI** : comment je sécurise le système d'IA que je suis en train de construire (le modèle, ses données d'entraînement, ses outils, sa chaîne d'approvisionnement logicielle).
- **La sécurisation de l'usage de l'IA une fois déployée dans l'entreprise** : comment j'encadre les employés — et maintenant les agents — qui utilisent l'IA au quotidien.

Ce cours suit cette distinction, en partant de la couche la plus technique (construire l'IA) vers la couche la plus organisationnelle (gouverner l'IA dans l'entreprise).

---

## Module 1 — Sécuriser la construction de l'IA

Quand une équipe construit une application d'IA (un agent, un chatbot, un copilote métier), elle assemble plusieurs briques : un modèle (souvent pré-entraîné, rarement construit from scratch), des sources de connaissance (RAG), des outils que le modèle peut appeler, et une chaîne d'approvisionnement logicielle (bibliothèques, serveurs MCP, etc.). Chacune de ces briques est un point d'entrée potentiel pour un attaquant.

### 1.1 Prompt injection vs jailbreak — une distinction utile

- **Prompt injection (au sens large)** : faire faire au modèle quelque chose qu'il n'était pas censé faire, en manipulant l'entrée qu'il reçoit (directement dans le chat, ou indirectement via un document, une page web, un email qu'il va lire).
- **Jailbreak** : un cas plus spécifique et plus grave, où l'on pousse le modèle à produire un contenu illégal ou gravement nuisible (violence, contenu sexuel impliquant des mineurs, fabrication d'armes, etc.) — un acte qui, s'il était commis par un humain, l'enverrait potentiellement en prison.

En résumé : tout jailbreak est une forme de prompt injection, mais toutes les prompt injections ne sont pas des jailbreaks.

### 1.2 Les surfaces d'attaque autour d'un agent

Quand on schématise un agent, on retrouve typiquement :

- **L'entrée (input)** : l'utilisateur écrit un prompt. Un attaquant peut y injecter des instructions cachées (texte en petite taille, texte inversé, instructions dissimulées dans des balises invisibles).
- **Les outils (tools)** : chaque outil que l'on connecte à l'agent (recherche web, exécution de code, accès à un CRM…) introduit ses propres vulnérabilités. Utiliser un outil, c'est hériter de ses failles.
- **La chaîne de connaissance (RAG / data sources)** : si l'on connecte l'agent à une documentation ou à un site web, et que ce site contient des instructions malveillantes cachées, l'agent peut les exécuter sans que l'utilisateur ait rien demandé de mal — c'est de l'injection indirecte.
- **La chaîne d'approvisionnement logicielle (supply chain)** : les serveurs MCP (Model Context Protocol) et les bibliothèques tierces (souvent open source) sont l'équivalent moderne des anciennes dépendances logicielles compromises. Un serveur MCP mal sécurisé peut aspirer toutes les données qui transitent par lui.
- **La sortie (output)** : l'agent peut malencontreusement révéler des secrets internes (clés, informations confidentielles) dans sa réponse.

### 1.3 Tester ces vulnérabilités

Des outils de *red teaming automatisé* (par exemple les fonctionnalités de scan disponibles dans les plateformes comme Azure AI Foundry) permettent de simuler des attaques après la conception d'un agent : injection de prompts, tentatives d'extraction de données, tests de résistance. Le résultat aide à savoir quelles attaques réussissent et lesquelles échouent, avant la mise en production.

**Pour aller plus loin** : consultez les référentiels spécialisés — l'OWASP Top 10 pour le Machine Learning, le référentiel MITRE ATLAS (l'équivalent de MITRE ATT&CK pour l'IA), et les travaux de centres de recherche dédiés à la sécurité de l'IA.

---

## Module 2 — Sécuriser l'usage direct de l'IA

Une fois l'IA construite et déployée (par exemple un assistant conversationnel interne connecté aux emails et documents de l'entreprise), un nouveau risque apparaît, indépendant des failles techniques du modèle : **l'usage** qu'en font les utilisateurs.

On distingue deux scénarios :

### 2.1 Le scénario « innocent »

Un utilisateur pose une question tout à fait légitime — « résume-moi les trois dernières réunions sur ce sujet » — et l'assistant, parce qu'il est très serviable et qu'il a accès à un large périmètre de documents, révèle par inadvertance une information que l'utilisateur n'aurait pas dû voir (par exemple la participation d'une personne à un projet confidentiel). Ici, il n'y a **aucune intention malveillante** : le problème vient d'un système qui a accès à trop de données par rapport à ce que l'utilisateur est autorisé à voir (excès de « sur-partage »/over-sharing), pas d'une attaque.

### 2.2 Le scénario malveillant (ou « shadow AI »)

Un employé, volontairement ou par simple commodité, contourne les règles :

- il envoie des documents internes vers un outil d'IA grand public non approuvé par l'entreprise (ChatGPT personnel, un modèle étranger moins cher…) ;
- il exfiltre des informations sensibles en les faisant passer par un canal externe sous couvert d'une demande anodine à l'IA.

Ce comportement soulève des enjeux :
- **De confidentialité** : on ne sait pas ce que le fournisseur tiers fait des données envoyées.
- **De conformité réglementaire** : certaines lois (notamment américaines, dans un contexte de contrôle des exportations technologiques) interdisent formellement à des entreprises travaillant avec certains clients (gouvernementaux, par exemple) d'envoyer des données vers des modèles hébergés dans certains pays.
- **De coût caché** : utiliser un service moins cher hors des canaux approuvés peut sembler économique à court terme, mais expose l'entreprise à un risque de non-conformité bien plus coûteux.

### 2.3 Une bonne pratique : ne pas simplement bloquer, mais canaliser

Bloquer purement et simplement l'accès aux outils d'IA pousse les employés à les utiliser depuis leur téléphone personnel, en dehors de tout contrôle. L'approche recommandée consiste plutôt à :
- autoriser les outils jugés sûrs (en fonction de critères objectifs : conformité, chiffrement, résidence des données…) ;
- bloquer sélectivement les outils non conformes au niveau réseau ;
- **empêcher les actions à risque plutôt que l'outil dans son ensemble** — par exemple, autoriser la conversation avec un assistant IA externe, mais bloquer l'upload d'un document classé « confidentiel » vers cet outil, grâce à des règles de labellisation et de prévention de perte de données (DLP).

---

## Module 3 — L'IA agentique : le « collègue numérique »

### 3.1 Le changement de paradigme

Une phrase attribuée au PDG de Microsoft résume l'évolution en cours : à terme, tout logiciel en mode SaaS deviendra un flux de travail agentique (« *agentic workflow* »). Concrètement : les outils métiers historiquement séparés (gestion des tickets IT, CRM commercial…) convergent tous vers un même modèle — un agent qui exécute des tâches, suit leur progression, et interagit avec les humains comme le ferait un collègue.

### 3.2 L'agent comme employé virtuel

La bonne manière de se représenter un agent d'entreprise moderne n'est pas « un chatbot qui répond et disparaît », mais un **collègue numérique** :
- il a une boîte mail, un compte Teams/Slack ;
- on lui assigne des tâches, il les exécute pendant que vous faites autre chose, puis revient avec un résultat ;
- il continue de travailler après que vous avez quitté la conversation.

Exemple concret : au lieu d'embaucher des employés temporaires pour rédiger des emails commerciaux personnalisés à partir des données du CRM, une entreprise peut désormais « embaucher » un agent, facturé à l'heure ou à l'usage, avec des exigences de disponibilité et de qualité (SLA) comparables à un prestataire humain.

### 3.3 Le risque insider amplifié

Le risque dit « insider » (une personne interne malveillante, un espion, un employé mécontent) existe depuis toujours. L'IA agentique l'amplifie fortement, car un agent :
- peut parcourir et copier des volumes de documents bien plus vite qu'un humain ;
- peut exécuter des actions destructrices (suppression de données de production, désactivation de systèmes) en quelques secondes ;
- n'a — c'est le point le plus important — **aucun bon sens (« common sense »)**. Un employé humain hésite instinctivement devant une demande suspecte, par réflexe de prudence ou par crainte des conséquences. Un agent, si on le lui demande poliment et sans qu'un garde-fou technique l'en empêche, obéira.

### 3.4 Les trois façons de créer un agent (illustration Microsoft)

1. Une équipe technique construit un agent sur mesure avec une plateforme de développement (ex. AI Foundry).
2. Une équipe métier utilise une plateforme low-code (ex. Copilot Studio).
3. Un simple utilisateur crée un agent « citoyen développeur », sans aucune compétence en programmation, directement depuis un outil de messagerie d'entreprise (ex. Teams), en lui donnant juste des instructions et éventuellement un accès à une base de connaissances.

Cette démocratisation de la création d'agents est une bonne nouvelle pour la productivité, mais elle multiplie le nombre de points d'entrée à surveiller.

---

## Module 4 — Les contrôles de sécurité appliqués aux agents

Comment sécurise-t-on un « collègue » qui n'a ni empreinte digitale, ni téléphone, ni bon sens ?

### 4.1 Authentification : les trois facteurs classiques

On distingue traditionnellement trois catégories de facteurs d'authentification :
- **Ce que vous savez** (mot de passe, code) ;
- **Ce que vous possédez** (téléphone, jeton matériel) ;
- **Ce que vous êtes** (empreinte digitale, visage, voix).

Un seul facteur, même très robuste, reste **plus faible qu'une combinaison de deux facteurs différents**, car un attaquant qui compromet une seule catégorie (par exemple en volant un mot de passe) n'a pas nécessairement accès aux autres. D'où l'importance systématique du MFA (authentification multi-facteurs) pour les humains.

**Problème** : un agent IA n'a ni téléphone, ni visage, ni mot de passe au sens humain. Les mécanismes classiques de MFA ne s'appliquent pas directement — d'où la nécessité de nouveaux contrôles, pensés spécifiquement pour les identités non humaines.

### 4.2 Conformité des appareils (device compliance)

Pour un humain, l'accès aux emails de l'entreprise est souvent conditionné à l'utilisation d'un appareil « inscrit » (enrôlé dans une solution de gestion des appareils mobiles, à jour, chiffré). Pour un agent, l'équivalent est : sur quelle infrastructure tourne-t-il ? Un agent qui s'exécute hors du réseau de confiance de l'entreprise pose un problème analogue à un appareil non conforme — sauf qu'aujourd'hui, le seul contrôle réellement disponible est souvent binaire : autoriser ou bloquer complètement la communication entre agents.

### 4.3 Contrôle réseau

Au niveau réseau, des solutions de type *Global Secure Access* (l'équivalent des solutions Zscaler ou Netskope) permettent de :
- bloquer l'accès à certains services d'IA non conformes (juridiquement ou en matière de sécurité) directement au niveau du navigateur ou du réseau de l'entreprise ;
- autoriser d'autres services jugés sûrs, tout en surveillant les flux (par exemple, empêcher qu'un document marqué comme sensible ne parte vers un service externe autorisé, même si l'outil lui-même n'est pas bloqué).

### 4.4 Labellisation automatique et prévention de perte de données (DLP)

Un des piliers de la sécurisation des données à l'ère de l'IA : la **labellisation de sensibilité** (sensitivity labels), idéalement automatique. Exemple concret :
- un site SharePoint est marqué comme « sensible » par défaut ;
- lorsqu'un utilisateur essaie d'en extraire un document via l'IA, le système bloque l'action en se basant uniquement sur ce label — sans même avoir besoin de relire le contenu du document.
- Un assistant IA peut lui-même analyser un document généré, détecter qu'il contient des informations sensibles (par exemple une évaluation de performance d'un collaborateur), et proposer de le marquer automatiquement comme confidentiel — la décision finale restant à l'utilisateur, mais avec un filet de sécurité qui réduit l'erreur humaine.

**Point clé** : la sécurité de la donnée doit se faire *le plus en amont possible*, avant même que l'agent ne la lise, car une fois qu'une information a fuité, on ne peut pas revenir en arrière — même si l'incident est détecté après coup.

### 4.5 Traçabilité et gouvernance des identités d'agents

Trois besoins de gouvernance émergent avec la multiplication des agents :

1. **Inventaire** : savoir qu'un agent a été créé, par qui, et où il tourne dans l'entreprise (registre centralisé des agents).
2. **Identité et permissions** : associer à chaque agent une identité propre, visible dans les outils de gestion des identités (par exemple Microsoft Entra), avec un historique de ses accès et de ses actions — de la même manière qu'on suit un employé.
3. **Réponse au comportement suspect** : si un agent affiche un schéma d'usage inhabituel (tentative répétée de faire déclasser des documents sensibles, requêtes en rafale sur des données confidentielles), le système peut restreindre dynamiquement son accès — comme on le ferait pour le compte d'un employé dont le comportement semble anormal — sans nécessairement bloquer tout le reste de son activité.

### 4.6 Gestion des risques internes (Insider Risk Management) appliquée aux agents

Les outils de gestion du risque interne, historiquement conçus pour détecter des comportements suspects chez les employés (téléchargements massifs, accès inhabituels), s'étendent désormais aux agents IA : un agent qui accumule des signaux de comportement à risque peut voir son accès restreint automatiquement, en attendant une vérification humaine.

---

## Module 5 — IA responsable (Responsible AI)

Il s'agit d'un pilier distinct de la sécurité technique : garantir que l'IA ne reproduit pas ou n'amplifie pas des biais discriminatoires (liés au genre, à l'origine, etc.).

**Exemple illustratif** : un modèle entraîné pour aider à décider de licenciements ou d'embauches peut apprendre, à partir de l'historique des données (rythme de promotion, ancienneté, formulations dans les CV, code postal…), des critères qui reflètent des biais sociétaux préexistants — sans que cela soit une intention du système, mais parce que les données d'entraînement elles-mêmes contiennent ces biais historiques. Un modèle mal calibré peut ainsi **reproduire et systématiser** une discrimination qui, chez des décideurs humains isolés, restait au moins variable ou corrigible au cas par cas.

La bonne pratique consiste à définir des critères de décision beaucoup plus profonds et explicitement auditée pour éviter que l'IA n'« apprenne » et n'automatise ces biais à grande échelle.

---

## Module 6 — Conformité réglementaire

Un quatrième axe de risque, distinct des trois précédents (sécurité de la construction, sécurité de l'usage, gouvernance des agents) : la **conformité légale**, un paysage en pleine évolution :

- Le règlement européen sur l'IA (IA Act) encadre déjà largement les usages à risque.
- Certains États américains commencent à légiférer spécifiquement sur l'usage de l'IA dans les décisions de recrutement ou de licenciement.
- Cette réglementation est aujourd'hui fragmentée : elle diffère d'un pays à l'autre, et parfois d'un État à l'autre au sein d'un même pays — un vrai défi pour toute entreprise multinationale qui doit adapter un même produit d'IA aux exigences différentes de chaque juridiction.

---

## Module 7 — ROI, automatisation de l'emploi et prise de recul

Cette section, plus prospective, replace le sujet dans un débat plus large :

- Une partie du débat public sur l'IA repose sur une vision anthropomorphisée de l'IA générale (AGI) — l'idée qu'une IA « consciente » menacerait directement l'humanité. Ce cadrage, bien que répandu chez certains dirigeants technologiques, n'est pas partagé par tous les acteurs du secteur, qui préfèrent une approche plus mesurée : l'IA comme accélérateur progressif de productivité, dont l'impact réel se mesurera sur plusieurs années à travers des indicateurs macroéconomiques (croissance du PIB liée à l'IA, par exemple), plutôt que par un scénario de rupture brutale.
- Sur le plan business : une large majorité des entreprises ayant lancé des projets pilotes d'IA générative n'ont pour l'instant pas mesuré de retour sur investissement direct et chiffré. Cela ne signifie pas que l'IA est inefficace — de nombreux retours individuels rapportent un vrai gain de productivité personnelle — mais que la transformation à l'échelle de l'organisation prend du temps à se traduire en résultats mesurables.

---

## Module 8 — Bonnes pratiques pour les architectes et développeurs

Deux principes pratiques ressortent particulièrement pour quiconque conçoit une solution d'IA agentique en entreprise :

### 8.1 Ne pas se contenter de reproduire un workflow existant

Avant d'automatiser un processus avec de l'IA, il faut se demander comment ce processus **devrait** fonctionner si on le repensait depuis le début — et non simplement automatiser chaque étape humaine telle quelle. Exemple : un processus qui passe aujourd'hui par une extraction manuelle vers un tableur avant réinjection en base de données peut souvent être simplifié radicalement (lecture directe et injection automatique), sans qu'un humain n'ait besoin d'intervenir « au milieu » simplement parce que c'était ainsi auparavant.

### 8.2 Ne pas ajouter de l'IA « pour faire de l'IA »

Sous la pression des effets de mode et des systèmes de récompense internes valorisant l'adoption de l'IA à tout prix, de nombreuses équipes construisent des agents ou des automatisations qui n'apportent aucune valeur réelle — juste pour « cocher la case IA ». Le bon réflexe est de partir d'un besoin réel : d'abord clarifier ce dont on a besoin (par exemple, une synthèse d'informations que l'on peut fournir directement), plutôt que d'imposer un nouveau processus rigide simplement parce qu'un agent existe.

### 8.3 Outils et ressources recommandées

- Un livre pratique orienté ingénierie (plutôt que théorie mathématique) sur la construction de systèmes à base de modèles de langage, souvent cité en référence dans la communauté (« AI Engineering », éd. O'Reilly).
- Des plateformes low-code d'automatisation permettant de construire des chaînes d'agents (recherche → décision → action) sans tout coder à la main.
- Des outils de gestion de la posture de sécurité, avec deux approches complémentaires : une approche « agent-based » installée à l'intérieur d'une machine ou d'un environnement pour une visibilité en temps réel (comme un endpoint agent classique), et une approche « agentless », qui analyse la configuration et les journaux depuis l'extérieur, plus légère mais avec un temps de détection potentiellement plus long en cas de compromission active.

---

## Glossaire

| Terme | Définition courte |
|---|---|
| Prompt injection | Manipuler l'entrée d'un modèle (directe ou indirecte) pour lui faire produire un comportement non prévu. |
| Jailbreak | Prompt injection visant spécifiquement à faire produire un contenu illégal ou gravement nuisible. |
| RAG (Retrieval-Augmented Generation) | Technique consistant à connecter un modèle à des sources de documents externes pour enrichir ses réponses. |
| MCP (Model Context Protocol) | Protocole standard permettant à un agent de se connecter à des outils ou sources de données externes. |
| Shadow AI | Usage d'outils d'IA non approuvés par l'entreprise, en dehors des canaux officiels. |
| Sensitivity label | Étiquette de classification (public, interne, confidentiel…) apposée sur un document ou un site. |
| DLP (Data Loss Prevention) | Ensemble de règles techniques empêchant la sortie non autorisée de données sensibles. |
| MFA (Multi-Factor Authentication) | Authentification combinant au moins deux facteurs parmi : savoir, possession, caractéristique physique. |
| Insider risk | Risque provenant d'une personne (ou, désormais, d'un agent) ayant un accès légitime mais un comportement à risque ou malveillant. |
| Responsible AI | Ensemble des pratiques visant à éviter les biais et discriminations dans les décisions assistées par IA. |
| Agent citoyen développeur | Agent créé par un utilisateur métier sans compétence en programmation, via un outil low-code/no-code. |

---

## Auto-évaluation

1. Expliquez en une phrase la différence entre prompt injection et jailbreak.
2. Citez deux surfaces d'attaque distinctes autour d'un agent construit avec du RAG et des outils externes.
3. Pourquoi bloquer entièrement un outil d'IA externe est-il souvent une moins bonne stratégie que de labelliser et contrôler les données qui peuvent y transiter ?
4. En quoi un agent IA amplifie-t-il le risque insider par rapport à un employé humain malveillant ?
5. Pourquoi les mécanismes classiques de MFA ne s'appliquent-ils pas directement à un agent IA ?
6. Donnez un exemple concret de labellisation automatique de document évoqué dans ce cours.
7. Quelle est la différence entre Responsible AI et Security for AI ?
8. Citez un principe à respecter avant d'automatiser un processus métier existant avec de l'IA.

---

*Fin du cours. N'hésitez pas à demander un approfondissement sur un module particulier (par exemple : mise en pratique du contrôle réseau, exemples de règles DLP, ou détail du fonctionnement de MITRE ATLAS).*
