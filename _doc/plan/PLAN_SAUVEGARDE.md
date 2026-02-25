<h1 id="planSauvegarde">Plan de sauvegarde</h1>

• Titre du document : Plan de sauvegarde | `projet7_orion`  
• Auteur : Pierre Cistac   
• Option choisie : Option B (Scénario Orion)   
• Date : 27/02/2026

## Ce qui doit être sauvegardé

**Données (si applicable)**
- Aucune base de données applicative n'est gérée directement dans la CI. Si l'application manipule des données persistantes (BDD, volumes Docker), celles-ci doivent être sauvegardées en dehors du pipeline, au niveau de l'infrastructure d'hébergement.

**Fichiers de configuration**
- `.github/workflows/ci.yml` — définition complète du pipeline CI/CD
- `package.json` / `package-lock.json` — dépendances Node.js (commitlint, semantic-release)
- `back/**/*.gradle` / `gradle-wrapper.properties` — configuration Gradle et wrapper
- `front/package.json` / `front/package-lock.json` — dépendances Angular
- `sonar-project.properties` (front) — configuration SonarCloud front-end
- Variables et secrets GitHub (`Settings → Secrets and variables → Actions`) : `SONAR_TOKEN`, `SONAR_PROJECT_KEY_BACK`, `SONAR_ORGANIZATION`, `REGISTRY`, `IMAGE_REPO`, `JAVA_VERSION`, `NODE_VERSION`, etc. — à documenter dans un coffre-fort sécurisé (ex. Bitwarden, Vault, 1Password)

**Artefacts de build**
- JAR Spring Boot : `back/build/libs/*.jar` — uploadé comme artefact GitHub Actions (`backend-jar`) avec rétention de `RETENTION_DAYS` jours
- Build Angular production : `front/dist/` — uploadé comme artefact GitHub Actions (`frontend-dist`) avec la même rétention
- Images Docker versionnées : publiées dans GitHub Container Registry (`ghcr.io`) avec les tags `<branch>-<sha>`, `<version>` et `latest`
- GitHub Releases : chaque release stable contient le JAR et le `frontend-dist.zip`, associés à un tag Git SemVer (`vX.Y.Z`)

---

## Procédure de sauvegarde

**Format**
- Artefacts CI : fichiers binaires (`.jar`, `.zip`) attachés aux GitHub Actions runs et GitHub Releases
- Images Docker : layers OCI stockés dans le GitHub Container Registry (GHCR), identifiables par digest SHA256
- Configuration : fichiers versionnés dans le dépôt Git (source de vérité unique)
- Secrets : exportés manuellement et stockés dans un gestionnaire de secrets externe (non versionné)

**Fréquence**
- À chaque push sur `main` ou `development` : build, publication d'image Docker et potentielle nouvelle release automatique via semantic-release
- Artefacts GitHub Actions : conservés `RETENTION_DAYS` jours (valeur par défaut : 7 jours, configurable)
- GitHub Releases : conservées indéfiniment sur le dépôt GitHub
- Images Docker : conservées dans GHCR jusqu'à suppression manuelle ; prévoir une politique de nettoyage des anciennes images (tags `branch-sha`) pour éviter l'accumulation

**Outils utilisés**
```bash
# Télécharger un artefact GitHub Actions via CLI
gh run download <run-id> --name backend-jar --dir ./restore/

# Lister les images disponibles dans GHCR
docker pull ghcr.io/<owner>/<repo>-back:<version>

# Sauvegarder une image Docker localement
docker save ghcr.io/<owner>/<repo>-back:<version> | gzip > back-<version>.tar.gz

# Restaurer une image Docker depuis une archive locale
docker load < back-<version>.tar.gz

# Lister les releases GitHub
gh release list

# Télécharger les assets d'une release
gh release download v<version> --dir ./restore/
```

---

## Procédure de restauration

**Scénario d'incident**
Un déploiement introduit une régression critique en production. L'équipe doit revenir à la dernière version stable connue.

**Étapes pour revenir à une version stable**

1. **Identifier la dernière version stable** via la liste des GitHub Releases (`gh release list` ou interface GitHub).
2. **Récupérer les artefacts de la release cible** :
   ```bash
   gh release download v<version-stable> --dir ./restore/
   ```
3. **Restaurer l'image Docker back** :
   ```bash
   docker pull ghcr.io/<owner>/<repo>-back:<version-stable>
   # Re-déployer cette image sur l'environnement cible
   ```
4. **Restaurer l'image Docker front** :
   ```bash
   docker pull ghcr.io/<owner>/<repo>-front:<version-stable>
   ```
5. **Re-déployer via docker-compose** en pointant vers les tags versionnés :
   ```bash
   IMAGE_TAG=<version-stable> docker-compose up -d
   ```
6. **Vérifier la santé des services** et valider le rollback.

Si les images Docker ne sont plus disponibles dans GHCR (suppression accidentelle), utiliser le JAR téléchargé depuis la GitHub Release pour reconstruire l'image localement.

**Limitations éventuelles**
- Les artefacts GitHub Actions (JAR, dist) expirent après `RETENTION_DAYS` jours : seules les GitHub Releases constituent une sauvegarde pérenne des binaires.
- Les secrets GitHub ne sont pas récupérables après création : ils doivent impérativement être archivés dans un gestionnaire de secrets externe dès leur création.
- Les images Docker taguées `branch-sha` (images intermédiaires) peuvent être supprimées par une politique de nettoyage GHCR sans impact sur les releases, à condition que les tags `<version>` et `latest` soient préservés.
- En l'absence de base de données sauvegardée, un rollback applicatif ne restaure pas les données créées entre deux versions.