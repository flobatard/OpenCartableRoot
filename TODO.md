# TODO

Dettes techniques acceptées « à terme », transverses au monorepo. Chaque entrée
renvoie vers le code ou le CLAUDE.md concerné ; retirer l'entrée quand c'est livré.

## Stratégie de purge des compteurs de messages IA (`ai_daily_usage`)

La table back `ai_daily_usage` compte les appels servis par l'**IA par défaut**
(une ligne par utilisateur actif et par jour UTC —
`OpenCartableBack/app/models/ai_daily_usage.py`). Aucune purge aujourd'hui :
le volume est négligeable à court terme, mais la croissance est **sans borne** —
il faudra mettre en place une stratégie de purge.

Contraintes et pistes :

- **Ne jamais toucher au jour UTC courant** : le quota quotidien
  (`_consume_default_quota` / `refund_default_quota`,
  `OpenCartableBack/app/ai_credentials/service.py`) et le compteur
  « utilisés / autorisés » de l'écran Paramètres → Assistant IA lisent cette ligne.
- Choisir une **rétention** (ex. 90 jours) : l'historique au-delà ne sert qu'à
  d'éventuelles statistiques — si des stats long terme sont voulues un jour,
  agréger avant de supprimer.
- Véhicule possible : job périodique hors API (cron système sur le Pi —
  `scripts/` du back est maintenu à la main par l'utilisateur) ; un simple
  `DELETE FROM ai_daily_usage WHERE day < now() - interval 'N days'` suffit,
  aucune dépendance nouvelle.

## Assistant IA de cours — suites du lot « contexte global »

- **Purge des conversations IA** (`ai_conversations`/`ai_messages`) : aucune
  purge ni pagination aujourd'hui (limite de liste 100, plafond 300
  messages/conversation) — croissance sans borne, même stratégie à définir que
  pour `ai_daily_usage` (le prof peut déjà supprimer manuellement).
- ⚠ **Taille des réponses d'outils persistées** : chaque tour `tool` stocke le
  **contenu complet** du résultat dans `ai_messages.content` — jusqu'à
  **40 000 caractères par lecture de PDF** (`PDF_MAX_CHARS`,
  `OpenCartableBack/app/course_assistant/tools.py`), rejoué au modèle à chaque
  reprise de conversation et servi intégralement par le détail. Une
  conversation qui enchaîne les lectures peut peser plusieurs Mo à elle seule
  (contrainte Pi : disque + RAM des selects). Pistes à arbitrer : tronquer à la
  persistance (en gardant un extrait « suffisant pour l'affichage/replay »),
  compresser, ou purger le `content` des tours tool au-delà d'une ancienneté —
  à traiter avec la stratégie de purge ci-dessus.
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
  proposition et sa revue. La résolution d'exercice élève est **exemptée de
  persistance** (décision produit) et relève du régime élève anonyme
  ci-dessous.
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
- **Chat élève anonyme** (reporté — décision utilisateur) : régime public sans
  JWT à concevoir, avec la question de l'imputation du quota d'un élève sans
  compte. Le front **réserve déjà l'emplacement de correction par question**
  dans la vue de résolution d'exercice (`ExerciseView`,
  `OpenCartableFront/src/app/shared/course-blocks-view/`, mode `solve` du bloc
  seul `blocks/:blockId`) : input `correctionEnabled` (défaut `false` — le
  bouton « Demander une correction » n'est pas rendu), map `corrections` par
  id de question (`pending` / `done` + retour markdown / `error`, rien de
  rendu sans entrée), output `correctionRequested {blockId, questionId,
  answer}`, relayés par `CourseBlocksView` (types dans
  `core/student/exercise-correction.ts`). Câblage prévu : `StudentBlock`
  porte l'état et appelle l'endpoint public à créer (quota à imputer), **sans
  persistance** (décision produit — rien dans `answer-storage.ts`).
- **Images lues par l'assistant (`read_resource_image`)** : aucun
  redimensionnement côté back (pas de Pillow — règle « pas de binaire par le
  backend ») — une image de plus de `IMAGE_MAX_BYTES` (3,5 Mo brut,
  `OpenCartableBack/app/course_assistant/tools.py`) est refusée au modèle, et
  une image lourde coûte cher en tokens à chaque lecture. Si les captures
  volumineuses deviennent courantes : réduire à l'upload côté front (canvas)
  ou accepter Pillow pour une miniature. L'image n'est **pas rejouée** aux
  tours suivants (décision actée, le modèle relit au besoin).

## Autres dettes déjà actées dans les CLAUDE.md

- **Job de réconciliation des orphelins S3** (un échec de purge S3 après commit
  laisse des objets orphelins dans le bucket) — cf. `OpenCartableBack/CLAUDE.md`,
  « Nettoyage S3 aux suppressions ».
- **Keepalive SSE périodique** sur les routes IA streamées (protection contre les
  timeouts du proxy pendant les longues générations) — cf.
  `OpenCartableBack/CLAUDE.md`, section « Client IA générique ». (Le flux HITL
  n'aggrave plus le cas : il se **ferme** à la proposition et la décision
  rouvre un flux — plus d'attente silencieuse sur connexion ouverte.)
- **Import : compat archives v1 (manifest français)** maintenue via
  `normalize_manifest_v1` (`OpenCartableBack/app/course_transfer/schemas.py`) —
  à retirer un jour si l'on décide de ne plus supporter les exports antérieurs
  à la nomenclature anglaise.
