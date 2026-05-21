# Documentation CI/CD — BobApp

## 1. Architecture du workflow

```
on: push / pull_request
         │
    ┌────┴────┐
    │         │
test-backend  test-frontend    ← s'exécutent en parallèle
(JaCoCo)     (Karma/lcov)
    │         │
    └────┬────┘
         │
  sonarcloud-analysis          ← attend les deux rapports de couverture
         │
         │ (uniquement sur push vers main)
         ▼
   build-and-push              ← déploiement sur Docker Hub
```

Le déploiement sur Docker Hub n'est déclenché que si toutes les étapes précédentes ont réussi,
et uniquement lors d'un merge vers la branche `main`.

---

## 2. Description des étapes

### Job 1 — `test-backend`
**Déclencheur :** push sur `dev` ou `main`, pull request vers `main`

Lance les tests unitaires du back-end Spring Boot et génère le rapport de couverture JaCoCo.

Étapes :
- Checkout du code source
- Installation de Java 11 (Eclipse Temurin)
- Exécution de `mvn verify` — compile, teste et génère le rapport JaCoCo dans `back/target/site/jacoco/`
- Upload du rapport JaCoCo comme artefact GitHub Actions

---

### Job 2 — `test-frontend`
**Déclencheur :** push sur `dev` ou `main`, pull request vers `main`

Lance les tests unitaires du front-end Angular et génère le rapport de couverture Istanbul au format lcov.

Étapes :
- Checkout du code source
- Installation de Node.js 18
- Installation des dépendances via `npm ci`
- Exécution des tests Karma en mode headless (sans navigateur graphique) avec génération de la couverture
- Upload du rapport lcov comme artefact GitHub Actions

---

### Job 3 — `sonarcloud-analysis`
**Déclencheur :** après la réussite de `test-backend` ET `test-frontend`

Analyse la qualité du code back et front via SonarCloud. Vérifie les bugs, code smells,
vulnérabilités, duplications et couverture de code.

Étapes :
- Checkout complet de l'historique Git (nécessaire pour SonarCloud)
- Téléchargement des rapports de couverture JaCoCo et lcov depuis les artefacts
- Compilation du back-end pour fournir les classes Java à SonarCloud
- Envoi du code et des métriques à SonarCloud via `SonarSource/sonarcloud-github-action`

---

### Job 4 — `build-and-push`
**Déclencheur :** après la réussite de `sonarcloud-analysis`, uniquement sur `main`

Construit les images Docker du back-end et du front-end et les publie sur Docker Hub.

Étapes :
- Checkout du code source
- Authentification sur Docker Hub
- Build et push de l'image back-end : `<user>/bobapp-back:latest`
- Build et push de l'image front-end : `<user>/bobapp-front:latest`

---

## 3. KPIs proposés

Ces seuils constituent la Quality Gate du projet. Une PR ne peut être mergée que si tous ces
seuils sont respectés.

| KPI | Seuil minimum | Justification |
|---|---|---|
| **Couverture de code (Coverage)** | ≥ 30 % | Seuil de départ réaliste compte tenu de la faible couverture initiale. Objectif à 6 mois : 80 %. |
| **New Blocker Issues** | 0 | Aucun bug bloquant ne doit entrer en production. |
| **Duplicated Lines (%)** | < 3 % | Limite la dette technique liée au copier-coller. |
| **Security Hotspots Reviewed** | 100 % | Toute vulnérabilité potentielle doit être examinée avant merge. |

---

## 4. Analyse des métriques

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

La couverture globale dépasse le seuil minimum fixé à 30 %. Elle reste perfectible,
notamment côté back-end où seul un test d'intégration (`contextLoads`) existe.

### Issues SonarCloud (7 au total)

| Sévérité | Type | Fichier | Description |
|---|---|---|---|
| **Critique** | Bug | `JokeService.java` L22 | Un objet `Random` est recréé à chaque appel au lieu d'être réutilisé. Cela dégrade la distribution aléatoire et les performances. |
| **Critique** | Code Smell | `BobappApplicationTests.java` L10 | Méthode de test vide sans commentaire explicatif. |
| **Majeur** | Code Smell | `Joke.java` L4 | Le champ `joke` porte le même nom que sa classe — source de confusion. |
| **Mineur** | Code Smell | `Joke.java` L4 | Le champ `joke` devrait être privé avec un accesseur (getter/setter). |
| **Mineur** | Code Smell | `Joke.java` L5 | Le champ `response` devrait être privé avec un accesseur (getter/setter). |
| **Mineur** | Code Smell | `JsonReader.java` L29 | L'ordre des modificateurs ne respecte pas la Java Language Specification. |
| **Info** | Code Smell | `JsonReader.java` L16 | Utilisation du pattern Singleton — à vérifier si c'est intentionnel. |

### Duplications

0,0 % — aucun copier-coller détecté dans le code.

### Ordre de priorité des corrections

1. **Bug `Random` dans `JokeService`** — impact direct sur la fiabilité des fonctionnalités
2. **Champs publics dans `Joke.java`** — violation des principes d'encapsulation orientée objet
3. **Méthode de test vide** — améliorer la couverture et la lisibilité des tests
4. **Conventions de nommage et modificateurs** — dette technique mineure

---

## 5. Analyse des retours utilisateurs

Les avis récents des utilisateurs de BobApp font état de plusieurs problèmes fonctionnels
récurrents qui dégradent fortement l'expérience.

### Retours collectés

> *"Je mets une étoile car je ne peux pas en mettre zéro ! Impossible de poster une suggestion
> de blague, le bouton tourne et fait planter mon navigateur !"*

> *"#BobApp j'ai remonté un bug sur le post de vidéo il y a deux semaines et il est encore
> présent ! Les devs vous faites quoi ????"*

> *"Ça fait une semaine que je ne reçois plus rien, j'ai envoyé un email il y a 5 jours mais
> toujours pas de nouvelles..."*

> *"J'ai supprimé ce site de mes favoris ce matin, dommage, vraiment dommage"*

### Problèmes identifiés

| Priorité | Problème | Impact |
|---|---|---|
| 🔴 P1 | **Bug bloquant sur la soumission de suggestion de blague** — le bouton ne répond plus et plante le navigateur | Fonctionnalité principale inutilisable |
| 🔴 P1 | **Bug non corrigé sur le post de vidéo** — signalé depuis plus de 2 semaines sans résolution | Perte de confiance des utilisateurs |
| 🟠 P2 | **Absence de contenu reçu depuis une semaine** — peut indiquer un problème de backend ou de notifications | Désengagement silencieux |
| 🟠 P2 | **Absence de réponse du support** — email sans réponse depuis 5 jours | Relation utilisateur dégradée |

### Conclusion

Les avis révèlent deux catégories de problèmes distincts :

**Technique :** des bugs fonctionnels graves (soumission de blague, post de vidéo) qui bloquent
les utilisateurs dans leurs actions principales. Ces corrections doivent être traitées en urgence.

**Organisationnel :** un manque de réactivité du support et de communication autour des bugs
connus. La mise en place de la CI/CD permettra d'accélérer les corrections et de regagner la
confiance des utilisateurs en livrant des correctifs plus rapidement et de façon plus fiable.
