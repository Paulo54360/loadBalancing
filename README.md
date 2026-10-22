# TP 1 — Reverse Proxy et Load Balancing avec Traefik et Docker

Ce projet implémente le TP complet de mise en place d'un reverse proxy et d'un répartiteur de charge (Load Balancer) avec **Traefik** et **Docker**, basé sur le cours Reverse Proxy (Mohammad Jabbari).

---

## 1. Structure du projet

```text
.
├── README.md
└── traefik-tp/
    ├── docker-compose.yml
    ├── config/
    │   └── traefik.yaml
    ├── app1/
    │   └── index.html
    ├── app1b/
    │   └── index.html
    └── app2/
        └── index.html
```

---

## 2. Démarrage rapide (KISS)

### Étape 1 : Créer le réseau Docker partagé (si non existant)
```bash
docker network create frontend
```

### Étape 2 : Lancer l'environnement
```bash
cd traefik-tp
docker compose up -d
```

### Étape 3 : Tester le routage et le Load Balancing
```bash
# Test du routage vers App1 (avec alternance Load Balancing entre App1 et App1b)
for i in $(seq 1 4); do curl -s http://app1.localhost; done

# Test du routage vers App2
curl -s http://app2.localhost

# Dashboard Traefik
open http://localhost:8080/dashboard/
```

---

## 3. Réponses aux Exercices Pratiques (Section 18)

### Exercice 1 — Déploiement initial
1. **Création du réseau :** `docker network create frontend`.
2. **Déploiement de Traefik :** Configuré dans `docker-compose.yml` avec l'entrypoint `web` (:80), `websecure` (:443) et le port d'administration `8080` lié à `127.0.0.1`.
3. **Accès au Dashboard :** Accessible sur `http://localhost:8080/dashboard/`.
4. **Ports exposés :** Seul Traefik expose des ports sur l'hôte (`80`, `443`, `127.0.0.1:8080`). Les conteneurs Nginx restent isolés sur le réseau Docker `frontend`.
5. **Différence entre un EntryPoint et un Router :**
   - **EntryPoint :** Définit la porte d'entrée réseau (port d'écoute et protocole, ex: port 80 pour HTTP).
   - **Router :** Règle d'aiguillage qui analyse la requête arrivant sur un entrypoint (ex: en-tête `Host: app1.localhost`) et décide vers quel service la transmettre.

---

### Exercice 2 — Routage par nom d'hôte
1. **Déploiement :** `app1` et `app2` démarrent sur le réseau `frontend`.
2. **Labels utilisés :**
   - `traefik.enable=true` : autorise explicitement Traefik à prendre en compte le conteneur.
   - `traefik.http.routers.<app>.rule=Host('<app>.localhost')` : route selon le nom d'hôte HTTP.
   - `traefik.http.routers.<app>.entrypoints=web` : écoute sur le port 80.
   - `traefik.http.services.<app>.loadbalancer.server.port=80` : indique le port interne du conteneur Nginx.
3. **Tests :**
   - `curl http://app1.localhost` -> Réponse de App1.
   - `curl http://app2.localhost` -> Réponse de App2.
4. **Observation :** Les routers `app1@docker` et `app2@docker` ainsi que leurs services associés apparaissent actifs dans le dashboard Traefik.

---

### Exercice 3 — Load Balancing
1. **Ajout de l'instance B :** Conteneur `app1b` ajouté avec les mêmes labels de router et de service (`app1`) que `app1`.
2. **Contenu distinct :**
   - `app1` -> `<h1>App1 - Bonjour depuis App1</h1>`
   - `app1b` -> `<h1>Bonjour depuis App1 - Instance B</h1>`
3. **Vérification Dashboard / API :** Le service `app1@docker` regroupe désormais 2 serveurs (`http://172.26.0.2:80` et `http://172.26.0.4:80`).
4. **Test en boucle :**
   ```bash
   for i in $(seq 1 6); do curl -s http://app1.localhost; done
   ```
   *Résultat :* alternance Round-Robin (50% instance A, 50% instance B).
5. **Différence entre Routage et Load Balancing :**
   - Le **routage** sélectionne l'application destinatrice en fonction de règles (ex: `Host`, `PathPrefix`).
   - Le **Load Balancing** distribue le trafic entre plusieurs instances identiques d'un même service applicatif pour absorber la charge et améliorer la disponibilité.

---

### Exercice 4 — Sécurité
1. **Risque de `api.insecure: true` :** Expose l'API d'administration et le dashboard sans mot de passe sur le port 8080. N'importe qui sur le réseau peut analyser les routes, les backends et l'infrastructure interne.
2. **Sensibilité du socket Docker (`/var/run/docker.sock`) :** Donner l'accès au socket Docker équivaut à donner les droits root sur l'hôte. Un attaquant qui compromettrait Traefik pourrait piloter le démon Docker et s'échapper du conteneur.
3. **Approche pour protéger le dashboard :**
   - Désactiver `insecure: true`.
   - Exposer le dashboard via un router sécurisé HTTPS avec un middleware d'authentification (`BasicAuth` ou OAuth/ForwardAuth) et une restriction d'adresses IP (`ipWhiteList`).
