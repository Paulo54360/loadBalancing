# Compte-Rendu TP4 — Networking Kubernetes : Pods, Services, NetworkPolicy, Ingress et Gateway API

**Auteur :** Paulo  
**Environnement :** Kind, Kubernetes v1.37.0, Envoy Gateway v1.9.2, Helm v3.14.4, Docker Desktop  
**Date :** 23 Octobre 2026  

---

## Sommaire
1. [Préparation de l'environnement](#1-préparation-de-lenvironnement)
2. [Partie 1 — Déployer les Pods et observer leurs IP](#2-partie-1--déployer-les-pods-et-observer-leurs-ip)
3. [Partie 2 — Créer un Service ClusterIP](#3-partie-2--créer-un-service-clusterip)
4. [Partie 3 — Contrôler les flux avec NetworkPolicy](#4-partie-3--contrôler-les-flux-avec-networkpolicy)
5. [Partie 4 — Comprendre Ingress](#5-partie-4--comprendre-ingress)
6. [Partie 5 — Installer Envoy Gateway et utiliser Gateway API](#6-partie-5--installer-envoy-gateway-et-utiliser-gateway-api)
7. [Partie 6 — Défis pratiques](#7-partie-6--défis-pratiques)
   - [Défi 1 — Ajouter un second Service](#défi-1--ajouter-un-second-service)
   - [Défi 2 — Modifier le routage HTTP](#défi-2--modifier-le-routage-http)
   - [Défi 3 — Réactiver la sécurité réseau](#défi-3--réactiver-la-sécurité-réseau)
   - [Défi 4 — Méthodologie et diagnostic de panne](#défi-4--méthodologie-et-diagnostic-de-panne)
8. [Questions de synthèse](#8-questions-de-synthèse)
9. [Bilan conceptuel comparatif](#9-bilan-conceptuel-comparatif)

---

## 1. Préparation de l'environnement

### 1.1 Création du cluster Kind et du namespace de travail
```bash
kind create cluster --name eshop
kubectl create namespace demo
```

**Sortie d'exécution :**
```text
Creating cluster "eshop" ...
 • Ensuring node image (kindest/node:v1.37.0) 🖼️  ...
 ✓ Ensuring node image (kindest/node:v1.37.0) 🖼️
 • Preparing nodes 📦   ...
 ✓ Preparing nodes 📦 
 • Writing configuration 📜  ...
 ✓ Writing configuration 📜
 • Starting control-plane 🕹️  ...
 ✓ Starting control-plane 🕹️
 • Installing CNI 🔌  ...
 ✓ Installing CNI 🔌
 • Installing StorageClass 💾  ...
 ✓ Installing StorageClass 💾
Set kubectl context to "kind-eshop"
namespace/demo created
```

---

## 2. Partie 1 — Déployer les Pods et observer leurs IP

### 2.1 Manifeste `01-nginx-deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-demo
  namespace: demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-demo
  template:
    metadata:
      labels:
        app: nginx-demo
    spec:
      containers:
      - name: nginx
        image: nginx:1.27
        ports:
        - name: http
          containerPort: 80
```

### 2.2 Déploiement et création du Pod client
```bash
kubectl apply -f 01-nginx-deployment.yaml
kubectl rollout status deployment/nginx-demo -n demo
kubectl run client -n demo --image=curlimages/curl:8.10.1 --restart=Never --command -- sleep 3600
kubectl wait -n demo --for=condition=Ready pod/client --timeout=120s
kubectl get deployment,pods -n demo -o wide
```

**Preuve d'exécution :**
```text
deployment.apps/nginx-demo created
Waiting for deployment "nginx-demo" rollout to finish: 0 of 2 updated replicas are available...
Waiting for deployment "nginx-demo" rollout to finish: 1 of 2 updated replicas are available...
deployment "nginx-demo" successfully rolled out

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES       SELECTOR
deployment.apps/nginx-demo   2/2     2            2           10s   nginx        nginx:1.27   app=nginx-demo

NAME                              READY   STATUS    RESTARTS   AGE   IP           NODE                  NOMINATED NODE   READINESS GATES
pod/client                        1/1     Running   0          6s    10.244.0.7   eshop-control-plane   <none>           <none>
pod/nginx-demo-67ff6498df-cg52m   1/1     Running   0          10s   10.244.0.5   eshop-control-plane   <none>           <none>
pod/nginx-demo-67ff6498df-9n22z   1/1     Running   0          10s   10.244.0.6   eshop-control-plane   <none>           <none>
```

### 2.3 Test de connectivité directe par IP de Pod
```bash
kubectl exec -n demo client -- curl -I --max-time 5 http://10.244.0.5
```
**Résultat :**
```text
HTTP/1.1 200 OK
Server: nginx/1.27.5
Date: Fri, 23 Oct 2026 14:01:32 GMT
Content-Type: text/html
Content-Length: 615
Connection: keep-alive
```

### 2.4 Réponses aux questions — Partie 1

1. Pourquoi une IP de Pod ne doit-elle pas être considérée comme une adresse permanente ?
Un pod est temporaire dans Kubernetes. S'il plante, s'il redémarre ou si on déploie une nouvelle version, l'ancien pod est détruit et un nouveau est recréé avec une toute nouvelle adresse IP. On ne peut donc pas se baser sur son IP pour le joindre de façon fiable.

2. Que fait le Deployment si un Pod est supprimé ?
Le Deployment surveille en permanence le nombre de réplicas demandés. Si un pod disparaît, il s'en aperçoit aussitôt et recrée un nouveau pod pour revenir automatiquement aux 2 réplicas configurés.

3. Le Deployment fournit-il lui-même une adresse réseau stable aux clients ?
Non. Le Deployment sert uniquement à gérer le cycle de vie et le nombre de pods. Il ne donne aucune IP stable ni point d'entrée unique. Pour avoir une adresse fixe et équilibrer la charge entre les pods, il faut obligatoirement créer un Service.

---

## 3. Partie 2 — Créer un Service ClusterIP

### 3.1 Manifeste `02-nginx-service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: demo
spec:
  type: ClusterIP
  selector:
    app: nginx-demo
  ports:
  - name: http
    port: 80
    targetPort: http
    protocol: TCP
```

### 3.2 Déploiement et inspection des EndpointSlices
```bash
kubectl apply -f 02-nginx-service.yaml
kubectl get service nginx-service -n demo
kubectl get endpointslices -n demo -l kubernetes.io/service-name=nginx-service
```

**Sortie d'exécution :**
```text
NAME            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
nginx-service   ClusterIP   10.96.95.194   <none>        80/TCP    0s

NAME                  ADDRESSTYPE   PORTS   ENDPOINTS               AGE
nginx-service-r75xf   IPv4          80      10.244.0.6,10.244.0.5   0s
```

### 3.3 Test de résolution DNS et appel de service
```bash
kubectl exec -n demo client -- curl -I --max-time 5 http://nginx-service
kubectl exec -n demo client -- nslookup nginx-service.demo.svc.cluster.local
```

**Résultat :**
```text
HTTP/1.1 200 OK
Server: nginx/1.27.5
Content-Type: text/html

Server:		10.96.0.10
Address:	10.96.0.10:53

Name:	nginx-service.demo.svc.cluster.local
Address: 10.96.95.194
```

### 3.4 Test de résilience du Service lors de la destruction des Pods
```bash
kubectl delete pod -n demo -l app=nginx-demo
kubectl rollout status deployment/nginx-demo -n demo
kubectl get pods -n demo -o wide
kubectl get endpointslices -n demo -l kubernetes.io/service-name=nginx-service
kubectl exec -n demo client -- curl -I --max-time 5 http://nginx-service
```

**Sortie d'observation :**
```text
pod "nginx-demo-67ff6498df-9n22z" deleted
pod "nginx-demo-67ff6498df-cg52m" deleted
deployment "nginx-demo" successfully rolled out

NAME                          READY   STATUS    IP           NODE
nginx-demo-67ff6498df-qmj9k   1/1     Running   10.244.0.8   eshop-control-plane
nginx-demo-67ff6498df-vsl8m   1/1     Running   10.244.0.9   eshop-control-plane

NAME                  ADDRESSTYPE   PORTS   ENDPOINTS               AGE
nginx-service-r75xf   IPv4          80      10.244.0.8,10.244.0.9   24s

HTTP/1.1 200 OK
Server: nginx/1.27.5
```
*Constat :* Bien que les deux Pods aient changé d'IP (passant de `.5` et `.6` à `.8` et `.9`), le Service a conservé sa ClusterIP fixe `10.96.95.194` et le EndpointSlice a automatiquement mis à jour ses adresses cibles sans aucune interruption de service.

### 3.5 Réponses aux questions — Partie 2

1. Quelle différence y a-t-il entre une IP de Pod et un ClusterIP ?
L'IP d'un pod est temporaire et change dès que le conteneur redémarre ou est recréé. La ClusterIP, c'est l'adresse virtuelle fixe du Service : elle ne bouge jamais pendant toute la vie du Service. Quand on envoie une requête vers la ClusterIP, Kubernetes l'envoie automatiquement à l'un des pods sains qui tournent derrière.

2. Quel objet sélectionne les Pods du Service ?
C'est le sélecteur de labels (le selector dans le YAML du Service, ici app: nginx-demo). Kubernetes l'utilise pour repérer tous les pods qui portent ce label et met à jour automatiquement la liste des adresses dans l'EndpointSlice.

3. Que se passe-t-il si le sélecteur du Service ne correspond à aucun Pod ?
Le Service existe toujours avec son IP, mais son EndpointSlice est vide. Si on essaie de lui envoyer une requête, la connexion échoue par timeout ou erreur réseau parce qu'il n'y a aucun pod derrière pour répondre.

4. Pourquoi est-il préférable d’appeler nginx-service plutôt que de mémoriser l’IP d’un Pod ?
Parce que le nom nginx-service ne change jamais et résout toujours vers l'adresse stable du Service grâce au DNS interne. En plus, le Service distribue les requêtes entre les différents pods. Si un pod plante ou change d'IP, l'appel continue de fonctionner de manière transparente sans rien casser côté client.

---

## 4. Partie 3 — Contrôler les flux avec NetworkPolicy

### 4.1 Étape 6 : Politique Default-Deny Ingress
Manifeste `03-nginx-network-policy.yaml` (blocage initial) :
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: nginx-ingress-policy
  namespace: demo
spec:
  podSelector:
    matchLabels:
      app: nginx-demo
  policyTypes:
  - Ingress
```

**Test du blocage :**
```bash
kubectl apply -f 03-nginx-network-policy.yaml
kubectl exec -n demo client -- curl -I --connect-timeout 3 --max-time 5 http://nginx-service
```
**Résultat :**
```text
curl: (28) Failed to connect to nginx-service port 80 after 3004 ms: Timeout was reached
command terminated with exit code 28
```
*Constat :* Le CNI Kindnet applique les règles et bloque tout flux entrant vers NGINX.

### 4.2 Étape 7 : Autoriser un client labellisé
Mise à jour de `03-nginx-network-policy.yaml` :
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: nginx-ingress-policy
  namespace: demo
spec:
  podSelector:
    matchLabels:
      app: nginx-demo
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          access: granted
    ports:
    - protocol: TCP
      port: 80
```

**Validation dynamique du label :**
```bash
# 1. Application de la politique et labellisation du client
kubectl apply -f 03-nginx-network-policy.yaml
kubectl label pod client -n demo access=granted --overwrite
kubectl exec -n demo client -- curl -I --connect-timeout 3 --max-time 5 http://nginx-service
# Résultat : HTTP/1.1 200 OK

# 2. Retrait du label
kubectl label pod client -n demo access-
kubectl exec -n demo client -- curl -I --connect-timeout 3 --max-time 5 http://nginx-service
# Résultat : curl: (28) Failed to connect after 3002 ms: Timeout was reached

# 3. Réattribution du label
kubectl label pod client -n demo access=granted --overwrite
```

### 4.3 Réponses aux questions — Partie 3

1. À quoi sert podSelector dans cette politique ?
Il sert à cibler les pods sur lesquels la politique s'applique, ici ceux qui ont le label app: nginx-demo. Dès qu'un pod est ciblé, il passe en mode verrouillé : Kubernetes bloque par défaut tout le trafic entrant qui n'est pas autorisé dans les règles.

2. La règle autorise-t-elle tous les Pods du namespace demo ou seulement ceux portant le label access=granted ?
Uniquement les pods du namespace demo qui portent le label access=granted. Si un pod du namespace n'a pas cette étiquette, ses requêtes sont bloquées.

3. Pourquoi la création de la ressource deployment ou pod ne garantit-elle pas que le trafic sera bloqué ?
Parce que de base dans Kubernetes, le réseau est entièrement ouvert et tout le monde peut parler à tout le monde. Déclarer un pod ou un deployment ne met aucune sécurité réseau en place. Il faut obligatoirement créer une NetworkPolicy, et il faut aussi que le plugin réseau du cluster (le CNI) gère bien l'application de ces politiques.

4. Pourquoi une politique restrictive peut-elle empêcher une Gateway d’accéder à son backend ?
Parce que la Gateway fait passer le trafic par des pods de proxy (ici Envoy) qui sont installés dans un autre namespace (envoy-gateway-system). Si la NetworkPolicy de notre application n'autorise que les pods locaux et n'ouvre pas l'accès au namespace et aux pods du proxy, la Gateway se fait bloquer et renvoie un timeout ou une erreur 502/504 au client.

---

## 5. Partie 4 — Comprendre Ingress

### 5.1 Réponse à la question — Partie 4

Quelle est la différence entre une ressource Ingress, un contrôleur Ingress et un Service ClusterIP ?

La ressource Ingress, c'est juste un fichier de configuration YAML dans lequel on écrit les règles de routage (par exemple tel nom de domaine ou tel chemin va vers tel service). Elle ne traite rien toute seule, c'est juste une description.

Le contrôleur Ingress, c'est le vrai programme (comme Nginx ou Traefik) qui tourne dans le cluster. Il lit ces fichiers YAML et applique concrètement la configuration sur son reverse proxy pour recevoir les requêtes de l'extérieur et les envoyer aux bons services.

Le Service ClusterIP, c'est l'aiguillage interne au cluster (au niveau TCP). Il donne une adresse IP fixe à l'application et répartit les connexions entre les différents pods qui tournent en arrière-plan.

---

## 6. Partie 5 — Installer Envoy Gateway et utiliser Gateway API

### 6.1 Installation via Helm OCI (v1.9.2)
```bash
helm install eg oci://docker.io/envoyproxy/gateway-helm \
  --version v1.9.2 \
  --namespace envoy-gateway-system \
  --create-namespace
```

**Vérification de l'installation :**
```bash
kubectl wait --namespace envoy-gateway-system --for=condition=Available deployment/envoy-gateway --timeout=5m
helm list -n envoy-gateway-system
kubectl get pods -n envoy-gateway-system
kubectl get crd | grep gateway.networking.k8s.io
```

**Sortie :**
```text
deployment.apps/envoy-gateway condition met
NAME   NAMESPACE              REVISION  STATUS    CHART                APP VERSION
eg     envoy-gateway-system   1         deployed  gateway-helm-v1.9.2  v1.9.2     

NAME                             READY   STATUS    RESTARTS   AGE
envoy-gateway-75b559cc4c-krsjc   1/1     Running   0          95s
```

### 6.2 Déclaration de la GatewayClass (`00-gatewayclass.yaml`)
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: eg
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
```
Statut : `Accepted: True` ("Valid GatewayClass").

### 6.3 Déclaration de la Gateway (`04-eshop-gateway.yaml`)
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: eshop-gateway
  namespace: demo
spec:
  gatewayClassName: eg
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    allowedRoutes:
      namespaces:
        from: Same
```

Statut de la Gateway :
- `Accepted: True` ("The Gateway has been scheduled by Envoy Gateway")
- Un pod proxy Envoy a été déployé automatiquement dans `envoy-gateway-system` : `pod/envoy-demo-eshop-gateway-fdfffb6e-84b9f4b7ff-wb5n8`.
- Un service proxy a été instancié : `service/envoy-demo-eshop-gateway-fdfffb6e`.

### 6.4 Déclaration de la HTTPRoute (`05-eshop-route.yaml`)
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: nginx-route
  namespace: demo
spec:
  parentRefs:
  - name: eshop-gateway
  hostnames:
  - shop.example.com
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: nginx-service
      port: 80
```

Statut de la route :
- `Accepted: True` ("Route is accepted")
- `ResolvedRefs: True` ("Resolved all the Object references for the Route")

### 6.5 Test de bout en bout via le proxy Envoy
```bash
# Redirection locale du port HTTP du proxy Envoy
kubectl port-forward -n envoy-gateway-system service/envoy-demo-eshop-gateway-fdfffb6e 8080:80

# Appel HTTP avec l'en-tête Host attendu
curl -i -H 'Host: shop.example.com' http://localhost:8080/
```

**Résultat :**
```text
HTTP/1.1 200 OK
server: nginx/1.27.5
date: Fri, 23 Oct 2026 14:06:41 GMT
content-type: text/html
content-length: 615

<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
```

---

## 7. Partie 6 — Défis pratiques

### Défi 1 — Ajouter un second Service
Déploiement d'une application catalogue (`catalog-demo`) avec son propre ConfigMap, Deployment et Service ClusterIP (`06-catalog-app.yaml`) :
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: catalog-html
  namespace: demo
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <head><title>Catalog Service</title></head>
    <body>
      <h1>Catalog Service API</h1>
      <p>Bienvenue sur le service catalogue E-Shop !</p>
    </body>
    </html>
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: catalog-demo
  namespace: demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: catalog-demo
  template:
    metadata:
      labels:
        app: catalog-demo
    spec:
      containers:
      - name: catalog
        image: nginx:1.27
        ports:
        - name: http
          containerPort: 80
        volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html
      volumes:
      - name: html
        configMap:
          name: catalog-html
---
apiVersion: v1
kind: Service
metadata:
  name: catalog-service
  namespace: demo
spec:
  type: ClusterIP
  selector:
    app: catalog-demo
  ports:
  - name: http
    port: 80
    targetPort: 80
    protocol: TCP
```

**Preuves de validation :**
1. **Les EndpointSlices sont distincts et peuplés :**
   ```text
   NAME                    ADDRESSTYPE   PORTS   ENDPOINTS                 AGE
   catalog-service-jtmb4   IPv4          80      10.244.0.13,10.244.0.14   1s
   nginx-service-r75xf     IPv4          80      10.244.0.8,10.244.0.9     5m
   ```
2. **Appel DNS depuis le pod client vers `catalog-service` :**
   ```text
   HTTP/1.1 200 OK
   <title>Catalog Service</title>
   <h1>Catalog Service API</h1>
   ```
3. **Le service `nginx-service` reste strictement isolé sur ses propres Pods (`app: nginx-demo`).**

---

### Défi 2 — Modifier le routage HTTP
Création d'une HTTPRoute spécifique pour router le chemin `/catalog` vers `catalog-service` avec réécriture d'URL (`07-eshop-catalog-route.yaml`) :
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: catalog-route
  namespace: demo
spec:
  parentRefs:
  - name: eshop-gateway
  hostnames:
  - shop.example.com
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /catalog
    filters:
    - type: URLRewrite
      urlRewrite:
        path:
          type: ReplaceFullPath
          replaceFullPath: /
    backendRefs:
    - name: catalog-service
      port: 80
```

**Tests et validation :**
1. **Requête racine `shop.example.com /` :**
   ```text
   HTTP/1.1 200 OK
   <title>Welcome to nginx!</title>
   ```
2. **Requête `/catalog` sur `shop.example.com /catalog` :**
   ```text
   HTTP/1.1 200 OK
   <title>Catalog Service</title>
   <h1>Catalog Service API</h1>
   ```
3. **Requête avec Host inconnu `other.example.com /` :**
   ```text
   HTTP/1.1 404 Not Found
   ```

---

### Défi 3 — Réactiver la sécurité réseau
Inspection préalable des labels réels des Pods et namespaces :
- Pod Envoy proxy : `app.kubernetes.io/component=proxy` dans le namespace `envoy-gateway-system`
- Client diagnostique : `access=granted` dans le namespace `demo`

Création du manifeste `08-nginx-defi3-policy.yaml` :
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: nginx-ingress-policy
  namespace: demo
spec:
  podSelector:
    matchLabels:
      app: nginx-demo
  policyTypes:
  - Ingress
  ingress:
  # 1. Autorise le client de diagnostic (portant le label access=granted) dans le namespace demo
  - from:
    - podSelector:
        matchLabels:
          access: granted
    ports:
    - protocol: TCP
      port: 80
  # 2. Autorise les pods du proxy Envoy depuis le namespace envoy-gateway-system
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: envoy-gateway-system
      podSelector:
        matchLabels:
          app.kubernetes.io/component: proxy
    ports:
    - protocol: TCP
      port: 80
```

**Preuve d'exécution et matrice de validation :**
| Source de test | Configuration | Résultat observé | Conformité |
|---|---|:---:|:---:|
| **Envoy Gateway** (port-forward) | Via proxy Envoy (`envoy-gateway-system`) | `HTTP/1.1 200 OK` | ✅ Autorisé |
| **Pod Client** (namespace `demo`) | Avec label `access=granted` | `HTTP/1.1 200 OK` | ✅ Autorisé |
| **Pod Client** (namespace `demo`) | Sans label `access` (non autorisé) | `Timeout was reached (exit 28)` | 🛡️ Bloqué |

---

### Défi 4 — Méthodologie et diagnostic de panne

Quand quelqu'un dit que curl http://nginx-service ne marche pas depuis un pod, voici l'ordre dans lequel on vérifie les choses et pourquoi :

1. Est-ce que le pod client tourne et est Ready ?
Commande : kubectl get pod client -n demo
Pourquoi : Si le conteneur du client a planté, redémarre en boucle ou n'a pas fini de démarrer, le problème vient tout simplement du client lui-même, pas du réseau ni du serveur distant.

2. Est-ce que le DNS résout nginx-service ?
Commande : kubectl exec -n demo client -- nslookup nginx-service
Pourquoi : Ça permet de savoir si CoreDNS fonctionne correctement dans le cluster. Si la commande ne trouve pas l'adresse IP associée au nom du service, le client ne saura jamais où envoyer ses requêtes.

3. Est-ce que le Service existe et a une ClusterIP ?
Commande : kubectl get svc nginx-service -n demo
Pourquoi : Ça vérifie que le service a bien été créé, qu'on ne s'est pas trompé de nom ou de namespace, et qu'il possède bien son adresse IP virtuelle et son port configuré.

4. Est-ce que les EndpointSlices contiennent des endpoints prêts ?
Commande : kubectl get endpointslices -n demo -l kubernetes.io/service-name=nginx-service
Pourquoi : Ça permet de vérifier que le sélecteur du service trouve bien des pods correspondants et que ces pods sont en état Ready. S'il n'y a aucun endpoint, le service ne sait pas où router le trafic.

5. Est-ce qu'une NetworkPolicy bloque le flux ?
Commande : kubectl get networkpolicy -n demo
Pourquoi : Si les pods sont prêts mais que les requêtes finissent en timeout, c'est presque toujours une règle de sécurité réseau qui bloque les connexions entrantes ou sortantes.

6. Est-ce que le réseau du cluster applique bien les NetworkPolicies ?
Pourquoi : Ça permet de s'assurer que le plugin réseau installé sur le cluster prend réellement en charge le filtrage. Certains plugins réseau ignorent totalement les NetworkPolicies.

---

## 8. Questions de synthèse

1. Pourquoi un Service peut-il conserver une adresse stable quand un Pod est remplacé ?
Parce que l'adresse IP du Service (la ClusterIP) est une adresse virtuelle qui appartient au Service lui-même, pas aux conteneurs. Quand un pod meurt et qu'un nouveau est recréé avec une IP différente, Kubernetes met simplement à jour la liste des destinations dans l'EndpointSlice en coulisses. Pour le client qui appelle, l'IP du Service et son nom DNS ne changent jamais.

2. Quel est le rôle de CoreDNS et quel est celui du datapath réseau ?
CoreDNS sert uniquement d'annuaire : il traduit le nom du service (comme nginx-service) en son adresse IP interne.
Une fois que le client a cette adresse IP, c'est le datapath réseau (le plugin réseau et les règles iptables ou eBPF) qui prend le relais pour acheminer physiquement les paquets jusqu'au bon pod.

3. Pourquoi une NetworkPolicy peut-elle bloquer une requête qui fonctionnait avant ?
Parce que de base dans Kubernetes, tout le monde peut parler à tout le monde. Mais dès qu'on applique une NetworkPolicy sur un pod, ce pod passe en mode verrouillé. Tout ce qui n'est pas explicitement autorisé dans les règles est automatiquement bloqué. Si notre client n'est pas dans la liste des exceptions autorisées, sa connexion est immédiatement coupée.

4. Pourquoi Gateway API nécessite-t-elle un contrôleur ?
Parce que les fichiers YAML de la Gateway API (GatewayClass, Gateway, HTTPRoute) sont juste des déclarations stockées dans Kubernetes. Ce sont des données passives qui ne font rien toutes seules. Il faut obligatoirement un contrôleur actif (comme Envoy Gateway) pour lire ces règles, lancer les pods de proxy (Envoy) et leur envoyer la configuration de routage.

5. Quelle différence fais-tu entre une Gateway et une HTTPRoute ?
La Gateway, c'est le point d'entrée d'infrastructure : elle définit sur quels ports on écoute (par exemple 80 ou 443), quels protocoles on accepte et quels certificats on utilise. C'est l'équipe infra qui la gère.
La HTTPRoute, c'est la règle de routage applicative : elle dit quel nom de domaine ou quel chemin d'URL doit aller vers quel service. C'est l'équipe de dev qui la rédige et qui la rattache à la Gateway. Ça permet de bien séparer les rôles entre ceux qui gèrent la plateforme et ceux qui déploient les applications.

---

## 9. Bilan conceptuel comparatif

| Concept / Ressource | Couche Réseau (OSI) | Rôle principal | Portée / Rôle type |
|---|:---:|---|---|
| **Pod IP** | Couche 3 (Réseau) | Adresse directe et éphémère d'un conteneur donné. | Éphémère, interne au nœud / cluster. |
| **Service ClusterIP** | Couche 4 (Transport) | Adresse IP virtuelle stable et équilibrage TCP/UDP interne entre réplicas. | Interne au cluster, point d'entrée stable. |
| **NetworkPolicy** | Couches 3-4 (IP / Port) | Pare-feu distribué SDN contrôlant les flux autorisés/bloqués entre Pods. | Sécurité réseau interne au cluster. |
| **Ingress** | Couche 7 (Applicatif) | Spécification historique monolithique de reverse proxy HTTP/HTTPS. | Externe vers interne (mono-ressource). |
| **Gateway API** | Couche 7+ (Applicatif) | Standard moderne, extensible et orienté rôles (GatewayClass, Gateway, HTTPRoute) piloté par un contrôleur avancé (ex: Envoy Gateway). | Point d'entrée de production moderne avec séparation des privilèges Infra / Dev. |
