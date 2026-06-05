# Documentation CI/CD — BobApp

## 1. Architecture du workflow

```
on: push (dev) / pull_request (main)
         │
    ┌────┴────┐
    │         │
test-backend  test-frontend    ← s'exécutent en parallèle
(JaCoCo)     (Karma/lcov)
    │         │
    └────┬────┘
         │
  sonarcloud-analysis          ← Quality Gate toujours active
         │
    ┌────┴────┐
    │         │
build-backend  build-frontend  ← vérification des builds Docker (sans push)
    │         │
    └────┬────┘
         │ (uniquement sur merge d'une PR vers main)
         ▼
    push-docker                ← publication sur Docker Hub
```

Les images Docker ne sont publiées que si toutes les étapes précédentes ont réussi,
et uniquement lors du merge d'une Pull Request validée vers `main`.

---

## 2. Outils utilisés

| Outil | Rôle dans le pipeline |
|---|---|
| **GitHub Actions** | Orchestrateur CI/CD hébergé par GitHub. Exécute automatiquement le pipeline à chaque push sur `dev` et à chaque événement de Pull Request ciblant `main`. |
| **SonarCloud** | Plateforme d'analyse statique du code. Détecte les bugs, code smells, vulnérabilités et calcule la couverture globale (back + front). La Quality Gate bloque la PR si les seuils ne sont pas atteints. |
| **JaCoCo** | Plugin Maven qui instrumente le bytecode Java pendant l'exécution des tests et génère un rapport de couverture XML (`jacoco.xml`) transmis à SonarCloud. |
| **Istanbul / Karma** | Istanbul mesure la couverture du code TypeScript/Angular (statements, branches, functions, lines). Karma est le test runner qui exécute les specs Jasmine dans Chrome Headless en CI. Le rapport est produit au format lcov et transmis à SonarCloud. |
| **Docker** | Outil de conteneurisation. Utilisé en deux temps : d'abord pour vérifier que les Dockerfiles sont valides (build sans push), puis pour construire et publier les images finales sur Docker Hub après validation complète. |
| **Docker Hub** | Registre public où sont publiées les images `bobapp-back:latest` et `bobapp-front:latest` après un merge validé vers `main`. |

---

## 3. Description des étapes

### Job 1 — `test-backend`
**Déclencheur :** push sur `dev` ou événement de PR vers `main` (hors fermeture sans merge)

Lance les tests unitaires du back-end Spring Boot et génère le rapport de couverture JaCoCo.

Étapes :
- Checkout du code source
- Installation de Java 11 (Eclipse Temurin)
- Exécution de `mvn verify` — compile, teste et génère le rapport JaCoCo dans `back/target/site/jacoco/`
- Upload du rapport JaCoCo comme artefact GitHub Actions

---

### Job 2 — `test-frontend`
**Déclencheur :** push sur `dev` ou événement de PR vers `main` (hors fermeture sans merge)

Lance les tests unitaires du front-end Angular et génère le rapport de couverture Istanbul au format lcov.

Étapes :
- Checkout du code source
- Installation de Node.js 18
- Installation des dépendances via `npm ci`
- Exécution des tests Karma en mode headless avec génération de la couverture
- Upload du rapport lcov comme artefact GitHub Actions

---

### Job 3 — `sonarcloud-analysis`
**Déclencheur :** après la réussite de `test-backend` ET `test-frontend`

Analyse la qualité du code back et front via SonarCloud.

Sur une **Pull Request vers `main`**, la Quality Gate est attendue : si les seuils ne sont pas
atteints, le job échoue et bloque le merge via les status checks obligatoires.

Sur un **push vers `dev`**, l'analyse est lancée mais la Quality Gate n'est pas attendue.
Il s'agit d'une limitation du plan gratuit SonarCloud : l'API de vérification de Quality Gate
n'est pas disponible pour les branches courtes ("short-lived branches"). Les résultats restent
visibles dans l'interface SonarCloud.

Étapes :
- Checkout complet de l'historique Git (nécessaire pour SonarCloud)
- Téléchargement des rapports de couverture JaCoCo et lcov depuis les artefacts
- Compilation du back-end pour fournir les classes Java à SonarCloud
- Envoi du code et des métriques à SonarCloud (avec attente de la Quality Gate sur PR uniquement)

---

### Job 4 — `build-backend`
**Déclencheur :** après la réussite de `sonarcloud-analysis`

Vérifie que le Dockerfile back-end est valide en construisant l'image localement, sans la publier.
Garantit que le build ne peut pas être mergé si le Dockerfile est cassé.

Étapes :
- Checkout du code source
- `docker build ./back -t bobapp-back:verify`

---

### Job 5 — `build-frontend`
**Déclencheur :** après la réussite de `sonarcloud-analysis` (en parallèle avec `build-backend`)

Vérifie que le Dockerfile front-end est valide en construisant l'image localement, sans la publier.

Étapes :
- Checkout du code source
- `docker build ./front -t bobapp-front:verify`

---

