# Reverse Proxy & Load Balancing (TP 1 & TP 2)

Dépôt des travaux pratiques sur le Reverse Proxy et le Load Balancing avec **Traefik**, **Nginx** et **Docker**, basé sur le cours Reverse Proxy (Mohammad Jabbari).

```text
.
├── README.md
├── traefik-tp/               # TP 1 : Traefik & Docker
│   ├── docker-compose.yml
│   ├── config/
│   │   └── traefik.yaml
│   ├── app1/
│   │   └── index.html
│   ├── app1b/
│   │   └── index.html
│   └── app2/
│       └── index.html
└── tp2-load-balancer/        # TP 2 : Nginx Load Balancer
    ├── docker-compose.yml
    └── nginx/
        └── nginx.conf
```

---

# TP 2 — Mise en place d’un Load Balancer avec Nginx et Docker

## 1. Architecture

```text
               Client (curl / navigateur)
                          │
                   Port 8080 hôte
                          │
                          ▼
            ┌───────────────────────────┐
            │   Nginx (Load Balancer)   │
            └─────────────┬─────────────┘
       ┌──────────────────┼──────────────────┐
       ▼                  ▼                  ▼
┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│  backend1    │   │  backend2    │   │  backend3    │
│  (port 5678) │   │  (port 5678) │   │  (port 5678) │
└──────────────┘   └──────────────┘   └──────────────┘
```

* Le client envoie toutes ses requêtes à un point d'entrée unique (`http://localhost:8080`).
* Nginx distribue les requêtes vers les 3 backends `http-echo` sur le réseau privé créé par Docker Compose.
* Les backends ne publient aucun port sur la machine hôte.

---

## 2. Démarrage rapide (KISS)

### Étape 1 : Lancer l'environnement
```bash
cd tp2-load-balancer
docker compose up -d
```

### Étape 2 : Tester la répartition Round Robin
```bash
for i in $(seq 1 9); do curl -s http://localhost:8080; done
```
*Résultat :* distribution équitable entre `backend 1`, `backend 2` et `backend 3`.

### Étape 3 : Arrêt des conteneurs
```bash
docker compose down
```

---

## 3. Synthèse des algorithmes de Load Balancing testés

| Algorithme | Directive Nginx | Critère principal | Disponible en Nginx Open Source |
| :--- | :--- | :--- | :--- |
| **Round Robin** | *(défaut)* | Alternance cyclique entre les serveurs | Oui |
| **Weighted Round Robin** | `weight=X` | Alternance proportionnelle aux capacités | Oui |
| **Least Connections** | `least_conn;` | Serveur ayant le moins de connexions actives | Oui |
| **Weighted Least Connections** | `least_conn;` + `weight=X` | Ratio connexions actives / poids | Oui |
| **IP Hash** | `ip_hash;` | Même adresse IP client vers le même backend | Oui |
| **Random** | `random;` | Tirage aléatoire indépendant à chaque requête | Oui |
| **Least Response Time** | `least_time header;` | Temps de réponse et nombre de connexions | **Non** (Nginx Plus uniquement) |
| **Sticky Cookie** | `sticky cookie ...` | Persistance de session par cookie applicatif | **Non** (Nginx Plus uniquement) |
| **Rate Limiting** | `limit_req_zone` + `limit_req` | Contrôle et limitation du débit de requêtes | Oui |

---

## 4. Réponses aux Questions du TP 2 (Étape 17 — À rendre)

1. **Quelle est la différence entre un reverse proxy et un Load Balancer ?**  
   - Le **reverse proxy** est un intermédiaire général qui s'interpose devant les serveurs pour centraliser l'accès, terminer le TLS, gérer la sécurité (WAF, headers) ou router selon le domaine/chemin.  
   - Le **Load Balancer** a pour objectif spécifique de distribuer la charge entre plusieurs instances exécutant le même service afin d'éviter la surcharge et d'optimiser les performances. Nginx et Traefik cumulent les deux rôles.

