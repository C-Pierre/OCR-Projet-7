# Plan de testing périodique

Objectif : garantir la qualité continue, détecter les régressions et sécuriser les déploiements.

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
Tests d’intégration | Spring Boot Test | Chaque CI |	Vérifier interactions services |
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