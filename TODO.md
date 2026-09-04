# TODO

Dettes techniques acceptées « à terme », transverses au monorepo. Chaque entrée
renvoie vers le code ou le CLAUDE.md concerné ; retirer l'entrée quand c'est livré.

## Purge des données — réglages laissés à l'opérateur

La politique de rétention est **livrée** (`OpenCartableBack/app/maintenance/`,
service `purge` du compose — cf. le CLAUDE.md du back). Ce qui reste à arbitrer
n'est plus du code mais des **valeurs** :

- **Deux tâches sont câblées mais désactivées par défaut** (`0` = désactivée) :
  `PURGE_AI_CONVERSATIONS_DAYS` (conversations sans activité — c'est du travail
  de prof) et `PURGE_EXERCISE_SUBMISSIONS_DAYS` (tentatives d'élèves — données
  personnelles). Les activer est une décision produit/vie privée, pas une dette
  technique ; l'effacement manuel existe déjà des deux côtés.
- **`PURGE_S3_ORPHANS_DRY_RUN` est à `true`** : la réconciliation ne fait que
  journaliser les orphelins. À basculer après relecture des logs d'une première
  passe en production.
- **Pas d'agrégation avant suppression** : l'historique de `ai_daily_usage`
  au-delà de la rétention est perdu, pas résumé. Si des statistiques long terme
  sont voulues un jour, agréger avant de purger.
- Si `PURGE_EXERCISE_SUBMISSIONS_DAYS` est un jour activé à grande échelle, un
  index sur `exercise_submissions.created_at` seul deviendra utile (l'index
  existant est `(user_id, block_id, question_id, created_at)`).

## Assistant IA de cours — suites du lot « contexte global »

