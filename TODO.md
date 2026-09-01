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
- **Contextes d'édition restants (exercice, module) et résolution d'exercice
  élève** : le contexte **bloc texte (`block_text`) est livré** et fixe le
  motif à suivre — flux **HITL par interrupt/resume LangGraph, checkpointer
  `InMemorySaver`** (arbitrage acté, révise le « sans checkpointer » du
  premier lot) : l'agent propose via un tool sans mutation (proposition dans
  les args du `tool_call`, persistés), le tool **fige le run**
  (`agent_interrupt`, `app/core/ai/`), le flux SSE émet `interrupt` et se
  ferme ; la route `POST .../proposals/{id}/decision` **reprend le run dans un
  nouveau flux** (`Command(resume=…)`) — le résultat du tool est la décision
  commentée. Registre des reprises :
  `OpenCartableBack/app/course_assistant/hitl.py` (TTL 6 h). Côté front,
  `CourseChat` a trois modes (global/block/placeholder), l'état est extrait en
  `AssistantChatState` instanciable par hôte (état `awaiting` +
  `pendingProposal`/`resumeProposal`) et la revue (`app-proposal-review`, diff
  à la place de l'éditeur) vit chez l'hôte — un nouveau contexte = étendre le
  `Literal` de `ConversationCreate`, un prompt/tool dédiés, un mode et une
  revue de plus.
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
  détecter à l'ouverture d'une conversation un round `propose_block_edit`
  sans tour `tool` et re-proposer la décision.
- **Chat élève anonyme** (reporté — décision utilisateur) : régime public sans
  JWT à concevoir, avec la question de l'imputation du quota d'un élève sans
  compte.
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
