# Stack ELK — Guide de mise en place

## Architecture

```
Spring Boot (back)
    │
    │  TCP JSON (port 5000)
    ▼
Logstash  ──────────────▶  Elasticsearch (port 9200)
                                  │
                                  ▼
                            Kibana (port 5601)
```

## Prérequis

- Docker + Docker Compose installés
- ~4 Go de RAM disponibles
- L'application tourne déjà via `docker-compose.yml`

---

## 1. Structure des fichiers

```
projet/
├── docker-compose-elk.yml         ← stack ELK séparée
├── .env.elk                       ← variables (copier en .env si utilisé seul)
├── elk/
│   └── logstash/
│       ├── pipeline/
│       │   └── logstash.conf      ← pipeline de traitement des logs
│       └── config/
│           └── logstash.yml       ← config générale Logstash
└── back/
    └── src/main/resources/
        └── logback-spring.xml     ← configuration logs Spring Boot
```

---

## 2. Configurer Spring Boot

### 2.1 Ajouter la dépendance dans `build.gradle`

```groovy
implementation 'net.logstash.logback:logstash-logback-encoder:7.4'
```

### 2.2 Ajouter le nom de l'application dans `application.properties`

```properties
spring.application.name=projet7-back
```

### 2.3 Configurer les variables Logstash

En local (hors Docker) :
```properties
LOGSTASH_HOST=localhost
LOGSTASH_PORT=5000
```

Si Spring Boot tourne aussi dans Docker sur le même réseau `elk` :
```properties
LOGSTASH_HOST=logstash
LOGSTASH_PORT=5000
```

---

## 3. Démarrer la stack ELK

```bash
cp .env.template .env

docker compose -f docker-compose-elk.yml --env-file .env up -d

docker compose -f docker-compose-elk.yml ps

docker logs logstash -f
```

---

## 4. Configurer Kibana

1. Ouvrir **http://localhost:5601**
2. Aller dans **Management > Stack Management > Index Patterns**
3. Créer un index pattern : `projet7-*`
4. Sélectionner `@timestamp` comme champ de temps
5. Aller dans **Discover** pour explorer les logs en temps réel

### Dashboards mis en place

**Volume de logs par niveau :**
- Visualisation : Bar chart
- Axe X : `@timestamp` (Date Histogram, interval Auto)
- Axe Y : Count
- Split series : `log_level.keyword`

**Compteur erreurs :**
- Visualisation : Metric
- Metric : Count
- Filtre : `tags: error`

**Top erreurs :**
- Visualisation : Data Table
- Metric : Count
- Split rows : `message.keyword` (Top 10)
- Filtre : `tags: error`

**Logs en temps réel :**
- Aller dans **Discover**
- Filtre : `log_level: ERROR OR log_level: WARN`
- Colonnes : `@timestamp`, `application`, `log_level`, `message`

---

## 5. Vérifier que les logs arrivent

```bash
echo '{"@timestamp":"2024-01-15T10:00:00Z","level":"INFO","message":"Test log","application":"projet7-back"}' | nc localhost 5000
```

Puis vérifier dans Kibana > Discover que le document apparaît.

---

## 6. Arrêter la stack

```bash
docker compose -f docker-compose-elk.yml down

# Supprimer aussi les données persistées
docker compose -f docker-compose-elk.yml down -v
```

---

## Points de vigilance

- **Ne pas intégrer ELK dans la CI/CD** — trop lourd, réservé à l'usage local.
- **xpack.security est désactivé** — ne jamais déployer cette config en production.
- **Heap JVM** — ne jamais dépasser 50% de la RAM du conteneur (règle ES/LS).
- **Le profil `test`** désactive l'appender Logstash pour ne pas polluer les tests unitaires.