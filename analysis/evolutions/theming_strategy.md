# Stratégie de Thématisation et Personnalisation Visuelle

L'objectif est de permettre une personnalisation simple de l'apparence de l'application (logo, couleurs primaires, etc.) pour qu'elle corresponde à une charte graphique spécifique (white-labeling).

## Analyse de l'Existant

-   **Frontend**: L'utilisation de **Tailwind CSS** est le pilier de cette stratégie. Tailwind est conçu pour être personnalisable. Le fichier `web/tailwind.config.js` est le point d'entrée central pour toute la configuration visuelle. La bibliothèque `next-themes` est également présente, ce qui suggère une capacité de basculement de thème (ex: clair/sombre) déjà intégrée.
-   **Backend**: Certains éléments visuels, comme les avatars des bases de connaissances ou des agents, sont stockés directement en base de données (souvent en base64). Ce n'est pas idéal pour une thématisation dynamique.

## Stratégie de Mise en Œuvre

### 1. Personnalisation des Couleurs et de la Typographie

Toute la personnalisation de base se fera dans `web/tailwind.config.js`.

-   **Couleurs**: Le fichier de configuration de Tailwind permet de définir ou d'étendre la palette de couleurs. Il suffira de modifier les couleurs primaires, secondaires, etc., pour qu'elles correspondent à la nouvelle charte graphique.
    ```javascript
    // web/tailwind.config.js
    module.exports = {
      theme: {
        extend: {
          colors: {
            primary: '#00A4A6', // Nouvelle couleur primaire
            secondary: '#FF7A5A', // Nouvelle couleur secondaire
          },
        },
      },
      // ...
    };
    ```
    Il faudra ensuite s'assurer que les composants utilisent ces variables (`bg-primary`, `text-secondary`, etc.) plutôt que des couleurs codées en dur.

-   **Typographie**: De la même manière, les polices de caractères peuvent être changées via le même fichier de configuration.

### 2. Personnalisation du Logo

-   **Logo principal**: Le logo est probablement un simple fichier image. Il suffira de le remplacer dans le code source. Les emplacements les plus probables sont :
    -   `web/public/logo.svg` (ou .png)
    -   `web/src/assets/logo.svg` (ou .png)
    Une recherche globale dans le code pour "logo.svg" confirmera l'emplacement exact.

### 3. Personnalisation Avancée (via le Backend)

Pour une personnalisation plus poussée et dynamique (par exemple, par tenant), une approche plus complexe est nécessaire.

-   **Créer un endpoint de configuration**:
    -   Développer une nouvelle route sur le backend Flask, par exemple `/api/v1/theme/config`, qui renverrait un objet JSON avec les paramètres du thème (couleurs primaires, URL du logo, etc.).
    -   Ces paramètres pourraient être stockés dans la table `Tenant` de la base de données.

-   **Charger la configuration au démarrage du Frontend**:
    -   Au chargement de l'application React, faire un appel à ce nouvel endpoint.
    -   Utiliser les informations reçues pour :
        -   Injecter dynamiquement les couleurs dans le DOM via des variables CSS.
        -   Afficher l'URL du logo reçue au lieu d'un logo statique.

-   **Refactorisation des avatars**:
    -   Modifier les parties du code qui stockent des images en base64. L'idéal serait de stocker ces images dans **MinIO** et de ne sauvegarder que leur URL dans la base de données relationnelle (MySQL). Cela allège la base de données et rend la gestion des images plus flexible.

## Implications et Charge de Travail

-   **Personnalisation de base (Facile)**: Changer les couleurs et le logo via la configuration de Tailwind et le remplacement de fichiers est très rapide et à faible impact.
-   **Personnalisation avancée (Moyenne)**: Mettre en place une configuration de thème dynamique via le backend demande plus de travail (modification de la BDD, création d'API, logique de chargement dans le frontend) mais offre une flexibilité maximale. C'est la voie à suivre pour une solution multi-tenant entièrement "white-label".
-   **Refactorisation des avatars (Modérée)**: Changer la manière dont les avatars sont stockés est un travail de refactorisation qui touchera plusieurs parties du code (upload, affichage), mais c'est une amélioration technique saine.
-   **Charge de travail (3 jours)**: Le travail est relativement simple s'il se limite aux couleurs et au logo.
-   **Impact**: Élevé sur les fichiers de style, ce qui peut engendrer des conflits de fusion lors des mises à jour.

## Considérations de Sécurité pour la Thématisation

- **Téléversement de logos** : Valider le type MIME (image/png, image/svg+xml). Pour les SVG, supprimer tout script ou élément `onload` → utiliser `SVGO --disable=removeUnknownsAndDefaults --enable=cleanupIDs`.
- **CSS Injection** : Si le backend permet de sauvegarder des couleurs hexadécimales dans la BDD, valider via regex `^#?[0-9a-fA-F]{3,6}$` pour éviter l'injection CSS/JS dans des attributs de style.
- **Séparation des tenants** : Les objets (logo, config thème) doivent être isolés par `tenant_id` dans MinIO et MySQL pour empêcher un locataire d'écraser le thème d'un autre.

## Stratégie de Maintenance & Impact Upstream

-   **Impact**: **Élevé**. Tout comme l'i18n, la thématisation touche à des fichiers (CSS, config Tailwind) qui sont fréquemment modifiés par `upstream`.
-   **Isolation du Code**:
    -   **CSS Overrides**: La meilleure approche est de ne **jamais modifier directement** les fichiers CSS ou Less de l'application. Créez un fichier `_custom_theme.less` qui est importé en **dernier** dans le fichier principal (`global.less` ou `app.less`). Ce fichier contiendra toutes vos surcharges de variables et de styles.
    -   **Configuration Tailwind**: Plutôt que de modifier `tailwind.config.js` directement, créez un `tailwind.custom.config.js` qui importe la configuration de base et la fusionne avec vos ajouts/surcharges.
    ```javascript
    // tailwind.custom.config.js
    const baseConfig = require('./tailwind.config.js'); // Le fichier original renommé ou de base
    const { merge } = require('lodash');

    const customConfig = {
      theme: {
        extend: {
          colors: {
            primary: '#ABCDEF', // Votre couleur primaire
          },
        },
      },
    };

    module.exports = merge(baseConfig, customConfig);
    ```
    -   **Logo**: Le logo est souvent un simple remplacement de fichier, ce qui est peu susceptible de créer des conflits. 