# Documentation technique

- **Document** : Documentation technique | `projet7_orion`
- **Auteur** : Pierre Cistac
- **Option choisie** : Option B (Scénario Orion)
- **Date** : 03/03/2026

---

## Sommaire

- [Introduction](#introduction)
- [Installation et utilisation](#installation-et-utilisation)
- [Étapes de mise en œuvre du pipeline CI/CD](#étapes-de-mise-en-œuvre-du-pipeline-cicd)
- [Stack ELK](#stack-elk)
- [Monitoring, métriques & KPI](#monitoring-métriques--kpi)
- [Plan de conteneurisation](#plan-de-conteneurisation)
- [Plan de déploiement](#plan-de-déploiement)
- [Plan de testing périodique](#plan-de-testing-périodique)
- [Plan de sécurité](#plan-de-sécurité)
- [Plan de sauvegarde des données](#plan-de-sauvegarde-des-données)
- [Plan de mise à jour](#plan-de-mise-à-jour)
- [Axes d'améliorations](#axes-daméliorations)
- [Conclusion](#conclusion)
- [Annexes](#annexes)

---

## Introduction

### Contexte du projet

`projet7_orion` est une application CRM (Customer Relationship Management) composée d'un back-end Spring Boot (Java 21) et d'un front-end Angular.
Elle est développée dans le cadre du scénario Orion (Option B) et suit une approche DevOps complète : industrialisation du build, automatisation des tests, analyse de qualité, conteneurisation et versioning sémantique.

### Objectifs de l'industrialisation

- Garantir la qualité du code à chaque commit via une analyse statique automatisée (SonarCloud)
- Détecter les vulnérabilités des images Docker avant tout déploiement (Trivy)
- Automatiser intégralement le cycle de release (semantic-release + SemVer)
- Assurer la reproductibilité des builds (lockfiles, Gradle wrapper, multi-stage Docker)
- Mettre en place une observabilité locale via la stack ELK

### Technologies principales

| Couche | Technologie |
|---|---|
| Back-end | Spring Boot 3, Java 21, Gradle 8 |
| Front-end | Angular, Node 20, npm |
| Conteneurisation | Docker, Docker Compose |
| CI/CD | GitHub Actions |
| Qualité | SonarCloud, JaCoCo, Jest/Istanbul |
| Sécurité images | Trivy |
| Versioning | SemVer, Conventional Commits, semantic-release |
| Observabilité | ELK (Elasticsearch, Logstash, Kibana) |
| Registry | GitHub Container Registry (GHCR) |

---

## Installation et utilisation

Une fois le dépôt Github récupéré, se placer à la racine du projet et exécuter la commande suivante :

    docker compose up -d --build

Ne plus tenir compte des instructions d'installation du fichier [README.md](/README.md) à la racine du projet.

---

## Étapes de mise en œuvre du pipeline CI/CD

### Présentation du pipeline CI/CD

Le pipeline est défini dans `.github/workflows/ci.yml` et se déclenche sur chaque push vers `main` ou `development`, ainsi que sur chaque pull request. Il enchaîne la validation des commits, le build et les tests des deux applications, l'analyse qualité SonarCloud (sur PR et `main` uniquement), la construction et le scan des images Docker, la publication automatique d'une release versionnée sur `main` et `development`, puis le tag sémantique final des images Docker.

### Structure du pipeline

```
push (main/development) / pull_request
        │
        ├── [Job 1] commitlint      ── Validation des messages de commit
        ├── [Job 2] backend         ── Gradle build + tests + Sonar (PR+main) + artefacts
        ├── [Job 3] frontend        ── npm ci + tests + build prod + Sonar (PR+main) + artefacts
        │
        ├── [Job 4] docker-build    (dépend de backend & frontend — main/development)
        │       ├── Build image back + scan Trivy + rapport HTML
        │       └── Build image front + scan Trivy + rapport HTML
        │
        ├── [Job 5] release         (semantic-release — main/development)
        │       ├── Génération version + changelog
        │       ├── Publication GitHub Release
        │       └── Upload artefacts (JAR + Angular zip)
        │
        └── [Job 6] docker-tag      (uniquement si nouvelle version publiée)
                ├── Tag images back avec version SemVer + latest
                └── Tag images front avec version SemVer + latest
```

Les jobs `backend` et `frontend` s'exécutent en parallèle pour réduire le temps total du pipeline.
Le job `docker-build` est conditionné à leur réussite respective. Le job `release` ne s'exécute que si `docker-build` a réussi. Le job `docker-tag` ne s'exécute que si `release` a publié une nouvelle version.

#### Job 1 — CommitLint

| Étape | Outil | Justification |
|---|---|---|
| Checkout complet | `actions/checkout@v4` | Permet d'analyser l'historique des commits |
| Setup Node + cache npm | `actions/setup-node@v4` | Installation rapide des dépendances |
| Cache npm dédié | `actions/cache@v4` | Clé de cache isolée pour commitlint |
| Installation commitlint | `npm` | Validation des messages Conventional Commits |
| Validation PR | commitlint `--from/--to` | Vérifie tous les commits de la PR |
| Validation push | commitlint `HEAD~1..HEAD` | Vérifie le dernier commit uniquement |

Objectif : garantir la compatibilité des messages de commit avec semantic-release.

#### Job 2 — Back-end

| Étape | Outil | Justification |
|---|---|---|
| Checkout (`fetch-depth: 0`) | `actions/checkout@v4` | Sonar nécessite tout l'historique git |
| Cache Gradle | `actions/cache@v4` | Réutilise `~/.gradle` entre les runs |
| JDK 21 | `actions/setup-java@v4` | Correspond à la version du Dockerfile back |
| Build + tests | `./gradlew build --no-daemon` | Compile et exécute les tests en une seule commande |
| Publier résultats JUnit | `actions/upload-artifact@v4` | Rapports de tests accessibles dans l'interface GitHub |
| Upload JAR | `actions/upload-artifact@v4` | Artefact utilisé pour la release |
| Sonar back *(PR + main uniquement)* | plugin Gradle `sonar` | Analyse qualité + sécurité du code Java via SonarCloud |

#### Job 3 — Front-end

| Étape | Outil | Justification |
|---|---|---|
| Checkout | `actions/checkout@v4` | Accès au code source complet |
| Cache npm | `actions/cache@v4` | Accélère `npm ci` en conservant `~/.npm` |
| Node 20 | `actions/setup-node@v4` | Version LTS correspondant au Dockerfile front |
| `npm ci` | npm | Installation reproductible et stricte (lockfile) |
| Tests headless | `npm test -- --watch=false --browsers=ChromeHeadless` | Exécution non-interactive en environnement CI |
| Publier résultats tests | `actions/upload-artifact@v4` | Rapports de couverture accessibles |
| Build prod | `npm run build -- --configuration production` | Vérifie que le build de production fonctionne |
| Upload dist | `actions/upload-artifact@v4` | Artefact utilisé pour la release |
| Sonar front *(PR + main uniquement)* | `SonarSource/sonarqube-scan-action@v6` | Analyse qualité + sécurité front-end via SonarCloud |

#### Job 4 — Docker Build & Scan

Conditionné à la réussite des jobs `backend` et `frontend` (`needs: [backend, frontend]`), et limité aux branches `main` et `development`.

- Utilise le cache GitHub Actions (`type=gha`) pour accélérer les builds successifs
- Les images sont poussées vers GHCR et taguées avec `$SAFE_BRANCH-${{ github.sha }}` (nom de branche normalisé + SHA complet)
- Trivy analyse les images construites sur les vulnérabilités configurables via `vars.SEVERITY`
- Les rapports Trivy sont générés au format HTML et uploadés comme artefacts GitHub Actions

#### Job 5 — Semantic Release

Exécuté sur `main` et `development`, conditionné à la réussite du job `docker-build`.

| Étape | Description |
|---|---|
| Installation semantic-release | Prépare les outils de versioning |
| Download artefacts | Récupère JAR et build Angular depuis les jobs précédents |
| Zip Angular | Prépare l'artefact release |
| Semantic release | Calcule la version SemVer + génère le changelog |
| GitHub Release | Publication automatique avec notes de version |
| Upload artefacts | Ajout JAR + frontend zip à la GitHub Release |

#### Job 6 — Docker Tag

Exécuté uniquement si le job `release` a publié une nouvelle version (`needs.release.outputs.new_release_published == 'true'`).

| Étape | Description |
|---|---|
| Login GHCR | Authentification au registry |
| Pull image `branch-sha` | Récupère les images construites au Job 4 |
| Tag `VERSION` + `latest` (back) | Appose la version SemVer et le tag `latest` sur l'image back |
| Tag `VERSION` + `latest` (front) | Appose la version SemVer et le tag `latest` sur l'image front |
| Push vers GHCR | Publication des images versionnées |

Ce job découple volontairement la construction des images (Job 4) du tag sémantique (Job 6), garantissant que seules les images associées à une release valide reçoivent un tag de version.

### Scripts d'automatisation

Le pipeline ne repose pas sur des scripts externes mais sur des commandes directement intégrées dans les steps YAML :

- `./gradlew build --no-daemon` — build et tests back-end
- `./gradlew sonar --no-daemon` — analyse SonarCloud back-end
- `npm ci` — installation des dépendances front-end
- `npm test -- --watch=false --browsers=ChromeHeadless --code-coverage` — tests Angular en mode headless
- `npm run build -- --configuration production` — build Angular de production
- `npx commitlint --from ... --to ... --verbose` — validation des messages de commit

Ces commandes sont toutes rejouables localement en dehors de la CI.

### Reproductibilité

Le pipeline est reproductible grâce aux mesures suivantes :

- **Lockfiles commités** : `package-lock.json` (front) et `gradle-wrapper.properties` (back) garantissent des versions de dépendances identiques à chaque run
- **Cache CI** : les caches Gradle (`~/.gradle`) et npm (`~/.npm`) sont restaurés entre les runs via `actions/cache@v4`
- **Images Docker fixées** : les images de base sont épinglées à des versions précises (`gradle:8.14-jdk21`, `eclipse-temurin:21-jre-alpine`, `node:20-alpine`, `nginx:alpine`)
- **Secrets** : tous les secrets sont définis dans `Settings → Secrets and variables → Actions`. Aucun secret n'apparaît en clair dans le code

| Secret | Usage |
|---|---|
| `SONAR_TOKEN` | Authentification SonarCloud |
| `SONAR_PROJECT_KEY_BACK` | Clé projet SonarCloud back-end |
| `SONAR_ORGANIZATION` | Organisation SonarCloud |
| `GITHUB_TOKEN` | Authentification GitHub (auto-injecté) |

| Variable | Valeur |
|---|---|
| `JAVA_VERSION` | 21 |
| `NODE_VERSION` | 20 |
| `DISTRIBUTION` | temurin |
| `REGISTRY` | ghcr.io |
| `IMAGE_REPO` | projet_7 |
| `RETENTION_DAYS` | 7 |
| `SEVERITY` | CRITICAL |

---

## Stack ELK

Développement local uniquement.
La stack ELK est isolée dans un fichier `docker-compose-elk.yml` distinct et ne fait pas partie du déploiement de production. Elle est démarrée à la demande en local :

```bash
docker compose -f docker-compose-elk.yml up -d
```

Architecture des logs :

```
Spring Boot (back)
    │  TCP JSON (port 5000)
    ▼
Logstash ──────────────► Elasticsearch (port 9200)
                                 │
                                 ▼
                           Kibana (port 5601)
```

Configuration Spring Boot requise : dépendance `logstash-logback-encoder:7.4` dans `build.gradle` et fichier `logback-spring.xml` pointant vers Logstash. Le profil `test` désactive l'appender Logstash pour ne pas polluer les tests unitaires.

Après démarrage, créer l'index pattern `projet7-*` dans Kibana (`Management → Stack Management → Index Patterns`) et sélectionner `@timestamp` comme champ de temps.

Points de vigilance :

- `xpack.security` est désactivé — ne jamais déployer cette configuration en production
- La heap JVM ne doit pas dépasser 50% de la RAM du conteneur (règle ES/LS)
- La stack ELK nécessite environ 4 Go de RAM disponibles

---

## Monitoring, métriques & KPI

### Métriques DORA

#### Deployment Frequency

Fréquence à laquelle une nouvelle version est déployée en production. Dans ce projet, chaque merge sur `main` ou `development` qui produit un nouveau tag SemVer via semantic-release constitue un déploiement.

Valeur observée : dépend du rythme des PRs vers `main`. L'outillage permet une fréquence quotidienne si les Quality Gates sont vertes.

#### Lead Time for Changes

Temps entre le premier commit d'une feature et son déploiement en production. Il comprend le temps de développement, la durée du pipeline CI (build + tests + scan + release + tag Docker) et la validation manuelle de la PR vers `main`.

Valeur observée : la durée du pipeline CI est de l'ordre de 8 à 12 minutes sur `main`. Le lead time total dépend principalement du cycle de review humain.

#### MTTR (Mean Time To Restore)

Temps moyen pour revenir à un état stable après un incident. Grâce aux images Docker versionnées et immuables stockées sur GHCR, le rollback se réduit à un `docker compose pull + up -d` avec le tag de la version précédente.

Valeur observée : rollback en moins de 5 minutes si les images sont déjà présentes sur le serveur.

#### Change Failure Rate

Proportion de déploiements qui entraînent un incident. La présence du scan Trivy, de la Quality Gate SonarCloud et des tests automatisés réduit mécaniquement ce taux en bloquant les artefacts défaillants avant production.

### KPI personnalisés

| KPI | Source | Valeur observée |
|---|---|---|
| Durée du pipeline complet | GitHub Actions | ~8-12 min sur `main` |
| Durée des tests back | GitHub Actions | 0.656s (2 tests) |
| Couverture back | JaCoCo / SonarCloud | 56.4% (nouveau code) |
| Couverture front | Istanbul | 33.08% (statements) |
| Nombre de CVE HIGH (images) | Trivy | 3 (back: 2, front: 1) |
| Nombre de CVE CRITICAL | Trivy | 0 |
| Security Hotspots ouverts | SonarCloud | 3 |
| Duplications nouveau code | SonarCloud | 7.9% |
| Quality Gate | SonarCloud | Failed (3 conditions) |

### Analyse synthétique du monitoring

**Points forts**

- Zéro vulnérabilité CRITICAL dans les images Docker
- Pipeline entièrement automatisé de la validation du commit jusqu'à la publication de la release et au tag sémantique des images
- Rollback rapide grâce aux images versionnées sur GHCR
- Rapports Trivy HTML uploadés comme artefacts GitHub Actions pour chaque build
- Observabilité locale complète via la stack ELK avec dashboards Kibana préconfigurés

**Points à améliorer**

- La Quality Gate SonarCloud est en échec sur trois conditions (couverture, duplication, hotspots non révisés) — le pipeline ne bloque pas réellement sur ces critères avec un token personnel
- La couverture de tests front est insuffisante, notamment sur les branches (9.52%)
- Les CVE HIGH dans les images ne bloquent pas le pipeline (seuil limité à CRITICAL)
- Aucun monitoring de production n'est en place (ELK réservé au local)
- Sonar n'est exécuté que sur PR et `main`, les pushs sur `development` ne bénéficient pas de l'analyse qualité

**Dashboards Kibana configurés (local)**

- Volume de logs par niveau (bar chart, `log_level.keyword`, date histogram)
- Compteur d'erreurs (metric, filtre `tags: error`)
- Top 10 des messages d'erreur (data table, `message.keyword`)
- Logs temps réel (Discover, filtre `log_level: ERROR OR WARN`)

**Alertes recommandées**

- Spike de logs `ERROR` au-dessus d'un seuil défini sur une fenêtre glissante de 5 minutes
- Échec du healthcheck `/actuator/health` sur le back
- Quality Gate SonarCloud en échec sur `main`

---

## Plan de conteneurisation

Voir le document [plan de conteneurisation](./plan/PLAN_CONTENEURISATION.md)

---

## Plan de déploiement

Voir le document [plan de déploiement](./plan/PLAN_DEPLOIEMENT.md)

---

## Plan de testing

Voir le document [plan de testing](./plan/PLAN_TESTING.md)

---

## Plan de sécurité

Voir le document [plan de sécurité](./plan/PLAN_SECURITY.md)

---

## Plan de sauvegarde des données

Voir le document [plan de sauvegarde](./plan/PLAN_SAUVEGARDE.md)

---

## Plan de mise à jour

Voir le document [plan de mise à jour](./plan/PLAN_MAJ.md)

---

## Axes d'améliorations

// TODO

---

## Conclusion

### Résumé des améliorations apportées

Le projet `projet7_orion` dispose d'un pipeline CI/CD complet en 6 jobs couvrant l'ensemble du cycle de vie logiciel : validation des commits, build, tests, analyse de qualité (sur PR et `main`), scan de sécurité avec rapports HTML, versioning sémantique automatique, publication des artefacts, et tag sémantique des images Docker dans un job dédié. La conteneurisation repose sur des bonnes pratiques établies (multi-stage build, images Alpine, utilisateur non-root côté back) et les images sont publiées sur GHCR avec des tags immuables et versionnés.

### Gains observés

- **Fiabilité** : chaque merge sur `main` est précédé d'un pipeline complet qui valide la compilation, les tests, la qualité SonarCloud et l'absence de CVE CRITICAL
- **Rapidité** : les jobs back et front s'exécutent en parallèle, les caches Gradle et npm réduisent les temps de build, le rollback en production prend moins de 5 minutes
- **Qualité** : SonarCloud identifie les hotspots de sécurité, les duplications et mesure la couverture à chaque commit sur PR et `main`
- **Traçabilité** : chaque release est associée à un tag Git, un changelog généré automatiquement, des artefacts binaires (JAR, zip Angular) et des images Docker versionnées
- **Observabilité** : les rapports Trivy au format HTML sont archivés comme artefacts GitHub Actions pour chaque build sur `main` et `development`

### Recommandations pour les itérations suivantes

1. Débloquer la Quality Gate en priorité : couverture back >= 80%, duplication < 3%, révision des hotspots CORS
2. Étendre le seuil Trivy à `CRITICAL,HIGH` pour bloquer les CVE HIGH corrigeables
3. Activer Sonar sur `development` pour détecter les régressions de qualité avant le merge sur `main`
4. Mettre en place un monitoring de production (instance ELK dédiée ou service externe) pour disposer d'une observabilité hors du contexte local
5. Activer Dependabot pour automatiser la veille sur les dépendances
6. Remplacer le PAT SonarCloud par un token de service d'organisation pour plus de robustesse

---

## Annexes

### Screenshots

Voir le dossier `./_doc/screenshots` à la racine du projet.

### Reports

Voir le dossier `./_doc/reports` à la racine du projet.

### Commandes utiles

```bash
# Lancer les tests back-end localement
cd back && ./gradlew test --no-daemon

# Lancer les tests front-end localement
cd front && npm test -- --watch=false --browsers=ChromeHeadless --code-coverage

# Analyser SonarCloud manuellement (back)
cd back && ./gradlew sonar --no-daemon \
  -Dsonar.projectKey=<key> \
  -Dsonar.organization=<org> \
  -Dsonar.host.url=https://sonarcloud.io

# Scanner les images Docker avec Trivy
trivy image projet_7-back
trivy image projet_7-front

# Démarrer l'application
docker compose up -d

# Démarrer la stack ELK (local uniquement)
docker compose -f docker-compose-elk.yml up -d

# Vérifier la santé du back-end
curl -s http://localhost:8080/actuator/health | jq .

# Valider les messages de commit
npx commitlint --from HEAD~1 --to HEAD --verbose
```