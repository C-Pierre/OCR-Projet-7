# Principes de conteneurisation et de déploiement

## Principes de conteneurisation

### Séparation des responsabilités

Un conteneur = un service :
- Frontend (Nginx)
- Backend (Spring Boot)
- ELK stack (observabilité)

Avantages :
- Scalabilité indépendante
- Isolation
- Maintenance facilitée

### Images reproductibles

Garanties par :
- Versions figées Node / Java
- npm ci
- Gradle wrapper
- Multi-stage Docker

Objectif : même image = même comportement partout.

### Images légères et sécurisées

Choix techniques :
- Alpine Linux
- Runtime sans outils de build
- Suppression dépendances inutiles

Résultat :
- Surface d’attaque réduite
- Déploiement rapide

### Exécution non-root

Backend :
- USER 1001
- Réduit l’impact d’une compromission.

## Principes de déploiement

### Déploiement automatisé

Pipeline :
-       Commit → CI → Tests → Docker build → Scan → Release → Registry
Aucune intervention humaine sur la version.

### Versionnement immuable

Images taguées :
-       back:1.2.0
-       front:1.2.0

Jamais modifiées après publication.

Permet :
- Rollback rapide
- Traçabilité complète
- Audit facilité

### Stratégie d’environnements

Branches :
-       feature → dev local
-       development → staging
-       main → production

Correspondance environnements :
-       Branche	→ Environnement
-       development	→ Intégration
-       main	→ Production

### Stratégie de rollback

Rollback simple :
-       docker pull image:version_precedente
-       docker compose up -d

Temps de restauration très faible → bon score MTTR DORA.

### Observabilité intégrée

Stack ELK :
- Logs applicatifs
- Erreurs runtime
- Monitoring comportement

Lien direct avec :
- MTTR
- Change Failure Rate