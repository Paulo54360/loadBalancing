# Reverse Proxy, Load Balancing & API Gateway (TP 1, TP 2, TP 3)

Dépôt complet des travaux pratiques sur les architectures de services, le reverse proxy, le load balancing et l'API Gateway avec **Traefik**, **Nginx**, **FastAPI**, **Docker Compose** et **GitLab CI/CD**.

```text
.
├── README.md
├── traefik-tp/               # TP 1 : Reverse Proxy & Load Balancing avec Traefik
│   ├── docker-compose.yml
│   ├── config/
│   │   └── traefik.yaml
│   ├── app1/
│   ├── app1b/
│   └── app2/
├── tp2-load-balancer/        # TP 2 : Load Balancer avec Nginx & algorithmes
│   ├── docker-compose.yml
│   └── nginx/
│       └── nginx.conf
└── eshop-api/                # TP 3 : API Gateway, BFF & Microservices sécurisés
    ├── users/
    │   ├── main.py
    │   ├── requirements.txt
    │   └── Dockerfile
    ├── orders/
    │   ├── main.py
    │   ├── requirements.txt
    │   └── Dockerfile
    ├── gateway/
    │   ├── main.py
    │   ├── requirements.txt
    │   └── Dockerfile
    ├── tests/
    │   └── test_gateway.py
    ├── docker-compose.yml
    ├── .dockerignore
    ├── .gitignore
    └── .gitlab-ci.yml
```

---

# TP 3 — API Gateway et microservices sécurisés avec Docker Compose et GitLab

## 1. Contexte & Architecture E-Shop

Ce TP implémente une architecture de microservices pour la boutique en ligne **E-Shop** :
* **Users Service** : microservice FastAPI gérant les utilisateurs (port interne `8000`).
* **Orders Service** : microservice FastAPI gérant les commandes (port interne `8000`).
* **API Gateway** : point d'entrée unique exposé sur `http://127.0.0.1:3000` assurant :
  * Le routage et la validation des requêtes.
  * L'authentification par jeton Bearer (`GATEWAY_TOKEN`).
  * L'agrégation de données selon le pattern **BFF (Backend For Frontend)** sur `/profile/{id}` avec exécution asynchrone concurrente (`asyncio.gather`).
  * La protection DevSecOps (timeout, masquage des erreurs internes, conteneurs durcis non-root).

```text
                       Client web / mobile
                                │
                 Appels HTTP vers localhost:3000
                                │
                                ▼
              ┌───────────────────────────────────┐
              │       API Gateway (FastAPI)       │
              │  Port publié : 127.0.0.1:3000     │
              │  Auth Bearer · BFF · Health check │
              └───────────────┬───────────────────┘
                              │ Réseau privé Docker
                ┌─────────────┴─────────────┐
                ▼                           ▼
      ┌──────────────────┐        ┌──────────────────┐
      │   Users Service  │        │  Orders Service  │
      │   Port 8000      │        │  Port 8000       │
      │   (non publié)   │        │  (non publié)    │
      └──────────────────┘        └──────────────────┘
```

---

## 2. Démarrage rapide (KISS)

### Étape 1 : Créer le fichier d'environnement `.env`
Générer un jeton secret et l'inscrire dans `eshop-api/.env` (ignoré par Git) :
```bash
cd eshop-api
echo "GATEWAY_TOKEN=$(openssl rand -hex 32)" > .env
```

### Étape 2 : Lancer l'architecture durcie
```bash
docker compose up --build -d
```
Les conteneurs démarrent avec des healthchecks. Compose attend la santé de `users` et `orders` avant de lancer la `gateway`.

### Étape 3 : Tester les endpoints
```bash
set -a && source .env && set +a

# 1. Health check public (sans authentification)
curl -s http://localhost:3000/health
# Réponse : {"status":"ok"}

# 2. Test refus sans token (401 Unauthorized)
curl -s -o /dev/null -w "Code HTTP: %{http_code}\n" http://localhost:3000/users
# Réponse : Code HTTP: 401

# 3. Récupérer les utilisateurs
curl -s -H "Authorization: Bearer $GATEWAY_TOKEN" http://localhost:3000/users

# 4. Récupérer les commandes
curl -s -H "Authorization: Bearer $GATEWAY_TOKEN" http://localhost:3000/orders

# 5. Tester l'agrégation BFF (/profile/1)
curl -s -H "Authorization: Bearer $GATEWAY_TOKEN" http://localhost:3000/profile/1
```

