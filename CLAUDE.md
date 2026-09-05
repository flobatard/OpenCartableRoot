# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Le projet

**OpenCartable** — plateforme pédagogique libre (AGPLv3), auto-hébergée sur Raspberry Pi ARM64 : un prof authentifié compose des cours par blocs et les partage à ses élèves via des liens publics (les élèves n'ont pas de compte ; un élève connecté dispose en plus d'un tuteur IA). L'utilisateur échange en **français**.

Trois répertoires, **trois dépôts git distincts** (les deux sous-projets sont ignorés par le dépôt racine — commiter chacun dans le sien) :

- **[OpenCartableBack/](OpenCartableBack/)** — API FastAPI + SQLAlchemy 2.0 async + Alembic + PostgreSQL (asyncpg). Voir [OpenCartableBack/CLAUDE.md](OpenCartableBack/CLAUDE.md).
- **[OpenCartableFront/](OpenCartableFront/)** — SPA Angular 22 (zoneless, SSR + prerender), Transloco (fr/en), OIDC. Voir [OpenCartableFront/CLAUDE.md](OpenCartableFront/CLAUDE.md).
- **[Assets/](Assets/)** — logos SVG uniquement.

Toujours lire le CLAUDE.md du sous-projet concerné avant d'y travailler. Ce fichier-ci ne couvre que ce qui traverse les deux.

## Où trouver quoi

| Besoin | Fichier |
|---|---|
| Spec produit et architecture cible (fait foi) | `OpenCartableBack/Descriptions.md` (travail back), `OpenCartableFront/Descriptions.md` (travail front/UI) — les deux copies divergent légitimement par leur angle |
| Design system (source de vérité UI) | `OpenCartableFront/DESIGN_SYSTEM.md` |
| Décisions d'architecture encore contraignantes | [docs/decisions.md](docs/decisions.md) |
| Ce qui a été livré, jalon par jalon | [docs/milestones.md](docs/milestones.md) |
| Dettes techniques acceptées | [TODO.md](TODO.md) |
| Approfondissements par paquet / feature | `OpenCartableBack/docs/architecture.md`, `OpenCartableFront/docs/architecture.md` |

## Le contrat entre front et back

L'API et la SPA sont couplées par des invariants transverses ; les casser d'un seul côté casse l'autre :

1. **La SPA est propriétaire du flow OIDC ; l'API est un resource server pur.** Le login (Authorization Code + PKCE vers Zitadel) est entièrement géré par le front ; l'API n'émet jamais de token et ne stocke aucune identité — elle valide le JWT de chaque requête. Corollaire côté back : `OIDC_AUDIENCE` doit correspondre au client id configuré côté front, et Zitadel doit émettre des **JWT access tokens** (pas les tokens opaques par défaut, sinon 401).

2. **Remplaçabilité de l'IdP confinée à un module de chaque côté** : back = `app/core/auth.py`, front = `AuthService` (`src/app/core/auth/`), seule couche autorisée à importer `angular-oauth2-oidc`. Ne jamais disperser de logique Zitadel ailleurs.

3. **Base d'URL API** : le front attaque `environment.apiUrl` (qui **inclut déjà `/api`**) ; le Bearer est attaché automatiquement par l'intercepteur OIDC à toute requête sous cette base. Côté back, les routes sont montées sous `settings.API_V1_PREFIX` (`/api/v1`). Route de référence du contrat de données : `GET /api/v1/subjects/tree`.

4. **Les liens publics élèves ne dépendent jamais de Zitadel.** Back = routes `/api/v1/public/*` **sans JWT** (visibilité du cours + token de partage vérifiés à chaque requête, 404 uniforme sans oracle) ; front = pages élèves sans Bearer (la `customUrlValidation` d'`app.config.ts` exclut `/v1/public/` — même un prof connecté consulte ces pages en anonyme). Le tuteur IA de l'élève connecté est l'exception voulue : routes `/api/v1/student/*` **avec** JWT mais accès au cours par le régime public (`?token=`).

5. **Le contrat SSE** des routes IA (`token`/`thinking`/`tool_call`/`tool_result`/`interrupt`/`done`/`error`, POST + `fetch`/`ReadableStream` côté front, Bearer posé à la main) est défini côté back par la docstring de référence de `app/ai/service.py` (étendu par `app/course_assistant/streaming.py`) et consommé côté front par le parseur de `core/course-assistant/sse.ts`. Le contrat est **additif** : le front tolère les événements inconnus.

6. **Deux miroirs à maintenir ensemble** : le prompt `MODULE_RUNTIME` du back décrit le bac à sable réel de `shared/module-runner/module-document.ts` (CSP, bridge) ; les plafonds partagés (taille d'export, extrait des résultats d'outils, clés camelCase de `preview_settings`) sont recopiés en constantes des deux côtés.

## Règles transverses

- Toute nouvelle décision d'architecture → une entrée en tête de [docs/decisions.md](docs/decisions.md), et `Descriptions.md` mis à jour si l'architecture cible change.
- Toute dette acceptée « à terme » → une ligne dans [TODO.md](TODO.md) ; retirer la ligne quand c'est livré.
- Les CLAUDE.md décrivent l'état **présent** (commandes, invariants, carte du code) : pas de récit de livraison, pas de « décision actée » — l'historique vit dans `docs/`.
- Cible Raspberry Pi ARM64 : déporter le lourd (URL S3 présignées, jobs hors API), images Docker multi-arch.

## Reverse proxy et infra

Le reverse proxy nginx (TLS, routage `/api` vers le back, protection SSRF SSR côté front) est fourni par l'infra, **hors de ce repo**. Les composes exposent l'API sur 8000 et le SSR sur 4000 sans proxy — c'est voulu.
