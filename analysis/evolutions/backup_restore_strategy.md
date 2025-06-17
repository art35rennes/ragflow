# Stratégie de Sauvegarde et de Restauration (Backup & Restore)

L'objectif est de mettre en place et de documenter des procédures fiables pour sauvegarder les données critiques de l'application et être capable de les restaurer en cas d'incident (Disaster Recovery).

## Identification des Données Critiques

L'architecture de RAGFlow repose sur deux types principaux de stockage de données persistantes :

1.  **Base de Données Relationnelle (MySQL)**:
    -   **Contenu**: Toutes les métadonnées de l'application (utilisateurs, tenants, configurations des KB et des agents, historique des conversations, logs d'utilisation des jetons, etc.).
    -   **Criticité**: Très élevée. Une perte de ces données équivaut à une perte totale du service.

2.  **Stockage d'Objets (MinIO)**:
    -   **Contenu**: Tous les fichiers bruts uploadés par les utilisateurs (PDF, DOCX, etc.).
    -   **Criticité**: Élevée. Sans ces fichiers, les bases de connaissances sont inutilisables.

3.  **Base de Données Vectorielle (Elasticsearch, etc.) - Optionnel**:
    -   **Contenu**: Les index vectoriels (embeddings) des documents.
    -   **Criticité**: Moyenne. En théorie, ces index peuvent être reconstruits à partir des fichiers sources stockés dans MinIO et des configurations dans MySQL. Cependant, ce processus de ré-indexation peut être très long et coûteux en ressources de calcul. Une sauvegarde directe est donc préférable.

## Stratégie de Sauvegarde

### 1. Sauvegarde de MySQL

-   **Outil**: `mysqldump`, l'outil standard de MySQL.
-   **Procédure**: Un script `cron` sur l'hôte Docker (ou un conteneur dédié) exécute la commande suivante quotidiennement :
    ```bash
    #!/bin/bash
    TIMESTAMP=$(date +%F)
    BACKUP_DIR="/path/to/backups/mysql"
    
    docker exec ragflow-mysql mysqldump \
      -u root -p"$MYSQL_ROOT_PASSWORD" \
      --all-databases > "$BACKUP_DIR/ragflow-dump-$TIMESTAMP.sql"
    
    # Optionnel: compresser le dump
    gzip "$BACKUP_DIR/ragflow-dump-$TIMESTAMP.sql"
    ```
-   **Rétention**: Conserver les 7 derniers jours de sauvegardes.

### 2. Sauvegarde de MinIO

-   **Outil**: `mc` (MinIO Client).
-   **Procédure**: Utiliser la commande `mc mirror` pour répliquer le contenu du bucket MinIO vers un stockage externe (un autre S3, un NAS, etc.).
    ```bash
    #!/bin/bash
    # Configurer mc pour se connecter à la source (MinIO local)
    mc alias set local http://localhost:9000 MINIO_USER MINIO_PASSWORD
    
    # Configurer mc pour la destination
    mc alias set destination_s3 https://s3.some-region.amazonaws.com AWS_ACCESS_KEY AWS_SECRET_KEY
    
    # Lancer la synchronisation miroir
    mc mirror --overwrite local/ragflow-bucket/ destination_s3/ragflow-backup/
    ```
-   **Fréquence**: Quotidienne.

### 3. Sauvegarde d'Elasticsearch (Recommandé)

-   **Outil**: Snapshots natifs d'Elasticsearch.
-   **Procédure**:
    1.  Configurer un "repository" de snapshot dans Elasticsearch (qui peut pointer vers un partage NFS ou un bucket S3 via un plugin).
    2.  Créer des snapshots via l'API d'Elasticsearch.
    ```bash
    # Créer un snapshot de tous les index
    curl -X PUT "localhost:9200/_snapshot/my_backup_repo/snapshot_1?wait_for_completion=true"
    ```
-   **Fréquence**: Hebdomadaire (car reconstructible en cas de besoin).

## Stratégie de Restauration

La procédure de restauration doit être **documentée et testée** au moins une fois par trimestre.

1.  **Stopper l'application**.
2.  **Restaurer MinIO**: Inverser la commande `mc mirror` pour copier les données du stockage de sauvegarde vers le bucket MinIO.
3.  **Restaurer MySQL**:
    ```bash
    # Décompresser si besoin
    gunzip /path/to/backup.sql.gz
    
    # Importer le dump
    docker exec -i ragflow-mysql mysql -u root -p"$MYSQL_ROOT_PASSWORD" < /path/to/backup.sql
    ```
4.  **Restaurer Elasticsearch**: Restaurer le snapshot désiré via l'API.
5.  **Redémarrer l'application** et vérifier l'intégrité des données.

## Implications

-   **Charge de travail (2-3 jours)**: Principalement de l'écriture de scripts et de la configuration de `cron`.
-   **Coûts de stockage**: Le stockage des sauvegardes a un coût qui doit être pris en compte.
-   **Discipline**: La valeur des sauvegardes réside dans la discipline de les tester régulièrement. 