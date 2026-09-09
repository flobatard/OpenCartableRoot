# TODO

Dettes techniques acceptées « à terme ». Une ligne par dette, avec le point d'entrée dans le code ; retirer la ligne quand c'est livré. Les décisions qui les motivent sont dans [docs/decisions.md](docs/decisions.md).

## Back

- **Checkpointer HITL en mémoire → `AsyncPostgresSaver` au passage multi-nœud** : `InMemorySaver` et le registre `app/course_assistant/hitl.py` sont process-locaux (mono-worker obligatoire, reprises perdues au redémarrage). Le jour venu : `langgraph-checkpoint-postgres` + psycopg3, tables créées hors Alembic, registre en table.
- **Keepalive SSE périodique** sur les routes IA streamées (timeouts de proxy pendant les longues générations) — `app/core/sse.py`.
- **Images lues par l'assistant sans redimensionnement** (pas de Pillow) : une image > `IMAGE_MAX_BYTES` est refusée, une image lourde coûte cher à chaque lecture — `app/course_assistant/tools.py`. Piste : réduire à l'upload côté front, ou accepter Pillow.
- **Injection par l'élève (risque assumé)** : le modèle du tuteur voit le corrigé de la question cible, et c'est lui qui décide de le dévoiler — `guard_reveal` ne fait que vérifier la cohérence de ce qu'il déclare (`verdict`/`effort`), tous deux issus du même tour manipulable. Un élève insistant ou une injection obtient donc soit une paraphrase, soit un `record_verdict(correct|sufficient, reveal=true)` qui fait servir le corrigé verbatim — `app/student_exercises/`. Pistes : second appel « juge » sans corrigé ni historique élève, filtre de similarité, plafond de révélations par question.
- **Pas de pagination du fil du tuteur** (plafond `MAX_TURNS_PER_QUESTION` = 100 tours par question, chargés d'un bloc) — `app/student_exercises/`.
- **Vue professeur des soumissions d'élèves** : seul un résumé par question (compteurs) existe ; aucune route ne lit les contenus. À concevoir avec la question de la vie privée (consentement, anonymisation).
- **Compat des archives d'export v1** (manifest français) maintenue par `normalize_manifest_v1` — `app/course_transfer/schemas.py` ; à retirer si l'on cesse de supporter ces exports.
- **Cours d'exemple non idempotent** : chaque `POST /courses/starter` crée un nouveau cours (pas de colonne `is_starter`, et le titre est renommable donc inexploitable comme marqueur) — `app/starter_course/`. À revoir seulement si des doublons remontent.
- **Manifeste du cours d'exemple vérifié en forme, jamais en rendu** : les tests valident la syntaxe des blocs de code (Mermaid, JSXGraph, TikZ) et l'absence de macro KaTeX, mais rien ne rend réellement le cours — une régression du pipeline de rendu front ne casserait aucun test back — `tests/test_starter_course.py`.
- **`app/ai/` (routes de smoke-test)** supprimable une fois ses tests de cascade config × quota portés au niveau service dans `tests/test_ai_credentials_api.py`.
- **Usage IA perdu sur erreur mid-stream** : l'accumulateur de tokens vit dans `_stream_agent` (`app/core/ai/client.py`) et l'exception ne le transporte pas ; `_AssistantTurn.failed` persiste le partiel sans usage (un tour en erreur affiche donc moins que consommé). Pistes : événement d'usage partiel avant `error`, ou accumulateur côté sink.
- **Tokens d'écriture de cache non distingués** : seuls les tokens *lus* en cache (`input_token_details.cache_read`) sont remontés dans `cached_input_tokens` ; l'écriture (Anthropic, facturée ×1,25) reste confondue avec l'entrée ordinaire — `app/core/ai/messages.py`. À exposer si le coût réel doit être affiché.
- **Tuteur : deux appels modèle par tour** (protocole `record_verdict` puis rédaction, la garde de révélation en dépend) : le second appel renvoie tout le contexte — bon marché avec un provider à cache, plein tarif sinon — `app/student_exercises/streaming.py`. Piste : verdict en fin de réponse structurée, si la garde serveur peut se contenter d'une vérification a posteriori.
- **Sommaire sans titres setext** : `markdown_outline` (`app/course_assistant/render.py`) ne détecte que les titres ATX (`#`) ; un bloc écrit avec des soulignés `===`/`---` apparaît sans plan (toujours lisible via `read_block`).
- **Catalogue de raisonnement à entretenir** : aucun provider ne publie ses niveaux d'effort par modèle — les règles par préfixe de `app/core/ai/reasoning.py` (OpenAI, Gemini, Ollama, alias Anthropic hors profil embarqué) sont à compléter à chaque nouvelle famille ; un modèle inconnu reçoit les options génériques du provider (`known=false`) et le provider tranche à l'appel (422 neutre). Restent non exposés : la bascule `openai_compatible` hors noms OpenAI (`chat_template_kwargs`, `include_reasoning`…), les préférences de l'IA par défaut (`AI_REASONING*`) au front.

## Front

- **Reprise HITL non ré-offerte après rechargement de page** : la proposition en attente vit dans l'état front ; le back garde la reprise jusqu'à son TTL. Piste : à l'ouverture d'une conversation, détecter un round d'un tool de proposition sans tour `tool` et re-proposer la décision.
- **Ctrl-Z partiel sur les propositions d'exercice** : seuls les champs markdown passent par Monaco ; corrigé, ajout et suppression passent par le formulaire (le prof rejette ou redemande). Le déplacement d'une question par l'IA n'est pas couvert.
- **Assistant de module sans retour d'exécution** : le modèle ne voit ni les erreurs JS ni la console de l'iframe. Piste : le bridge de `shared/module-runner/module-document.ts` capture `window.onerror`/`console.error` et les remonte au tour — touche le contrat du bridge et le prompt `MODULE_RUNTIME` du back, à traiter comme un lot à part.
- **Fil du tuteur sans dévoilement progressif** : le texte streamé est re-rendu à chaque token (pas le lissage du chat prof) — à reprendre si le rendu paraît saccadé.

## Opérateur (réglages, pas du code)

- **`PURGE_S3_ORPHANS_DRY_RUN`** est à `true` : la réconciliation ne fait que journaliser. À basculer après relecture des logs d'une première passe en production.
- **Purges désactivées par défaut** (`PURGE_AI_CONVERSATIONS_DAYS`, `PURGE_EXERCISE_SUBMISSIONS_DAYS` = 0) : les activer est une décision produit / vie privée. Si la seconde est activée à grande échelle, un index sur `exercise_submissions.created_at` seul deviendra utile.
- **Pas d'agrégation avant purge** de `ai_daily_usage` : l'historique au-delà de la rétention est perdu, pas résumé.
