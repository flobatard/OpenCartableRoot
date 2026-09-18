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

## Hors jalon — Scheduler de maintenance

Back (2026-09-18), décision 37 : la boucle shell de purge devient un scheduler résident.

- Service compose `scheduler` (ex-`purge`) : `AsyncIOScheduler` in-process, **une expression cron par job** dans `config/*.yaml` (`PURGE_INTERVAL_SECONDS` supprimé) — purges légères quotidiennes vers 03 h, l'`UPDATE` des tours d'outil et le balayage du bucket répartis sur deux nuits de week-end distinctes. Arrêt propre sur SIGTERM (handler explicite, passe en cours tenue derrière un `shield` puis drainée, `engine.dispose()`).
- Registre unique des neuf jobs (noms anglais, libellés français pour les logs), runner partagé par le scheduler et le one-shot `python -m app.maintenance [job…]`, garde de schéma Alembic rejouée **avant chaque job** (statut `skipped`, sans jamais attendre).
- Table `maintenance_job_state` : une ligne par job, upsertée à chaque passe sur une session dédiée (statut, compte, durée, erreur tronquée et expurgée, échecs consécutifs, total, détail JSONB).
- Deux contrôles en **lecture seule** : `storage_inventory` (lignes par table, octets sur disque, objets et octets par préfixe S3) et `missing_s3_objects` (lignes `available` pointant un objet S3 absent — HEAD à concurrence bornée, plafond par passe, curseur tournant).

## Hors jalon — Consommation de tokens de l'assistant

