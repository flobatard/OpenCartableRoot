# Jalons livrés

Récit court de ce qui a été construit, jalon par jalon, avec l'endroit où ça vit. La spec de référence (architecture cible, modèle de données, roadmap) reste `Descriptions.md` dans chaque sous-projet ; les décisions qui ont infléchi cette roadmap sont dans [decisions.md](decisions.md). Ordre chronologique de livraison.

## J0 — Socle d'authentification

- Back : validation du JWT Zitadel (`app/core/auth.py`), comptes applicatifs auto-provisionnés au premier appel et profil d'onboarding (`app/users/`), configuration en couches env > `.env` > `config/<APP_ENV>.yaml`.
- Front : `AuthService` (OIDC Code + PKCE), callback, onboarding bloquant, page profil, i18n Transloco fr/en, thème clair/sombre, SSR + prerender de la home.

## J1 — Contenu

- Taxonomie des matières (~475 nœuds, `app/subjects/`) et niveaux d'étude par système scolaire (`app/education_levels/`), seeds append-only, arbres servis en une requête.
- Modèle des cours : blocs ordonnés de quatre types (`text`, `exercise`, `document`, `module`) et bibliothèque de ressources S3 découplée des blocs (`app/courses/`, `app/resources/`, `app/core/storage.py` — upload presigné direct navigateur → S3).
- Front : « Mes cours », page cours à onglets Blocs | Ressources | Modules | Aperçu, éditeurs texte (Monaco, markdown + KaTeX), exercice (questions à corrigé) et document ; réordonnancement par glisser-déposer ; export PDF par impression native ; réglages de style de lecture par cours.

## J4 (anticipé) — Modules interactifs

- Bibliothèque de modules HTML/CSS/JS par cours, code stocké en base (`app/modules/`), blocs `module` pointeurs, insertion `oc-module:<id>` dans le markdown.
- Front : éditeur à trois Monaco avec aperçu live, exécution en iframe sandbox à origine opaque sans réseau (`shared/module-runner/`), bridge postMessage (auto-resize, événements).

## J2 — Liens de partage publics

- Visibilité par cours (`public`/`private`/`draft`), liens à token opaque avec expiration (`app/share_links/`), régime public **sans JWT** (`app/public/` : détail filtré des corrigés, presign des ressources, code des modules, catalogue par prof opt-in).
- Front : pages élèves `/:lang/shared/:token` et `/:lang/p/courses/:id` — coquille à onglets Sommaire | Ressources | Modules | Cours entier, navigation bloc par bloc, résolution d'exercice avec brouillon local, page dédiée par module ; catalogue `/:lang/p/:profId`.

## Hors jalon — Export / import de cours

- Archive `.zip` (manifest versionné + binaires) assemblée et parsée par l'API (`app/course_transfer/`) ; le réimport recrée un cours neuf (ids régénérés, références `oc-*` réécrites, taxonomie remappée par code).
- Front : bouton Exporter de la page cours, modale d'import de « Mes cours ».

## J3 — Recherche

- FTS Postgres en configuration `french_unaccent`, vecteurs sur `courses`/`blocks` maintenus par triggers, routes publiques paginées `/public/search/{courses,teachers}` (`app/search/`) ; profs cherchables sur opt-in.
- Front : page `/:lang/search` (onglets Cours | Professeurs, facettes matière/niveau, état dans l'URL), case « cherchable » du profil.

## Hors jalon — Client IA générique

- Client multi-provider BYO token sur LangChain 1.x (`app/core/ai/` : anthropic, openai, google, mistral, ollama, openai_compatible, huggingface), appels classiques et streaming, erreurs traduites au bord, Langfuse opt-in, routes de smoke-test `app/ai/`.
- Credential IA chiffré par utilisateur et quota quotidien de l'IA par défaut (`app/ai_credentials/`) ; écran « Réglages IA » du front (test de connexion, liste des modèles).

## J5 — IA (cinq briques)

1. **Assistant de cours, contexte global** : conversations persistées par cours, agent LangGraph avec tools de lecture (bloc, PDF, image, module), citations de sources en références courtes réécrites en flux (`app/course_assistant/`) ; front : panneau flottant persistant monté dans le shell, premier client SSE.
2. **Édition d'un bloc texte** (`block_text`) : premier flux HITL par interrupt/resume ; revue en diff Monaco à la place de l'éditeur, application annulable par Ctrl-Z.
3. **Édition d'un exercice** (`block_exercise`) : propositions par question (sujet, énoncé/corrigé, ajout, suppression), revue structurée ; les contextes d'édition deviennent des descripteurs (`app/course_assistant/editing/`).
4. **Édition d'un module** (`module`) : propositions par fichier, l'aperçu sandbox exécutant déjà le code proposé.
5. **Tuteur d'exercice côté élève** (`app/student_exercises/`) : élève authentifié, tentatives persistées par tour, verdict structuré, révélation du corrigé bornée par le serveur ; effacement des tentatives par l'élève et par le prof.

## Purge des données

- Job hors API (`app/maintenance/`, service compose `purge`) : sept tâches de rétention réglées par `PURGE_*`, garde de schéma contre la course avec les migrations, réconciliation des orphelins S3 (en dry-run par défaut).

## Reste du J5

Vue professeur des soumissions d'élèves et RAG éventuel — voir [../TODO.md](../TODO.md).
