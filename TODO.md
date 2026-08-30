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
