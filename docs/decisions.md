# Journal des décisions

Décisions d'architecture encore contraignantes, une entrée chacune : le contexte, la décision, ce qu'elle impose au code, où elle vit. Les entrées sont datées « avant 2026-09 » quand la date exacte n'a pas été conservée ; les nouvelles décisions s'ajoutent **en tête** avec leur date. Ce qui a été livré et quand est dans [milestones.md](milestones.md) ; les dettes acceptées sont dans [../TODO.md](../TODO.md).

Format : **Titre** · Contexte · Décision · Conséquences · Code.

---

## 20. Style de lecture = propriété du cours (avant 2026-09)

- Contexte : le prof veut régler typographie et largeur de lecture de ses cours ; ça doit suivre au PDF.
- Décision : les réglages sont une **propriété du cours** (JSONB `courses.preview_settings`, remplacement complet par `PUT /courses/{id}/preview`), pas une préférence du lecteur ; appliqués en `[style]` inline sur le conteneur de rendu, jamais sur `:root`.
- Conséquences : facteurs d'échelle sans unité (un seul jeu de réglages pour écran et impression) ; scope par cours, aucune fuite vers les pages de doc ; le clone d'impression emporte le style.
- Code : back `app/models/course.py` (contrat commenté) ; front `CourseStyleService`, `course-style-dialog`.

## 19. Pages élèves en rendu client explicite, sans query param (avant 2026-09)

- Contexte : DOMPurify sans `window` (SSR) renvoie le HTML **non filtré** ; `@angular/ssr` résout un `redirectTo` en chaîne relative en un 302 cassé.
- Décision : toute route élève a son entrée `RenderMode.Client` explicite dans `app.routes.server.ts` (rien ne retombe dans le catch-all serveur) ; l'état de navigation vit dans le chemin (une route par onglet), jamais en query param ; les redirections sont des **fonctions** `redirectTo`.
- Conséquences : ajouter une page élève = deux déclarations (routes + routes serveur), gardées par `app.routes.spec.ts` ; les pages élèves n'injectent que les résolveurs `COURSE_*_RESOLVER`, jamais un service prof.
- Code : `app.routes.ts` (`PUBLIC_COURSE_CHILDREN`), `app.routes.server.ts`, `core/course-content/`.

## 18. Panneau assistant monté une fois dans le shell (avant 2026-09)

- Contexte : le prof navigue entre la page cours et les éditeurs en gardant sa conversation ; un remontage ferait perdre le fil, le scroll et la saisie.
- Décision : le panneau flottant est monté **hors du router-outlet** par une balise du shell ; le cours courant est dérivé de l'URL ; l'état plié/déplié et la conversation vivent dans un service root.
- Conséquences : les deux éditeurs portent `remountOnParamChange` (voir 17) — c'est la page, pas le panneau, qui doit remonter quand une citation change de bloc ; le corps du panneau est derrière `@defer` pour rester hors du bundle initial.
- Code : `assistant-outlet`, `assistant-panel`, `CourseAssistantService`.

## 17. Remontage des éditeurs au changement de param (avant 2026-09)

