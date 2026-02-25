<h1 id="planMaj">Plan de mise à jour</h1>

• Titre du document : Plan de mise à jour | `projet7_orion`  
• Auteur : Pierre Cistac   
• Option choisie : Option B (Scénario Orion)   
• Date : 27/02/2026

## Mise à jour de l'application

**Dépendances Maven / npm**

- Back-end (Gradle) : utiliser la commande suivante pour identifier les dépendances obsolètes :
  ```bash
  ./gradlew dependencyUpdates
  ```
  Mettre à jour les versions dans `build.gradle`, valider avec `./gradlew build --no-daemon` et vérifier que la Quality Gate SonarCloud reste verte.

- Front-end (npm) : auditer et mettre à jour les dépendances Angular et Jest/Karma :
  ```bash
  npm outdated          # lister les packages à jour
  npm audit             # détecter les vulnérabilités connues
  npm audit fix         # appliquer les correctifs non-breaking
  ```
  Committer les changements de `package-lock.json` avec un message de type `chore(deps): update npm dependencies`.

**Mises à jour Angular / Spring Boot**

- Suivre les cycles de release LTS : Angular (tous les 6 mois, support étendu 18 mois) et Spring Boot (tous les 6 mois, support 12 mois).
- Pour Angular, utiliser le guide de migration officiel via `ng update` :
  ```bash
  ng update @angular/core @angular/cli
  ```
- Pour Spring Boot, mettre à jour la version dans `build.gradle` et valider l'absence de breaking changes via les notes de release.
- Adapter la version JDK dans le workflow si nécessaire (`vars.JAVA_VERSION`).

**Mises à jour Docker (images)**

- Mettre à jour les images de base dans les Dockerfiles à chaque nouvelle version LTS disponible :
  - `gradle:8-jdk21-alpine` → vérifier les nouvelles versions Gradle
  - `eclipse-temurin:21-jre-alpine` → suivre les releases Eclipse Temurin
  - `node:20-alpine` → préférer le tag LTS (`node:lts-alpine`)
  - `nginx:stable-alpine` → utiliser le tag `stable` pour la prévisibilité
- Après chaque mise à jour d'image de base, le scan Trivy du Job 4 validera l'absence de nouvelles vulnérabilités `CRITICAL`.

---

## Mise à jour du pipeline CI/CD

**Versions des actions GitHub**

Les actions GitHub sont épinglées à des versions majeures (`@v4`, `@v3`, etc.). Les maintenir à jour réduit les risques de sécurité (supply chain) et de dépréciation :

| Action | Version actuelle | Vérification |
|---|---|---|
| `actions/checkout` | v4 | Stable, aucune action requise |
| `actions/setup-node` | v4 | Stable |
| `actions/setup-java` | v4 | Stable |
| `actions/cache` | v4 | Stable |
| `actions/upload-artifact` | v4 | Stable |
| `actions/download-artifact` | v4 | Stable |
| `docker/build-push-action` | v6 | Vérifier les releases |
| `docker/login-action` | v3 | Vérifier les releases |
| `aquasecurity/trivy-action` | 0.33.1 | Épinglé à une version mineure, à mettre à jour régulièrement |
| `SonarSource/sonarqube-scan-action` | v6 | Suivre les releases SonarSource |
| `cycjimmy/semantic-release-action` | v4 | Vérifier la compatibilité avec semantic-release |
| `softprops/action-gh-release` | v2 | Vérifier les releases |

Mettre à jour les versions en ouvrant une PR dédiée sur `development` pour valider le pipeline avant de merger.

**Versions des scripts**

- `semantic-release` : épinglé à `^22.0.0` — surveiller les releases majeures sur [npmjs.com/package/semantic-release](https://www.npmjs.com/package/semantic-release)
- `@commitlint/cli` et `@commitlint/config-conventional` : mettre à jour conjointement pour garantir la cohérence
- `@semantic-release/changelog` et `@semantic-release/github` : mettre à jour avec `semantic-release`

**Maintenance du workflow**

- Documenter toute modification du fichier `ci.yml` avec un message de commit de type `ci: <description>` pour que semantic-release ne génère pas de nouvelle version de l'application.
- Tester les modifications du workflow sur une branche `feature/ci-*` avant de merger sur `development`.
- Revoir annuellement les permissions accordées aux jobs (`contents: write`, `packages: write`, etc.) et les restreindre au minimum nécessaire.

---

## Fréquence & bonnes pratiques

**Fréquence recommandée**

- **Hebdomadaire** : vérifier les alertes de sécurité GitHub Dependabot et les résultats du dashboard SonarCloud
- **Mensuelle** : auditer les dépendances npm (`npm audit`) et Gradle, vérifier les nouvelles versions des actions GitHub utilisées
- **Trimestrielle** : mettre à jour les images Docker de base, revoir les seuils de sévérité Trivy (`vars.SEVERITY`), vérifier l'expiration des tokens SonarCloud
- **Semestrielle** : évaluer les migrations de versions majeures (Angular LTS, Spring Boot, JDK), revoir la stratégie de branches et la politique de rétention des artefacts

**Bonnes pratiques**

- Activer **GitHub Dependabot** pour les alertes de sécurité automatiques sur `package.json` et `build.gradle`
- Ne jamais modifier directement `main` : toute mise à jour passe par une PR validée sur `development` d'abord
- Utiliser des messages de commit Conventional Commits pour toutes les mises à jour (`chore(deps):`, `ci:`, `fix:`) afin de garantir un changelog propre
- Documenter les décisions de mise à jour différée (ex. dépendance avec breaking change) dans un fichier `DECISIONS.md` ou dans les issues GitHub
- Conserver une trace des versions testées et des éventuelles incompatibilités rencontrées lors des migrations
- Vérifier que la Quality Gate SonarCloud reste verte après chaque mise à jour significative avant de merger sur `main`