### Job 6 — `push-docker`
**Déclencheur :** après la réussite de `build-backend` ET `build-frontend`, uniquement lors du merge effectif d'une PR vers `main`

Construit les images Docker finales et les publie sur Docker Hub.

Étapes :
- Checkout du code source
- Authentification sur Docker Hub
- Build et push de l'image back-end : `<user>/bobapp-back:latest`
- Build et push de l'image front-end : `<user>/bobapp-front:latest`

---

## 4. KPIs proposés

Ces seuils constituent la Quality Gate du projet. Une PR ne peut être mergée que si tous ces
seuils sont respectés.

| KPI | Seuil minimum | Justification |
|---|---|---|
| **Couverture de code (Coverage)** | ≥ 80 % | Seuil professionnel garantissant que la majorité des chemins d'exécution sont couverts par des tests automatisés. |
| **New Blocker Issues** | 0 | Aucun bug bloquant ne doit entrer en production. |
| **Duplicated Lines (%)** | < 3 % | Limite la dette technique liée au copier-coller. |
| **Security Hotspots Reviewed** | 100 % | Toute vulnérabilité potentielle doit être examinée avant merge. |

> **Note :** Le seuil de couverture doit également être mis à jour dans l'interface SonarCloud
> (Quality Gates → condition Coverage) pour être cohérent avec cette documentation.

---

## 5. Analyse des métriques

Métriques obtenues après la première exécution complète du pipeline sur la branche `main`.

### Couverture de code

**Back-end (JaCoCo)**

| Package | Couverture instructions |
|---|---|
| `controller` | 54 % |
| `data` | 49 % |
| `bobapp` (main) | 37 % |
| `service` | 25 % |
| `model` | **0 %** |
| **Global back-end** | **32 %** |

Le package `model` est entièrement non testé. Le `service` ne couvre que 25 % — le bug
`Random` dans `JokeService` provient d'un code jamais vérifié par les tests.

**Front-end (Istanbul/Karma)**

| Métrique | Couverture |
|---|---|
| Statements | 76.92 % |
| Branches | 100 % (aucune branche conditionnelle) |
| Functions | 57.14 % |
| Lines | 83.33 % |

Le front-end présente une meilleure couverture grâce aux tests de composants Angular. Cependant,
57 % des fonctions ne sont pas couvertes, notamment dans `app.component`.

**Couverture globale SonarCloud (back + front) : 38.8 %**

La couverture globale est en dessous du seuil cible de 80 %. L'effort principal porte sur le
back-end, qui ne dispose que d'un seul test d'intégration (`contextLoads`).

### Issues SonarCloud (7 au total)

| Sévérité | Type | Fichier | Description |
|---|---|---|---|
| **Critique** | Bug | `JokeService.java` L22 | Un objet `Random` est recréé à chaque appel au lieu d'être déclaré comme attribut de classe. Cela génère une allocation inutile à chaque requête et dégrade les performances sous charge. |
| **Critique** | Code Smell | `BobappApplicationTests.java` L10 | Méthode de test vide sans commentaire explicatif. |
| **Majeur** | Code Smell | `Joke.java` L4 | Le champ `joke` porte le même nom que sa classe — source de confusion. |
| **Mineur** | Code Smell | `Joke.java` L4 | Le champ `joke` devrait être privé avec un accesseur (getter/setter). |
| **Mineur** | Code Smell | `Joke.java` L5 | Le champ `response` devrait être privé avec un accesseur (getter/setter). |
| **Mineur** | Code Smell | `JsonReader.java` L29 | L'ordre des modificateurs ne respecte pas la Java Language Specification. |
| **Info** | Code Smell | `JsonReader.java` L16 | Utilisation du pattern Singleton — à vérifier si c'est intentionnel. |

### Duplications

0,0 % — aucun copier-coller détecté dans le code.

### Ordre de priorité des corrections

1. **Bug `Random` dans `JokeService`** — impact direct sur les performances
2. **Champs publics dans `Joke.java`** — violation des principes d'encapsulation orientée objet
3. **Méthode de test vide** — améliorer la couverture et la lisibilité des tests
4. **Conventions de nommage et modificateurs** — dette technique mineure

---

## 6. Analyse des retours utilisateurs

Les avis récents des utilisateurs de BobApp font état de plusieurs problèmes fonctionnels
récurrents. Chaque retour est analysé ci-dessous en lien avec le code source.

---

### Utilisateur 1

> *"Je mets une étoile car je ne peux pas en mettre zéro ! Impossible de poster une suggestion
> de blague, le bouton tourne et fait planter mon navigateur !"*

**Analyse du code :**
Le back-end n'expose qu'un seul endpoint : `GET /api/joke` dans `JokeController.java`. Il n'existe
aucun endpoint `POST` permettant de soumettre une suggestion de blague — la fonctionnalité est
entièrement absente du code.

Côté front-end, le bouton "VITE UNE AUTRE 😂" dans `app.component.html` déclenche `getRandomJoke()`
dans `jokes.service.ts`, qui appelle `.subscribe()` sans aucun gestionnaire d'erreur :