*Sortie de l'agrégation BFF `/profile/1` :*
```json
{
  "user": {
    "id": 1,
    "name": "Alice",
    "email": "alice@test.com"
  },
  "orders": [
    {"id": 101, "userId": 1, "item": "Laptop"},
    {"id": 103, "userId": 1, "item": "Keyboard"}
  ]
}
```

---

## 3. Tests automatisés pytest

Les tests unitaires mockent les appels HTTP via `monkeypatch` pour valider le comportement de la Gateway en isolation.

```bash
docker run --rm -v "$(pwd):/app" -w /app python:3.11-slim sh -c \
  "pip install -q -r gateway/requirements.txt pytest && export PYTHONPATH=. && export GATEWAY_TOKEN=ci-test-token && pytest -v"
```

*Résultat des 5 tests validés :*
```text
tests/test_gateway.py::test_health_is_public PASSED                      [ 20%]
tests/test_gateway.py::test_users_requires_authentication PASSED         [ 40%]
tests/test_gateway.py::test_users_with_invalid_token PASSED              [ 60%]
tests/test_gateway.py::test_profile_aggregates_user_and_orders PASSED    [ 80%]
tests/test_gateway.py::test_profile_unknown_user PASSED                  [100%]
========================= 5 passed in 0.23s =========================
```

---

## 4. Pipeline GitLab CI/CD (`.gitlab-ci.yml`)

Le fichier `.gitlab-ci.yml` automatise la chaîne DevSecOps :
1. **Stage `test`** : exécute `pytest` avec une image `python:3.11-slim` et un token fictif de test.
2. **Stage `build`** : utilise Docker-in-Docker (`dind`) pour construire les 3 images (`users`, `orders`, `gateway`) et les publier sur le GitLab Container Registry taguées avec `$CI_COMMIT_SHA`.

```yaml
stages:
  - test
  - build

test:
  stage: test
  image: python:3.11-slim
  variables:
    GATEWAY_TOKEN: "ci-test-token"
  script:
    - pip install --no-cache-dir -r gateway/requirements.txt pytest
    - pytest -q

build:
  stage: build
  image: docker:24.0.5
  services:
    - docker:24.0.5-dind
  variables:
    DOCKER_TLS_CERTDIR: ""
  before_script:
    - echo "$CI_REGISTRY_PASSWORD" | docker login -u "$CI_REGISTRY_USER" --password-stdin "$CI_REGISTRY"
  script:
    - docker build -t "$CI_REGISTRY_IMAGE/users:$CI_COMMIT_SHA" ./users
    - docker build -t "$CI_REGISTRY_IMAGE/orders:$CI_COMMIT_SHA" ./orders
    - docker build -t "$CI_REGISTRY_IMAGE/gateway:$CI_COMMIT_SHA" ./gateway
    - docker push "$CI_REGISTRY_IMAGE/users:$CI_COMMIT_SHA"
    - docker push "$CI_REGISTRY_IMAGE/orders:$CI_COMMIT_SHA"
    - docker push "$CI_REGISTRY_IMAGE/gateway:$CI_COMMIT_SHA"
```

---

## 5. Checklist de sécurité DevSecOps (10/10)

| Règle de sécurité | Implémentation | Statut |
| :--- | :--- | :---: |
| **Exclusion Git du secret** | `.env` exclu dans `.gitignore` et `.dockerignore` | ✅ |
| **Jeton Bearer obligatoire** | Middleware `require_token` avec `secrets.compare_digest` | ✅ |
| **Pas de publication des backends** | `users` et `orders` n'exposent aucun port sur l'hôte | ✅ |
| **Réseau privé dédié** | Réseau interne `backend_net` pour isoler les conteneurs | ✅ |
| **Utilisateur non-root** | `RUN useradd -u 1000 appuser && USER appuser` dans les Dockerfiles | ✅ |
| **Timeouts configurés** | `timeout=5.0` sur `httpx.AsyncClient` contre le blocage réseau | ✅ |
| **Masquage des erreurs internes** | Erreurs HTTP 502 génériques sans trace technique renvoyée au client | ✅ |
| **Variables Registry sécurisées** | `$CI_REGISTRY_USER` et `$CI_REGISTRY_PASSWORD` sans mot de passe en dur | ✅ |
| **Traçabilité des images** | Tags d'images liés au `$CI_COMMIT_SHA` | ✅ |
| **Conteneurs durcis** | `read_only: true`, `cap_drop: ALL`, `no-new-privileges: true` | ✅ |

