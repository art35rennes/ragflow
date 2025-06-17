# Guide de Développement Local

Ce guide a pour but de vous aider à configurer un environnement de développement pour RAGFlow sur votre machine locale. L'objectif est de reproduire aussi fidèlement que possible l'environnement défini dans les `Dockerfile` et `docker-compose` pour assurer la cohérence et minimiser les surprises lors du déploiement.

## Prérequis

Avant de commencer, vous devez avoir les outils suivants installés sur votre système :
1.  **Docker et Docker Compose**: Essentiel pour lancer l'infrastructure de services (bases de données, MinIO, etc.).
2.  **Git**: Pour cloner le projet.
3.  **Python 3.10**: La version de Python utilisée dans le projet. Il est fortement recommandé d'utiliser un gestionnaire de versions comme `pyenv` pour installer la version exacte.
4.  **Node.js 20.x**: La version de Node.js utilisée pour construire le frontend. Il est recommandé d'utiliser un gestionnaire de versions comme `nvm`.
5.  **Rust**: Requis par certaines dépendances. L'installation se fait via `rustup`.
6.  **Outils de build**: Un compilateur C/C++ et les outils de développement (`build-essential` sur Debian/Ubuntu, ou les "Command Line Tools for Xcode" sur macOS).

## Étape 1 : Cloner le Dépôt

Clonez votre fork du projet RAGFlow sur votre machine locale.
```bash
git clone <URL_de_votre_fork>
cd ragflow
```

## Étape 2 : Lancer l'Infrastructure de Base

Tous les services de support (bases de données, stockage, etc.) sont gérés par Docker Compose.
1.  **Naviguez vers le répertoire docker**:
    ```bash
    cd docker
    ```
2.  **Créez un fichier `.env`**: Le projet utilise un fichier `.env` pour configurer les services. Vous pouvez copier le fichier d'exemple s'il en existe un, ou créer le vôtre en vous basant sur les variables utilisées dans `docker-compose-base.yml` (par exemple, `MYSQL_PASSWORD`, `ES_PORT`, etc.).
3.  **Lancez les services**:
    ```bash
    docker-compose --profile elasticsearch up -d
    ```
    - L'argument `--profile elasticsearch` active la base de données vectorielle Elasticsearch, qui est une configuration courante. Vous pourriez choisir `opensearch` ou `infinity` à la place.
    - Le `-d` lance les conteneurs en arrière-plan.

À la fin de cette étape, vous devriez avoir MySQL, Elasticsearch, MinIO, et Redis qui tournent sur votre machine via Docker.

## Étape 3 : Télécharger les Dépendances Lourdes

Le projet dépend de plusieurs modèles de machine learning et d'autres gros fichiers. Le script `download_deps.py` est fourni pour les télécharger.

1.  **Revenez à la racine du projet**.
2.  **Créez un environnement virtuel Python** (bonne pratique) :
    ```bash
    python3.10 -m venv .venv
    source .venv/bin/activate
    ```
3.  **Installez les dépendances du script et exécutez-le**:
    ```bash
    pip install huggingface-hub nltk argparse
    python download_deps.py
    ```
    - Si vous êtes en Chine, utilisez `python download_deps.py --china-mirrors`.

Cette étape peut prendre beaucoup de temps et d'espace disque, car elle télécharge plusieurs gigaoctets de données dans des dossiers comme `huggingface.co/` et `nltk_data/`. Ces fichiers ne sont **pas** versionnés avec Git.

## Étape 4 : Configurer le Backend Python

Le backend utilise **UV** pour gérer les dépendances Python.

1.  **Installez UV**:
    ```bash
    pip install uv
    ```
2.  **Installez les dépendances du projet**:
    ```bash
    uv sync --python 3.10 --all-extras
    ```
    Cette commande va lire le fichier `uv.lock` et installer toutes les dépendances Python requises dans votre environnement virtuel.

3.  **Lancez le serveur backend**:
    Le point d'entrée est `ragflow_server.py`.
    ```bash
    python api/ragflow_server.py
    ```
    Le serveur devrait démarrer et se connecter aux services Docker que vous avez lancés à l'étape 2.

## Étape 5 : Configurer le Frontend

Le frontend est une application Node.js gérée avec `npm`.

1.  **Ouvrez un nouveau terminal**.
2.  **Naviguez vers le répertoire `web`**:
    ```bash
    cd web
    ```
3.  **Installez les dépendances NPM**:
    ```bash
    npm install
    ```
4.  **Lancez le serveur de développement**:
    ```bash
    npm run dev
    ```
    Le serveur de développement UmiJS démarrera. Il est configuré (via `.umirc.ts`) pour rediriger les appels API vers votre backend Flask qui tourne localement.

## Conclusion

À ce stade, vous devriez avoir :
- Les services d'infrastructure tournant dans Docker.
- Le serveur backend Flask tournant dans un terminal.
- Le serveur de développement frontend tournant dans un autre terminal.

Vous pouvez maintenant accéder à l'application dans votre navigateur (généralement à une adresse comme `http://localhost:8000`) et commencer à développer. 