# Plan de sécurité

Objectif : réduire la surface d’attaque et garantir l’intégrité du système.

## Sécurité du code

Mesures existantes :
- Analyse SonarCloud (bugs + vulnérabilités)
- Quality Gate obligatoire
- Conventional commits contrôlés
- Dépendances verrouillées (npm ci, Gradle wrapper)

Mesures recommandées :
- Dependabot activé
- SAST supplémentaire (optionnel : Snyk)

## Sécurité CI/CD

Bonnes pratiques déjà respectées :
- Secrets GitHub protégés
- Aucun secret dans le code
- Tokens limités aux besoins
- Images Docker scannées avant push

Améliorations possibles :
- OIDC pour registry (sans secrets persistants)
- Signature des images Docker (cosign)
- Protection branches main et develop

## Sécurité des conteneurs

Mesures en place :
- Images officielles maintenues
- Alpine minimal
- Multi-stage build
- Utilisateur non-root backend
- Scan Trivy CVE CRITICAL

Mesures recommandées :
- Read-only filesystem
- Drop Linux capabilities
- Healthchecks Docker
- Scan SBOM (Software Bill of Materials)

## Sécurité infrastructure & réseau

Architecture :
- Réseau Docker isolé
- Communication interne front → back
- ELK séparé

Bonnes pratiques production :
- HTTPS obligatoire (reverse proxy)
- Authentification Kibana activée
- Firewall ports exposés
- Rotation des logs
- Sauvegardes Elasticsearch

## Gestion des incidents

Détection :
- Logs ELK
- Alertes Kibana sur ERROR spike
- Monitoring CI failures

Réponse :
- Identification version déployée
- Analyse logs
- Rollback Docker image précédente
- Correctif hotfix
- Post-mortem