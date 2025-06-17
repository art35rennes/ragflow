# Stratégie de Sécurisation pour la Production

L'objectif est de passer en revue et de renforcer la sécurité de l'application à tous les niveaux pour une mise en production sereine.

## Analyse de l'Existant

-   L'architecture est déjà bien conçue avec une isolation par conteneurs (Docker) et un service de `sandbox-executor-manager` pour l'exécution de code à risque.
-   L'utilisation de `https/TLS` est prévue via Nginx.
-   La configuration est externalisée dans des fichiers `.env`.

## Stratégie de Renforcement

### 1. Gestion des Secrets

-   **Problématique**: Le stockage des secrets (mots de passe de BDD, clés d'API, client secrets OAuth) dans des fichiers `.env` n'est pas idéal pour la production.
-   **Solution**:
    -   **Docker Swarm / Kubernetes**: Utiliser les mécanismes natifs de gestion des secrets (`Docker Secrets` ou `Kubernetes Secrets`). Ces secrets sont montés en mémoire dans les conteneurs et ne sont pas stockés sur disque.
    -   **Vault (HashiCorp)**: Pour un niveau de sécurité supérieur, utiliser un gestionnaire de secrets externe comme Vault. L'application s'authentifierait auprès de Vault au démarrage pour récupérer ses secrets.

### 2. Sécurité des Dépendances

-   **Problématique**: Les dépendances (NPM et Python) peuvent contenir des vulnérabilités connues.
-   **Solution**: Mettre en place un pipeline de CI/CD qui inclut des étapes de scan de sécurité.
    -   **Scan de code statique (SAST)**: Utiliser des outils pour détecter les mauvaises pratiques de sécurité dans le code source.
    -   **Analyse de composition logicielle (SCA)**: Utiliser des outils comme **`Snyk`**, **`Dependabot`** (intégré à GitHub), ou **`Trivy`** pour scanner `uv.lock` et `package-lock.json` à la recherche de vulnérabilités connues dans les dépendances. Bloquer le build si des vulnérabilités critiques sont trouvées.

### 3. Sécurité des Conteneurs

-   **Problématique**: Les images Docker de base peuvent contenir des failles de sécurité.
-   **Solution**:
    -   **Scan d'images**: Intégrer un scanner d'images comme **`Trivy`** ou **`Clair`** dans le pipeline de CI/CD pour analyser les images Docker après leur construction.
    -   **Principe du moindre privilège**: Revoir les `Dockerfile` pour s'assurer que les conteneurs tournent avec des utilisateurs non-root (`USER ragflow`) partout où c'est possible.
    -   **Images "Distroless" / minimalistes**: Pour l'image de production finale, envisager d'utiliser une image de base plus petite (comme les images `distroless` de Google ou `python:3.10-slim`) qui ne contient que le strict nécessaire pour exécuter l'application, réduisant ainsi la surface d'attaque.

### 4. Sécurité Applicative (API)

-   **Protection contre le Brute-Force**: Mettre en place un rate-limiting sur les endpoints sensibles, en particulier `/api/v1/login`. La bibliothèque `Flask-Limiter` est une excellente option.
-   **En-têtes de Sécurité HTTP**: Configurer Nginx pour qu'il ajoute des en-têtes de sécurité aux réponses HTTP, comme `Content-Security-Policy` (CSP), `Strict-Transport-Security` (HSTS), `X-Frame-Options`, etc.
-   **Validation des Entrées**: Le projet utilise déjà un ORM qui protège contre les injections SQL, mais il faut s'assurer que toutes les entrées (paramètres d'URL, corps des requêtes) sont systématiquement validées (type, longueur, format), potentiellement avec une bibliothèque comme `Pydantic` ou `Flask-Marshmallow`.

### 5. Pare-feu Applicatif Web (WAF)

-   **Problématique**: Protection contre les attaques web courantes (XSS, CSRF, Injection SQL, etc.).
-   **Solution**: Déployer un WAF devant Nginx. Les fournisseurs de cloud (AWS, GCP, Azure) proposent des services de WAF managés. Il existe aussi des solutions open-source comme `ModSecurity`.

### 6. Remplacer le Serveur de Développement (Point Critique)

