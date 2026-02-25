<h1 id="planSecurity">Plan de sécurité</h1>

• Titre du document : Plan de sécurité | `projet7_orion`  
• Auteur : Pierre Cistac   
• Option choisie : Option B (Scénario Orion)   
• Date : 27/02/2026

## Résultats SonarQube

> Projet analysé : **Pierre.C / microcrm** — branche `main` — 226 lignes de code (Java)
> Dernière analyse : 25/02/2026 — Quality Gate : **Failed** (3 conditions non respectées)

### Vulnérabilités identifiées

La note **Security** est **A (0 vulnérabilité)** sur l'ensemble du code. Aucune faille de sécurité avérée n'a été détectée par SonarCloud.

En revanche, **3 Security Hotspots** sont présents, tous classés en priorité **Low**, catégorie **Insecure Configuration**, et à 0% révisés (condition d'échec de la Quality Gate) :

| # | Fichier | Règle | Problème |
|---|---------|-------|----------|
| 1 | `OrganizationRepository.java` (ligne 8) | `java:S5122` | `@CrossOrigin` sans restriction d'origine — politique CORS permissive |
| 2 | `OrganizationRepository.java` | `java:S5122` | Même annotation, second point d'exposition |
| 3 | `OrganizationRepository.java` | `java:S5122` | Même annotation, troisième occurrence |

Les trois hotspots sont liés à l'usage de `@CrossOrigin` sans paramètre `origins` explicite sur le repository Spring Data REST, ce qui autorise potentiellement toutes les origines à interroger l'API.

### Code Smells critiques

La note **Maintainability** est **A (8 code smells)**. Aucun n'est critique au sens sécurité, mais leur accumulation peut nuire à la lisibilité et à la maintenabilité du code.

### Zones de complexité / duplication

Le dashboard Duplications révèle **9.2% de duplication globale** (7.9% sur le nouveau code — condition d'échec de la Quality Gate), avec **28 lignes dupliquées** réparties sur 2 blocs :

| Fichier | Duplication |
|---------|-------------|
| `Organization.java` | 16.5% — 14 lignes dupliquées |
| `Person.java` | 12.3% — 14 lignes dupliquées |

Ces deux entités présentent vraisemblablement des champs et méthodes redondants (getters/setters, annotations JPA) qui mériteraient une extraction vers une classe abstraite parente.

### Couverture des tests

#### Back-end (Jacoco / Gradle)

| Indicateur | Valeur |
|---|---|
| Tests exécutés | 2 |
| Échecs | 0 |
| Taux de succès | 100% |
| Coverage sur nouveau code (SonarCloud) | **56.4%** (requis : >= 80%) ❌ |
| Line Coverage | 60.0% |
| Lignes non couvertes | 28 / 70 |

Détail de la couverture par fichier :

| Fichier | Coverage | Lignes non couvertes |
|---------|----------|----------------------|
| `InitialDataFixture.java` | 92.3% ✅ | 0 |
| `SpringDataRestCustomization.java` | 100% ✅ | 0 |
| `Organization.java` | 36.4% ⚠️ | 11 |
| `Person.java` | 45.2% ⚠️ | 15 |
| `MicroCRMApplication.java` | 33.3% ⚠️ | 2 |

#### Front-end (Jest / Istanbul)

| Indicateur | Valeur |
|---|---|
| Statements | **33.08%** (44/133) ❌ |
| Branches | **9.52%** (2/21) ❌ |
| Functions | **28.94%** (11/38) ❌ |
| Lines | **30.76%** (36/117) ❌ |

Détail par module :

| Module | Statements | Branches | Functions | Lines |
|--------|-----------|----------|-----------|-------|
| `app` (racine) | 20.33% ❌ | 0% ❌ | 33.33% ❌ | 18.96% ❌ |
| `app/main-dashboard` | 77.77% ⚠️ | 100% ✅ | 50% ⚠️ | 100% ✅ |
| `app/organization-details` | 45.83% ❌ | 20% ❌ | 28.57% ❌ | 40% ❌ |
| `app/person-details` | 34.14% ❌ | 8.33% ❌ | 16.66% ❌ | 33.33% ❌ |

---

## Résultats Trivy

> Scans réalisés localement le 26/02/2026 — Trivy v0.69 — OS détecté : Alpine 3.23.3

### Image back (`projet_7-back`)

Basée sur `eclipse-temurin:21-jre-alpine` (runtime), buildée depuis `gradle:8.14-jdk21`. Multi-stage build — le build stage n'est pas embarqué dans l'image finale. Utilisateur non-root `1001` configuré.

| Couche | Type | Vulnérabilités |
|--------|------|----------------|
| Alpine 3.23.3 | OS | 3 (HIGH: 2, MEDIUM: 1) |
| `app/app.jar` | JAR | 2 (MEDIUM: 1, LOW: 1) |
| **Total** | | **5** |

Détail des CVE :

| Bibliothèque | CVE | Sévérité | Version installée | Version corrigée | Description |
|---|---|---|---|---|---|
| `gnutls` | CVE-2026-1584 | HIGH | 3.8.11-r0 | 3.8.12-r0 | Déni de service distant via ClientHello PSK malformé |
| `gnutls` | CVE-2025-14831 | MEDIUM | 3.8.11-r0 | 3.8.12-r0 | DoS par consommation excessive lors de la vérification de certificat |
| `libpng` | CVE-2026-25646 | HIGH | 1.6.54-r0 | 1.6.55-r0 | Heap buffer overflow dans `png_set_quantize` |
| `tomcat-embed-core` | CVE-2025-66614 | MEDIUM | 10.1.47 | 10.1.49 | Contournement de vérification de certificat client via virtual host mapping |
| `tomcat-embed-core` | CVE-2026-24733 | LOW | 10.1.47 | — | Contournement de contrainte de sécurité via HTTP/0.9 |

Aucun secret détecté. Aucune vulnérabilité CRITICAL.

### Image front (`projet_7-front`)

Basée sur `nginx:alpine`. Multi-stage build — le build Node.js n'est pas embarqué dans l'image finale. Aucun utilisateur non-root configuré explicitement.

| Couche | Type | Vulnérabilités |
|--------|------|----------------|
| Alpine 3.23.3 | OS | 1 (HIGH: 1) |
| **Total** | | **1** |

Détail des CVE :

| Bibliothèque | CVE | Sévérité | Version installée | Version corrigée | Description |
|---|---|---|---|---|---|
| `libpng` | CVE-2026-25646 | HIGH | 1.6.54-r0 | 1.6.55-r0 | Heap buffer overflow dans `png_set_quantize` |

Aucun secret détecté. Aucune vulnérabilité CRITICAL. Aucune dépendance applicative exposée (le build Angular n'embarque pas de packages npm dans le runtime Nginx).

### Bilan Trivy

| | Back | Front |
|---|---|---|
| CRITICAL | 0 ✅ | 0 ✅ |
| HIGH | 2 ⚠️ | 1 ⚠️ |
| MEDIUM | 2 | 0 |
| LOW | 1 | 0 |
| Secrets | 0 ✅ | 0 ✅ |

La configuration `exit-code: 1` sur sévérité `CRITICAL` dans le pipeline ne bloque pas le build actuellement, ce qui est cohérent avec les résultats (0 CRITICAL). Les vulnérabilités HIGH sont toutes corrigées dans des versions disponibles d'Alpine 3.23.

---

## Analyse des risques

### Vulnérabilités applicatives

**CORS non restreint (`@CrossOrigin` sans `origins`)** — Risque : Moyen
L'annotation sans paramètre sur `OrganizationRepository` autorise les requêtes cross-origin depuis n'importe quel domaine. En production exposée publiquement, cela ouvre la porte à des attaques CSRF ou à des accès non autorisés depuis des domaines tiers.

**Couverture de tests insuffisante** — Risque : Élevé
Avec 56.4% back et moins de 33% front, une large portion du comportement applicatif n'est pas validée automatiquement. Des régressions peuvent être introduites sans détection, notamment dans `person-details` et `organization-details` (front) et les entités `Person`, `Organization` (back).

**Duplication de code** — Risque : Faible à Moyen
La duplication dans `Organization.java` et `Person.java` augmente le risque d'incohérence lors des corrections de bugs : un correctif appliqué sur un seul endroit dupliqué laisse l'autre vulnérable.

### Vulnérabilités des conteneurs

**`libpng` CVE-2026-25646 (HIGH) — back ET front** — Risque : Moyen
Heap buffer overflow exploitable dans le traitement d'images PNG. Présent dans les deux images. Une version corrigée (`1.6.55-r0`) est disponible — un simple rebuild des images sur une base Alpine à jour suffit à l'éliminer.

**`gnutls` CVE-2026-1584 (HIGH) — back uniquement** — Risque : Moyen
Déni de service distant via un handshake TLS malformé. Exploitable si l'API est exposée directement sans reverse proxy TLS terminant la connexion en amont. Corrigé dans `gnutls 3.8.12-r0`.

**`tomcat-embed-core` CVE-2025-66614 (MEDIUM) — back uniquement** — Risque : Faible à Moyen
Contournement de vérification de certificat client. Pertinent uniquement si l'authentification mutuelle TLS (mTLS) est utilisée. Corrigé dans Spring Boot embarquant Tomcat 10.1.49+.

**Image front sans utilisateur non-root** — Risque : Faible
Le Dockerfile front n'explicite pas d'utilisateur non-root, contrairement au back (`USER 1001`). Nginx tourne par défaut en root pour son processus maître, ce qui peut être durci.

**Seuil Trivy limité à CRITICAL** — Risque : Faible
La variable `vars.SEVERITY` est positionnée sur `CRITICAL` uniquement. Les 3 vulnérabilités HIGH actuellement présentes ne font pas échouer le pipeline, bien qu'elles soient toutes corrigeables.

### Risques liés au pipeline CI/CD

**Security Hotspots non révisés (0%)** — Risque : Moyen
La Quality Gate échoue sur ce critère. L'absence de processus de révision formel signifie que des hotspots de sécurité s'accumulent sans évaluation.

**Secrets GitHub non sauvegardés** — Risque : Élevé
Les secrets (`SONAR_TOKEN`, `SONAR_PROJECT_KEY_BACK`, `SONAR_ORGANIZATION`) ne sont pas récupérables après création dans GitHub. En l'absence d'un gestionnaire de secrets externe, leur perte implique une interruption complète du pipeline.

**Token SonarCloud personnel (PAT)** — Risque : Moyen
Si le compte propriétaire est compromis ou désactivé, l'ensemble des analyses Sonar est interrompu. Un token de service dédié serait plus robuste.

**`npm install` sans lockfile dans les jobs CI** — Risque : Moyen
Les jobs `commitlint` et `release` utilisent `npm install --save-dev` au lieu de `npm ci`, ce qui ne garantit pas des versions reproductibles et expose à des attaques de type dependency confusion.

**Permissions du job `release` trop larges** — Risque : Faible à Moyen
Le job `release` cumule `contents: write`, `packages: write`, `issues: write`, `pull-requests: write` et `id-token: write`. En cas de compromission via une dépendance malveillante, l'attaquant dispose de droits étendus sur le dépôt.

---

## Plan d'action / Remédiation

### Actions immédiates

**1. Rebaser les images Docker sur Alpine à jour**
Les trois CVE de couche OS (`gnutls`, `libpng`) sont corrigées dans des versions disponibles. Un simple rebuild sans cache suffit — aucune modification de code nécessaire :
```bash
docker build --no-cache -t projet_7-back ./back
docker build --no-cache -t projet_7-front ./front
```
Cela élimine d'un coup CVE-2026-1584, CVE-2025-14831 et CVE-2026-25646.

**2. Réviser les 3 Security Hotspots CORS sur SonarCloud**
Ouvrir chaque hotspot et statuer explicitement. Si l'API est exposée publiquement, restreindre `@CrossOrigin` :
```java
// Avant
@CrossOrigin

// Après
@CrossOrigin(origins = "${app.cors.allowed-origins}")
```
Définir la valeur dans `application.properties` et surcharger selon les profils (`dev`, `prod`). Marquer les hotspots comme "Fixed" sur SonarCloud pour débloquer cette condition de la Quality Gate.

**3. Archiver les secrets dans un gestionnaire externe**
Exporter et stocker immédiatement les secrets CI (`SONAR_TOKEN`, `SONAR_PROJECT_KEY_BACK`, `SONAR_ORGANIZATION`) dans un gestionnaire sécurisé (Bitwarden, 1Password, HashiCorp Vault) avant toute rotation ou expiration.

**4. Étendre le seuil Trivy à HIGH**
Modifier la variable `vars.SEVERITY` pour bloquer le pipeline sur les vulnérabilités HIGH corrigeables :
```
SEVERITY: CRITICAL,HIGH
```

---

### Actions à court terme

**5. Mettre à jour `tomcat-embed-core` vers 10.1.49+**
Mettre à jour la version de Spring Boot dans `build.gradle` pour embarquer Tomcat 10.1.49 minimum, ce qui corrige CVE-2025-66614.

**6. Augmenter la couverture de tests back-end (objectif : >= 80%)**
Prioriser `Organization.java` (36.4%) et `Person.java` (45.2%). Ajouter des tests unitaires sur les constructeurs, getters/setters et méthodes métier pour débloquer la condition Coverage de la Quality Gate.

**7. Augmenter la couverture de tests front-end**
Prioriser `person-details` et `organization-details`. Commencer par les branches (9.52% global) qui représentent le risque de régression le plus élevé et le gap le plus important.

**8. Réduire la duplication dans `Organization.java` et `Person.java`**
Extraire les champs et méthodes communs dans une classe abstraite parente (ex. `BaseEntity`) pour passer sous le seuil de 3% de duplication requis par la Quality Gate.

**9. Remplacer `npm install` par `npm ci` dans les jobs CI**
```yaml
# Avant
run: npm install --save-dev @commitlint/cli @commitlint/config-conventional

# Après (avec package.json + lockfile commités)
run: npm ci
```

**10. Ajouter un utilisateur non-root dans le Dockerfile front**
En cohérence avec la bonne pratique déjà appliquée sur le back :
```dockerfile
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
```

**11. Conserver les rapports Trivy comme artefacts CI**
Ajouter un step `upload-artifact` après chaque scan pour disposer d'une traçabilité à chaque build :
```yaml
- name: Scan Docker image back
  uses: aquasecurity/trivy-action@0.33.1
  with:
    image-ref: ...
    severity: ${{ vars.SEVERITY }}
    exit-code: 1
    format: template
    template: "@/contrib/html.tpl"
    output: trivy-report-back.html

- name: Upload rapport Trivy back
  uses: actions/upload-artifact@v4
  if: always()
  with:
    name: trivy-report-back
    path: trivy-report-back.html
```

---

### Actions à long terme

**12. Activer GitHub Dependabot**
Configurer les alertes de sécurité automatiques sur les dépendances npm et Gradle :
```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
  - package-ecosystem: "gradle"
    directory: "/back"
    schedule:
      interval: "weekly"
```

**13. Remplacer le PAT SonarCloud par un token de service d'organisation**
Créer un compte de service dédié sur SonarCloud pour découpler l'authentification CI du compte personnel du développeur et éviter toute interruption en cas de changement de compte.

**14. Intégrer la Quality Gate comme bloquant sur `main`**
Configurer une règle de protection de branche exigeant que la Quality Gate SonarCloud soit verte avant tout merge vers `main`.

**15. Réduire les permissions du job `release`**
Appliquer le principe du moindre privilège : supprimer `issues: write` et `pull-requests: write` si ces permissions ne sont pas strictement nécessaires à semantic-release.

**16. Mettre en place une politique de nettoyage des images GHCR**
Configurer une GitHub Action de nettoyage périodique pour supprimer les images taguées `branch-sha` de plus de 30 jours, tout en préservant les tags `latest` et versionnés (`vX.Y.Z`).

**17. Étendre l'analyse SonarCloud au front-end Angular**
Le front-end n'apparaît pas dans le dashboard SonarCloud actuel. Valider que la configuration `sonar-project.properties` du front est correctement rattachée à un projet SonarCloud distinct et que les résultats sont publiés à chaque run CI.


---

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