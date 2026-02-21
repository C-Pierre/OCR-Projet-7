# Choix techniques

## Mise en place de la CI

### Vue d'ensemble

Le pipeline est défini dans `.github/workflows/ci.yml`.  
Il s'exécute à chaque **push** ou **pull request** sur `main` et `develop`.

---

### Structure du pipeline

```
push / pull_request
        │
        ├── [Job 1] backend   ──── Gradle build + tests + Sonar (back)
        ├── [Job 2] frontend  ──── npm ci + tests + build prod + Sonar (front)
        │
        └── [Job 3] docker-build  (dépend de 1 & 2, branches main/develop uniquement)
                ├── Build image back + scan Trivy
                └── Build image front + scan Trivy
```

---

### Détail des jobs

#### Job 1 — Back-end

| Étape | Outil | Justification |
|---|---|---|
| Checkout (fetch-depth: 0) | `actions/checkout@v4` | Sonar nécessite tout l'historique git pour calculer les métriques de nouveaux bugs |
| Cache Gradle | `actions/cache@v4` | Réduit le temps de build en réutilisant le cache `~/.gradle` |
| JDK 21 | `actions/setup-java@v4` | Correspond à la version utilisée dans le Dockerfile back |
| Build + tests | `./gradlew build --no-daemon` | Compile **et** exécute les tests en une seule commande Gradle |
| Publier résultats JUnit | `actions/upload-artifact@v4` | Permet de récupérer les rapports de tests via l’interface GitHub Actions |
| Sonar back | plugin Gradle `sonar` | Analyse qualité + sécurité du code Java côté SonarCloud |

#### Job 2 — Front-end

| Étape | Outil | Justification |
|---|---|---|
| Checkout | `actions/checkout@v4` | Accès au code source complet |
| Cache npm | `actions/cache@v4` | Accélère `npm ci` en conservant `~/.npm` entre les runs |
| Node 20 | `actions/setup-node@v4` | Version LTS correspondant au Dockerfile front |
| `npm ci` | npm | Installation reproductible et stricte (lockfile) |
| Tests headless | `npm test -- --watch=false` | Exécution non-interactive en environnement CI |
| Publier résultats tests | `actions/upload-artifact@v4` | Permet de récupérer les rapports de couverture |
| Build prod | `npm run build -- --configuration production` | Vérifie que le build de production fonctionne correctement |
| Sonar front | `SonarSource/sonarqube-scan-action@v5` | Analyse qualité + sécurité du code front-end via SonarCloud |

#### Job 3 — Docker Build & Scan

Conditionné à la réussite des deux jobs précédents (`needs: [backend, frontend]`).  
- Utilise le **cache GitHub Actions** (`type=gha`) pour accélérer les builds Docker successifs.  
- Les images ne sont **pas poussées** (`push: false`) — ce job valide uniquement la cohérence des Dockerfiles en CI.  
- **Trivy** est utilisé pour scanner les images Docker back et front sur les vulnérabilités **CRITICAL**.  
- Les images sont taguées avec `${{ github.sha }}` pour correspondre à l’état exact du commit scanné.

---

### Analyse / Qualité du code

#### 1. Choix de l'outil
- **SonarCloud** : analyse statique multi-langages (Java, TypeScript, HTML, CSS)
- Détecte :
  - Bugs
  - Vulnérabilités
  - Code smells
  - Duplication de code
  - Couverture de tests

#### 2. Intégration CI/CD

##### Back-end (Java / Gradle)
- Commande : `./gradlew sonar -Dsonar.coverage.jacoco.xmlReportPaths=build/reports/jacoco/test/jacocoTestReport.xml`
- Avantages :
  - Réutilise la compilation existante
  - Intègre les rapports de tests Jacoco
  - Workflow unifié et reproductible

##### Front-end (Angular / TypeScript)
- Outil : SonarScanner CLI
- Configuration : `projectBaseDir` pointant vers le front-end
- Avantages :
  - Analyse multi-langages front-end
  - Intègre les rapports de tests Jest / Karma
  - Compatible avec GitHub Actions

#### 3. Quality Gate
- **Back-end** : `-Dsonar.qualitygate.wait=true`
- **Front-end** : `sonarqube-quality-gate-action` GitHub
- Fonction :
  - Vérifie que le code respecte les règles de qualité avant de continuer le pipeline (build Docker, release)
- Limitation :
  - Tokens gratuits (personnels) permettent l’analyse mais pas le blocage automatique
  - Tokens organisationnels (payants) nécessaires pour full-blocking

#### 4. Authentification
- **Token** : Personal Access Token (`OCR_TOKEN`)
- Usage :
  - Soumettre les analyses vers SonarCloud
  - Garantir l’accès sécurisé à l’organisation Sonar