2. **Quel est le rôle du bloc `upstream` dans Nginx ?**  
   Il définit un pool logique de serveurs backend avec leurs adresses/ports internes et permet de spécifier l'algorithme de répartition de charge ainsi que les options associées (`weight`, `backup`, `max_fails`, etc.).

3. **Pourquoi les backends peuvent-ils communiquer avec Nginx sans publier leurs ports sur la machine ?**  
   Parce que Docker Compose crée un réseau virtuel partagé (bridge). À l'intérieur de ce réseau, Nginx et les conteneurs backend communiquent directement via leurs adresses IP privées et leurs noms de service DNS internes (`backend1:5678`). Les ports n'ont pas besoin d'être exposés sur la machine hôte.

4. **Quelle est la différence entre Round Robin et Weighted Round Robin ?**  
   - *Round Robin* traite chaque serveur de façon équivalente et distribue les requêtes de manière égale en cycle (1-2-3-1-2-3).  
   - *Weighted Round Robin* attribue un poids (`weight=N`) à chaque serveur pour refléter une différence de capacité matérielle (CPU/RAM). Un serveur de poids 3 recevra 3 fois plus de requêtes qu'un serveur de poids 1.

5. **Dans quel cas l’algorithme `least_conn` est-il pertinent ?**  
   Lorsque les requêtes ont des temps d'exécution très inégaux (ex: téléchargements lourds, calculs complexes, websockets). Le Round Robin risquerait de surcharger un serveur déjà occupé par des requêtes longues, alors que `least_conn` achemine les nouveaux flux vers le serveur le plus disponible.

6. **Quel est l’objectif d’`ip_hash` ?**  
   Assurer une persistance de session basique en garantissant qu'un client (identifié par son adresse IP) soit toujours dirigé vers la même instance backend, évitant de perdre des données de session stockées localement en mémoire.

7. **Que se passe-t-il lorsqu’on arrête le backend 2 ?**  
   Lors de la première tentative vers le backend arrêté, la connexion échoue ou timeout. Nginx détecte passivement l'erreur (`max_fails` atteint), bascule la requête sur un autre serveur valide (`backend 1` ou `backend 3`), et marque temporairement `backend 2` comme indisponible pendant `fail_timeout` (10s par défaut).

8. **Pourquoi un Load Balancer ne suffit-il pas à garantir une haute disponibilité complète ?**  
   - Le Load Balancer lui-même constitue un point unique de défaillance (**SPOF**) s'il n'est pas lui-même redondé (ex: VRRP / Keepalived ou DNS Anycast).  
   - Si tous les backends plantent ou si la base de données centrale tombe, le load balancer ne peut plus rien servir.  
   - Sans contrôles de santé actifs (*active health checks*), le client peut subir des lenteurs ou erreurs temporaires lors d'une panne.

9. **Quelle est la différence entre Load Balancing et Session Persistence ?**  
   - Le **Load Balancing** vise à disperser le trafic uniformément entre les serveurs.  
   - La **Session Persistence** contraint au contraire un même utilisateur à rester lié au même serveur pour maintenir son état applicatif.

10. **Pourquoi `ip_hash` peut-il être moins fiable lorsque plusieurs utilisateurs partagent la même adresse IP publique ?**  
    Derrière un NAT d'entreprise, un proxy d'école ou un réseau 4G/5G mobile, des centaines d'utilisateurs distincts ont la même adresse IP publique. Avec `ip_hash`, tout ce groupe sera dirigé vers un seul et même serveur, créant un fort déséquilibre de charge.

11. **Pourquoi une session stockée uniquement en mémoire peut-elle poser problème si le backend tombe en panne ?**  
    Si le backend stocke la session en RAM et s'arrête brutalement, la session est irrémédiablement perdue (panier vidé, utilisateur déconnecté). En production, on utilise un magasin de sessions externe partagé (ex: Redis, Memcached ou base de données).

