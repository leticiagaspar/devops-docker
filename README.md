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