- Back : l'événement `interrupt` porte l'usage des rounds déjà joués et le segment persisté à l'interruption le conserve (un tour HITL = plusieurs segments dont la somme est le tour) ; `stream_usage` forcé pour la famille OpenAI (`openai` derrière une `base_url`, `openai_compatible`), sans lequel aucun usage n'arrivait en flux.
- Front : ligne « entrée · sortie » sous chaque tour de l'assistant prof et total de la conversation dans le pied du chat (`core/course-assistant/usage.ts`, sommes par tour et par conversation sur les messages servis par l'API).
- Back (2026-09-08) : le cours n'entre plus dans le contexte qu'en **sommaire structuré** (jamais le contenu d'un bloc, cible du tour en entier, lecture à la demande par `read_block`), system prompt statique par contexte et contexte du tour en tête du message utilisateur, replay abrégé à hystérésis, prompts rationalisés (règles partagées assistant/tuteur, protocole HITL énoncé une fois), référence de la question ajoutée donnée au modèle sans relecture ; cache de prompt Anthropic (middleware, system prompt passé au graphe) et `cached_input_tokens` dans l'usage SSE et les colonnes. Sur le cours d'exemple : −40 à −60 % par appel modèle, jusqu'à −80 % sur un gros cours (décision 23).
- Front (2026-09-08) : « dont N en cache » sur la ligne du tour et le total de la conversation quand le provider relaie des tokens lus en cache.

## Hors jalon — Cours d'exemple à l'onboarding

- Back : `app/starter_course/` — manifeste v2 embarqué sans aucun binaire (neuf blocs : formules, diagrammes, figures, schémas, exercice, module interactif, référence `oc-module:`), seed best-effort à la première complétion d'un profil de prof, route de rattrapage `POST /courses/starter`. La phase base de données de l'import devient `insert_manifest_course`, partagée par les deux chemins.
- Front : bouton « Charger le cours d'exemple » dans l'état vide de « Mes cours », et en entrée discrète (bouton ghost) sous une liste non vide.

## Hors jalon — Raisonnement et effort des modèles

- Back : préférences `reasoning` (défaut / activé et affiché / coupé) et `reasoning_effort` (niveau natif du provider : jusqu'à minimal…xhigh/max) persistées avec le credential personnel (`users.ai_reasoning`, `ai_reasoning_effort`, migration) et transmises par la cascade `effective_config` à l'assistant et au tuteur ; capacités déclarées par provider (`PROVIDERS_WITH_REASONING_TOGGLE` / `PROVIDER_REASONING_EFFORTS`, 422 hors capacités), catalogue des options par couple (provider, modèle) (`app/core/ai/reasoning.py`, adossé aux profils embarqués de langchain, servi avec le credential et par `POST /users/me/ai-credentials/reasoning-options`) et encodage par provider dans `app/core/ai/providers.py` (Anthropic par paliers du profil embarqué, Gemini par famille, OpenAI `reasoning_effort`, Ollama `think`) ; pour l'IA par défaut, réglages opérateur `AI_REASONING` / `AI_REASONING_EFFORT` (absents = comportement historique) résolus par `resolve_config` avec la même règle de gating (décision 24).
- Front : deux `<select>` natifs après le champ modèle de Réglages IA (options du catalogue pour le couple saisi, re-sondées au changement de provider et au blur du modèle, enregistrées avec le formulaire) et deux sélecteurs compacts dans le pied du chat qui enregistrent aussitôt par le PUT du credential reconstruit ; formulaire non modifié réaligné sur le signal quand le pied écrit.

## Hors jalon — Configurations IA nommées

- Back (2026-09-09) : table `ai_configurations` (plusieurs configurations nommées par utilisateur, clé chiffrée avec un sel par ligne, au plus une active par index partiel unique, aucune active = IA par défaut), migration copiant le credential unique de `users.ai_*` en première configuration active puis supprimant les colonnes ; `/users/me/ai-credentials` devient une collection (enveloppe `configurations` + `active_id`, `POST` crée et active, `PUT /active` bascule, `PUT`/`DELETE /{id}`, sondes avec `config_id`), plafond de 10 ; la cascade `effective_config` lit la configuration active (décision 25).
- Front : écran Réglages IA en liste de radios-cartes (IA par défaut + une carte par configuration, cocher bascule aussitôt, Modifier/Supprimer par carte) avec éditeur de création/modification (nom après le champ clé), signal `AiCredentialsService` en enveloppe (`create`/`update`/`remove`/`activate`), pied du chat affichant le nom de la configuration active avec un menu rapide de bascule dans l'engrenage (puis « Gérer les configurations… » vers la modale) et un sélecteur de modèle à autocomplétion (modèles du provider listés avec la clé de l'active) qui change le modèle de l'active en place ; les sélecteurs de raisonnement du pied écrivent sur l'active.

## Hors jalon — Mode « Édition auto » des propositions

- Front (2026-09-10) : interrupteur « Édition auto » dans le pied des chats d'édition (bloc texte, exercice, module) ; activé, chaque proposition HITL est appliquée dans l'éditeur et acceptée sans revue (`ProposalModeService`, préférence du navigateur `oc-assistant-proposal-mode` ; `ProposalHost.autoAccept`), sauf la suppression de question, toujours revue ; cible disparue ou envoi en échec = repli sur la revue manuelle. Aucun changement back (décision 27).

## Hors jalon — Langages pluridisciplinaires

Langages du markdown de cours pour les matières autres que les maths, un commit par langage et par dépôt ; chacun apporte sa page de doc, sa section dans l'aide de l'éditeur, sa clause dans le catalogue de l'assistant et son bloc dans le cours d'exemple.

- mhchem (2026-09-10) : notation chimique dans les formules KaTeX — `$\ce{…}$` (équations, ions, états, équilibres) et `$\pu{…}$` (grandeurs et unités) ; extension `katex/contrib/mhchem` chargée avec KaTeX, page de doc intégrée `mhchem`, règle commune `MATH_RULE` de l'assistant (chat global et tuteur compris), bloc « Écrire de la chimie » et garde « jamais hors d'une formule » sur le manifeste.
- timeline (2026-09-10) : frise chronologique ```` ```timeline ```` — `period=début,fin,libellé` en bandes sous l'axe, `event=date,libellé` au-dessus (dates `AAAA`, négatives avant J.-C., ou `AAAA-MM-JJ`), `start`/`end`/`step` optionnels ; SVG dessiné par le template sans dépendance (couloirs gloutons contre les chevauchements, largeur calée sur le conteneur, thème sombre natif, imprimable, liste accessible), lignes invalides comptées ; clause du catalogue d'édition, bloc « Construire une frise chronologique » et garde de forme sur le manifeste.
- smiles (2026-09-11) : molécules ```` ```smiles ```` — une formule topologique par ligne, `SMILES | légende` ; SmilesDrawer (MIT, chunk paresseux ~56 ko gzip) en formule développée complète (`compactDrawing: false`), SVG re-sanitisé sans son `<style>` global, planche claire fixe `--figure-board` (nouveau token, TikZ migré), notice par molécule invalide, imprimable grâce à l'unicité des ids SVG à l'impression (`uniquifySvgIds`, correctif préalable qui profite aussi à Mermaid) ; clause du catalogue, bloc « Dessiner une molécule » et garde de forme.
- vegalite (2026-09-11) : graphiques de données ```` ```vegalite ```` — spécification Vega-Lite JSON rendue en SVG statique durci (décision 28 : clé `url` refusée, loader bloqué, expressions interprétées, SVG re-sanitisé sur planche claire), largeur calée sur le conteneur, fence coloré en JSON dans l'éditeur, erreurs JSON / spec / données externes distinguées ; ~250 ko gzip paresseux ; clause du catalogue, bloc « Tracer un graphique de données » et garde « JSON en ligne sans url ».
- abc (2026-09-11) : partitions ```` ```abc ```` — notation ABC gravée par abcjs (~149 ko gzip paresseux) sur la largeur de la planche avec retour à la ligne des mesures, re-sanitisée ; lecture au piano au clic (`AudioContext` à la demande, un seul lecteur par page) avec une banque de sons FluidR3 (CC BY 3.0, 88 notes, ~2 Mo) auto-hébergée dans `public/abcjs-soundfont/` — seules les notes jouées sont téléchargées, aucune requête hors origine —, directives `%%MIDI` retirées (piano forcé), avertissements de l'analyseur repliés ; clause du catalogue, bloc « Écrire une partition » et garde (en-têtes `X:`…`K:`, pas de MIDI).
- sql (2026-09-11) : requêtes SQLite exécutables ```` ```sql ```` (décision 29) — préparation repliée avant `-- @query`, requête modifiable, résultats en tableaux plafonnés à 200 lignes, erreurs de préparation et de requête distinguées, délai de 5 s ; sql.js (~0,7 Mo) servi depuis `/assets/sqljs/` et téléchargé au premier clic, dans un worker classique tenu par `SqlRuntime` ; brique commune `RunnableCode` ; `server.ts` pose `no-cache` sur les runtimes et une CSP d'en-tête sur les scripts de worker ; clause du catalogue, bloc « Interroger une base de données » et garde qui exécute le fence dans `sqlite3`.
- python (2026-09-11) : programmes Python exécutables ```` ```python ```` (décision 29) — Pyodide 314 dans un worker module, numpy et matplotlib (12 wheels vérifiées par sha256, préparées au postinstall par `scripts/prepare-pyodide.mjs`, lock élagué), `input()` alimenté par le champ Entrées avec écho, traceback réduit au code de l'élève, figures matplotlib en PNG, délai de 15 s et bouton Arrêter ; ~6 Mo gzip au premier clic, ~13 Mo de plus au premier import de numpy/matplotlib ; vérifié sous la CSP de production (réseau hors origine bloqué). Clause du catalogue, bloc « Programmer en Python » et garde (compilation, imports limités aux paquets hébergés). Le cours d'exemple compte désormais seize blocs.
- circuits électriques (2026-09-11) : pas de nouveau fence — la bibliothèque TikZ `circuits.ee.IEC` (symboles IEC) et `circuits.logic.*` (portes logiques) du TikZJax embarqué, documentées par une page complémentaire `tikz-circuits` (galerie des dipôles, valeurs et unités, DEL/rhéostat/capteurs, demi-additionneur ; `MarkdownExtensionDoc.guides`) ; les `\usetikzlibrary{…}` d'un fence montent au préambule (`data-tikz-libraries`), sans quoi la figure est décalée ; lien depuis l'aide et la page TikZ, clause « circuitikz indisponible » dans le catalogue de l'assistant, bloc « Dessiner un circuit électrique » dans le cours d'exemple (désormais dix-sept blocs) et garde des bibliothèques TikZ embarquées.
- encadrés (2026-09-11) : pas de fence — syntaxe des alertes GitHub dans le pipeline markdown (décision 31) : une citation ouverte par `> [!DEFINITION]`, `[!RETENIR]`, `[!METHODE]`, `[!EXEMPLE]`, `[!REMARQUE]` ou `[!ATTENTION]` devient un encadré (fond teinté, liseré, icône par type, titre par défaut dans la langue de l'interface — un changement de langue rejoue le rendu), texte après le marqueur en titre personnalisé, contenu markdown complet ; mots-clés sans casse ni accents, alias GitHub (`NOTE`, `TIP`, `IMPORTANT`, `WARNING`, `CAUTION`) ; imprimés en couleurs claires, insécables. Au passage : `--text-warning` sombre du contenu (manquant), tokens sémantiques re-forcés en clair à l'impression, blocs de code du markdown qui défilent dans leur cadre au lieu d'élargir la page. Page de doc intégrée `callouts`, section d'aide, clause du catalogue d'édition, bloc « Encadrer l'essentiel » et garde « mot-clé connu, en tête de citation » sur le manifeste.
- passage (2026-09-11) : extrait à commenter ```` ```passage ```` — une ligne de source par ligne numérotée, numéro dans la marge toutes les 5 lignes (« l. 12 »), lignes vides non comptées, `start=`/`step=` optionnels en tête du bloc ; rendu par le template sans dépendance, numéros non sélectionnables et annoncés « Ligne n », suite d'une ligne repliée en retrait profond, imprimable. Clause du catalogue, bloc « Numéroter les lignes d'un texte » (Hugo, La Fontaine) et garde « au moins un numéro affiché » sur le manifeste.
- colonnes (2026-09-17) : pas de fence — conteneur `::: columns` … `+++` … `:::` (décision 35), ratio facultatif `1:2`/`2:1`, chaque colonne en markdown complet (encadrés, formules, figures, code exécutable, images, modules) ; empilement selon la largeur disponible (seuil 26em, sans requête de largeur), côte à côte et insécables à l'impression ; marqueurs colorés dans l'éditeur, bouton « Insérer des colonnes » (la sélection devient la colonne de gauche, une seule étape d'annulation). Page de doc intégrée `columns`, section d'aide, clause du catalogue d'édition, bloc « Mettre en colonnes » (une définition et son exemple, puis une suite récurrente à côté du module qui en calcule les termes) et garde de forme sur le manifeste ; le cours d'exemple compte désormais vingt-huit blocs.

## Hors jalon — Librairies des modules

- Front et back (2026-09-11) : six librairies préinstallées dans le bac à sable des modules — Matter.js, Chart.js, p5.js 2.x, JSXGraph, D3, Three.js — déclarées par le pragma `// @oc-libs: …` en tête du JS et inlinées par le runtime sans rouvrir la CSP (décision 30) ; fichiers préparés au postinstall (`scripts/prepare-module-libs.mjs`, Three bundlé en IIFE), lus une fois par session et servis `no-cache` ; note sous l'iframe si une lecture échoue, aide de l'onglet JS de l'éditeur (pragma, noms disponibles, noms inconnus signalés) ; puce « Bibliothèques » et pièges de l'auto-resize dans `MODULE_RUNTIME`. Cours d'exemple : bloc « Des bibliothèques dans les modules » et six modules (chute et rebonds, proies et prédateurs, diffusion à travers une membrane, nombre dérivé et tangente, arbre de parenté des vertébrés, géométrie des molécules), désormais vingt-quatre blocs ; tests de pragma, d'usage et de couverture du catalogue.

## Hors jalon — Expérience mobile

Front (2026-09-12), cinq lots (décision 32) : le site défilait horizontalement sur toutes ses pages (574 px de contenu pour un viewport de 375 px).

- Fondations : partiel `styles/_breakpoints.scss` (mixins `mobile` ≤ 640, `stacked` ≤ 900, `wide`, `touch`, sans sortie CSS, résolu par `includePaths`) et §12 du design system ; header à symbole + burger (disclosure, nav et préférences dans un seul markup basculé en CSS) ; une seule gouttière (`.app-main`) sur téléphone, les pages retirent leur padding latéral ; `.tabs` défilant avec l'onglet actif ramené en vue (`ocActiveTabInView`) ; bandeau de consentement empilé, pied de page en colonne, `100svh`, `text-size-adjust`.
- Pages élèves et contenu de cours : tableaux GFM enveloppés dans `div.course-table` défilant (override du renderer `table`), mots longs et URL qui passent à la ligne, citation simple à filet, listes resserrées ; boutons imprimer / style de lecture visibles et dans le flux au tactile ; lecture en haut de page à l'arrivée sur un bloc, navigation Précédent/Suivant en deux lignes ; retraits de l'exercice et du tuteur resserrés, champs à 16 px minimum (zoom iOS) ; carte document qui passe à la ligne, repli du PDF inline sans visionneuse intégrée (Chrome Android) ; module interactif agrandissable en calque plein écran ; JSXGraph en 16:9 qui laisse défiler la page à un doigt, GeoGebra au ratio de sa config, TikZ/Vega en `safe center`.
- Recherche et documentation : grilles à minimum borné, facettes repliées par défaut sous 900 px, pagination en deux lignes, playground en `minmax(0, 1fr)`.
- Pages prof (cible « utilisable ») : rangées d'actions des listes sur leur propre ligne (l'état « Confirmer … » débordait), cibles tactiles (flèches, poignées, chevrons), éditeurs de bloc et de module en **bascule de vue** Éditeur ⇄ Assistant plutôt qu'en empilement, hauteurs en `dvh`.
- Assistant et notifications : `pointer-events: none` sur l'hôte du panneau flottant (sa bande transparente avalait les taps du bas de l'écran), panneau effacé sur les pages d'éditeur sous 900 px, `interactive-widget=resizes-content` (clavier virtuel), pied du chat à 36 px au tactile, légendes de diff masquées quand Monaco passe en vue inline, snackbar au-dessus de la pastille.