12. **Quelle est la différence entre la persistance par IP et la persistance par cookie ?**  
    - *Persistance par IP (`ip_hash`)* : Basée sur l'adresse IP réseau de la requête. Problématique en cas de NAT partagé ou d'IP dynamique.  
    - *Persistance par cookie (`sticky cookie`)* : Basée sur un cookie HTTP injecté dans le navigateur. Beaucoup plus fine et précise, car elle identifie réellement le client navigateur même si plusieurs personnes partagent la même adresse IP.

---

## 5. Réponses aux Questions Intermédiaires du Cours

* **Étape 11 (Random) : Pourquoi Random ne garantit-il pas une répartition égale sur 10 requêtes ?**  
  Parce que chaque tirage est indépendant au sens probabiliste. Sur un faible échantillon (10 requêtes), la variance est élevée. La convergence vers l'équiprobabilité (1/3) n'apparaît que sur la loi des grands nombres (des centaines ou milliers de requêtes).
* **Étape 12 (Weighted Least Connections) : Pourquoi un serveur à poids élevé peut-il être moins sollicité s'il a beaucoup de connexions ?**  
  L'algorithme compare le ratio `connexions_actives / poids`. Si le serveur de poids 3 a déjà 6 connexions (ratio 2) alors que le serveur de poids 1 a 0 connexion (ratio 0), la nouvelle requête ira vers le serveur à 0 connexion malgré son poids inférieur.
* **Étape 13.3 (Least Response Time) :**  
  1. *Pourquoi le nombre de connexions ne suffit pas ?* Deux serveurs peuvent avoir 10 connexions chacun, mais un serveur peut être sur un CPU lent et répondre en 800ms tandis que l'autre répond en 50ms.  
  2. *Least Connections vs Least Response Time :* Least Conn s'intéresse au nombre de requêtes en cours ; Least Response Time mesure le temps réel nécessaire pour répondre (latence).  
  3. *Pourquoi `least_time` ne fonctionne pas en Nginx Open Source ?* Il s'agit d'un module propriétaire exclusif à Nginx Plus.
* **Étape 14.4 (Rate Limiting) :**  
  1. *Différence entre `rate=5r/s` et `burst=10` :* `rate=5r/s` impose le rythme moyen soutenable (1 requête toutes les 200ms). Le `burst=10` autorise un pic temporaire de 10 requêtes simultanées au-delà de ce rythme.  
  2. *Pourquoi le Rate Limiting protège une API ?* Il protège les backends contre les attaques DoS/brute-force, le scraping abusif et les boucles infinies de clients défaillants.  
  3. *Pourquoi la limite par IP peut être insuffisante ?* En cas de NAT, tous les utilisateurs légitimes d'une même entreprise partagent le même quota et peuvent être bloqués collectivement. Une limitation par clé d'API (API Key) ou jeton JWT est souvent préférable.

---

## 6. Exercice Bonus : Backends principaux et serveur de secours (`backup`)

Dans Nginx, le mot-clé `backup` marque un serveur qui ne recevra aucune requête tant qu'au moins l'un des serveurs principaux est opérationnel.

Configuration dans `nginx.conf` :
```nginx
upstream backend_pool {
    server backend1:5678;
    server backend2:5678;
    server backend3:5678 backup; # Secours uniquement
}
```
*Comportement vérifié :*
- En fonctionnement normal : seules les instances `backend1` et `backend2` reçoivent le trafic (50/50).
- Si `backend1` et `backend2` tombent en panne : Nginx bascule automatiquement 100% des requêtes sur `backend3`.

---

# TP 1 — Reverse Proxy et Load Balancing avec Traefik et Docker

*(Voir section précédente ci-dessous pour l'historique complet du TP 1)*

## Démarrage rapide TP 1
```bash
docker network create frontend  # si pas déjà créé
cd traefik-tp
docker compose up -d
for i in $(seq 1 4); do curl -s http://app1.localhost; done
curl -s http://app2.localhost
```
* Dashboard Traefik : [http://localhost:8080/dashboard/](http://localhost:8080/dashboard/)
* App1 (Load balanced entre app1 et app1b) : `http://app1.localhost`
* App2 : `http://app2.localhost`