4. **Intérêt de `traefik.enable=true` (`exposedByDefault: false`) :** Principe du moindre privilège : évite d'exposer publiquement par erreur des conteneurs internes ou des bases de données.
5. **TLS activé vs Certificat valide :**
   - *TLS activé* : le flux est chiffré, mais le certificat peut être auto-signé (alerte de sécurité dans le navigateur).
   - *Certificat valide* : certificat signé par une autorité reconnue (ex: Let's Encrypt), vérifié pour le nom de domaine et sans avertissement pour l'utilisateur.

---

### Exercice 5 — Évolution de l'architecture

Pour répondre aux besoins cibles (3 applications, 2 backends pour l'application principale, HTTPS, middleware d'auth, health check et dashboard sécurisé) :

#### Schéma d'architecture
```text
                  Clients (Internet)
                          │
                   Ports :80 / :443
                          │
                          ▼
                 ┌──────────────────┐
                 │  Traefik Proxy   │ ◄─── Dashboard (Protégé par BasicAuth)
                 └────────┬─────────┘
        ┌─────────────────┼─────────────────┐
        ▼                 ▼                 ▼
 ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
 │ Application1 │  │ Application2 │  │ Application3 │
 │ (2 instances │  │  (1 instance)│  │  (1 instance)│
 │ Load Balanced│  └──────────────┘  └──────────────┘
 │ + HealthCheck│
 └──────────────┘
```

#### Configuration des labels d'évolution (Exemple app1 sécurisée) :
```yaml
labels:
  - "traefik.enable=true"
  # Route HTTPS
  - "traefik.http.routers.app1.rule=Host(`app1.example.com`)"
  - "traefik.http.routers.app1.entrypoints=websecure"
  - "traefik.http.routers.app1.tls=true"
  - "traefik.http.routers.app1.tls.certresolver=letsencrypt"
  # Redirection HTTP vers HTTPS automatique
  - "traefik.http.routers.app1-http.rule=Host(`app1.example.com`)"
  - "traefik.http.routers.app1-http.entrypoints=web"
  - "traefik.http.routers.app1-http.middlewares=redirect-to-https"
  - "traefik.http.middlewares.redirect-to-https.redirectscheme.scheme=https"
  # Health Check du service
  - "traefik.http.services.app1.loadbalancer.server.port=80"
  - "traefik.http.services.app1.loadbalancer.healthcheck.path=/"
  - "traefik.http.services.app1.loadbalancer.healthcheck.interval=10s"
```

---

## 4. Réponses aux 15 Questions de Compréhension (Section 19)

1. **Qu’est-ce qu’un reverse proxy ?**  
   C'est un serveur intermédiaire placé devant les serveurs web/applicatifs qui reçoit les requêtes des clients et les achemine vers le bon backend.

2. **Quelle est la différence entre un reverse proxy et un Load Balancer ?**  
   Le reverse proxy gère le point d'entrée, le routage logique, la terminaison TLS et la sécurité. Le Load Balancer répartit la charge entre plusieurs serveurs d'une même application. Traefik cumule les deux rôles.

3. **Quel est le rôle de Traefik dans une architecture Docker ?**  
   Il agit comme reverse proxy dynamique : il écoute le démon Docker pour détecter automatiquement l'apparition et la disparition des conteneurs et met à jour ses règles de routage sans redémarrage.

4. **Quelle est la différence entre la configuration statique et dynamique ?**  
   - *Statique* : définie au démarrage de Traefik (entrypoints, providers, logs).
   - *Dynamique* : règles de routage, services et middlewares découverts en continu via les providers (labels Docker).

5. **À quoi sert un provider ?**  
   C'est une source de données (Docker, Kubernetes, fichiers) permettant à Traefik de découvrir les services et de générer sa configuration dynamique.

6. **Quel est le rôle d’un entrypoint ?**  
   Il définit un port d'écoute réseau sur Traefik (ex: `:80` pour HTTP, `:443` pour HTTPS).

7. **Quelle est la différence entre un router et un service ?**  
   Le router décide *où* envoyer une requête selon des règles (`Host`, `Path`) ; le service définit la *destination* réelle (adresses IP et ports des conteneurs) et la façon de répartir la charge.

8. **Pourquoi utilise-t-on des labels Docker ?**  
   Pour attacher directement les règles Traefik aux conteneurs au sein du fichier `docker-compose.yml`, éliminant ainsi le besoin de fichiers de configuration manuels et statiques.

9. **Quel est l’intérêt de `exposedByDefault: false` ?**  
   Garantir la sécurité en n'exposant que les conteneurs explicitement annotés avec `traefik.enable=true`.

10. **Comment Traefik peut-il découvrir automatiquement un nouveau conteneur ?**  
    En écoutant le flux d'événements Docker via `/var/run/docker.sock`.

11. **Quelle est la différence entre routage et Load Balancing ?**  
    Le routage choisit l'application destinataire (ex: App1 vs App2). Le Load Balancing choisit l'instance au sein de cette application (ex: App1-Instance A vs App1-Instance B).

12. **Pourquoi faut-il surveiller la santé des backends ?**  
    Pour éviter d'envoyer des requêtes vers un conteneur en panne ou en surcharge, assurant une disponibilité continue aux utilisateurs.

13. **Pourquoi le dashboard doit-il être protégé ?**  
    Parce qu'il expose la structure interne, les routes, les adresses réseau privées et l'état des services, ce qui constitue une fuite d'informations critique.

14. **Pourquoi un router TLS ne garantit-il pas à lui seul un certificat valide ?**  
    `tls=true` ordonne seulement à Traefik de négocier du TLS (souvent avec un certificat auto-signé). Un certificat valide exige une autorité de certification de confiance (ex: Let's Encrypt).

15. **Comment ajouter une nouvelle instance backend sans changer le point d’entrée client ?**  
    En démarrant un nouveau conteneur portant le même nom de service Traefik (`traefik.http.services.<nom>...`). Traefik l'ajoute automatiquement au pool du load balancer sans interrompre le trafic ni modifier l'URL publique.
