# TP 2 : premiers conteneurs Docker

| | |
|---|---|
| **Cours** | Introduction au Cloud (L3 SI RIST & BDGL) |
| **Binôme** | [À COMPLÉTER : NOM Prénom 1, NOM Prénom 2] |
| **Date** | 26/10/2026 |
| **Environnement** | VM `ubuntu-srv` (Ubuntu Server sous VirtualBox), utilisateur `etudiant` |
| **Image publiée** | https://hub.docker.com/r/issiaka01/api-fil-rouge |

---

## Partie A : découverte de Docker dans la VM `ubuntu-srv`

### A1. Installation de Docker

```bash
sudo apt update
sudo apt install -y docker.io
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
docker --version
```

| Commande | Rôle |
|---|---|
| `apt install -y docker.io` | Installe Docker. |
| `systemctl enable --now docker` | Démarre Docker maintenant et à chaque démarrage de la VM. |
| `usermod -aG docker $USER` | Ajoute l'utilisateur au groupe `docker` pour utiliser Docker sans `sudo` (reconnexion nécessaire). |

> **[À COMPLÉTER]** Résultat de `docker --version` (capture ou copier-coller).

### A2. Premier conteneur : `hello-world`

```bash
time docker run --rm hello-world
```

![hello-world et temps d'exécution](img/a2-hello-world-time.png)

Le message affiché décrit ce qui s'est passé :
1. le client Docker contacte le démon Docker ;
2. le démon télécharge l'image `hello-world` depuis Docker Hub (si elle est absente en local) ;
3. le démon crée un conteneur à partir de cette image et l'exécute ;
4. la sortie est renvoyée au terminal.

Temps mesuré : **`real` = 1,421 s** (`user` = 0,016 s, `sys` = 0,100 s). L'option `--rm` supprime le conteneur dès qu'il se termine.

### A3. Nginx sur le port 8080

```bash
docker run -d --name web -p 8080:80 nginx
docker ps
curl http://localhost:8080
```

| Option | Rôle |
|---|---|
| `-d` | Exécute le conteneur en arrière-plan. |
| `--name web` | Donne le nom `web` au conteneur. |
| `-p 8080:80` | Relie le port 8080 de la VM au port 80 du conteneur. |
| `nginx` | Image utilisée. |

> **[À COMPLÉTER]** Capture de `docker ps` (conteneur `web` en état `Up`) et de `curl http://localhost:8080` (page « Welcome to nginx! »).

### A4. Inspection du conteneur

```bash
docker ps            # conteneurs actifs
docker logs web      # journaux du conteneur
docker exec -it web bash   # shell dans le conteneur
docker stats --no-stream   # consommation de ressources
```

| Commande | Rôle |
|---|---|
| `docker ps` | Liste les conteneurs en cours d'exécution (`-a` pour inclure les arrêtés). |
| `docker logs web` | Affiche les journaux (requêtes reçues par Nginx). |
| `docker exec -it web bash` | Ouvre un shell dans un conteneur déjà lancé. |
| `docker stats --no-stream` | Affiche CPU, mémoire, réseau et disque en un instantané. |

> **[À COMPLÉTER]** Capture de `docker logs web`.

**`docker exec`** : le prompt passe de `etudiant@ubuntu-srv` à `root@ef748b8758b9`, ce qui montre que l'on est à l'intérieur du conteneur (`ef748b8758b9` est l'identifiant du conteneur `web`).

![docker exec dans le conteneur web](img/a3-exec.png)

**`docker stats`** (exécuté depuis la VM, pas depuis le conteneur) :

![docker stats](img/a4-stats.png)

| Mesure | Valeur |
|---|---|
| CPU | 0,00 % |
| Mémoire utilisée | 21,18 MiB sur 2,886 GiB (0,72 %) |
| Réseau (entrée / sortie) | 1,46 kB / 126 B |
| Disque (lecture / écriture) | 18,3 MB / 8,19 kB |
| Processus (PIDS) | 5 |

### A5. Comparaison conteneur / machine virtuelle

Les tailles d'images proviennent de la commande `docker images` (voir la capture de la partie B, section B4).

| Critère | Conteneur (Nginx) | Machine virtuelle (`ubuntu-srv`) |
|---|---|---|
| Taille sur disque | Image `nginx` : **243 MB** (66,8 MB de contenu compressé) | [À COMPLÉTER : taille du disque de la VM, via VirtualBox ou `df -h`] |
| Démarrage | Environ **1,4 s** (mesuré avec `hello-world`, image déjà en local) | [À COMPLÉTER : temps de démarrage de la VM] |
| Mémoire | **21,18 MiB** | [À COMPLÉTER : RAM allouée à la VM] (la VM voit 2,886 GiB au total) |
| Système d'exploitation | Aucun noyau propre, il utilise celui de l'hôte | Système complet avec son propre noyau |
| Isolation | Par processus (namespaces et cgroups) | Complète, via l'hyperviseur |

**Analyse.** Un conteneur ne embarque pas de système d'exploitation complet : il partage le noyau de l'hôte et ne contient que l'application et ses dépendances. Il est donc beaucoup plus léger, démarre en environ une seconde et consomme quelques dizaines de Mo de mémoire. En contrepartie, l'isolation est moins forte que celle d'une VM, puisque le noyau est partagé.

### Nettoyage

```bash
docker stop web
docker rm web
```

---

## Partie B : conteneuriser l'API fil rouge

### B1. Les fichiers de l'API

L'API est écrite en Python avec Flask. Les fichiers sont dans le dossier `~/api` de la VM.

**`app.py`**
```python
from flask import Flask, jsonify

app = Flask(__name__)

@app.route("/")
def home():
    return jsonify(message="API operationnelle")

@app.route("/health")
def health():
    return jsonify(status="ok")

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

**`requirements.txt`**
```
flask==3.0.3
```

L'API écoute sur `0.0.0.0` : avec `127.0.0.1`, elle ne serait pas joignable depuis l'extérieur du conteneur, et la redirection de port `-p 5000:5000` serait inutile.

### B2. Le Dockerfile

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 5000
CMD ["python", "app.py"]
```

| Instruction | Rôle |
|---|---|
| `FROM python:3.12-slim` | Image de base : Python 3.12 sur un Linux allégé (version `slim`, plus petite). |
| `WORKDIR /app` | Dossier de travail dans le conteneur. |
| `COPY requirements.txt .` | Copie la liste des dépendances en premier. |
| `RUN pip install --no-cache-dir -r requirements.txt` | Installe Flask pendant la construction ; `--no-cache-dir` réduit la taille de l'image. |
| `COPY . .` | Copie le code de l'API dans l'image. |
| `EXPOSE 5000` | Documente le port utilisé (ne publie pas le port à lui seul). |
| `CMD ["python", "app.py"]` | Commande exécutée au démarrage du conteneur. |

**Pourquoi copier `requirements.txt` avant le reste du code ?** Chaque instruction crée une couche mise en cache. Si seul `app.py` change, Docker réutilise la couche des dépendances et ne réinstalle pas Flask, ce qui accélère les constructions suivantes.

**`RUN` ou `CMD` ?** `RUN` s'exécute une seule fois, à la construction de l'image. `CMD` s'exécute à chaque démarrage d'un conteneur.

### B3. Construction de l'image

```bash
cd ~/api
docker build -t api-fil-rouge:1.0 .
```

![docker build](img/b2-build.png)

Le build se déroule en 7 étapes, une par instruction du Dockerfile. Flask et ses dépendances (Werkzeug, Jinja2, click, itsdangerous, blinker, MarkupSafe) sont installés à l'étape 4. Il se termine par `Successfully built 18de320f18ec` puis `Successfully tagged api-fil-rouge:1.0`.

Les deux avertissements en rouge sont sans gravité : pip signale qu'il s'exécute en tant que `root` (normal dans un conteneur de construction) et qu'une nouvelle version de pip existe.

### B4. Lancement et test

```bash
docker rm -f api
docker run -d --name api -p 5000:5000 api-fil-rouge:1.0
curl http://localhost:5000
docker logs api
```

La commande `docker rm -f api` supprime un ancien conteneur du même nom : deux conteneurs ne peuvent pas porter le même nom.

![Lancement du conteneur et appel de l'API](img/b3-run-curl.png)

L'API répond : `{"message":"API operationnelle"}`.

Les journaux du conteneur confirment le fonctionnement :

![Journaux du conteneur api](img/b4-logs.png)

- Flask écoute sur toutes les adresses (`0.0.0.0`) : `127.0.0.1:5000` et `172.17.0.2:5000`, cette dernière étant l'adresse du conteneur sur le réseau interne de Docker.
- La requête `GET / HTTP/1.1` a reçu le code **200**.
- L'avertissement « development server » est normal : Flask intégré convient au développement, pas à la production, où l'on utiliserait un serveur WSGI comme Gunicorn.

### B5. Publication sur Docker Hub

**Connexion**

```bash
docker login
```

![docker login](img/b5-login.png)

La connexion par navigateur n'a pas démarré du premier coup (`failed to start web-based login - falling back to command line login`). Au second essai, Docker a affiché un code de confirmation à saisir sur `https://login.docker.com/activate`, puis `Login Succeeded`. Le compte Docker Hub ayant été créé avec Google, aucun mot de passe n'existe : l'authentification passe par le navigateur (ou par un jeton d'accès personnel).

Docker prévient aussi que les identifiants sont stockés non chiffrés dans `~/.docker/config.json`.

**Nommage et envoi**

```bash
docker tag api-fil-rouge:1.0 issiaka01/api-fil-rouge:1.0
docker images
docker push issiaka01/api-fil-rouge:1.0
```

`docker tag` ne copie rien : il ajoute un second nom à la même image. Un nom publiable sur Docker Hub doit commencer par l'identifiant du compte (`issiaka01/`).

![docker images et docker push](img/b6-images-push.png)

Lecture de la capture :
- `api-fil-rouge:1.0` et `issiaka01/api-fil-rouge:1.0` ont le **même identifiant** (`18de320f18ec`) : c'est la même image avec deux noms (198 MB sur disque, 48,3 MB de contenu).
- `docker push` envoie uniquement les couches nouvelles (`Pushed`). Les couches de l'image de base sont déjà présentes sur Docker Hub (`Mounted from library/python`, `Mounted from library/nginx`), ce qui évite de les retransmettre.
- Le message final donne l'empreinte de l'image : `digest: sha256:18de320f18ec22a1…` (taille 2054).

**Lien de l'image publiée :** https://hub.docker.com/r/issiaka01/api-fil-rouge

Pour la récupérer et la lancer depuis n'importe quelle machine disposant de Docker :
```bash
docker run -d -p 5000:5000 issiaka01/api-fil-rouge:1.0
```

---

## Conclusion

Docker permet d'empaqueter une application avec ses dépendances dans une image portable, qui s'exécute de la même façon sur toute machine disposant de Docker. Comparé à une machine virtuelle, un conteneur est plus léger (quelques dizaines de Mo de mémoire), démarre en environ une seconde et se partage facilement via un registre comme Docker Hub. En contrepartie, il partage le noyau de l'hôte, donc son isolation est moins forte.

Le Dockerfile rend la construction de l'image reproductible, et l'ordre de ses instructions (dépendances avant le code) permet de tirer parti du cache pour accélérer les constructions.
