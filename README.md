# DevOps - Docker

## TD - Discover Docker

Prise en main de Docker : lancer des containers à partir d'images existantes (`alpine`, `dockersamples/static-site`), publier des ports, puis créer sa propre image. L'objectif final est de conteneuriser une petite application Python Flask qui affiche un GIF de chat aléatoire, en écrivant son Dockerfile, en buildant l'image et en exposant l'app sur `localhost:8888`.

### Commandes principales
| Commande | Rôle |
|---|---|
| `docker pull <img>` | Télécharge une image depuis le registry |
| `docker run -it <img> sh` | Container interactif (debug) |
| `docker run -d -p HOST:CONT --name X <img>` | Container en arrière-plan avec port publié |
| `docker ps -a` | Liste les containers (y compris arrêtés) |
| `docker stop / rm / rm -f` | Arrête / supprime un container |
| `docker build -t user/app .` | Construit une image depuis un Dockerfile |
| `docker logs X` | Affiche la sortie du container |

### Flask app (`td/flask-app`)

```bash
docker build -t marialeticiagaspar/myfirstapp td/flask-app
docker run -d -p 8888:5000 --name myfirstapp marialeticiagaspar/myfirstapp
# http://localhost:8888
```

**Choix du Dockerfile**
- Image de base `alpine:3.21.0` : légère, version fixée (pas de `latest`)
- `requirements.txt` copié avant le code pour profiter du cache des layers
- `--no-cache` / `--no-cache-dir` pour réduire la taille de l'image
- `venv` : dépendances Python isolées du système
- `.dockerignore` : exclut `.git` et `__pycache__` du contexte de build

**`-p 8888:5000`** - `PORT_HÔTE:PORT_CONTAINER`
- `5000` : port où Flask écoute dans le container
- `8888` : port ouvert sur la machine, redirigé par Docker vers le 5000

`EXPOSE 5000` ne publie rien (documentation seulement), c'est `-p` qui ouvre le port.
`host="0.0.0.0"` est nécessaire pour que Flask accepte les connexions venant de l'extérieur du container.

**Affichage des chats :** chaque requête sur `/` appelle `index()`, qui tire une URL au hasard (`random.choice`) et l'injecte dans `{{url}}` du template. Un refresh correspond donc à un nouveau tirage.

## TP part 01 - Docker

Application 3-tiers conteneurisée : un reverse proxy **Apache httpd**, une API **Spring Boot** (Java 21) et une base **PostgreSQL**, orchestrés avec **docker-compose** et publiés sur Docker Hub.

```
tp/
├── database/     # PostgreSQL + scripts d'init
├── java-hello/   # Hello World Java (simple + multistage)
├── backend/      # API Spring Boot
└── httpd/        # Reverse proxy Apache
docker-compose.yml
.env.example      # modèle des variables (.env est ignoré par git)
```

### Lancer l'application
```bash
cp .env.example .env
docker compose up -d --build
curl http://localhost/departments/IRC/students
```

---

### Database (`tp/database`)

Image `postgres:17.2-alpine` ; les scripts `sql/01-CreateScheme.sql` et `sql/02-InsertData.sql` sont copiés dans `/docker-entrypoint-initdb.d/` et exécutés par ordre alphabétique au premier démarrage (uniquement si le volume est vide).

```bash
docker network create app-network
docker build -t marialeticiagaspar/database tp/database
docker run -d --name database --network app-network --env-file .env \
  -v db-data:/var/lib/postgresql/data marialeticiagaspar/database
docker run -d --name adminer --network app-network -p 8090:8080 adminer   # http://localhost:8090
```

**1-1 Pourquoi `-e` plutôt que `ENV` dans le Dockerfile ?**
Un `ENV` est figé dans l'image : le mot de passe serait visible dans le code versionné et via `docker inspect` / `docker history` par toute personne ayant l'image. Avec `-e` (ou `--env-file`), les secrets restent hors de l'image et la même image peut être utilisée dans plusieurs environnements avec des valeurs différentes.

**1-2 Pourquoi un volume ?**
Le filesystem d'un container est éphémère : supprimer le container supprime les données. Le volume `db-data`, monté sur `/var/lib/postgresql/data`, stocke les données hors du container ; elles survivent à sa suppression ou à une mise à jour de l'image.

**1-3 Essentiels du container database**
- Dockerfile : `FROM postgres:17.2-alpine` + `COPY sql/ /docker-entrypoint-initdb.d/`
- Pas de port publié : seuls les containers du même réseau y accèdent, via le hostname `database`
- Vérification : `docker exec -it database psql -U usr -d db -c "SELECT * FROM students;"`

---

### Backend (`tp/java-hello`, `tp/backend`)

- `java-hello/Dockerfile` : `.class` compilé sur la machine, exécuté dans une image JRE.
- `java-hello/Dockerfile.multistage` : compilation dans un stage JDK, exécution dans un stage JRE.
- `backend` : API Spring Boot 3.4.5 buildée par Maven en multistage. Dans `application.yml`, la datasource lit les variables d'environnement (`${POSTGRES_DB}`, `${POSTGRES_USER}`, `${POSTGRES_PASSWORD}`) et se connecte à `jdbc:postgresql://database:5432/...` : aucun identifiant dans le code.