Vérifié en Chromium headless (émulation mobile 360 / 375 / 414 px, thèmes clair et sombre) sur la home, la recherche, la documentation, les pages élèves et l'espace prof : `scrollWidth === clientWidth` partout, desktop inchangé à 1280 px.

## Hors jalon — Export HTML autonome

Front (2026-09-12), décision 33 : second format d'export à côté du PDF, pour ce que le papier perd — les modules interactifs.

- `shared/export-html/` clone le DOM rendu comme `PrintService`, puis sérialise : chrome morte retirée (miroir des blocs `@media print`), liens relatifs absolutisés (le fichier s'ouvre en `file://`), médias et PDF embarqués remplacés par une note vers l'URL front stable, images de la bibliothèque intégrées en `data:` sous un budget de 25 Mio, CSS pris sur le **CSSOM vivant** (`@media print` de l'app retiré, polices d'interface abandonnées au profit des piles système, polices KaTeX embarquées seulement si le contenu porte des formules).
- **Modules vivants** : chaque hôte `data-oc-module-id` devient une iframe `srcdoc` composée par `composeModuleDocument` (CSP `default-src 'none'` inchangée, librairies `@oc-libs` inlinées), plus un script de dix lignes dans la page exportée qui applique les messages `oc-module:resize`. `composeModule` est extrait de `ModuleRunner` et partagé par le bac à sable, l'export et l'éditeur.
- UI : modale « Exporter » commune aux deux formats (`shared/export-dialog/`), branchée sur l'aperçu prof, la page « Cours entier » de l'élève et le bouton flottant de chaque bloc (monté à la demande) ; « Télécharger en HTML » dans l'éditeur de module, sur le code courant des trois Monaco.
- Vérifié en vrai navigateur, fichier ouvert en `file://` **DNS coupé** : maths, Mermaid, encadré et tableau rendus, `Chart` chargé et canvas peint dans l'iframe, hauteur ajustée par le pont, zéro requête réseau, aucun débordement à 375 px, page toujours imprimable. Poids : 652 ko pour deux blocs avec formules et un module Chart.js (338 ko de polices KaTeX, 226 ko de librairie).

## Hors jalon — Questions de l'assistant

Back et front (2026-09-13), décision 34 : l'assistant pose ses questions au professeur au lieu de deviner.

- Back : tool bloquant `ask_questions` dans les quatre contextes (1 à 4 questions à choix unique ou multiple, suggestions validées, « Autre » ajouté par l'interface), route de réponse `…/questions/{tool_call_id}/answer` contrôlée contre la forme attendue avant consommation (422 réessayable, 404 si rien n'attend), genre `kind` sur l'interrupt et le registre ; tout tour d'assistant checkpointé avec `durability="exit"`, threads purgés à l'abandon du flux, à la suppression d'une conversation et à l'expiration ; garde « un seul outil bloquant par réponse » dans le graphe agent (appels parallèles écartés, reprise toujours possible) ; replay qui replie les rounds incomplets et garde les réponses entières ; règle de prompt `QUESTIONS_RULE`.
- Front : formulaire en étapes à la place du composer (cartes de choix radio ou cases, « Autre » coché à la saisie, Précédent / Suivant / Répondre, croix du refus), carte de la série dans le fil (en attente, réponses, ou sans réponse), chat ou panneau replié déplié à l'arrivée d'une série, questions reproposées à la réouverture d'une conversation (notice si elles ont expiré), mesure en compteurs et formulaire masqué des replays.

## Hors jalon — Édition globale par sous-assistants

Back et front (2026-09-17), décision 36 : l'assistant du cours fait modifier des blocs et des modules par des sous-assistants d'édition, le professeur validant chaque proposition où qu'il soit dans le cours.

- Back : tools bloquants `edit_block`/`edit_module` du contexte `course` à `allow_edit` (prompt d'édition globale, cibles éligibles en `enum`, consignes bornées), driver du flux SSE qui enchaîne l'assistant → le sous-assistant (descripteur d'édition de la cible, thread propre, événements tagués `agent`) → la reprise de l'assistant avec le compte rendu ; registre à `delegation` (deux threads purgés ensemble, usage reporté d'un flux sans ligne), reprise du sous-assistant par les routes de décision et de réponse, plafond de délégations par tour, transcription du sous-assistant non persistée (seuls l'appel et son compte rendu le sont).
- Front : interrupteur « Édition globale » du panneau flottant (préférence du navigateur, `allow_edit` par tour), carte de délégation dans le fil (cible, consignes, activité du sous-assistant imbriquée en direct, compte rendu), fenêtre de revue globale `<dialog>` hébergée par l'outlet à côté du panneau (revues texte, exercice et module réutilisées, rouverte par « Revoir »), application sur la cible via l'éditeur monté (registre des appliqueurs, Ctrl-Z, flush d'autosave) ou en headless (PATCH du bloc ou du module, signaux patchés — la page cours suit), cible préchargée dès la délégation et original figé par proposition, mode « Édition auto » commun.

## Hors jalon — Backoffice : rôles et jobs de maintenance

Back et front (2026-09-18), décision 39 : un rôle de plateforme ouvre un backoffice, dont le premier écran suit le scheduler de maintenance et en lance les jobs à la main.

- Back : colonne `users.platform_role` (`public` / `super_admin`) exposée par `/users/me`, jamais écrite par une route, posée par la CLI `python -m app.users.roles grant|revoke|list` ; garde `require_super_admin` (403) sur le router `/admin` ; `GET /admin/maintenance/jobs` (statut du scheduler, plan, demande en attente et dernière passe de chaque job du registre) et `POST /admin/maintenance/jobs/{job}/run` (202, dépôt d'une demande — 409 si le job est déjà demandé ou en cours, 503 si Redis ne répond pas). Canal de contrôle par un **Redis** partagé (nouveau service des deux compose, sans persistance, client confiné dans `app/core/kv.py`) : demandes en `SET NX` à expiration, relevées par le scheduler quand aucune passe n'est en vol (prise en `GETDEL`, au plus une exécution), statut publié au démarrage et autour de chaque passe (plan en UTC, passe en cours), boucle asyncio à côté des crons, panne du canal journalisée une fois. Ni battement ni vérification de vie, et plus aucun accès périodique à Postgres : la base managée (Neon) peut se mettre en veille entre deux passes. Alembic importé paresseusement pour que l'API ne le charge pas.
- Front : entrée « Administration » du menu utilisateur réservée au rôle, `superAdminGuard` fail-closed derrière `authGuard`, coquille à menu latéral (`/:lang/admin`, entrée « Jobs ») et page des jobs : bandeau du scheduler (statut publié — démarrage, fuseau des crons, passe en cours —, aucun statut, ou canal de contrôle injoignable), une carte par job (libellé et description traduits, cron et prochaine passe, rétention, dernière passe — statut en toutes lettres, durée, compte au sens propre à chaque job, erreur ou raison du saut, échecs consécutifs, détail JSON dépliable), lancement en deux clics, rafraîchissement automatique toutes les 5 s tant qu'un job est demandé ou en cours ; page masquée des replays de session.

Vérifié en navigateur headless sur fausse API : redirection de `/fr/admin` vers `/fr/admin/jobs`, compte `public` renvoyé à l'accueil, lancement en deux temps et demande affichée, thèmes clair et sombre, aucun débordement à 375 px.

## Reste du J5

Vue professeur des soumissions d'élèves et RAG éventuel — voir [../TODO.md](../TODO.md).
