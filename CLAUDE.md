# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Monorepo

**OpenCartable** — plateforme pédagogique libre (AGPLv3), auto-hébergée sur Raspberry Pi ARM64 : un prof authentifié compose des cours par blocs et les partage à ses élèves via des liens publics (les élèves n'ont pas de compte). L'utilisateur échange en **français**.

Trois répertoires, deux applications déployables :

- **[OpenCartableBack/](OpenCartableBack/)** — API FastAPI + SQLAlchemy 2.0 async + Alembic + PostgreSQL (asyncpg). Voir **[OpenCartableBack/CLAUDE.md](OpenCartableBack/CLAUDE.md)** pour les commandes, l'architecture et les décisions actées.
- **[OpenCartableFront/](OpenCartableFront/)** — SPA Angular 22 (zoneless, SSR + prerender), Transloco (fr/en), OIDC. Voir **[OpenCartableFront/CLAUDE.md](OpenCartableFront/CLAUDE.md)**.
- **[Assets/](Assets/)** — logos SVG (variantes mono/dark/horizontal) et documents de cadrage de référence (`Descriptions.md`, `DESIGN_SYSTEM.md`).

Chaque sous-projet est autonome (son propre `docker-compose.yml`, `Dockerfile`, `.git`-ignore, tests) et possède un **CLAUDE.md détaillé** : toujours lire celui du sous-projet concerné avant d'y travailler. Ce fichier-ci ne couvre que ce qui traverse les deux.

## Le contrat entre front et back

L'API et la SPA sont couplées par deux invariants transverses ; les casser côté d'un seul répertoire casse l'autre :

1. **La SPA est propriétaire du flow OIDC ; l'API est un resource server pur.** Le login (Authorization Code + PKCE vers Zitadel) est entièrement géré par le front ; l'API n'émet jamais de token et ne stocke aucune identité — elle valide le JWT de chaque requête. Corollaire côté back : `OIDC_AUDIENCE` doit correspondre au client id configuré côté front, et Zitadel doit émettre des **JWT access tokens** (pas les tokens opaques par défaut, sinon 401).

2. **Remplaçabilité de l'IdP confinée à un module de chaque côté** (exigence du cahier des charges) : back = `app/core/auth.py`, front = `AuthService` (`src/app/core/auth/`), seule couche autorisée à importer `angular-oauth2-oidc`. Ne jamais disperser de logique Zitadel ailleurs.

3. **Base d'URL API** : le front attaque `environment.apiUrl` (qui **inclut déjà `/api`**) ; le Bearer est attaché automatiquement par l'intercepteur OIDC à toute requête sous cette base. Côté back, les routes sont montées sous `settings.API_V1_PREFIX` (`/api/v1`). Route de référence du contrat de données : `GET /api/v1/subjects/tree`.

4. **Les liens publics élèves ne dépendent jamais de Zitadel.** Le régime d'accès élève (J2, livré) a sa propre autorisation des deux côtés : back = routes `/api/v1/public/*` **sans JWT** (visibilité du cours + token de partage vérifiés à chaque requête, `app/public/`) ; front = pages élèves sans Bearer (la `customUrlValidation` d'`app.config.ts` exclut `/v1/public/` de l'attachement du token — même un prof connecté consulte ces pages en anonyme).

## Cadrage produit et jalons

Le cahier des charges complet (architecture cible, modèle de données, roadmap J0→J5) fait foi et doit être mis à jour quand une décision d'archi change. **Attention : `Descriptions.md` et `DESIGN_SYSTEM.md` existent en plusieurs copies divergentes** (dans `Assets/`, `OpenCartableBack/`, `OpenCartableFront/`). Pour un travail back, se référer à `OpenCartableBack/Descriptions.md` ; pour un travail front/UI, à `OpenCartableFront/Descriptions.md` + `OpenCartableFront/DESIGN_SYSTEM.md`.

Les jalons J0→J2 (+ J4 anticipé) sont livrés : socle auth (J0), taxonomie des matières, modèle BDD des cours, **CRUD cours** (`/api/v1/courses`), **page cours à deux onglets Blocs | Ressources**, espace blocs (création/suppression/réordonnancement — quatre types : `texte`, `exercice`, `document`, `module`), **éditeurs texte/exercice** (Monaco, markdown + formules LaTeX `$…$`/`$$…$$` rendues par KaTeX) et **document** (picker de ressource + légende/affichage), et la **bibliothèque de ressources S3** (`/api/v1/courses/{id}/resources` : upload presigned direct navigateur→S3, liste, renommage, téléchargement, suppression — **découplée des blocs** : un bloc `document` pointe une ressource via `resource_id` nullable, FK `CASCADE`) sont en place, ainsi que le **J4 anticipé** (livré avant J2/J3) : **modules interactifs HTML/CSS/JS** — bibliothèque par cours (onglet Modules, table `modules` : code **stocké en base**, décision actée qui remplace le bundle S3 du cadrage initial), API `/api/v1/courses/{id}/modules`, éditeur dédié à 3 Monaco (HTML/CSS/JS) avec preview live, blocs `module` pointeurs (`module_id`, FK CASCADE), insertion `oc-module:<id>` dans le markdown, exécution en **iframe sandbox à origine opaque** (jamais `allow-same-origin`, **réseau sortant bloqué par CSP dans le srcdoc** — modules self-contained, décision actée —, bridge postMessage auto-resize + événements) ; et le **J2 — liens de partage publics** : visibilité par cours (`public`/`prive`/`en_cours`, défaut `en_cours` qui **suspend** les liens sans les supprimer, `PUT /api/v1/courses/{id}/visibility`), liens à token opaque 256 bits (`/api/v1/courses/{id}/share-links` — expiration obligatoire 9 mois, révocation soft), routes publiques **sans JWT** `/api/v1/public/*` (détail de cours filtré — jamais les `reponse_attendue` —, presign ressources, code des modules, catalogue par prof via `users.nom_public` opt-in ; 404 uniforme sans oracle), pages élèves front (`/:lang/shared/:token`, `/:lang/p/courses/:id`, `/:lang/p/:profId` — toutes `RenderMode.Client`) et réponses d'exercice élève en **localStorage uniquement** (les soumissions serveur arrivent au J5) ; s'y ajoute, hors jalon, l'**export/import de cours** — archive `.zip` (manifest versionné + binaires des ressources) assemblée/parsée **par l'API** (back `app/course_transfer/`, exception actée à la règle « pas de binaire par le backend ») ; le réimport recrée un cours au contenu identique : nouveaux ids, références `oc-resource:`/`oc-module:` réécrites, taxonomie remappée par `code` (front : bouton Exporter de la page cours — premier download blob — et modale d'import de « Mes cours » — premier POST multipart) ; restent J3 (recherche) et J5 (IA).

## Reverse proxy et infra

Le reverse proxy nginx (TLS, routage `/api` vers le back, protection SSRF SSR côté front) est fourni par l'infra, **hors de ce repo**. Les composes exposent l'API sur 8000 et le SSR sur 4000 sans proxy — c'est voulu.