```typescript
this.httpClient.get<Joke>(this.pathService).subscribe((joke: Joke) => this.subject.next(joke));
```

Si le back-end retourne une erreur ou est indisponible, l'Observable échoue silencieusement :
aucun message d'erreur n'est affiché, l'interface ne donne aucun retour à l'utilisateur.
De plus, `getRandomJoke()` est appelée deux fois à l'initialisation (constructeur du service +
`ngOnInit` du composant), ce qui génère deux requêtes HTTP simultanées à chaque chargement.

**Cause :** fonctionnalité POST absente du back-end + absence de gestion d'erreur dans le service Angular.

**Solutions :**
- Implémenter l'endpoint `POST /api/joke` dans `JokeController` et `JokeService` si la feature est souhaitée
- Ajouter un opérateur `catchError` dans `jokes.service.ts` pour afficher un message d'erreur visible
- Supprimer le double appel à `getRandomJoke()` à l'initialisation
- Écrire des tests unitaires couvrant le cas d'erreur HTTP

---

### Utilisateur 2

> *"#BobApp j'ai remonté un bug sur le post de vidéo il y a deux semaines et il est encore
> présent ! Les devs vous faites quoi ????"*

**Analyse du code :**
Aucune fonctionnalité vidéo n'existe dans l'application — ni endpoint back-end, ni composant
front-end. Le code source ne contient aucune référence à un upload ou affichage de vidéo.

Ce type de bug non détecté s'explique par le faible taux de couverture du back-end (32 %) :
avec un seul test (`contextLoads` qui vérifie uniquement le démarrage du contexte Spring),
aucune régression fonctionnelle ne peut être détectée automatiquement.

**Cause :** fonctionnalité jamais implémentée ou supprimée, sans détection possible faute de tests.

**Solutions :**
- Déterminer si la feature vidéo est prévue dans le produit : si oui, implémenter les endpoints
  REST et les composants Angular correspondants, couverts par des tests
- Si la feature n'est plus prévue, s'assurer qu'aucune référence à elle n'est visible côté UI
- Augmenter la couverture de tests back-end : avec la CI/CD en place, tout bug sur un endpoint
  testé serait détecté avant le merge

---

### Utilisateur 3

> *"Ça fait une semaine que je ne reçois plus rien, j'ai envoyé un email il y a 5 jours mais
> toujours pas de nouvelles..."*

**Analyse du code :**
Dans `JsonReader.java`, le constructeur charge `jokes.json` via `getJsonFile()` mais n'attrape
les exceptions que pour les afficher en console (`e.printStackTrace()`). Si le fichier est absent
ou corrompu, `jsonFile` reste `null` sans qu'aucune erreur ne remonte.

Lors de l'appel suivant à `getJokes()`, la ligne `this.jsonFile.get("jokes")` déclenche une
`NullPointerException` non rattrapée : le back-end retourne une erreur HTTP 500.

Côté front-end, le `subscribe()` sans gestionnaire d'erreur fait que `subject` reste à `null`,
et la condition `*ngIf="(joke$ | async) as joke"` dans `app.component.html` n'affiche simplement
rien — l'écran est vide, sans aucun message d'erreur pour l'utilisateur.

**Cause :** erreur silencieuse dans `JsonReader` (exception avalée) qui se propage jusqu'au
front-end sous forme d'écran vide.

**Solutions :**
- Dans `JsonReader.java` : remplacer `e.printStackTrace()` par une exception explicite, et
  ajouter une vérification `jsonFile != null` dans `getJokes()` avant d'appeler `.get("jokes")`
- Dans `jokes.service.ts` : ajouter un `catchError` qui met à jour un état d'erreur visible dans l'UI
- Écrire des tests unitaires couvrant le cas de fichier JSON absent ou invalide
- Mettre en place un healthcheck sur l'endpoint back-end pour détecter ce type de panne proactivement

---

### Utilisateur 4

> *"J'ai supprimé ce site de mes favoris ce matin, dommage, vraiment dommage"*

**Analyse :**
Ce retour est la conséquence directe de l'accumulation des trois problèmes précédents non
résolus, aggravée par l'absence de réponse du support (cinq jours sans retour).

Il ne s'agit pas d'un problème technique à corriger dans le code, mais d'un problème humain
et organisationnel : la réactivité du support dépend de Bob, pas du pipeline.

La CI/CD ne résout pas directement le temps de réponse du support. En revanche, en automatisant
les builds, les tests et les déploiements, elle libère du temps développeur que Bob peut
consacrer au traitement des retours utilisateurs et à la correction des bugs signalés.

**Solutions :**
- Mettre en place un canal de support avec un délai de réponse défini
- Communiquer auprès des utilisateurs sur les corrections déployées
- Grâce à la CI/CD, les correctifs peuvent être testés, validés et déployés plus rapidement,
  réduisant le délai entre le signalement d'un bug et sa résolution en production
