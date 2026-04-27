# Rapport : Containerisation de wordsmith

## Exercice 1 : Écriture des Dockerfiles

Trois Dockerfiles ont été créés, un par service.

### web (`web/Dockerfile`)
Le service est un serveur web en Go. Le Dockerfile utilise un **multi-stage build** :
- **Stage `build`** : compile `dispatcher.go` avec l'image `golang:1.21-alpine`
- **Stage final** : image `scratch` (image vide, 0 octet de base) qui ne contient que le binaire compilé et le dossier `static/`

`scratch` est possible ici car Go compile en binaire statique autonome (`CGO_ENABLED=0`) qui n'a besoin d'aucune librairie système.

### words (`words/Dockerfile`)
Le service est une API REST en Java compilée avec Maven. Le Dockerfile utilise un **multi-stage build** en 3 étapes :
- **Stage `build`** : compile le projet avec `maven:3.9-eclipse-temurin-21`
- **Stage `jre-build`** : construit un JRE minimal avec `jlink` (outil Java qui ne garde que les modules nécessaires)
- **Stage final** : image `alpine` avec uniquement le JRE minimal + les JARs

### db (`db/Dockerfile`)
Le service est une base de données PostgreSQL. Le Dockerfile est minimal :
- Image de base `postgres:15-alpine`
- Variable `POSTGRES_HOST_AUTH_METHOD=trust` pour autoriser les connexions sans mot de passe
- Le fichier `words.sql` est copié dans `/docker-entrypoint-initdb.d/` : PostgreSQL exécute automatiquement tous les scripts SQL placés dans ce dossier au premier démarrage

---

## Exercice 2 : Optimisation de la taille des images

| Service | Avant optimisation | Après optimisation | Objectif "très bon" |
|---------|-------------------|-------------------|---------------------|
| web     | ~900 MB (golang)  | **7.81 MB**       | 10 MB               |
| words   | ~500 MB (maven)   | **57.6 MB**       | 50 MB               |
| db      | 274 MB            | **274 MB**        | 300 MB              |

### web
L'image finale utilise `FROM scratch` : elle ne contient que le binaire Go et les fichiers statiques. Résultat exceptionnel de 7.81 MB.

### words
Deux optimisations clés :
1. **`jlink`** : génère un JRE sur-mesure avec seulement les modules Java nécessaires (`java.base`, `java.logging`, `java.sql`, `java.naming`, `jdk.httpserver`) au lieu d'un JRE complet (~200 MB)
2. **Image de base musl-compatible** : `eclipse-temurin:21-alpine` (musl libc) pour le stage `jre-build` au lieu de `eclipse-temurin:21` (glibc). Cela garantit que le JRE généré est nativement compatible avec Alpine Linux qui utilise musl libc

### db
L'image `postgres:15-alpine` est déjà la version optimisée officielle (Alpine au lieu de Debian).

---

## Exercice 3 : Optimisation du temps de build

L'objectif est que le rebuild après modification du code source prenne moins de 10 secondes.

La technique clé est l'**ordre des instructions dans le Dockerfile** : Docker met en cache chaque layer. Si un layer n'a pas changé, il est réutilisé sans être recalculé.

### words
```dockerfile
COPY pom.xml .               # layer 1 : rarement modifié
RUN mvn dependency:go-offline # layer 2 : caché si pom.xml inchangé (~2 min de téléchargement)
COPY src ./src               # layer 3 : invalidé si le code change
RUN mvn verify               # layer 4 : re-run uniquement après modif du code
```
Après modification du code : seuls les layers 3 et 4 re-tournent, les dépendances Maven sont déjà en cache.

### web
```dockerfile
COPY go.mod* go.sum* ./      # layer 1 : rarement modifié
RUN go mod download           # layer 2 : caché si go.mod inchangé
COPY . .                     # layer 3 : invalidé si le code change
RUN go build dispatcher.go   # layer 4 : re-run uniquement après modif du code
```

---

## Exercice 4 : Écriture du fichier Compose

Le fichier `compose.yaml` à la racine du projet orchestre les 3 services :

