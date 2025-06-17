# Répartition des Fonctionnalités par Technologie

Ce document a pour but de fournir une vue d'ensemble claire de quelle fonctionnalité est gérée par quelle technologie au sein de l'écosystème RAGFlow.

| Fonctionnalité | Technologie / Langage | Composants / Fichiers / Services Clés |
| :--- | :--- | :--- |
| **Interface Utilisateur (Frontend)** | **React, TypeScript, UmiJS** | `web/src/`, `web/.umirc.ts` |
| **Styling et Composants UI** | **Ant Design, TailwindCSS, Radix UI** | `web/src/components/`, `web/tailwind.config.js` |
| **Serveur Web & API REST** | **Python, Flask** | `api/ragflow_server.py`, `api/apps/` |
| **Logique Métier & Orchestration** | **Python** | `api/db/services/`, `agent/canvas.py` |
| **Base de Données Relationnelle** | **SQL (MySQL)** | Service Docker `mysql`, `api/db/db_models.py` |
| **Interaction avec la BDD (ORM)** | **Python, Peewee** | `api/db/db_models.py` |
| **Stockage & Recherche Vectorielle (RAG)** | **Elasticsearch / OpenSearch / Infinity** | Service Docker `es01` / `opensearch01` / `infinity` |
| **Analyse de Documents (Parsing)** | **Python, Java (via Tika)** | `deepdoc/parser/`, `tika-server-standard.jar` |
| **OCR & Analyse de mise en page** | **Python (libs C++)** | `deepdoc/vision/` (utilisant probablement des bibliothèques comme OpenCV) |
| **Stockage de Fichiers Bruts** | **MinIO (compatible S3)** | Service Docker `minio` |
| **Cache & Verrouillage Distribué** | **Redis / Valkey** | Service Docker `redis` |
| **Exécution de Code Sécurisé** | **Docker** | Service Docker `sandbox-executor-manager` |
| **Interaction avec les LLM / Embeddings** | **Python** | `rag/llm/`, `agent/component/llm.py` |
| **Traitement du Langage Naturel (NLP)** | **Python (NLTK, etc.)** | `rag/nlp/`, `download_deps.py` (téléchargement de `nltk_data`) |
| **Dépendances Python** | **Python, UV (Rust)** | `pyproject.toml`, `uv.lock` |
| **Dépendances Frontend** | **JavaScript, NPM** | `web/package.json`, `web/package-lock.json` |
| **Conteneurisation** | **Docker** | `Dockerfile`, `docker/docker-compose-*.yml` |
| **Déploiement en Production** | **Kubernetes, Helm** | `helm/` |

---

### Résumé par Langage/Technologie

- **Python**: C'est le langage principal pour tout le backend. Il gère la logique métier, l'API, l'interaction avec toutes les bases de données, l'orchestration des agents, l'analyse des documents et les appels aux modèles de langage. Les frameworks clés sont **Flask** pour le web et **Peewee** pour l'ORM.

- **TypeScript/JavaScript**: C'est le langage exclusif du frontend. Il est utilisé avec le framework **React** et la boîte à outils **UmiJS** pour construire l'intégralité de l'interface utilisateur.

- **SQL**: Utilisé par **MySQL** pour stocker et récupérer les données structurées. Le code Python interagit avec lui via l'ORM Peewee.

- **Java**: Utilisé indirectement via le serveur **Apache Tika** pour l'analyse de contenu de certains types de fichiers (notamment les documents Office).

- **Docker**: Fondamental pour l'architecture. Il définit l'environnement de chaque service et les fait communiquer entre eux. Il est également utilisé de manière dynamique par le `Sandbox Executor` pour l'exécution de code isolé.

- **Rust**: Utilisé comme outil de support pour l'environnement de développement Python via **UV**, un gestionnaire de paquets et d'environnements virtuels très performant. 