- **Résolution d'exercice élève (dernier contexte du cadrage)** : les trois
  contextes d'édition — **bloc texte (`block_text`), exercice
  (`block_exercise`) et module (`module`) — sont livrés** et fixent le motif
  à suivre — flux **HITL par interrupt/resume LangGraph, checkpointer
  `InMemorySaver`** (arbitrage acté, révise le « sans checkpointer » du
  premier lot) : l'agent propose via un tool sans mutation (proposition dans
  les args du `tool_call`, persistés), le tool **fige le run** (`hitl_gate`,
  `OpenCartableBack/app/course_assistant/editing/base.py` — seul appelant
  d'`agent_interrupt`), le flux SSE émet `interrupt` et se ferme ; la route
  `POST .../proposals/{id}/decision` **reprend le run dans un nouveau flux**
  (`Command(resume=…)`) — le résultat du tool est la décision commentée.
  Registre des reprises : `OpenCartableBack/app/course_assistant/hitl.py`
  (TTL 6 h — porte aussi la numérotation `Q…` du tour pour l'exercice,
  `PendingProposal.question_refs`). Côté back, un contexte d'édition est un
  **descripteur** `EditContext` (`editing/` : cible `target` bloc|module,
  type de bloc attendu, system prompt, tools de proposition — spec,
  validation, réécriture des args) : un nouveau contexte = un module sous
  `editing/` + une entrée au registre + étendre le `Literal` de
  `ConversationCreate`. Côté front, `CourseChat` a trois modes
  (global/edit/placeholder), l'état est extrait en `AssistantChatState`
  instanciable par hôte (état `awaiting` + `pendingProposal`/
  `resumeProposal`), les propositions sont parsées par tool
  (`core/course-assistant/proposals.ts`, union `AssistantPendingProposal`) et
  les revues (diff texte, revue structurée d'exercice, diff de code d'un
  module — briques `shared/proposal/`) sont orchestrées chez l'hôte par le
  `ProposalHost<V>` générique (`core/course-assistant/proposal-host.ts`) — un
  nouveau contexte = une portée de plus dans `AssistantContext`, un genre de
  proposition et sa revue. La résolution d'exercice élève est **livrée hors de
  ce motif** (tuteur d'exercice, cf. section dédiée ci-dessous).
- **Propositions d'exercice : Ctrl-Z partiel** — seuls les champs markdown
  (sujet, énoncé d'une question) sont appliqués via Monaco (annulables) ; le
  corrigé, l'ajout et la suppression d'une question passent par le formulaire
  (pas d'undo : le prof rejette, ou redemande). Le **déplacement d'une
  question par l'IA** n'est pas couvert (aucun tool — le prof réordonne par
  glisser-déposer). Les propositions de **module**, elles, passent toutes par
  Monaco (Ctrl-Z complet sur les trois fichiers).
- **Assistant de module : pas de retour d'exécution** (décision utilisateur,
  lot module) — l'IA lit le code et propose, le prof juge sur l'aperçu qui
  exécute le code proposé ; le modèle ne voit ni les erreurs JS ni la console
  de l'iframe. Piste si le besoin se confirme : le bridge `oc-module:*` de
  `OpenCartableFront/src/app/shared/module-runner/module-document.ts` capture
  `window.onerror`/`console.error` et les `postMessage` au parent, l'éditeur
  les tient et les joint au tour (message système ou tool `read_module_errors`).
  Touche le code de sécurité du runner et le contrat du bridge — à traiter
  comme un lot à part. Le prompt `MODULE_RUNTIME`
  (`OpenCartableBack/app/course_assistant/prompts.py`) est le **miroir** du
  contrat de `module-document.ts` (CSP, bridge) : toute évolution de l'un
  doit être reportée dans l'autre.
- ⚠ **Checkpointer HITL InMemory → `AsyncPostgresSaver` au passage
  multi-nœud** (décision utilisateur) : l'`InMemorySaver` du client IA et le
  registre `hitl.py` sont **process-locaux** — mono-worker obligatoire (la
  reprise doit arriver sur le worker qui tient le checkpoint), un redémarrage
  perd les reprises en attente (le tour partiel persisté reste un round
  incomplet, replié au replay), et une proposition abandonnée n'est purgée du
  checkpointer qu'à son TTL (fuite mémoire bornée par l'usage). Le jour du
  multi-nœud : dépendance `langgraph-checkpoint-postgres` + driver psycopg3,
  tables créées hors Alembic (`.setup()`), et remplacer le registre in-memory
  par une table.
- **Reprise HITL non ré-offerte après un rechargement de page** : la
  proposition en attente vit dans l'état front (`pendingProposal`) — un
  reload la perd, alors que le back garde la reprise jusqu'à son TTL. Piste :
  détecter à l'ouverture d'une conversation un round d'un tool de proposition
  (`PROPOSAL_TOOLS` — texte ou exercice) sans tour `tool` et re-proposer la
  décision.
- **Images lues par l'assistant (`read_resource_image`)** : aucun
  redimensionnement côté back (pas de Pillow — règle « pas de binaire par le
  backend ») — une image de plus de `IMAGE_MAX_BYTES` (3,5 Mo brut,
  `OpenCartableBack/app/course_assistant/tools.py`) est refusée au modèle, et
  une image lourde coûte cher en tokens à chaque lecture. Si les captures
  volumineuses deviennent courantes : réduire à l'upload côté front (canvas)
  ou accepter Pillow pour une miniature. L'image n'est **pas rejouée** aux
  tours suivants (décision actée, le modèle relit au besoin).

## Tuteur IA d'exercice élève — suites du lot

Le tuteur (`OpenCartableBack/app/student_exercises/`, front `core/student/` +
`ExerciseView`) est livré pour l'**élève authentifié**, sur **sa** config IA
(le régime anonyme n'a pas d'IA — décision actée, qui clôt la question de
l'imputation du quota) ; les tentatives sont persistées par tour dans
`exercise_submissions` (révise le « sans persistance » du cadrage initial).

- **Pas de pagination du fil** (plafond 100 tours par question,
  `MAX_TURNS_PER_QUESTION`) : le fil complet d'une question est chargé d'un
  bloc. La purge périodique existe mais est **désactivée par défaut** pour
  cette table (cf. « Purge des données » plus haut).
- **Vue professeur des soumissions** (hors lot) : seul un **résumé par
  question** (compteurs, `GET /courses/{id}/blocks/{id}/submissions/summary`)
  alimente les boutons d'effacement de l'éditeur d'exercice ; aucune route
  prof ne lit les contenus (réponses, verdicts, progression par élève). À
  concevoir avec la question de la vie privée des élèves (consentement,
  anonymisation) avant d'exposer.
- ⚠ **Injection par l'élève — risque assumé** : le modèle voit le corrigé de
  la question cible (il doit juger) ; la révélation est **bornée côté serveur**
  (`guard_reveal` : jamais sans réponse juste ni effort suffisant, corrigé
  joint par le back seulement si `revealed`) et le prompt ordonne d'ignorer
  les instructions de l'élève, mais un modèle faible peut paraphraser le
  corrigé dans son texte. Pistes si le cas se présente : second appel
  « juge » sans corrigé, ou filtre de similarité entre le retour et le corrigé
  avant émission.
- **Fil dévoilé sans reveal progressif** : le texte streamé est concaténé et
  re-rendu par `app-markdown-view` à chaque token (`[courseId]="null"`, oc-*
  inertes) — pas le lissage `STREAM_REVEAL_*` du chat prof ; à reprendre si le
  rendu paraît saccadé.

## Autres dettes déjà actées dans les CLAUDE.md

- **Keepalive SSE périodique** sur les routes IA streamées (protection contre les
  timeouts du proxy pendant les longues générations) — cf.
  `OpenCartableBack/CLAUDE.md`, section « Client IA générique ». (Le flux HITL
  n'aggrave plus le cas : il se **ferme** à la proposition et la décision
  rouvre un flux — plus d'attente silencieuse sur connexion ouverte.)
- **Import : compat archives v1 (manifest français)** maintenue via
  `normalize_manifest_v1` (`OpenCartableBack/app/course_transfer/schemas.py`) —
  à retirer un jour si l'on décide de ne plus supporter les exports antérieurs
  à la nomenclature anglaise.
