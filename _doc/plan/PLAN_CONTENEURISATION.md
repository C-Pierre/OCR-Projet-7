<h1 id="planConteneurisation">Plan de conteneurisation</h1>

• Titre du document : Plan de conteneurisation | `projet7_orion`  
• Auteur : Pierre Cistac   
• Option choisie : Option B (Scénario Orion)   
• Date : 27/02/2026

## Séparation des responsabilités

Un conteneur = un service :
- Frontend (Nginx)
- Backend (Spring Boot)
- ELK stack (observabilité)

Avantages :
- Scalabilité indépendante
- Isolation
- Maintenance facilitée

---

## Images reproductibles

Garanties par :
- Versions figées Node / Java
- npm ci
- Gradle wrapper
- Multi-stage Docker

Objectif : même image = même comportement partout.

---

## Images légères et sécurisées

Choix techniques :
- Alpine Linux
- Runtime sans outils de build
- Suppression dépendances inutiles

Résultat :
- Surface d’attaque réduite
- Déploiement rapide

---

## Exécution non-root

Backend :
- USER 1001
- Réduit l’impact d’une compromission.

---

## Dockerfiles

### Back-end

Multi-stage build en deux étapes :

- **Build stage** (`gradle:8.14-jdk21`) : compile l'application via `./gradlew build --no-daemon -x test`. L'image Gradle complète n'est pas embarquée dans le runtime final.
- **Runtime stage** (`eclipse-temurin:21-jre-alpine`) : image JRE minimale Alpine. Un utilisateur non-root (`USER 1001`) est créé pour réduire la surface d'attaque en cas de compromission. Le port 8080 est exposé (standard Spring Boot).

Le multi-stage build garantit que ni Gradle ni les sources Java ne sont présents dans l'image de production.

### Front-end

Multi-stage build en deux étapes :

- **Build stage** (`node:20-alpine`) : installe les dépendances via `npm ci` et compile Angular en mode production (`npm run build -- --configuration production`).
- **Runtime stage** (`nginx:alpine`) : sert les fichiers statiques compilés. Le build Node.js complet n'est pas embarqué dans le runtime. Un fichier `nginx.conf` personnalisé est copié pour la configuration du serveur. Les ports 80 et 443 sont exposés.

---

## docker-compose.yml

Deux services applicatifs orchestrés sur un réseau interne `microcrm-net` :

- **`back`** : démarre le back-end Spring Boot, expose `APP_PORT` (8080), inclut un healthcheck sur `/actuator/health` (interval 30s, timeout 5s, 5 retries)
- **`front`** : démarre Nginx, expose les ports 80 et 443, dépend du service `back` via `depends_on`

Toute la configuration est externalisée dans un fichier `.env` (non versionné) :

```env
APP_NAME=microcrm
APP_HOST=localhost
APP_PORT=8080
APP_FRONT_PORT=80:80
APP_FRONT_PORT_2=443:443
DOCKER_RESTART=always
DOCKER_NETWORK=microcrm-net
```

Lancer l'application localement :

```bash
# Copier et remplir le fichier de variables
cp .env.template .env

# Démarrer les services
docker compose up -d

# Vérifier l'état
docker compose ps
docker compose logs --tail=50 back
```