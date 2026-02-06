# Microservices Observables et Résilients

## Vue d'ensemble

Ce TP construit 2 microservices :

### **pricing-service**
Un petit service qui renvoie un prix. Il peut "tomber en panne" volontairement pour tester la robustesse.

### **book-service**
Un service qui gère des livres (titre, auteur, stock) dans une base MySQL.
Quand on emprunte un livre, book-service :
- décrémente le stock en base
- appelle pricing-service pour récupérer un prix
- si pricing-service est en panne, book-service ne doit pas planter ⇒ il continue avec un fallback.

### Déploiement Docker Compose :
- une base MySQL (avec volume pour garder les données)
- pricing-service
- 3 instances de book-service (comme en production)
- wait strategy pour éviter que book-service démarre avant MySQL

---

## Ce que le lab produit à la fin

| Service | Port Machine |
|---------|--------------|
| pricing-service | 8082 |
| book-service (instance 1) | 8081 |
| book-service (instance 2) | 8083 |
| book-service (instance 3) | 8084 |
| mysql | 3306 |

---

## Ce que l'étudiant apprend

1. **Actuator** : "comment vérifier si un service est vivant et prêt"
2. **Healthcheck Docker** : "comment Docker sait qu'un service est OK"
3. **Profiles Spring** : "comment changer la config selon l'environnement"
4. **MySQL volume** : "comment garder les données après redémarrage"
5. **Wait strategy** : "comment attendre MySQL avant démarrer l'app"
6. **Résilience** : retry + circuit breaker + fallback (Resilience4j)
7. **Multi-instances** : pourquoi synchronized ne marche plus et comment faire avec un verrou DB

---

## Prérequis

- JDK 17 ou 21
- Maven 3.9+
- Docker + Docker Compose (v2 conseillé)

---

## Convention de packages

| Service | Package |
|---------|---------|
| pricing-service | `com.example.pricingservice` |
| book-service | `com.example.bookservice` |

---

## Arborescence finale

```
TP26/
├── book-service/
│   ├── src/
│   ├── pom.xml
│   └── Dockerfile
├── pricing-service/
│   ├── src/
│   ├── pom.xml
│   └── Dockerfile
├── docker-compose.yml
└── README.md
```

---

## Étapes du TP

### Étape 1 — Création de pricing-service

#### 1.1 Démarrer le service
```bash
cd pricing-service
mvn spring-boot:run
```

#### 1.2 Checkpoints
```bash
# Health
curl http://localhost:8082/actuator/health
curl http://localhost:8082/actuator/health/readiness

# Prix
curl http://localhost:8082/api/prices/1
curl "http://localhost:8082/api/prices/1?fail=true"  # erreur forcée
```

---

### Étape 2 — Test book-service en local (profil dev avec H2)

#### 2.1 Démarrer les deux services

**Terminal 1 - pricing-service :**
```bash
cd pricing-service
mvn spring-boot:run
```

**Terminal 2 - book-service (H2) :**
```bash
cd book-service
$env:SPRING_PROFILES_ACTIVE="dev"
mvn spring-boot:run
```

#### 2.2 Checkpoints
```bash
# Health
curl http://localhost:8081/actuator/health

# Créer un livre
curl -X POST http://localhost:8081/api/books -H "Content-Type: application/json" -d "{\"title\":\"Dune\",\"author\":\"Herbert\",\"stock\":2}"

# Emprunter
curl -X POST http://localhost:8081/api/books/1/borrow
```

---

### Étape 3 — Exécution Docker Compose

```bash
cd TP26
docker compose up -d --build
```

#### 3.1 Checkpoints santé
```bash
curl http://localhost:8082/actuator/health
curl http://localhost:8081/actuator/health
curl http://localhost:8083/actuator/health
curl http://localhost:8084/actuator/health
```

#### 3.2 Checkpoint multi-instances
```bash
curl http://localhost:8081/api/debug/instance
curl http://localhost:8083/api/debug/instance
curl http://localhost:8084/api/debug/instance
```
**Attendu :** 3 `instance=<hostname>` différents.

---

### Étape 4 — Scénarios de validation

#### 4.1 Données partagées (même MySQL)

```bash
# Créer un livre via instance 1
curl -X POST http://localhost:8081/api/books -H "Content-Type: application/json" -d "{\"title\":\"Dune\",\"author\":\"Herbert\",\"stock\":3}"

# Lire via instance 2 et 3
curl http://localhost:8083/api/books
curl http://localhost:8084/api/books
```
**Attendu :** même liste (DB commune).

#### 4.2 Stock cohérent (verrou DB)

```bash
curl -X POST http://localhost:8081/api/books/1/borrow
curl -X POST http://localhost:8083/api/books/1/borrow
curl -X POST http://localhost:8084/api/books/1/borrow
curl -X POST http://localhost:8083/api/books/1/borrow
```
**Attendu :**
- 3 succès (stock 3 → 0)
- 4e : HTTP 409 + `{"error":"Plus d'exemplaires"}`

#### 4.3 Résilience : pricing down (fallback)

```bash
# Stop pricing
docker compose stop pricing-service

# Emprunter (doit fonctionner avec fallback price=0.0)
curl -X POST http://localhost:8081/api/books/1/borrow

# Relancer pricing
docker compose start pricing-service
```

#### 4.4 Volume MySQL : persistance après redémarrage

```bash
docker compose down
docker compose up -d
curl http://localhost:8081/api/books
```
**Attendu :** le livre existe encore.

---

## Remarques pédagogiques

| Concept | Explication |
|---------|-------------|
| **Profiles** | `dev` = H2, `docker` = MySQL container |
| **Actuator** | health/readiness/liveness sert à l'infra |
| **Wait strategy** | évite les démarrages "trop tôt" |
| **Multi-instances** | verrou JVM inutile → utiliser verrou DB |
| **Résilience** | fallback empêche une panne externe de casser tout le service |

---

## Arrêter le stack

```bash
docker compose down
```

## Supprimer les données MySQL

```bash
docker compose down -v
```
