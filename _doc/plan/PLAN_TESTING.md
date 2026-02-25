<h1 id="planTesting">Plan de testing périodique</h1>

• Titre du document : Plan de testing | `projet7_orion`  
• Auteur : Pierre Cistac   
• Option choisie : Option B (Scénario Orion)   
• Date : 27/02/2026
• Objectif : garantir la qualité continue, détecter les régressions et sécuriser les déploiements.

## Tests automatisés en continu (CI)

Déclenchés à chaque :
- Push
- Pull Request
- Merge vers development ou main

Tests exécutés
			
| Type | Outil | Fréquence | Objectif |
|---|---|---|---|
Tests unitaires backend | Gradle + JUnit | Chaque CI | Validation logique métier |
Tests unitaires frontend | Karma/Jest |	Chaque CI |	Validation composants Angular |
Tests d’intégration | Spring Boot Test | Chaque CI | Vérifier interactions services |
Analyse statique | SonarCloud |	Chaque CI |	Détection bugs et vulnérabilités |
Scan sécurité images | Trivy | Chaque build Docker | Détection CVE critiques |
Validation commits | Commitlint | Chaque CI | Cohérence versionning |

Condition bloquante : échec CI = merge interdit.

## Tests périodiques planifiés

Même avec une CI forte, certains tests doivent être planifiés indépendamment.

### Tests hebdomadaires

Rebuild complet sans cache (détection dépendances cassées)
Scan Trivy complet avec toutes sévérités
Vérification Quality Gate SonarCloud
Revue logs Kibana erreurs WARN/ERROR

### Tests mensuels

Tests de charge (ex : k6 / Gatling)
Tests de performance API
Vérification métriques DORA
Audit dépendances (npm audit, Gradle dependency check)

### Tests trimestriels

Tests de reprise après incident (chaos testing léger)
Simulation rollback Docker
Test restauration sauvegardes Elasticsearch

## Tests avant mise en production

Avant merge develop → main :
- Validation fonctionnelle manuelle
- Smoke tests automatisés
- Vérification dashboards Kibana
- Validation release candidate

---

## Types de tests automatisés

**Back-end (Java / JUnit / JaCoCo)**

- Tests unitaires : validation du comportement des classes métier (`Person`, `Organization`)
- Tests d'intégration : `PersonRepositoryIntegrationTest` — validation de la couche de persistance Spring Data
- Rapport de couverture : généré par JaCoCo au format XML, transmis à SonarCloud

**Front-end (Angular / Jest / Istanbul)**

- Tests unitaires : validation des composants Angular (spec files)
- Couverture de code : générée par Istanbul, transmise à SonarCloud
- Exécution headless : `ChromeHeadless` pour compatibilité CI

**Analyse statique (SonarCloud)**

- Bugs, vulnérabilités, code smells, duplications, couverture : analysés sur back et front à chaque push
- Security Hotspots : identifiés et soumis à révision manuelle

**Sécurité des images (Trivy)**

- Scan des images Docker back et front sur les CVE (CRITICAL configuré, extension à HIGH recommandée)
- Exécuté après chaque build Docker sur `main` et `development`

## Fréquence d'exécution

| Déclencheur | Tests exécutés |
|---|---|
| Push sur toute branche | Tests back + tests front + SonarCloud |
| Pull request | Idem + validation de tous les commits de la PR (commitlint) |
| Push sur `main` / `development` | Idem + build Docker + scan Trivy + release |
| Avant release | Tous les jobs doivent être verts (Quality Gate + Trivy) |

Aucun nightly build n'est configuré actuellement. Les tests sont entièrement déclenchés par les événements Git.

## Critères de réussite

| Critère | Seuil requis | État actuel |
|---|---|---|
| Tests back | 100% succès | ✅ 2/2 |
| Tests front | 100% succès | ✅ |
| Coverage back (nouveau code) | >= 80% | ❌ 56.4% |
| Coverage front (statements) | >= 80% | ❌ 33.08% |
| Duplication (nouveau code) | <= 3% | ❌ 7.9% |
| Security Hotspots révisés | 100% | ❌ 0% |
| Vulnérabilités CRITICAL (Trivy) | 0 | ✅ |
| Messages de commit | Conventional Commits | ✅ |

## Objectifs des tests

- **Qualité** : maintenir la Quality Gate SonarCloud verte sur `main` avant tout merge
- **Non-régression** : détecter les régressions introduites par les nouvelles fonctionnalités avant merge
- **Sécurité** : garantir l'absence de CVE CRITICAL dans les images déployées
- **Reproductibilité** : s'assurer que le build de production Angular fonctionne à chaque run