-   **Problématique (identifiée lors de l'analyse)**: L'application est lancée via `werkzeug.serving.run_simple` dans `api/ragflow_server.py`. **Ceci est un serveur de développement et ne doit JAMAIS être utilisé en production.** Il est lent, mono-threadé par défaut, et ne dispose pas des protections nécessaires contre les attaques de déni de service.
-   **Solution**: Remplacer Werkzeug par un serveur d'application WSGI de production comme **Gunicorn**. Gunicorn est le standard de l'industrie pour le déploiement d'applications Flask.

-   **Stratégie de mise en œuvre**:
    1.  **Ajouter la dépendance**: Ajouter Gunicorn aux dépendances du projet.
        ```bash
        uv pip install gunicorn
        ```
    2.  **Modifier le point d'entrée du conteneur**: Le `Dockerfile` utilise `entrypoint.sh` pour lancer l'application. Il faut modifier ce script pour qu'il lance Gunicorn au lieu de `python api/ragflow_server.py`. Gunicorn se chargera d'importer et de servir l'objet `app` depuis `api.apps`.
    3.  **Exemple de commande Gunicorn dans `entrypoint.sh`**:
        ```bash
        #!/bin/bash
        # ... autres logiques du script ...

        # Lancer Gunicorn au lieu du serveur de dev Flask
        exec gunicorn 'api.apps:app' \
            --bind '0.0.0.0:8000' \
            --workers 4 \
            --timeout 120
        ```
        -   `api.apps:app`: Indique à Gunicorn où trouver l'objet application Flask (`app` dans le module `api.apps`).
        -   `--bind '0.0.0.0:8000'`: Gunicorn écoute sur le port 8000 à l'intérieur du conteneur. Nginx redirigera ensuite le trafic vers ce port.
        -   `--workers 4`: Lance 4 processus "workers" pour gérer les requêtes en parallèle. Le nombre de workers est un paramètre de performance clé.
        -   `--timeout 120`: Augmente le délai d'attente pour les requêtes longues, ce qui peut être nécessaire pour les opérations de RAG.

-   **Implications**:
    -   Cette modification est **essentielle et non négociable** pour un passage en production.
    -   Elle améliorera considérablement les performances et la stabilité de l'application.
    -   Le travail est relativement simple et très bien isolé au niveau de la configuration du déploiement.

### 7. CORS Trop Permissif

- **Problématique** : `flask_cors.CORS(app, supports_credentials=True)` sans restriction d'origines autorise n'importe quel site à appeler l'API avec des cookies/tokens.
- **Correctif** :
  1. Limiter l'option `origins` à la liste exacte de vos domaines (ex.: `origins=['https://app.exemple.com']`).
  2. Désactiver `supports_credentials` sauf nécessité absolue.
  3. Ajouter un middleware d'audit qui loggue les Origin reçus pour détecter les tentatives suspectes.

### 8. Absence de Protection CSRF

- **Problématique** : Les requêtes POST authentifiées peuvent être forgées depuis un site tiers.
- **Correctif** :
  1. Passer à un schéma « stateless » strict : toutes les requêtes portent un header `Authorization: Bearer <token>` et **aucun cookie d'auth** n'est envoyé.
  2. Ou, si des cookies sont indispensables, intégrer `Flask-WTF` ou `Flask-Seasurf` pour générer un jeton CSRF synchronisé.

### 9. Exposition de Ports Internes

- **Problématique** : `docker-compose-base.yml` expose MySQL, Redis, MinIO, Elasticsearch/OpenSearch/Infinity sur l'hôte.
- **Correctif** :
  1. Supprimer les directives `ports:` pour ces services en production et basculer vers un réseau Docker interne.
  2. Seul Nginx (port 443) reste exposé.
  3. Pour l'accès admin : ouvrir un tunnel VPN/bastion ou `docker exec`.

### 10. Taille d'Upload Excessive

- **Problématique** : `MAX_CONTENT_LENGTH = 1 Go` ➜ DoS (RAM, disque).
- **Correctif** :
  1. Réduire la limite (ex.: 50 Mo) ; ajouter une white-list MIME.
  2. Uploader directement vers MinIO en mode multipart (pré-signed URL) pour ne pas transiter par la RAM du backend.

### 11. Exécution SQL Libre dans `agent/component/exesql.py`

- **Problématique** : `cursor.execute(single_sql)` sans sanitation ➜ injection SQL si l'utilisateur peut injecter du DSL.
- **Correctif** :
  1. Restreindre la liste des commandes SQL autorisées (SELECT ... LIMIT N).  
  2. Implémenter un parseur SQL (ex.: `sqlparse`) et rejeter toute clause dangereuse (INSERT, UPDATE, DROP, ; etc.).  
  3. Sinon, supprimer le composant.

### 12. Sandbox Executor `privileged:true`

- **Problématique** : Un conteneur privilégié peut échapper au host si un exploit Docker est trouvé.
- **Correctif** :
  1. Retirer `privileged:true`; appliquer un profil `seccomp` strict, capabilities limitées et `no-new-privileges:true`.
  2. Activer AppArmor/SELinux.

## Implications et Charge de Travail

-   **Élevée**: La sécurisation est un processus continu et non une tâche unique.
-   **Expertise Requise**: Nécessite des compétences en sécurité applicative (AppSec) et en sécurité des infrastructures (DevSecOps).
-   **Impact sur le CI/CD**: La mise en place de scans automatisés va modifier en profondeur le pipeline de construction et de déploiement.
-   **Charge de travail (5 jours)**: L'effort combine de la configuration d'infrastructure (Gunicorn, Nginx), de la configuration d'application (CORS, CSRF) et potentiellement de légères modifications de code pour la validation.
-   **Indispensable**: Ces mesures ne sont pas optionnelles pour un environnement de production.

## Stratégie de Maintenance & Impact Upstream

-   **Impact**: **Faible**. La plupart de ces changements concernent la configuration de l'infrastructure (fichiers `docker-compose.yml`, configuration Nginx, scripts de démarrage) et non le code source Python ou JavaScript de l'application.
-   **Isolation du Code**:
    -   **Fichiers de surcharge Docker Compose**: Utilisez des fichiers de surcharge comme `docker-compose.override.yml` pour injecter vos configurations de production (Gunicorn, variables d'environnement, limites de ressources). Ce fichier n'est pas versionné par `upstream` et ne créera jamais de conflit.
    -   **Configuration Nginx**: La configuration du reverse proxy est entièrement externe à l'application RAGFlow. Elle est donc isolée par nature.
    -   **Dépendances de sécurité**: L'ajout de `flask-talisman` ou `flask-limiter` se fait dans `requirements.txt` et leur initialisation dans `ragflow_server.py`. Ces ajouts peuvent être encapsulés dans une fonction `setup_production_middlewares(app)` appelée uniquement si un flag `ENABLE_HARDENING=true` est actif. 