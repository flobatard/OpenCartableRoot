# TODO

Dettes techniques acceptées « à terme ». Une ligne par dette, avec le point d'entrée dans le code ; retirer la ligne quand c'est livré. Les décisions qui les motivent sont dans [docs/decisions.md](docs/decisions.md).

## Back

- **Checkpointer HITL en mémoire → `AsyncPostgresSaver` au passage multi-nœud** : `InMemorySaver` et le registre `app/course_assistant/hitl.py` sont process-locaux (mono-worker obligatoire, reprises perdues au redémarrage). Le jour venu : `langgraph-checkpoint-postgres` + psycopg3, tables créées hors Alembic, registre en table.
- **Keepalive SSE périodique** sur les routes IA streamées (timeouts de proxy pendant les longues générations) — `app/core/sse.py`.
- **Images lues par l'assistant sans redimensionnement** (pas de Pillow) : une image > `IMAGE_MAX_BYTES` est refusée, une image lourde coûte cher à chaque lecture — `app/course_assistant/tools.py`. Piste : réduire à l'upload côté front, ou accepter Pillow.
- **Injection par l'élève (risque assumé)** : le modèle du tuteur voit le corrigé de la question cible ; la révélation est bornée serveur (`guard_reveal`) mais un modèle faible peut paraphraser — `app/student_exercises/`. Pistes : second appel « juge » sans corrigé, filtre de similarité.
- **Pas de pagination du fil du tuteur** (plafond `MAX_TURNS_PER_QUESTION` = 100 tours par question, chargés d'un bloc) — `app/student_exercises/`.
- **Vue professeur des soumissions d'élèves** : seul un résumé par question (compteurs) existe ; aucune route ne lit les contenus. À concevoir avec la question de la vie privée (consentement, anonymisation).
- **Compat des archives d'export v1** (manifest français) maintenue par `normalize_manifest_v1` — `app/course_transfer/schemas.py` ; à retirer si l'on cesse de supporter ces exports.
- **`app/ai/` (routes de smoke-test)** supprimable une fois ses tests de cascade config × quota portés au niveau service dans `tests/test_ai_credentials_api.py`.

## Front

- **Reprise HITL non ré-offerte après rechargement de page** : la proposition en attente vit dans l'état front ; le back garde la reprise jusqu'à son TTL. Piste : à l'ouverture d'une conversation, détecter un round d'un tool de proposition sans tour `tool` et re-proposer la décision.
- **Ctrl-Z partiel sur les propositions d'exercice** : seuls les champs markdown passent par Monaco ; corrigé, ajout et suppression passent par le formulaire (le prof rejette ou redemande). Le déplacement d'une question par l'IA n'est pas couvert.
- **Assistant de module sans retour d'exécution** : le modèle ne voit ni les erreurs JS ni la console de l'iframe. Piste : le bridge de `shared/module-runner/module-document.ts` capture `window.onerror`/`console.error` et les remonte au tour — touche le contrat du bridge et le prompt `MODULE_RUNTIME` du back, à traiter comme un lot à part.
- **Fil du tuteur sans dévoilement progressif** : le texte streamé est re-rendu à chaque token (pas le lissage du chat prof) — à reprendre si le rendu paraît saccadé.

## Opérateur (réglages, pas du code)

- **`PURGE_S3_ORPHANS_DRY_RUN`** est à `true` : la réconciliation ne fait que journaliser. À basculer après relecture des logs d'une première passe en production.
- **Purges désactivées par défaut** (`PURGE_AI_CONVERSATIONS_DAYS`, `PURGE_EXERCISE_SUBMISSIONS_DAYS` = 0) : les activer est une décision produit / vie privée. Si la seconde est activée à grande échelle, un index sur `exercise_submissions.created_at` seul deviendra utile.
- **Pas d'agrégation avant purge** de `ai_daily_usage` : l'historique au-delà de la rétention est perdu, pas résumé.
