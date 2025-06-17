# Stratégie de Contrôle d'Accès Basé sur les Rôles (RBAC)

L'objectif est de mettre en place un système de rôles simple pour restreindre l'accès à certaines fonctionnalités et données, une exigence fondamentale pour un usage en entreprise.

## Analyse de l'Existant

-   Le projet gère les utilisateurs et l'authentification.
-   Le modèle `UserTenant` définit un rôle, mais il semble lié à la gestion d'équipe et non à des permissions applicatives globales.
-   Il n'existe pas de mécanisme de restriction d'accès basé sur des rôles au niveau de l'API.

## Stratégie de Mise en Œuvre

Nous allons implémenter un système de rôles simple avec deux niveaux au départ : `admin` et `user`.

### 1. Modifications du Backend

-   **Adapter le modèle `User`**:
    -   Ajouter un champ `role` au modèle `User` dans `api/db/db_models.py`.
    ```python
    class User(DataBaseModel, UserMixin):
        # ... champs existants
        role = CharField(max_length=32, default="user", index=True) # Ajout du champ 'role'
    ```
    -   Générer et appliquer la migration de la base de données.

-   **Créer un décorateur de permission**:
    -   Dans un fichier utilitaire (ex: `api/utils/api_utils.py`), créer un décorateur Python qui vérifiera le rôle de l'utilisateur connecté.
    ```python
    from functools import wraps
    from flask_login import current_user
    from flask import abort

    def admin_required(fn):
        @wraps(fn)
        def wrapper(*args, **kwargs):
            if not current_user.is_authenticated or current_user.role != 'admin':
                abort(403)  # Forbidden
            return fn(*args, **kwargs)
        return wrapper
    ```

-   **Protéger les routes de l'API**:
    -   Appliquer le nouveau décorateur `@admin_required` aux routes de l'API qui ne doivent être accessibles qu'aux administrateurs.
    -   Exemple dans un `*_app.py`:
    ```python
    from api.utils.api_utils import admin_required

    @manager.route('/system/settings', methods=['POST'])
    @login_required
    @admin_required
    def update_system_settings():
        # ... code de la fonction ...
    ```

-   **API pour la gestion des rôles**:
    -   Créer un nouvel endpoint (protégé par `@admin_required`) permettant à un admin de changer le rôle d'un autre utilisateur.

### 2. Modifications du Frontend

-   **Récupérer le rôle de l'utilisateur**:
    -   L'API de login ou de profil (`/api/v1/user/profile`) doit maintenant renvoyer le champ `role` de l'utilisateur.
    -   Stocker ce rôle dans le store d'état du frontend (Zustand).

-   **Masquer les éléments d'interface**:
    -   Utiliser le rôle stocké pour masquer conditionnellement les boutons, liens ou sections de l'interface qui mènent à des fonctionnalités d'administration.
    -   Exemple en React:
    ```jsx
    // Dans un composant
    const userRole = useUserStore(state => state.role);

    return (
      <nav>
        <Link to="/dashboard">Dashboard</Link>
        {userRole === 'admin' && <Link to="/admin/settings">Admin Settings</Link>}
      </nav>
    );
    ```

## Implications et Charge de Travail

-   **Charge de travail (4-5 jours)**:
    -   Backend (2j) : Modification du modèle, création du décorateur, protection des routes initiales.
    -   Frontend (2j) : Adaptation de l'état global et masquage conditionnel des éléments d'UI.
    -   Tests (1j) : Vérification que les permissions sont bien appliquées.
-   **Impact**: Faible si bien implémenté. C'est une fonctionnalité additive.
-   **Sécurité**: Améliore considérablement la sécurité en appliquant le principe du moindre privilège.

## Stratégie de Maintenance & Impact Upstream

-   **Impact**: **Moyen**. La modification du modèle `User` crée un point de conflit potentiel mais gérable avec les migrations.
-   **Isolation du Code**:
    -   Le décorateur `@admin_required` doit être dans un fichier utilitaire dédié (`api/utils/custom_auth_utils.py`) pour ne pas modifier les fichiers core.
    -   Les nouvelles routes pour la gestion des rôles doivent être dans un **Blueprint Flask séparé** (`api/apps/rbac_app.py`) pour éviter les conflits de routage.
-   **Feature Flag**:
    -   L'application du décorateur peut être rendue conditionnelle via une variable d'environnement `ENABLE_RBAC=true`.
    ```python
    # api/utils/custom_auth_utils.py
    import os

    def admin_required(fn):
        if os.environ.get('ENABLE_RBAC') != 'true':
            return fn # Ne fait rien si le RBAC n'est pas activé
        # ... reste de la logique ...
    ```
    -   Cela permet de désactiver rapidement la fonctionnalité en cas de problème après une mise à jour. 