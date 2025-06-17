# Architecture du Frontend

Le frontend de RAGFlow est une application monopage (SPA) moderne construite avec **React** et l'écosystème qui l'entoure. Elle est responsable de toute l'interface utilisateur et interagit avec le backend via une API RESTful.

## Technologies et Frameworks Clés

- **Framework**: [UmiJS](https://umijs.org/), un framework React de niveau entreprise. UmiJS gère le routage, le processus de build et la configuration du développement.
- **Bibliothèque de base**: [React](https://react.dev/).
- **Langage**: [TypeScript](https://www.typescriptlang.org/) est utilisé pour la robustesse et la maintenabilité du code.
- **Kit d'interface utilisateur**: Le projet utilise une approche hybride :
  - [Ant Design](https://ant.design/): Utilisé comme bibliothèque de composants principale pour les éléments d'interface utilisateur courants.
  - [Radix UI](https://www.radix-ui.com/) et [Tailwind CSS](https://tailwindcss.com/): Radix fournit des primitives d'interface utilisateur accessibles et non stylées, qui sont ensuite stylées avec Tailwind CSS pour créer des composants personnalisés et cohérents.
- **Gestion d'état**: [Zustand](https://github.com/pmndrs/zustand) est utilisé pour la gestion d'état globale, offrant une API simple et performante.
- **Récupération et mise en cache des données**: [React Query (TanStack Query)](https://tanstack.com/query/latest) est utilisé pour gérer les requêtes API, la mise en cache, et la synchronisation des données serveur.
- **Routage**: Géré par UmiJS, avec une configuration de routage centralisée dans `src/routes.ts`.
- **Internationalisation (i18n)**: `i18next` et `react-i18next` sont intégrés, ce qui indique que l'application est conçue pour supporter plusieurs langues.

## Structure du Projet (`web/`)

- `.umirc.ts`: Le fichier de configuration principal pour UmiJS. Il définit le routage, les plugins, les alias et, surtout, la configuration du **proxy** qui redirige les appels API (`/api`, `/v1`) vers le serveur backend Flask pendant le développement.
- `package.json`: Définit tous les scripts (`dev`, `build`, etc.) et les dépendances du projet.
- `src/`: Le répertoire principal du code source de l'application.
  - `pages/`: Contient les composants React pour chaque page/route de l'application. La structure de ce dossier correspond aux routes définies.
  - `layouts/`: Contient les composants de layout globaux (par exemple, la barre de navigation, le menu latéral). `routes.ts` montre comment différents layouts sont appliqués à différentes sections de l'application.
  - `components/`: Contient des composants React réutilisables partagés entre plusieurs pages.
  - `services/`: Contient la logique pour effectuer des appels API au backend. UmiJS a une bibliothèque `umi-request` intégrée, mais le projet peut également utiliser `axios`.
  - `hooks/`: Contient des hooks React personnalisés pour encapsuler et réutiliser la logique avec état.
  - `wrappers/`: Contient des composants d'enrobage de haut niveau, comme le `auth` wrapper qui protège les routes contre l'accès non authentifié.
  - `routes.ts`: Le fichier de configuration centralisé pour toutes les routes de l'application.

## Communication avec le Backend

La communication avec le backend Flask se fait via des appels API RESTful. Pendant le développement, le serveur de développement de UmiJS utilise la configuration `proxy` dans `.umirc.ts` pour rediriger les requêtes vers le serveur backend, évitant ainsi les problèmes de CORS. En production, le serveur web (par exemple, Nginx) est généralement configuré pour servir les fichiers statiques du frontend et rediriger les appels API vers le backend.

## Points d'extension possibles

- **Personnalisation de l'interface utilisateur**: La plupart des styles sont gérés par Tailwind CSS. La personnalisation de l'apparence peut être effectuée en modifiant `tailwind.config.js` et les classes CSS dans les composants. L'ajout de nouveaux thèmes (par exemple, un thème sombre) serait également simple grâce à `next-themes` et à la configuration de Tailwind.
- **Ajout de nouvelles pages**: Pour ajouter une nouvelle page, il faudrait créer un nouveau composant dans `src/pages/` et ajouter une nouvelle entrée dans le fichier `src/routes.ts`.
- **Internationalisation (i18n)**: Pour ajouter une nouvelle langue, il faudrait ajouter de nouveaux fichiers de traduction dans le répertoire `locales` (s'il est configuré de cette manière) et utiliser le hook `useTranslation` de `react-i18next` dans les composants pour afficher le texte traduit. 