- Contexte : les éditeurs de bloc et de module lisent leurs params en snapshot ; une citation `oc-block:` du panneau peut naviguer d'un éditeur à un autre sur la même route.
- Décision : une `RouteReuseStrategy` dédiée détruit et remonte les routes marquées `data: { remountOnParamChange: true }` (flush d'autosave au destroy, puis init).
- Conséquences : les composants conçus pour survivre à un changement de param (`DocsShell`, `StudentBlock`, paramMap observé) ne posent **pas** le flag ; un changement de query params seuls ne remonte rien.
- Code : `core/routing/remount-on-param-change.strategy.ts`.

## 16. Monaco : AMD depuis les assets, jamais sous `@if`, propositions appliquées en édits annulables (avant 2026-09)

- Contexte : Monaco (~16 Mo) ne doit pas entrer dans le bundle ; recréer l'éditeur à chaque bascule d'onglet perdrait l'état ; une proposition de l'IA appliquée par `setValue` viderait la pile d'annulation.
- Décision : Monaco est servi en AMD depuis `/monaco/vs` avec un `baseUrl` absolu ; ses hôtes le masquent par `[hidden]` ou une classe, **jamais** par `@if` ; une proposition acceptée est appliquée par `executeEdits` **entre deux undo stops** (Ctrl-Z l'annule comme une frappe), avec repli `setValue` si Monaco n'est pas prêt.
- Conséquences : options à référence stable (le wrapper recrée l'éditeur à chaque nouvelle référence), thème par `setTheme` global, garde anti-écho sur `writeValue` ; en jsdom Monaco est inerte, les specs pilotent les `FormControl` publics.
- Code : `shared/markdown-editor/`, `shared/proposal/`.

## 15. Dépendances et outillage laissés à l'utilisateur (avant 2026-09)

- Décision : les dépendances IA encore inutilisées de `requirements.txt` (sentence-transformers, chromadb, redis, psycopg2-binary) sont **conservées** ; `scripts/` du back est écrit à la main par l'utilisateur ; les commandes alembic (révision, upgrade) sont **toujours** exécutées par l'utilisateur.
- Conséquences : ne pas purger ces dépendances, ne pas modifier `scripts/` sans demande, ne jamais lancer alembic à sa place.

## 14. Purge des données : un job hors API, qui vérifie le schéma au lieu d'ordonner les conteneurs (avant 2026-09)

- Contexte : la réconciliation des orphelins S3 énumère tout le bucket (contrainte Pi : déporter le lourd) ; `depends_on: service_healthy` n'ordonne que la première mise en route (reboot du Pi, déploiement avec nouvelle migration alors que le job tourne déjà).
- Décision : la purge est un **service compose `purge`** (même image, boucle shell qui lance `python -m app.maintenance` puis dort ; entrypoint écrasé pour ne pas rejouer les migrations) ; avant d'écrire, le job compare la révision alembic en base à la tête de l'image et **renonce** si elles diffèrent. Défauts prudents : conversations IA et tentatives d'élèves **désactivées** (0), réconciliation S3 en `dry_run`, plancher de 2 jours sur `ai_daily_usage` (quota vivant + remboursement de la veille). Tous les délais vivent dans `config/*.yaml`.
- Conséquences : ne pas remettre la purge dans le process uvicorn ; le service doit figurer dans le compose de preprod (sinon `--remove-orphans` le détruit) ; l'allègement des tours `tool` est un `UPDATE` (la ligne doit survivre pour rester appariée).
- Code : `app/maintenance/` (`schema.py` = garde de schéma).

## 13. Images lues par l'assistant : brutes, bornées, jamais rejouées (avant 2026-09)

- Contexte : le tool `read_resource_image` montre une image de la bibliothèque au modèle ; pas de Pillow côté back (règle « pas de binaire par le backend »).
- Décision : plafond brut `IMAGE_MAX_BYTES` (≈ 3,5 Mo, sous les 5 Mo en base64 d'Anthropic), aucun redimensionnement ; l'image voyage en artefact du `ToolMessage` puis en message utilisateur marqué ; elle **n'est pas rejouée** aux tours suivants (le modèle relit au besoin).
- Code : `app/course_assistant/tools.py`, `app/core/ai/client.py` (middleware image).

## 12. Tuteur d'exercice : élève authentifié, imputé à sa config, persistant, révélation bornée serveur (avant 2026-09)

- Contexte : le cadrage prévoyait une résolution « sans persistance » et posait la question du quota d'un élève anonyme.
- Décision : le tuteur exige un **JWT** (le régime anonyme n'a pas d'IA — ce qui clôt l'imputation : cascade `effective_config` de l'élève, credential BYO ou IA par défaut sous SON quota) tout en accédant au cours par le régime public (`?token=`) ; chaque tour est **persisté** (`exercise_submissions`, une ligne par tour) ; le modèle déclare un verdict structuré par tool et le back **n'expose le corrigé que si** réponse juste ou effort suffisant ; le modèle tutoie l'élève.
- Conséquences : redaction structurelle — l'instantané vu par le modèle passe par `public_content` (aucun `expected_answer` hors la question cible) ; routes sous `/student/…`, hors `/public/` ; le corrigé n'est jamais copié en table.
- Code : `app/student_exercises/` (`guard_reveal`), front `core/student/`.

## 11. Granularité des propositions HITL (avant 2026-09)

- Décision : bloc texte = **remplacement entier** du markdown ; exercice = **une opération par question** (sujet, énoncé/corrigé, ajout, suppression — jamais un remplacement de l'exercice) ; module = **un fichier à la fois** (HTML, CSS ou JS, contenu intégral). Une proposition à la fois, le modèle enchaîne après chaque décision. Pendant la revue d'un module, l'aperçu sandbox **exécute déjà le code proposé**.
- Conséquences : Ctrl-Z complet pour le texte et les trois fichiers d'un module ; partiel pour l'exercice (corrigé/ajout/suppression passent par le formulaire) ; les questions sont désignées par des références `Q…` stables le temps d'un tour.
- Code : `app/course_assistant/editing/{block_text,block_exercise,module}.py`, front `core/course-assistant/proposals.ts`.

## 10. HITL par interrupt/resume LangGraph, checkpointer en mémoire (avant 2026-09)

- Contexte : l'IA doit proposer une modification que le prof accepte ou rejette, sans jamais muter le bloc elle-même.
- Décision : le tool de proposition **fige le run** (`interrupt`), le flux SSE émet `interrupt` et se ferme sans `done` ; la décision rouvre un flux qui **reprend** le run (`Command(resume=…)`) ; **le résultat du tool est la décision** (acceptée/rejetée + commentaire). Checkpointer `InMemorySaver` + registre in-process avec TTL.
- Conséquences : **mono-worker** obligatoire (la reprise doit arriver sur le worker qui tient le checkpoint), reprises perdues au redémarrage, un tour HITL = un appel compté (config réutilisée à la reprise) ; le tool est ré-exécuté depuis le début à la reprise (validation idempotente). Passage à `AsyncPostgresSaver` le jour du multi-nœud (TODO).
- Code : `app/core/ai/client.py` (`agent_interrupt`), `app/course_assistant/editing/base.py` (`hitl_gate`, seul appelant), `app/course_assistant/hitl.py`.

## 9. Références courtes côté modèle, jamais d'UUID (avant 2026-09)

- Contexte : les modèles déforment les UUID recopiés (~20 tokens aléatoires).
- Décision : l'instantané du tour numérote blocs `B1…`, ressources `R1…`, modules `M1…`, questions `Q1…` ; prompt, tools (avec `enum`) et citations n'utilisent que ces références ; un résolveur tolérant (préfixe d'UUID, titre exact ou approchant) liste les candidats en cas d'échec ; les citations `oc-block:B3` sont **réécrites en UUID en flux**.
- Conséquences : texte streamé = texte persisté ; contrat SSE et front inchangés ; les ids hallucinés sont filtrés des `sources`.
- Code : `app/course_assistant/refs.py`.

## 8. Client IA BYO token, cascade de configuration et quota quotidien (avant 2026-09)

- Contexte : le cadrage initial disait « aucune clé persistée » ; en pratique il faut une clé par prof, et un fallback serveur pour les élèves ou l'essai.
- Décision : cascade **config explicite de la requête > credential utilisateur persistant chiffré** (AES-256-GCM, clé dérivée HKDF d'une clé maître, sel par utilisateur) **> fallback serveur `AI_*`**, ce dernier seul **compté** sous un quota quotidien par utilisateur (`users.ai_daily_call_quota`, NULL = défaut, 0 = illimité, **sans CHECK SQL**, jamais écrit par une route) ; réservation atomique avant l'appel, remboursement si le provider échoue, un flux qui a produit du contenu **reste compté**.
- Conséquences : `app/core/ai/` reste stateless (la cascade vit dans `ai_credentials.service.effective_config`) ; seul `app/core/ai/` importe langchain/langgraph/langfuse, seul `app/core/crypto.py` importe `cryptography` ; une clé refusée par le provider est un **400** (401 réservé au JWT) ; la clé n'apparaît dans aucun schéma de réponse.
- Code : `app/core/ai/`, `app/ai_credentials/`, `app/models/ai_daily_usage.py`.

## 7. Partager un cours expose toute sa bibliothèque (avant 2026-09)

- Décision : le détail public d'un cours embarque toutes ses ressources `available` et les titres de tous ses modules, blocs ou pas ; le front y adosse ses onglets Ressources/Modules.
- Conséquences : assumé et signalé par un avertissement dans l'UI de partage ; le code des modules reste servi module par module.
- Code : `app/public/service.py`.

## 6. Liens de partage : capability URL, expiration obligatoire, 404 uniforme (avant 2026-09)

- Décision : token opaque 256 bits stocké **en clair** (recopiable à tout moment), `expires_at` obligatoire (270 jours), révocation soft ; la visibilité `draft` **suspend** les liens sans les supprimer ; toute erreur du régime public est un **404 « Cours introuvable »**, jamais 401/403/410 (aucun oracle) ; le token voyage en `?token=`.
- Conséquences : validité vérifiée à chaque accès en Python ; les routes `/api/v1/public/*` ne portent aucune dépendance JWT ; le front n'y attache jamais de Bearer, même pour un prof connecté.
- Code : `app/share_links/`, `app/public/service.py`, front `app.config.ts` (`customUrlValidation`).

## 5. Un bloc pointeur meurt avec sa cible (avant 2026-09)

- Décision : un bloc `document` pointe une ressource et un bloc `module` un module par une **colonne** FK nullable en `ON DELETE CASCADE` ; supprimer la cible supprime les blocs pointeurs.
- Conséquences : le front recharge le détail du cours après une suppression de ressource/module ; les CHECKs de cohérence limitent `resource_id` aux blocs `document` et `module_id` aux blocs `module`.
- Code : `app/models/block.py`.

## 4. Modules interactifs : code en base, bac à sable sans réseau (avant 2026-09)

- Contexte : le cadrage §5.5 prévoyait un bundle `.zip` sur S3 avec versionnage par clé et des CDN autorisés.
- Décision : un module = **code HTML/CSS/JS stocké en base** et édité dans l'app ; exécution en **iframe sandbox à origine opaque** (`sandbox` statique **sans** `allow-same-origin`) avec une **CSP dans le srcdoc** : `default-src 'none'`, inline et `data:`/`blob:` seuls autorisés, `'unsafe-eval'` toléré (grapheurs), `form-action 'none'` — **aucun réseau sortant**.
- Conséquences : modules self-contained (aucun CDN, aucune police distante) ; le prompt `MODULE_RUNTIME` du back est le miroir du contrat de `module-document.ts` — toute évolution de l'un se reporte dans l'autre ; une future CSP d'en-tête posée par l'infra devra tolérer `'unsafe-eval'` ; le srcdoc est posé impérativement (jamais `[srcdoc]`, le sanitizer Angular striperait les scripts).
- Code : `app/modules/`, front `shared/module-runner/module-document.ts`.

## 3. Pas de binaire par le backend — sauf l'export/import de cours (avant 2026-09)

- Contexte : cible Raspberry Pi (RAM, bande passante) ; les uploads sont directs navigateur → S3 par URL présignée.
- Décision : l'API ne transporte jamais de binaire, **à une exception près** : l'archive `.zip` d'export/import est assemblée et parsée par l'API (volumes bornés par `TRANSFER_MAX_ZIP_BYTES`, assemblage dans un fichier temporaire spoolé, jamais une archive entière en RAM).
- Conséquences : seules deux méthodes de bytes dans `storage.py` (`read_object_into`, `put_object`) ; ordre transactionnel de l'import : inserts sans commit → put S3 → commit (au pire un orphelin bucket, jamais une référence DB vers un objet absent) ; la suppression purge S3 **après** commit.
- Code : `app/course_transfer/`, `app/core/storage.py`.

## 2. Une seule extension Postgres, pas de pgvector (avant 2026-09)

- Décision : aucune extension Postgres, à une exception actée pour la recherche : contrib `unaccent` + configuration `french_unaccent` (stemming + insensibilité aux accents). Pas de pgvector ; si la vectorisation se fait, ce sera probablement ChromaDB.
- Conséquences : toute autre extension reste à arbitrer explicitement ; les vecteurs FTS sont maintenus par triggers, jamais par l'ORM, et n'indexent jamais `expected_answer`.
- Code : migration FTS, `app/search/`.

## 1. Resource server pur, IdP remplaçable (avant 2026-09)

- Contexte : le cahier des charges exige de pouvoir changer d'IdP en ne touchant qu'un module par application.
- Décision : la SPA porte tout le flow OIDC (Authorization Code + PKCE vers Zitadel) ; l'API n'émet jamais de token, ne stocke aucune identité et valide le JWT de chaque requête ; la logique IdP est **confinée** dans `app/core/auth.py` (back) et `AuthService` (front, seul importeur d'`angular-oauth2-oidc`).
- Conséquences : `OIDC_AUDIENCE` = client id du front ; Zitadel doit émettre des JWT access tokens ; `get_current_user` ne touche jamais la base (la résolution `sub → users` vit dans `app/users/`) ; token invalide → 401 + `WWW-Authenticate`, IdP injoignable → 503, jamais 500, jamais le token dans les logs.
- Code : `app/core/auth.py`, front `src/app/core/auth/`.
