# Stratégie d'Authentification SSO avec Microsoft (OAuth2)

L'objectif est de permettre aux utilisateurs de se connecter via leur compte Microsoft (Azure Active Directory), en complément de la connexion par email/mot de passe existante.

## Analyse de l'Existant

-   Le document [`authentication_flow.md`](../authentication_flow.md) détaille le fonctionnement de l'authentification actuelle par jeton.
-   L'architecture est prête à être étendue : l'utilisation de Blueprints dans Flask permet d'ajouter de nouvelles routes d'authentification de manière propre.
-   Le modèle `User` de Peewee peut être facilement étendu pour lier un compte externe.

## Stratégie de Mise en Œuvre

Le flux utilisé sera le **"Authorization Code Flow"**, qui est le standard sécurisé pour les applications web.

### 1. Enregistrement de l'Application sur Azure

-   Il faudra d'abord enregistrer une nouvelle "Application" dans le portail Azure AD.
-   Cet enregistrement fournira deux informations cruciales à conserver secrètement :
    -   `Client ID` (identifiant de l'application)
    -   `Client Secret` (mot de passe de l'application)
-   Lors de l'enregistrement, il faudra configurer une "URI de redirection". Celle-ci doit pointer vers une nouvelle route de notre backend Flask, par exemple `https://votre-domaine.com/api/v1/auth/microsoft/callback`.

### 2. Modifications du Backend (Flask)

-   **Ajouter des dépendances**:
    ```bash
    uv pip install authlib
    ```
    `Authlib` est une bibliothèque puissante et populaire pour gérer les clients et fournisseurs OAuth en Python.

-   **Stocker les secrets**: Le `Client ID` et le `Client Secret` devront être stockés de manière sécurisée (variables d'environnement, gestionnaire de secrets) et chargés dans les `settings.py`. Ne jamais les commiter dans le code Git.

-   **Créer un nouveau Blueprint d'authentification**:
    -   Créer un fichier `api/apps/auth/microsoft_app.py`.
    -   Ce fichier contiendra deux routes :
        1.  `/login/microsoft`: Cette route initie le processus. Elle utilise `Authlib` pour générer l'URL de connexion de Microsoft et y redirige l'utilisateur.
        2.  `/auth/microsoft/callback`: C'est l'URI de redirection. Microsoft redirigera l'utilisateur ici après une connexion réussie. Cette route :
            -   Récupère le `code` d'autorisation fourni par Microsoft.
            -   Utilise `Authlib` pour échanger ce code (avec le client secret) contre un `access_token` et un `id_token`.
            -   Décode l'`id_token` pour obtenir les informations de l'utilisateur (email, nom, etc.).
            -   Appelle un service (`UserService`) pour trouver ou créer l'utilisateur dans la base de données RAGFlow (logique de "provisionnement Just-In-Time").
            -   Génère un jeton d'accès RAGFlow interne (exactement comme pour la connexion par mot de passe).
            -   Redirige l'utilisateur vers le frontend avec ce jeton à stocker.

-   **Adapter le modèle `User`**:
    -   Il peut être utile d'ajouter un champ pour stocker l'identifiant de l'utilisateur Microsoft (`oid` ou `sub` du token) afin de faire le lien entre les comptes.

### 3. Modifications du Frontend (React)

-   **Ajouter un bouton de connexion**:
    -   Sur la page de connexion (`web/src/pages/login/index.tsx`), ajouter un bouton "Se connecter avec Microsoft".
    -   Ce bouton doit pointer vers la route de login du backend : `/api/v1/login/microsoft`.

-   **Gérer le callback**:
    -   Le backend redirigera l'utilisateur vers une page spéciale du frontend après le callback, en passant le jeton RAGFlow dans les paramètres de l'URL.
    -   Cette page devra simplement récupérer le jeton de l'URL, le stocker dans le `localStorage`, puis rediriger l'utilisateur vers la page d'accueil de l'application.

## Implications et Charge de Travail

-   **Complexité modérée**: La logique OAuth peut être complexe, mais l'utilisation de bibliothèques comme `Authlib` la simplifie grandement.
-   **Configuration Externe**: Nécessite un accès au portail Azure pour configurer l'application.
-   **Sécurité**: Essentiel de valider scrupuleusement les jetons, le `state` et d'utiliser le flow PKCE pour éviter les failles CSRF et de rejeu.
-   **Dépendances**: Ajout de `Authlib` ou une librairie similaire.
-   **Travail bien isolé**: Les modifications sont majoritairement contenues dans un nouveau Blueprint et quelques modifications mineures sur le frontend, ce qui limite l'impact sur le code existant.

## Considérations de Sécurité Spécifiques au SSO

- **Paramètre `state` obligatoire** : Chaque initiation de login doit générer un `state` aléatoire stocké côté serveur (session ou Redis) et vérifié lors du callback pour prévenir les attaques CSRF/OAuth.
- **Paramètre `nonce`** : Pour OpenID Connect, ajouter un `nonce` et le vérifier dans l'`id_token` afin d'empêcher le replay.
- **PKCE** : Activer PKCE (Proof-Key for Code Exchange) même pour les applications serveur pour se prémunir d'interceptions du `code`.
- **Scopes minimaux** : Demander uniquement `openid email profile`. Pas d'accès `offline_access` si les Refresh-Tokens ne sont pas nécessaires.
- **Stockage du token** : Ne jamais stocker l'`access_token` Microsoft côté frontend. Seul le jeton RAGFlow interne (à usage limité) est stocké dans le navigateur.
- **Rotation des Refresh-Tokens** : Si vous utilisez des Refresh-Tokens Microsoft, configurer la rotation et la révocation.
- **Validation d'audience (`aud`) et d'émetteur (`iss`)** : Vérifier que l'`id_token` provient bien de `https://login.microsoftonline.com/{tenant}/v2.0` et que l'audience correspond au `Client ID` enregistré.
- **Limitation des redirections** : La route `/login/microsoft` ne doit accepter qu'une `next` URL interne validée par whitelist afin d'éviter les open-redirect.
- **Déconnexion (Single Logout)** : Implémenter l'endpoint `/logout` pour révoquer le token interne et rediriger vers `https://login.microsoftonline.com/{tenant}/oauth2/v2.0/logout?post_logout_redirect_uri=...`.

## Stratégie de Maintenance & Impact Upstream

-   **Impact**: **Moyen**. L'essentiel du code est nouveau et isolé, mais la modification de la logique de connexion et du modèle `User` peut créer des conflits.
-   **Isolation du Code**:
    -   **Blueprint Flask**: Toute la logique backend du SSO (callback, login, logout) **doit** être contenue dans un Blueprint (`api/apps/sso_app.py`). Ce Blueprint n'est enregistré que si le SSO est activé.
    -   **Service dédié**: Créer un `SSOUserService` pour gérer la création/liaison des utilisateurs à partir des informations du jeton, plutôt que de modifier directement les services existants.
-   **Feature Flag**:
    -   Le `docker-compose.yml` doit contenir `ENABLE_SSO=true` pour activer la fonctionnalité.
    -   **Backend**: `ragflow_server.py` ne chargera le Blueprint SSO que si la variable est présente.
    -   **Frontend**: Le bouton "Login with Microsoft" ne s'affichera que si une variable d'environnement `REACT_APP_ENABLE_SSO` (transmise au build) est à `true`. 