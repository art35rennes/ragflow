# Stratégie d'Internationalisation (i18n) en Français

L'objectif est de traduire l'intégralité de l'interface et potentiellement les communications du backend en français.

## Analyse de l'Existant

-   **Frontend**: Le projet utilise déjà `i18next` et `react-i18next`, comme vu dans `web/package.json`. C'est une excellente base, l'architecture est prête pour l'i18n.
-   **Backend**: Le modèle `User` contient un champ `language`. Cependant, le code backend ne semble pas l'utiliser pour traduire les messages d'erreur ou autres chaînes de caractères renvoyées par l'API.

## Stratégie de Mise en Œuvre

### 1. Frontend

La tâche principale se situe ici.

-   **Créer les fichiers de traduction**:
    -   Dans le répertoire `web/src/locales/`, il faudra créer un nouveau dossier `fr`.
    -   À l'intérieur, créer des fichiers JSON (par ex. `translation.json`) qui contiendront les traductions. La structure devra être une copie conforme des fichiers de la langue par défaut (probablement `en`).
    -   Exemple de `web/src/locales/fr/translation.json`:
        ```json
        {
          "login": {
            "title": "Connexion",
            "email_placeholder": "Adresse e-mail"
          },
          "knowledge": {
            "title": "Bases de connaissances"
          }
        }
        ```
-   **Utiliser les traductions dans le code**:
    -   Identifier toutes les chaînes de caractères codées en dur dans les composants React (`.tsx`).
    -   Les remplacer par le hook `useTranslation` fourni par `react-i18next`.
    -   Exemple:
        ```typescript
        // Avant
        return <h1>Login</h1>;

        // Après
        import { useTranslation } from 'react-i18next';
        // ...
        const { t } = useTranslation();
        return <h1>{t('login.title')}</h1>;
        ```
-   **Sélecteur de langue**:
    -   Le `UserSetting` (`/user-setting/locale`) semble déjà avoir une section pour cela. Il faudra s'assurer qu'elle est bien fonctionnelle et qu'elle met à jour la configuration `i18next` ainsi que le champ `language` du modèle `User` via un appel API.

### 2. Backend

Bien que moins prioritaire, pour une expérience utilisateur parfaite, les messages d'erreur de l'API devraient aussi être traduits.

-   **Intégrer Flask-Babel**: C'est la bibliothèque standard pour l'i18n dans Flask.
-   **Configurer Flask-Babel**: L'initialiser pour qu'il utilise la langue stockée dans `g.user.language` (Flask-Login stocke l'utilisateur actuel dans `g`).
-   **Marquer les chaînes à traduire**: Utiliser la fonction `gettext` (souvent importée comme `_`) pour marquer les chaînes de caractères dans le code Python qui doivent être traduites.
    -   Exemple : `raise Exception(_('Invalid credentials'))`
-   **Générer les fichiers de traduction**: Utiliser les commandes en ligne de commande de Flask-Babel pour extraire les chaînes marquées et créer les fichiers de traduction `.po` et `.mo`.

## Implications et Charge de Travail

-   **Charge de travail (5 jours)**: L'effort principal n'est pas technique mais réside dans l'extraction et la traduction de toutes les chaînes de l'interface.
-   **Impact**: Élevé sur le frontend car il touche de nombreux composants.

## Considérations de Sécurité (i18n)

- **Injection de format/translation** : Éviter les traductions contenant des placeholders non contrôlés (`{0}`, `%s`) concaténés directement. Toujours passer par les fonctions de format de `i18next` qui échappent les valeurs.
- **XSS via fichiers de traduction** : Les fichiers JSON sont sous contrôle du dépôt ; toutefois, si un serveur de traduction externe est utilisé plus tard, s'assurer de filtrer tout contenu HTML.
- **Locale spoofing** : Le backend doit valider la valeur envoyée par le client (`Accept-Language` ou champ `language` du profil) contre une liste blanche (`['en', 'fr']`).

## Stratégie de Maintenance & Impact Upstream

-   **Impact**: **Élevé**. C'est l'une des évolutions les plus susceptibles de créer des conflits de fusion, car elle modifie un grand nombre de fichiers de l'interface utilisateur qui sont souvent mis à jour par le projet `upstream`.
-   **Isolation du Code**:
    -   **Fichiers de langue dédiés**: Le fichier de traduction `fr.json` est un nouveau fichier, il ne créera donc pas de conflit direct.
    -   **Wrapper de composant**: Plutôt que d'éditer chaque composant, créer un composant `TranslatableText` qui prend une clé de traduction. Cela limite les modifications, mais nécessite une refactorisation importante. Une approche plus pragmatique est d'accepter les modifications directes.
-   **Stratégie de Fusion (Merge)**:
    -   Lors d'une mise à jour depuis `upstream`, les conflits sur les fichiers de composants devront être résolus manuellement.
    -   **Script de validation**: Un script peut être créé pour vérifier que toutes les clés de traduction utilisées dans le code existent bien dans `en.json` et `fr.json`, ce qui aide à détecter les clés manquantes après une fusion.
    -   Il est crucial de ne **pas** modifier les fichiers de langue `en.json` de `upstream`. Les ajouts doivent aller dans un fichier `en_custom.json` qui est fusionné au build. 