<h1 id="planDeploiement">Plan de déploiement</h1>

• Titre du document : Plan de déploiement | `projet7_orion`  
• Auteur : Pierre Cistac   
• Option choisie : Option B (Scénario Orion)   
• Date : 27/02/2026

## Déploiement automatisé

Pipeline :
-       Commit → CI → Tests → Docker build → Scan → Release → Registry
Aucune intervention humaine sur la version.

---

## Versionnement immuable

Images taguées :
-       back:1.2.0
-       front:1.2.0

Jamais modifiées après publication.

Permet :
- Rollback rapide
- Traçabilité complète
- Audit facilité

---

## Stratégie d’environnements

Branches :
-       feature → dev local
-       development → staging
-       main → production

Correspondance environnements :
-       Branche	→ Environnement
-       development	→ Intégration
-       main	→ Production

---

## Stratégie de rollback

Rollback simple :
-       docker pull image:version_precedente
-       docker compose up -d

Temps de restauration très faible → bon score MTTR DORA.

---

## Observabilité intégrée

Stack ELK :
- Logs applicatifs
- Erreurs runtime
- Monitoring comportement

Lien direct avec :
- MTTR
- Change Failure Rate

---

## Stratégie d’environnements 2 : scénarion production

### Introduction

Ceci est un scénario de déploiement, il n'a pas été réalisé dans la pratique.

---

### Architecture cible

Deux environnements distincts, correspondant aux branches principales du dépôt :

| Environnement | Branche | Usage |
|---|---|---|
| Staging | `development` | Intégration, validation fonctionnelle |
| Production | `main` | Déploiement stable, utilisateurs finaux |

Chaque environnement est hébergé sur un VPS dédié. La stack applicative est composée de deux services (`back`, `front`) orchestrés via Docker Compose. La stack ELK est réservée à l'usage local de développement et n'est pas déployée en production.

---

### Prérequis du serveur

Avant tout déploiement, le VPS doit disposer de :

- Docker Engine et Docker Compose installés
- Accès SSH configuré avec clé (pas de mot de passe)
- Accès au GitHub Container Registry (GHCR) depuis le serveur
- Un fichier `.env` de production présent sur le serveur (non versionné)
- Les ports applicatifs ouverts dans le firewall (`APP_PORT`, `APP_FRONT_PORT`)

---

### Variables d'environnement

Le `docker-compose.yml` repose entièrement sur des variables d'environnement. Le fichier `.env` de production doit être créé manuellement sur le serveur et ne jamais être commité dans le dépôt.

Variables minimales requises :

```env
APP_NAME=microcrm
APP_HOST=localhost
APP_PORT=8080
APP_FRONT_PORT=80:80
APP_FRONT_PORT_2=443:443
DOCKER_RESTART=always
DOCKER_NETWORK=microcrm-net
```

---

### Processus de déploiement

#### Déploiement automatique (CI/CD)

Le pipeline CI/CD gère le déploiement de bout en bout à chaque push sur `main` ou `development`. Le flux est le suivant :

```
push sur main / development
        │
        ├── Tests back + front
        ├── Build images Docker
        ├── Scan Trivy (CRITICAL bloquant)
        ├── Semantic Release → tag vX.Y.Z
        └── Images publiées sur GHCR
              ghcr.io/<owner>/microcrm-back:<version>
              ghcr.io/<owner>/microcrm-front:<version>
```

Une fois les images publiées sur GHCR, le déploiement sur le VPS peut être déclenché via un job CI supplémentaire (SSH + `docker compose pull && docker compose up -d`) ou manuellement.

#### Déploiement manuel sur le VPS

```bash
# 1. Se connecter au serveur
ssh user@<ip-vps>

# 2. S'authentifier sur GHCR
echo $GITHUB_TOKEN | docker login ghcr.io -u <username> --password-stdin

# 3. Récupérer la dernière version des images
docker compose pull

# 4. Relancer les services sans interruption
docker compose up -d --remove-orphans

# 5. Vérifier l'état des conteneurs
docker compose ps
docker compose logs --tail=50 back
```

---

### Healthcheck & validation

Le service `back` expose un endpoint Spring Boot Actuator utilisé par Docker pour valider l'état du conteneur :

```
GET http://<APP_HOST>:<APP_PORT>/actuator/health
Interval : 30s — Timeout : 5s — Retries : 5
```

Le service `front` dépend de `back` via `depends_on`. Docker attend que le back soit `healthy` avant de démarrer le front.

Après chaque déploiement, valider manuellement :

```bash
# Statut des conteneurs
docker compose ps

# Santé du back
curl -s http://localhost:8080/actuator/health | jq .

# Accès au front
curl -I http://localhost:80
```

---

### Rollback

En cas de régression détectée après déploiement, revenir à la version précédente en ciblant son tag GHCR :

```bash
# Identifier la version stable précédente
# (depuis les GitHub Releases ou les tags Docker GHCR)

# Modifier le docker-compose.yml pour pointer vers la version stable
# ou utiliser directement la commande pull avec le tag cible

docker pull ghcr.io/<owner>/microcrm-back:<version-stable>
docker pull ghcr.io/<owner>/microcrm-front:<version-stable>

# Redémarrer avec les images récupérées
docker compose up -d
```

Le temps de restauration est minimal grâce aux images versionnées et immuables stockées sur GHCR.

---

### Stack ELK (développement local uniquement)

La stack ELK (`elasticsearch`, `logstash`, `kibana`) est isolée dans un fichier `docker-compose-elk.yml` distinct et n'est pas déployée en production. Elle est démarrée à la demande en local :

```bash
docker compose -f docker-compose-elk.yml up -d
```

Pour une observabilité en production, prévoir à terme un service de logs centralisé externe (Datadog, Grafana Cloud, ou une instance ELK dédiée sur un VPS séparé avec authentification Kibana activée).