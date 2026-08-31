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
- **Contextes d'édition (bloc texte, exercice, module) et résolution d'exercice
  élève** avec leurs flux HITL : lever la garde `globalMode` de `CourseChat`,
  étendre le `Literal` de `ConversationCreate` (le CHECK en base accepte déjà
  les 4 contextes persistés). Pour les interrupts HITL inter-requêtes, arbitrer
  l'introduction d'un **checkpointer LangGraph** (tables hors Alembic, driver
  psycopg) — non introduit au lot 1, l'état est reconstruit depuis nos tables.
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
  `OpenCartableBack/CLAUDE.md`, section « Client IA générique ».
- **Import : compat archives v1 (manifest français)** maintenue via
  `normalize_manifest_v1` (`OpenCartableBack/app/course_transfer/schemas.py`) —
  à retirer un jour si l'on décide de ne plus supporter les exports antérieurs
  à la nomenclature anglaise.