```bash
docker build -t marialeticiagaspar/backend tp/backend
docker run -d --name backend --network app-network --env-file .env -p 8080:8080 marialeticiagaspar/backend
curl http://localhost:8080/departments/IRC/students
```

**1-4 Pourquoi un multistage build ?**
Pour builder l'application dans Docker (pas besoin de JDK ni de Maven sur la machine) tout en gardant une image finale légère : JDK, Maven et sources restent dans le stage de build ; l'image finale ne contient que le JRE et le `.jar` (plus petite, moins de surface d'attaque).

| Étape | Rôle |
|---|---|
| `FROM eclipse-temurin:21-jdk-alpine AS myapp-build` | Stage de build avec le JDK, nommé pour être référencé |
| `ENV MYAPP_HOME` / `WORKDIR` | Dossier de travail `/opt/myapp` |
| `RUN apk add --no-cache maven` | Installe Maven |
| `COPY pom.xml` + `RUN mvn dependency:go-offline` | Télécharge les dépendances dans une layer en cache tant que le `pom.xml` ne change pas |
| `COPY src` + `RUN mvn package -DskipTests` | Compile et génère le `.jar` |
| `FROM eclipse-temurin:21-jre-alpine` | Stage final : runtime Java uniquement |
| `COPY --from=myapp-build ... myapp.jar` | Récupère seulement le `.jar` |
| `ENTRYPOINT ["java", "-jar", "myapp.jar"]` | Lance l'application |

---

### HTTP server (`tp/httpd`)

Image `httpd:2.4-alpine` avec un `index.html` et un `httpd.conf` récupéré du container (`docker cp httpd:/usr/local/apache2/conf/httpd.conf`) puis complété :
- `ServerName localhost` : supprime l'avertissement AH00558
- `LoadModule proxy_module` / `proxy_http_module` placés **avant** le `VirtualHost`, sinon Apache ne reconnaît pas `ProxyPass`
- `ProxyPass / http://backend:8080/` : toutes les requêtes sont transmises au backend

**1-5 Pourquoi un reverse proxy ?**
C'est le seul point d'entrée exposé : backend et base ne sont pas accessibles directement depuis l'extérieur. Il permet aussi de centraliser le HTTPS, le load balancing entre plusieurs backends, le cache, et de servir front-end et API sous le même domaine.

---

### Docker Compose (`docker-compose.yml`)

- Seul `httpd` publie un port (`80`) : backend et database restent fermés à l'hôte.
- Deux réseaux : `frontend-network` (httpd ↔ backend) et `backend-network` (backend ↔ database) ; httpd ne peut pas joindre la base.
- `env_file: .env` : identifiants hors du code et de l'image.
- Volume nommé `db-data` : persistance de la base.
- `healthcheck` (`pg_isready`) + `depends_on: condition: service_healthy` : le backend attend que la base soit prête.
- `restart: unless-stopped` : redémarrage automatique en cas de crash.

**1-6 Pourquoi docker-compose est-il important ?**
Il décrit toute l'application (services, réseaux, volumes, variables, dépendances) dans un seul fichier versionné. Une commande suffit pour tout builder, démarrer ou arrêter dans le bon ordre, de manière reproductible, au lieu d'enchaîner des `docker run` à la main.

**1-7 Commandes principales**
| Commande | Rôle |
|---|---|
| `docker compose up -d --build` | Build et démarre les services en arrière-plan |
| `docker compose ps` | État des services |
| `docker compose logs -f <service>` | Suit les logs d'un service |
| `docker compose stop` / `start` | Arrête / relance sans supprimer |
| `docker compose down` | Supprime containers et réseaux (volume conservé) |
| `docker compose down -v` | Supprime aussi les volumes (reset de la base) |

**1-8 Le fichier docker-compose**
- `backend` : buildé depuis `tp/backend`, sur les deux réseaux, démarre quand `database` est *healthy*.
- `database` : buildé depuis `tp/database`, volume `db-data`, uniquement sur `backend-network`, healthcheck `pg_isready`.
- `httpd` : buildé depuis `tp/httpd`, seul service exposé (`80:80`), uniquement sur `frontend-network`.
- `networks` / `volumes` : déclaration des deux réseaux et du volume nommé.

---

### Publication (Docker Hub)

```bash
docker login
docker tag marialeticiagaspar/database marialeticiagaspar/database:1.0
docker push marialeticiagaspar/database:1.0
# idem pour backend et httpd
```

Images publiées : `database:1.0`, `backend:1.0`, `httpd:1.0` sur https://hub.docker.com/u/marialeticiagaspar

**1-9 Commandes de publication**
`docker login` (authentification), `docker tag` (nom du dépôt + version explicite, ex. `1.0`), `docker push` (envoi vers Docker Hub). Versionner évite de dépendre de `latest` et permet de revenir à une version précise.

**1-10 Pourquoi publier les images dans un registry en ligne ?**
Pour les partager : équipe, serveurs et CI/CD récupèrent exactement la même image avec un `docker pull`, sans rebuild (« build once, deploy everywhere »). Le registry garde l'historique des versions, ce qui facilite déploiements et rollbacks.