---

## 6. Réponses aux 4 Questions de Compréhension (Étape 18)

### Question 1 — Quels avantages l’API Gateway apporte-t-elle ?
* **Point d'entrée unique** : Simplifie l'architecture pour les clients qui n'ont qu'une seule adresse à contacter.
* **Centralisation de la sécurité** : Mutualise l'authentification (jeton Bearer), la validation des requêtes et le rate limiting sans devoir les réimplémenter dans chaque microservice.
* **Agrégation BFF (Backend For Frontend)** : Regroupe les données de plusieurs microservices (ex: profil utilisateur + commandes) en un seul appel, réduisant considérablement la latence et les allers-retours réseau (round-trips) côté client.
* **Découplage** : Masque la topologie interne, permettant de refactorer, découper ou renommer les microservices sans impacter les clients externes.

### Question 2 — Quels sont les inconvénients d’une API Gateway ?
* **Point unique de défaillance (SPOF)** : Si la Gateway est indisponible et n'est pas répliquée, l'ensemble des services devient inaccessible aux clients.
* **Latence supplémentaire** : Chaque requête traverse un saut réseau additionnel pour l'analyse, le déchiffrement et le routage.
* **Risque de complexité (monolithe masqué)** : Si trop de règles métier ou de transformations y sont intégrées, la Gateway devient complexe à maintenir et ralentit les déploiements.
* **Goulot d'étranglement** : Elle doit être dimensionnée et supervisée avec attention pour supporter la charge cumulée de toutes les applications.

### Question 3 — Comment gérer l’authentification et la sécurité ?
* **Au niveau de la Gateway** : Valider les jetons d'accès (Bearer token, JWT court ou OAuth2/OIDC), appliquer HTTPS obligatoirement, limiter le débit (Rate Limiting) et configurer des timeouts sur les appels aval.
* **Au niveau de l'infrastructure** : Isoler les microservices sur un réseau Docker privé sans aucun port exposé sur l'hôte, et faire tourner les conteneurs avec un utilisateur non-root et un système de fichiers en lecture seule.
* **Au niveau des microservices** : Ne pas se reposer uniquement sur la Gateway ; vérifier les autorisations fines d'accès aux données (RBAC) pour s'assurer qu'un utilisateur n'accède qu'à ses propres ressources.

### Question 4 — Quelle différence entre une API Gateway et un reverse proxy NGINX ?
* **Reverse Proxy NGINX** : Agit principalement aux couches 4 et 7 pour le routage d'URL, la terminaison TLS, la mise en cache statique de haute performance et la répartition de charge (Round Robin, IP Hash).
* **API Gateway** : Orientée couche applicative pour les APIs. Elle offre des fonctionnalités plus avancées comme l'authentification applicative fine (validation de jetons, OAuth2), la manipulation de payloads JSON, et l'agrégation de données de multiples backends (pattern BFF).  
*(Note : NGINX peut être configuré ou étendu pour assumer certaines fonctions de Gateway, mais une solution dédiée comme FastAPI Gateway apporte une flexibilité complète pour la logique d'agrégation asynchrone).*

---

# TP 2 — Mise en place d’un Load Balancer avec Nginx et Docker

## Architecture & Démarrage
```bash
cd tp2-load-balancer
docker compose up -d
for i in $(seq 1 9); do curl -s http://localhost:8080; done
```
* **Backends** : 3 instances `http-echo` réparties en Round Robin via Nginx (`:8080`).
* **Algorithmes étudiés** : Round Robin, Weighted (`weight=X`), Least Connections (`least_conn;`), IP Hash (`ip_hash;`), Random (`random;`), Least Response Time (Nginx Plus), et serveur de secours (`backup`).

---

# TP 1 — Reverse Proxy et Load Balancing avec Traefik et Docker

## Architecture & Démarrage
```bash
docker network create frontend
cd traefik-tp
docker compose up -d
curl http://app1.localhost
curl http://app2.localhost
```
* **Dashboard Traefik** : [http://localhost:8080/dashboard/](http://localhost:8080/dashboard/)
* **Découverte automatique Docker** via `/var/run/docker.sock` et labels.
* **Load Balancing** entre `app1` et `app1b`.
