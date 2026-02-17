# Choix techniques

## Conteneurisation - Exerice 1 - Partie 3

### Dockerfiles

--- 

#### Backend (./back/Dockerfile)

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

#### Frontend (./front/Dockerfile)

Build stage : node:20-alpine
Node officiel, version fixe 20
Alpine = légère

npm ci pour build reproductible
Runtime stage : caddy:2-alpine
Serveur web officiel, Alpine minimaliste
Multi-stage build pour ne pas embarquer Node ou Angular build dans runtime

Ports exposés : 80 / 443
HTTPS en dev : optionnel, HTTP suffisant pour développement

Reverse proxy : Caddy peut servir le front et rediriger /api/* vers backend

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

Images officielles, maintenues et fiables : Node, Gradle, Eclipse Temurin, Caddy
Alpine ou slim → minimaliste → réduit surface d’attaque
Multi-stage build → runtime propre et léger
Suppression de packages build inutiles dans runtime
Utilisateur non-root pour backend
Ports exposés uniquement nécessaires

### Utilisation de Trivy pour la sécurité des images Docker

Pour sécuriser les images Docker du projet, utilisation de Trivy.
Ce scanner open-source détecte les vulnérabilités dans le système d’exploitation et les dépendances applicatives.
Les images front-end et back-end sont analysées après build, et les résultats sont documentés pour garantir un pipeline de build sûr et reproductible.
Pour lancer l'analyse des images, une fois `trivy` installé, exécuter les commandes suivantes :
-       trivy image projet_7-back
        trivy image projet_7-front
        trivy image --format html --output trivy-report-back.html --scanners vuln projet_7-back
        trivy image --format html --output trivy-report-front.html --scanners vuln projet_7-front

---

### Résultat attendu

Commande :
-       docker-compose up --build

Application front accessible sur http://localhost/
API backend accessible sur http://localhost:8080

Architecture sécurisée et légère
Images scannées et valides