#### 5. Rapports et suivi
- Fichier : `.scannerwork/report-task.txt`
- Contenu :
  - Résultat de l’analyse
  - Lien vers le dashboard SonarCloud
- Permet un suivi clair et la consultation des résultats pour chaque branche ou pull request

---

### Secrets GitHub requis

Tous les secrets sont configurés dans **Settings → Secrets and variables → Actions** du dépôt.  
**Aucun secret ne doit apparaître en clair dans le code.**

| Secret | Valeur attendue | Usage |
|---|---|---|
| `SONAR_TOKEN` | Token d'authentification SonarCloud | Authentification pour les deux analyses Sonar |
| `SONAR_PROJECT_KEY_BACK` | Clé projet SonarCloud du back | Identifie le projet back dans SonarCloud |
| `SONAR_PROJECT_KEY_FRONT` | Clé projet SonarCloud du front | Identifie le projet front dans SonarCloud |
| `SONAR_ORGANIZATION` | Identifiant organisation SonarCloud | Requis pour SonarCloud (hébergement cloud) |

#### Création du token SonarCloud

1. Se connecter sur [sonarcloud.io](https://sonarcloud.io)
2. Mon compte → **Security** → **Generate Token**
3. Copier la valeur et la coller dans le secret GitHub `SONAR_TOKEN`

---

### Choix techniques justifiés

#### Jobs parallèles
Les jobs `backend` et `frontend` s'exécutent **en parallèle** pour réduire le temps total du pipeline.

#### Pourquoi `fetch-depth: 0` pour Sonar ?
SonarQube calcule le *blame* git pour dater les problèmes et identifier les auteurs.  

#### Deux projets Sonar distincts
- Back-end : Java/Gradle  
- Front-end : TypeScript/Angular  
Séparer les projets Sonar permet des Quality Gates et rapports de couverture indépendants.

#### Cache Gradle/npm
Réduit drastiquement le temps de build en réutilisant les dépendances téléchargées.

#### Docker Build + Trivy dans JOB 3
- Valide que les Dockerfiles compilent sans pousser vers un registry.  
- Trivy analyse les images construites sur les vulnérabilités **CRITICAL**.  
- Tag avec SHA pour correspondre exactement au commit testé.

---


## Conteneurisation

### Dockerfiles

--- 

#### Backend

Build stage : gradle:8-jdk17-alpine
Image officielle Gradle + JDK 17
Alpine = légère
Multi-stage build pour ne pas embarquer Gradle dans l’image finale

Runtime stage : eclipse-temurin:17-jre-alpine
OpenJDK officiel maintenu par Eclipse Foundation
Alpine = minimaliste et sécurisé

Permissions : chmod +x gradlew → nécessaire pour exécuter Gradle wrapper
Utilisateur non-root : USER 1001 → réduit le risque de compromission
Port exposé : 8080 → standard Spring Boot

---

#### Frontend

Build stage : node:20-alpine
Node officiel, version fixe 20
Alpine = légère

npm ci pour build reproductible
Runtime stage : nginx:2-alpine
Serveur web officiel, Alpine minimaliste
Multi-stage build pour ne pas embarquer Node ou Angular build dans runtime

Ports exposés : 80 / 443
HTTPS en dev : optionnel, HTTP suffisant pour développement

Reverse proxy : nginx peut servir le front et rediriger /api/* vers backend

---

### docker-compose.yml

Orchestration front + back séparés
Réseau interne (microcrm-net) pour communication front → back
depends_on → front attend que back soit lancé

Ports mappés :
- Front → 80/443
- Back → 8080

Avantages : possibilité de scale chaque service indépendamment, logs séparés, pas de supervisor nécessaire

---

### Bonnes pratiques sécurité

Images officielles, maintenues et fiables : Node, Gradle, Eclipse Temurin, Nginx
Alpine ou slim → minimaliste → réduit surface d’attaque
Multi-stage build → runtime propre et léger
Suppression de packages build inutiles dans runtime
Utilisateur non-root pour backend
Ports exposés uniquement nécessaires

---

### Utilisation de Trivy pour la sécurité des images Docker

Pour sécuriser les images Docker du projet, utilisation de Trivy.
Ce scanner open-source détecte les vulnérabilités dans le système d’exploitation et les dépendances applicatives.
Les images front-end et back-end sont analysées après build, et les résultats sont documentés pour garantir un pipeline de build sûr et reproductible.
Pour lancer l'analyse des images, une fois `trivy` installé, exécuter les commandes suivantes :
-       trivy image projet_7-back
        trivy image projet_7-front
        trivy image --format html --output trivy-report-back.html --scanners vuln projet_7-back
        trivy image --format html --output trivy-report-front.html --scanners vuln projet_7-front
