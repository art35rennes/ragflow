# Stratégie d'Observabilité et de Suivi

L'objectif est de mettre en place les outils nécessaires pour comprendre le comportement de l'application en production, diagnostiquer les problèmes, et suivre les métriques clés (y compris les coûts liés aux LLM). L'observabilité repose sur trois piliers : les **Logs**, les **Métriques** et les **Traces**.

## 1. Logs : Que s'est-il passé ?

-   **Problématique actuelle**: Les logs sont actuellement émis sur la sortie standard des conteneurs, ce qui est suffisant pour le développement mais inutilisable en production pour l'analyse.
-   **Stratégie**:
    1.  **Logs Structurés**: Modifier la configuration du logging en Python (dans `api/utils/log_utils.py`) pour émettre des logs au format JSON. Cela permet de les parser et de les requêter facilement.
    2.  **Centralisation des Logs**: Déployer une pile de centralisation des logs. Étant donné que le projet supporte déjà Elasticsearch, la pile **ELK (Elasticsearch, Logstash, Kibana)** est un choix naturel. Une alternative plus légère est la pile **Grafana Loki**. Les logs de tous les conteneurs (backend, Nginx, BDD...) seraient collectés et envoyés vers ce système.

## 2. Métriques : Comment se porte le système ?

-   **Problématique actuelle**: Aucune métrique de performance n'est exposée par l'application.
-   **Stratégie**:
    1.  **Exposition des Métriques**: Intégrer la bibliothèque `prometheus-flask-exporter` dans l'application Flask. Elle exposera automatiquement un endpoint `/metrics` avec des métriques de base (latence des requêtes, nombre de requêtes par endpoint, codes de statut HTTP...).
    2.  **Collecte et Visualisation**: Déployer un serveur **Prometheus** pour "scraper" (collecter) régulièrement les métriques de tous les services. Utiliser **Grafana** pour se connecter à Prometheus et créer des tableaux de bord pour visualiser l'état de santé de l'application en temps réel (taux d'erreur, latence, utilisation CPU/mémoire des conteneurs...).

## 3. Traces : Où le temps est-il passé ?

-   **Problématique actuelle**: Dans une architecture de microservices, il est difficile de suivre une requête qui passe à travers plusieurs services (Nginx -> Backend -> BDD).
-   **Stratégie**:
    1.  **Instrumentation du Code**: Intégrer le standard **OpenTelemetry**.
        -   Dans le backend Flask, utiliser l'instrumentation automatique d'OpenTelemetry pour capturer les traces des requêtes entrantes, des appels à la base de données, etc.
        -   Pour une vue complète, on peut même l'intégrer au frontend pour tracer le parcours depuis le navigateur de l'utilisateur.
    2.  **Collecteur et Visualisation**: Envoyer ces traces à un collecteur OpenTelemetry, qui les exportera ensuite vers un système de traçage distribué comme **Jaeger** ou **Zipkin** pour la visualisation.

## 4. Suivi Spécifique au Métier : Coût des LLM

Ceci est un besoin de suivi spécifique et crucial pour ce type d'application.

-   **Problématique actuelle**: Le coût des appels aux API des LLM n'est pas suivi.
-   **Stratégie**:
    1.  **Créer un Modèle de Données**: Ajouter une nouvelle table dans la base de données MySQL, par exemple `TokenUsageLog`, via un nouveau modèle dans `api/db/db_models.py`. Cette table contiendrait des colonnes comme : `id`, `user_id`, `tenant_id`, `model_name`, `input_tokens`, `output_tokens`, `cost`, `timestamp`.
    2.  **Logique d'Enregistrement**: Modifier les clients LLM dans `rag/llm/` et `agent/component/llm.py`. Après chaque appel réussi à une API externe, en plus de renvoyer la réponse, il faudra créer une nouvelle entrée dans la table `TokenUsageLog` avec les informations sur la consommation de jetons. La plupart des API LLM renvoient cette information dans leur réponse.
    3.  **Tableaux de Bord**: Créer des tableaux de bord (dans l'application RAGFlow elle-même ou dans Grafana en se connectant à la BDD MySQL) pour visualiser la consommation par utilisateur, par tenant ou par modèle.

## Implications

-   **Infrastructure supplémentaire**: Nécessite le déploiement de nouveaux services (Prometheus, Grafana, Jaeger, etc.).
-   **Instrumentation**: Demande un travail de modification du code, en particulier pour le suivi des jetons.
-   **Valeur ajoutée immense**: C'est un investissement indispensable pour opérer une application complexe en production de manière fiable et contrôlée.

## Considérations de Sécurité pour l'Observabilité

- **Données sensibles dans les logs** : Mettre en place un filtre de log pour masquer ou hacher les PII (emails, tokens). Exemple : middleware Flask qui remplace toute correspondance avec regex `[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+` par `[EMAIL]`.
- **Protection de l'endpoint `/metrics`** : Exposer ce chemin uniquement sur le réseau interne Docker/K8s et/ou le protéger par Basic-Auth ou mTLS pour empêcher une fuite d'informations.
- **Traçage distribuée & RGPD** : S'assurer que les spans OpenTelemetry ne contiennent pas de payload utilisateur (messages, prompts). Utiliser un sampler pour réduire la quantité de données.
- **RBAC Grafana/Jaeger** : Activer l'authentification Grafana et Jaeger avec un SSO ou un annuaire, limiter les rôles « viewer » vs « admin ».

## 5. Piste d'Audit (Audit Trail)

En plus de l'observabilité technique, il est crucial pour la conformité et la sécurité de savoir **"qui a fait quoi"**.

-   **Problématique actuelle**: Les actions des utilisateurs (création de KB, changement de permissions, etc.) ne sont pas tracées de manière persistante et interrogeable.
-   **Stratégie**:
    1.  **Créer un Modèle de Données `AuditLog`**:
        -   Dans `api/db/db_models.py`, ajouter un nouveau modèle Peewee.
        ```python
        class AuditLog(DataBaseModel):
            user_id = CharField(max_length=32, index=True)
            tenant_id = CharField(max_length=32, index=True)
            action = CharField(max_length=128, index=True) # Ex: 'KB_CREATE', 'USER_LOGIN_SUCCESS'
            details = JSONField(null=True) # Ex: {'kb_id': 'xyz', 'kb_name': 'Rapport Annuel'}
            
            class Meta:
                db_table = "audit_log"
        ```
        -   Générer et appliquer la migration de la base de données.

    2.  **Créer un Service d'Audit**:
        -   Créer un service `AuditService` dans `api/db/services/` avec une méthode statique `log()`.
        ```python
        from flask_login import current_user

        class AuditService:
            @staticmethod
            def log(action, details=None):
                if not current_user.is_authenticated:
                    return

                AuditLog.create(
                    user_id=current_user.id,
                    tenant_id=current_user.tenant_id, # Assumant que l'utilisateur est lié à un tenant
                    action=action,
                    details=details or {}
                )
        ```

    3.  **Instrumenter le Code Métier**:
        -   Identifier les opérations critiques dans les services et y ajouter des appels à `AuditService.log()`.
        -   **Exemples d'actions à tracer**:
            -   Connexion réussie / échouée.
            -   CRUD sur les `Knowledgebase`.
            -   CRUD sur les `User`.
            -   Changement de rôle d'un utilisateur.
            -   Génération/révocation de clés d'API.

-   **Implications**:
    -   **Charge de travail (3-4 jours)**: L'effort principal consiste à identifier les points d'instrumentation et à y ajouter les appels de log.
    -   **Stockage**: La table `audit_log` peut grandir rapidement. Une politique de rétention (archivage ou suppression après 1 an, par exemple) est nécessaire.
    -   **Sécurité et Conformité**: C'est une exigence non-négociable pour de nombreuses certifications (ISO 27001, SOC 2, etc.).

## Stratégie de Maintenance & Impact Upstream (Observabilité)

-   **Impact**: **Faible à Moyen**.
    -   **Faible** pour la partie Logs & Métriques, qui repose sur des standards et des configurations externes.
    -   **Moyen** pour la Piste d'Audit, qui ajoute un nouveau modèle de données et des appels de service dans le code métier.
-   **Isolation du Code**:
    -   **Piste d'Audit**: Le modèle `AuditLog` et le service `AuditService` sont des ajouts nets, sans modification du code `upstream`. Ils ne créeront pas de conflits. Les appels `AuditService.log()` sont des ajouts sur une seule ligne et sont faciles à gérer lors des fusions.
    -   **Instrumentation (OpenTelemetry)**: L'instrumentation doit être activée via une configuration et non codée en dur. Les décorateurs sont une bonne pratique pour ajouter des traces de manière non invasive.
-   **Feature Flag**:
    -   L'activation de l'instrumentation OpenTelemetry et de la journalisation d'audit doit être contrôlée par des variables d'environnement (`ENABLE_TRACING=true`, `ENABLE_AUDIT_LOG=true`).
    -   Cela permet de désactiver ces fonctionnalités si elles causent des problèmes de performance ou des conflits après une mise à jour. 