```yaml
services:
  db:
    build: ./db

  words:
    build: ./words
    depends_on:
      - db

  web:
    build: ./web
    ports:
      - "${WEB_PORT:-80}:80"
    volumes:
      - ./web/static:/static
    depends_on:
      - words
```

Points importants :
- **Résolution DNS automatique** : Compose crée un réseau interne où chaque service est accessible par son nom. `web` peut joindre `words` via `http://words:8080` et `words` peut joindre `db` via `db:5432` — ce qui correspond exactement aux URLs codées dans le code source
- **`depends_on`** : garantit l'ordre de démarrage (db → words → web)
- **`ports`** : seul `web` expose un port vers l'extérieur (`80:80`), les autres services ne sont accessibles que depuis l'intérieur du réseau Compose

Un bug a été découvert et corrigé : le Dockerfile de `words` ne copiait que `words.jar` mais pas le dossier `dependency/` contenant les JARs tiers (driver PostgreSQL, Guava). Le manifest du JAR référence ce dossier dans son classpath, donc l'application crashait avec `ClassNotFoundException`.

---

## Exercice 5 : Développement itératif avec Compose

Pour éviter de reconstruire l'image à chaque modification des fichiers HTML/CSS, un **bind mount** est ajouté au service `web` :

```yaml
volumes:
  - ./web/static:/static
```

Le bind mount "écrase" le dossier `/static` embarqué dans l'image par le dossier local `web/static/`. Toute modification d'un fichier dans `web/static/` est immédiatement visible dans le container sans rebuild ni restart. Il suffit de recharger la page dans le navigateur.

---

## Exercice 6 : Déployer plusieurs stacks

Pour lancer plusieurs instances de l'application sur la même machine, deux problèmes doivent être résolus :
1. **Conflit de noms** : Compose préfixe les containers avec le nom du projet. Si deux stacks ont le même nom, elles entrent en conflit
2. **Conflit de ports** : deux services ne peuvent pas écouter sur le même port hôte

### Solution

Le port est paramétré dans `compose.yaml` avec une variable d'environnement :
```yaml
ports:
  - "${WEB_PORT:-80}:80"   # 80 par défaut, surchargeable
```

Pour chaque nouvelle instance, deux petits fichiers suffisent :

**`compose.instance2.yaml`** (surcharge le nom du projet) :
```yaml
name: wordsmith2
```

**`instance2.env`** (surcharge le port) :
```
WEB_PORT=81
```

Commande de déploiement :
```bash
docker compose -f compose.yaml -f compose.instance2.yaml --env-file instance2.env up -d
```

Compose fusionne les deux fichiers : `compose.instance2.yaml` apporte le nouveau nom de projet, `instance2.env` injecte le port. Résultat : deux stacks complètement indépendantes sur les ports 80 et 81.

---

## Exercice 7 : Déploiement sur Kubernetes

Kubernetes a été déployé via **kind** (Kubernetes in Docker), qui crée un cluster dans un container Docker.

Le fichier `k8s.yaml` définit 6 ressources Kubernetes (1 Deployment + 1 Service par service applicatif).

### Structure

Pour chaque service, le pattern est identique :

**Deployment** : décrit comment lancer les containers
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: web
          image: jpetazzo/wordsmith-web:latest
```

**Service** : expose le Deployment sur le réseau interne (ou externe)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  type: LoadBalancer   # expose à l'extérieur du cluster
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
```

### Types de Services

| Service | Type        | Accessible depuis          |
|---------|-------------|---------------------------|
| db      | ClusterIP   | Intérieur du cluster seulement |
| words   | ClusterIP   | Intérieur du cluster seulement |
| web     | LoadBalancer| Extérieur (localhost:80)   |

`ClusterIP` est le type par défaut : le service n'est joignable que depuis l'intérieur du cluster. `LoadBalancer` demande à l'infrastructure sous-jacente (ici Docker Desktop / kind) de créer un point d'entrée externe.

Comme en Compose, la résolution DNS fonctionne par nom de Service : `words` joindra `db` via `db:5432` et `web` joindra `words` via `words:8080`.

### Déploiement

```bash
kubectl apply -f k8s.yaml   # crée toutes les ressources
kubectl get pods             # vérifie que les pods sont Running
```
