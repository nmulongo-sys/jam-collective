# jam-collective — plateforme de reprises de classe

Page unique et autonome pour monter des reprises à plusieurs : chaque morceau est un
projet où l'on s'inscrit à un pupitre, dépose partitions et vidéos, propose des
répétitions et discute. Pensée pour une classe de formation musicale, sans compte ni
installation.

**En ligne** : https://nmulongo-sys.github.io/jam-collective/
**Statut** : révision du 2026-07-23 • `index.html` unique, sans build ni dépendance embarquée.

## Utilisation

Ouvrir l'URL — rien à installer. On saisit un prénom (stocké localement), on choisit une
jaquette sur l'accueil, on prend un ou plusieurs rôles, on coche ses disponibilités. Un
guide de première connexion s'ouvre au tout premier passage et reste rouvrable via
« Guide » dans la navigation. Fonctionne sur mobile.

## Fonctions

- **Accueil** : mur de jaquettes façon étagère de DVD, une par morceau, chacune avec son
  identité visuelle. Tri récent/alphabétique, filtres, bandeau playlist YouTube de classe
  (compteur de morceaux dynamique), et formulaire de suggestion d'un titre.
- **Page projet** : en-tête thématisé selon l'univers du morceau, fiche (tonalité, tempo,
  proposé par), **histoire du morceau**, rôles, participants, partitions, vidéos, audios,
  grilles, répétitions et messages.
- **Rôles** : plusieurs personnes par rôle, niveau de besoin (indispensable / souhaitable /
  bonus), statut personnel (apprentissage / jouable / maîtrisé).
- **Vidéos** : liens YouTube par rôle ou en général, vignette et lecture intégrée
  (`youtube-nocookie`) au clic.
- **Disponibilités et répétitions** : créneaux partagés, propositions de répétition et
  réponses de présence.

## Architecture & conventions

- `index.html` unique, aucun build, aucune dépendance embarquée (polices Google Fonts
  uniquement). Échappement HTML systématique de toute donnée affichée.
- Données partagées via **Supabase** (projet « Music Noel », `hifqtzxhmboxbruraiab`), API
  REST PostgREST en `fetch` direct — pas de SDK. RLS activée, politiques anon ouvertes :
  app conviviale de classe, comme verrevacances.
- Tables lues au démarrage : `hub_projets`, `hub_personnes`, `hub_suggestions`,
  `hub_dispos`, `proj_pupitres`, `proj_inscriptions`, `proj_membres`, `proj_liens`,
  `proj_partitions`, `proj_audios`, `proj_grilles`, `proj_messages`, `proj_repets`,
  `proj_presences`.
- Synchronisation par sondage + rafraîchissement au retour d'onglet, suspendu pendant la
  saisie (`saisieEnCours()`) pour ne pas effacer un champ en cours de frappe.
- Identité légère : prénom en `localStorage`, aucune authentification.
- Persistance locale : `fm-theme` (thème clair/sombre), `jam-onboard-vu` (guide déjà vu).

### Convention « univers »

Chaque projet porte un champ `univers` (colonne `hub_projets.univers`) qui pilote
entièrement son apparence. Un univers = **quatre points d'ancrage dans le code**, à
renseigner ensemble sous peine de rendu incomplet :

1. `.tuile[data-univers="…"]` — quatre règles CSS : fond de `.art`, filet `::after`,
   dégradé de `.spine`, couleur de `.embleme`. C'est la jaquette d'accueil.
2. `.scope-projet[data-univers="…"]` — le jeu de variables CSS (`--bg`, `--panel`, `--ink`,
   `--acc`, `--acc2`, `--staff`…) plus le fond de `.projet-hero`. Les variables sont
   surchargées **localement** par le conteneur : le hub reste en thème FM quoi qu'il arrive.
3. `case "…"` dans `motifInner(u)` — le motif SVG, en coordonnées `viewBox="0 0 100 100"`,
   couleurs en `currentColor`. Sert à la fois de vignette et de motif de hero.
4. Une entrée d'animation du motif de hero (`scintilleMotif`, `flotteMotif`,
   `respireMotif`, `balanceMotif`, `tourneMotif`), sous `prefers-reduced-motion`.

Plus un `<option>` dans le sélecteur d'univers du formulaire de création. `motifSVG()`
renvoie une chaîne vide pour un univers inconnu : l'absence de motif dégrade proprement.

**Règle de conception : un univers par morceau.** Aucune jaquette ne doit en dupliquer
une autre. Ajouter un morceau implique donc de créer son univers, et le motif se tire de
préférence du *texte* de la chanson plutôt que de son genre musical — c'est ce qui rend
la vignette identifiable au premier coup d'œil.

## Journal de développement

### 2026-07-19 — création (v1)
- Tables et politiques créées via `Supabase apply_migration` (à noter : `apply_migration` passe les `CREATE TABLE` là où `execute_sql` échouait en juillet).
- Six pupitres semés d'après l'instrumentation de l'original (LinnDrum, synth bass, lead, voix talkbox couplets + voix Lovefoxxx refrain).
- Tempo/tonalité vérifiés sur bases BPM publiques (≈ 91–92 BPM, La mineur).
- Interface outrun : soleil rayé CSS, titre néon Monoton, palette nuit/rose/cyan/or ; mobile-first ; `prefers-reduced-motion` respecté.
- Validation headless jsdom : 16/16 (structure, parsing d'URL YouTube — watch/youtu.be/shorts/params, échappement, rendus pupitres/liens/messages).
- Test bout-en-bout REST réel : lecture, insertion, suppression OK avec la clé anon embarquée.

### 2026-07-19 — v2 : inscriptions multiples, dispos, répétitions, invitation
- **Pupitres** repensés en relation N–N : nouvelle table `nightcall_inscriptions` (`pupitre_id`, `pseudo`, `statut`, unique `(pupitre_id, pseudo)`). Plusieurs personnes par instrument, plusieurs instruments par personne. Chaque inscrit gère son propre statut et peut se retirer.
- **Disponibilités** : table `nightcall_dispos` (texte libre + note), sur le modèle « qui est libre quand ».
- **Répétitions** : table `nightcall_repets` (date/heure, lieu, note, proposeur) + `nightcall_presences` (réponse oui / peut-être / non, unique `(repet_id, pseudo)`). Chacun répond, décompte et noms affichés, le proposeur peut retirer sa répét. Réponse gérée par **upsert PostgREST** (`on_conflict=repet_id,pseudo`, `resolution=merge-duplicates`) — pas de doublon.
- **Invitation** : bouton utilisant `navigator.share` (mobile) avec repli sur copie presse-papier du lien. Pas d'e-mail : quiconque a le lien participe.
- Cascade `on delete` : supprimer une répét efface ses présences ; supprimer un pupitre efface ses inscriptions.
- Validation headless jsdom : 20/20. Test REST réel : upsert présence (fusion), rejet doublon inscription (409), cascade, nettoyage — OK.
- Fond outrun conservé tel quel (validé v1).

### 2026-07-19 — v3 : espace Partitions indexé par pupitre
- **Section Partitions** dédiée : on choisit d'abord le pupitre (menu déroulant alimenté par les pupitres existants), puis on joint le fichier via l'explorateur. Les partitions s'affichent groupées sous leur pupitre.
- **Stockage réel** via un bucket Supabase public `nightcall-partitions` (20 Mo/fichier), politiques anon lecture/écriture/suppression sur `storage.objects`. Table d'index `nightcall_partitions` (`pupitre_id` FK `on delete set null`, `nom_fichier`, `chemin`, `pseudo`, `taille`).
- Chemin de stockage : `{pupitre_id}/{timestamp}-{nom_assaini}` ; URL publique construite avec encodage par segment (préserve le `/`). Téléchargement direct (`download`), taille lisible (o/Ko/Mo), retrait réservé au déposant (supprime l'objet storage + la ligne d'index). Fichiers dont le pupitre a été supprimé regroupés sous « Non classé ».
- Types acceptés côté explorateur : PDF, images, MusicXML — non contraignant.
- Test storage réel (curl, clé anon) : upload 200, lecture publique 200, suppression 200. Validation headless jsdom : 18/18 (remplissage du menu, groupement par pupitre, URL encodée, tailles, permissions de retrait, cas non classé/vide).

### 2026-07-19 — v4 : vidéos indexées par pupitre
- **Vidéos** sur le même modèle que les partitions : on choisit le pupitre, puis on colle le lien. Colonne `pupitre_id` ajoutée à `nightcall_liens` (`on delete set null`).
- Affichage groupé par pupitre, avec un groupe **Général (tout le monde)** en tête pour l'original et les reprises non ciblées. La catégorie (original / reprise / tuto) reste comme badge secondaire ; vignette et lecture intégrée conservées.
- Sélecteur de pupitre factorisé (`remplirSelectPupitre(elId, avecGeneral)`) : partitions sans option « Général », vidéos avec.
- Validation headless jsdom : 17/17 (groupement, ordre Général en tête, badges, permissions de retrait, lecteur intégré, non-régression partitions).

### 2026-07-19 — v5 : liste des participants + badge pupitre principal
- Nouvelle table `nightcall_participants` (`pseudo` unique, `pupitre_principal_id` FK `on delete set null`). Chacun est enregistré dès qu'il indique son prénom (upsert `on_conflict=pseudo`, idempotent — ne réécrit pas le pupitre principal au rechargement).
- **Section Participants** : liste de tous ceux qui ont rejoint, triée alphabétiquement, l'utilisateur courant mis en avant. Sous chaque nom, les autres pupitres où il est inscrit (« aussi : … »).
- **Badge kawaii** du pupitre principal à côté de chaque nom : pastille pastel arrondie (dégradé rose→lilas→cyan, étoile ✦, léger rebond au survol) tranchant volontairement avec le néon synthwave. Le pupitre principal se choisit dans la carte d'identité en haut (« Mon pupitre principal »), enregistré par `PATCH` sur le participant.
- Validation headless jsdom : 17/17. Test REST réel : upsert idempotent, PATCH du pupitre principal, préservation après ré-upsert — OK.

### 2026-07-19 — v6 : passage en PLATEFORME multi-projets
Bascule de l'appli mono-projet vers une plateforme (fichier unique `plateforme.html`, SPA routée par ancre). L'ancien schéma `nightcall_*` est supprimé ; Nightcall devient le premier projet.

**Architecture données**
- *Hub (global)* : `hub_personnes` (prénom unique = identité partagée partout), `hub_dispos` (créneaux JSONB, dispos globales), `hub_projets` (slug, titre, auteur, tonalité, tempo, actif), `hub_suggestions` (titre, auteur, proposeur).
- *Projet* (préfixe `proj_`, FK `projet_id` → hub_projets, cascade) : pupitres, membres (avec `pupitre_principal_id`), inscriptions, répétitions, présences, partitions, liens, messages. Tous les noms sont résolus via `personne_id` → hub_personnes, donc un prénom unique partout.
- Bucket unique `projets-fichiers`, chemins `{projet_id}/{pupitre_id}/{fichier}`.

**Routes** : `#/` accueil (vignettes projets + bandeau suggestions + formulaire auteur/titre), `#/dispos` (grille hebdo), `#/gestion` (admin), `#/p/{slug}` (template projet complet).

**Identité globale** : on s'identifie une fois (prénom → upsert `hub_personnes`), ça vaut pour tous les projets. Rejoindre un projet = upsert `proj_membres` à l'ouverture.

**Dispos graphiques** : grille 7 jours × 3 créneaux (matin/après-midi/soir), heatmap verte (intensité = nombre de personnes libres), contour cyan = vous, noms des personnes libres en infobulle. Sert directement à repérer les créneaux de répétition.

**Bandeau suggestions** : défilement continu (dupliqué pour boucle sans couture), pause au survol, statique si `prefers-reduced-motion`. Toute personne identifiée propose un morceau (auteur+titre) ; il défile → signal pour créer le projet.

**Page Gestion** (route non listée + code `ADMIN_CODE`, à changer dans le source) : créer un projet (avec pupitres de base semés), transformer une suggestion en projet en un clic, masquer/réafficher, supprimer (cascade). Self-service → plus besoin de solliciter le backend à chaque nouveau projet.

**Validation** : 26/26 en headless (slug/route/heatmap/bandeau/tuiles/gestion + rendus du template). Test REST réel : identité, dispo, création projet+seed, adhésion, pupitre principal, suppression en cascade, nettoyage — tous OK.

Note : `index.html` (appli mono-projet Nightcall v5) reste dans le dossier comme référence ; la plateforme la remplace.

### 2026-07-19 — v7 : univers graphique Portail FM + déploiement
- Re-skin complet de la plateforme dans l'identité du **Portail Formation Musicale** (fini l'outrun synthwave) : palette crème/caramel (`--bg #efe7d8`, `--acc #b3763b`…), variante **sombre** (`#0e0e10` / or `#c9a24b`), polices **Cormorant Garamond** (serif titres) + **Work Sans** (UI) + **JetBrains Mono** (eyebrows), en-tête à **portées musicales** (repeating-linear-gradient), vignettes façon `fm-card` (survol relevé + ombre), badges et heatmap re-teintés caramel.
- Reprise du **contrat de thème du portail** (`fm/theme.js`) : mêmes jetons, respect de `?theme=clair|sombre` et `localStorage fm-theme`, bouton bascule intégré. La plateforme s'ouvre donc dans le même thème que le portail quand on y accède depuis lui.
- Fonctions **inchangées** ; seule la couche visuelle a bougé. Validation headless : 23/23 (thème appliqué, bascule clair/sombre, heat caramel, polices, intégrité des fonctions, rendu conservé).
- **Déployé** sur GitHub Pages : `https://nmulongo-sys.github.io/jam-collective/` (dépôt `nmulongo-sys/jam-collective`, `index.html` = la plateforme).
- Rappel « Failed to fetch » : c'était l'ouverture en `file://` (le navigateur bloque les requêtes réseau depuis un fichier local). En https (Pages) tout fonctionne.

### 2026-07-19 — v8 : correctif 401 + identité désambiguïsée + tuiles DVD
- **Bug 401 corrigé** (« No API key found ») : dans `api()`, `...options` réécrasait les en-têtes fusionnés et supprimait l'`apikey`/`Authorization` sur les écritures (POST/PATCH/upsert). Les lectures passaient, pas les écritures — d'où l'échec à l'identification. Corrigé en fusionnant les en-têtes en dernier.
- **Identité désambiguïsée** : deux champs **Prénom + Nom** ; l'identité affichée devient « Prénom I. » (initiale du nom forcée). À l'identification, si un homonyme (« Jean D. ») existe déjà, un avertissement propose de confirmer (c'est vous) ou d'annuler et changer d'initiale.
- **Vignettes en jaquette de DVD** : format portrait (27/40), tranche colorée à gauche, jaquette lustrée (dégradé caramel + reflet + motif portées), titre serif en réserve claire, emblème ♪, pied crème avec auteur (mono) et participants. Survol = léger soulèvement/rotation façon boîtier qu'on sort de l'étagère.
- Validation headless : 10/10 (apikey conservé sur écriture, identité « Jean D. », alerte homonyme, structure DVD, champs prénom+nom).

### 2026-07-19 — v9 : thème par projet (hub FM, projets à identité propre)
- Clarification : l'**accueil / hub** (accueil, dispos, gestion, en-tête, nav) reste en univers **Formation Musicale** ; chaque **projet** garde son **identité propre**. Nightcall repasse en **outrun / Kavinsky**.
- Colonne `univers` ajoutée à `hub_projets` (défaut `'fm'`) ; `nightcall` = `'outrun'`. Sélecteur d'univers (FM crème / Outrun néon) dans le formulaire de création de projet.
- La page projet est désormais enveloppée dans un conteneur `.scope-projet[data-univers]` : les jetons de couleur/police sont surchargés **localement** (le hub autour n'est pas affecté). Pack outrun = nuit violette, néons rose/cyan/or, titre **Monoton** avec halo, **soleil + horizon** dans le héros (animation « lever »). Pack fm = crème, titres Cormorant, héros à portées, et suit le mode clair/sombre du hub.
- Héros de projet à identité (soleil outrun / portées FM) remplaçant le simple titre. Le lien « ← Tous les projets » reste hors scope (chrome FM).
- Validation headless : 14/14 (scope + univers outrun/fm, hub sans scope, tuiles DVD intactes, création avec univers).

### 2026-07-19 — v10 : anti-effacement pendant la saisie + jaquettes thématiques
- **Bug corrigé** : l'auto-rafraîchissement (toutes les 8 s) reconstruisait la vue et effaçait le texte en cours de frappe (ex. lien YouTube). Ajout de `saisieEnCours()` : le rafraîchissement périodique (et celui au retour d'onglet) est suspendu si un champ a le focus, si un champ texte/URL/nombre/date contient du texte, ou si un fichier est sélectionné. Il reprend dès que les champs sont vidés/soumis.
- **Jaquettes thématiques** : chaque vignette d'accueil (`data-univers`) reprend l'univers de son projet. Nightcall (outrun) = jaquette nuit violette, tranche rose→cyan, titre à halo néon, ligne d'horizon cyan ; projets FM = jaquette caramel. Le « boîtier » (pied crème) reste uniforme, comme une étagère de DVD.
- Validation headless : 9/9.

### 2026-07-19 — v11 : infobulles + tutoriel première connexion
- **Infobulles** (`aide()`) : petites pastilles « ? » accessibles (survol *et* focus clavier/tactile) sur les points clés — barre d'identité (prénom + initiale), titre Disponibilités (lecture de la heatmap), Pupitres (niveaux indispensable/souhaitable/bonus), sélecteur d'univers à la création. Bulle sombre lisible, flèche, variante `.bas` pour la nav.
- **Tutoriel première connexion** : carrousel modal de 7 étapes (bienvenue → identité → choisir un morceau → prendre un pupitre → dispos → partager → c'est à vous), points de progression, Précédent/Suivant/Passer, navigation clavier (←/→/Échap), fermeture au clic sur le fond. S'ouvre automatiquement au tout premier passage (flag `localStorage jam-onboard-vu`) et **rouvrable à tout moment via « Guide »** dans la nav.
- Nettoyage : suppression des derniers résidus de jetons outrun hors scope (la légende des dispos utilisait encore `--cyan` / « cases les plus vertes » → corrigé en `--acc2` / « les plus foncées »).
- Validation headless : 16/16 + 5/5 sur le second lot (dernier écran, flag réel, non-réouverture).

### 2026-07-23 — deux morceaux, deux univers inédits, champ « histoire »
- **Ajout de deux morceaux** : *Je l'aime à mourir* (Francis Cabrel, 1979, do majeur, 76 BPM)
  et *Maybe Tomorrow* (Stereophonics, 2003, sol mineur, 81 BPM), avec leurs cinq rôles
  standard et leur lien YouTube de référence. Le répertoire passe à 20 morceaux.
- **Deux univers graphiques inédits**, pour tenir la règle « une jaquette unique par
  morceau » — les 18 univers existants étaient tous pris :
  - `jardin-papier` (Cabrel) — cocotte en papier survolant une arche, sous deux étoiles ;
    motif tiré du texte (« des cocottes en papier », « des ponts entre nous et le ciel »).
    Palette vert-jardin crépusculaire + miel, tranche crème → vert tendre. Animation :
    scintillement.
  - `nuage-vagabond` (Stereophonics) — petit nuage qui pleut, route s'éloignant vers une
    lueur (« these little black clouds keep walking around with me » / « maybe tomorrow
    I'll find my way home »). Dégradé ardoise froide en haut, réchauffé vers le bas.
    Animation : flottement.
- **Champ `histoire`** : nouvelle colonne `hub_projets.histoire` (migration
  `add_histoire_to_hub_projets`), affichée sur la page projet dans une carte
  « L'histoire du morceau » (lettrine sur la couleur d'accent de l'univers). Peuplée pour
  les 20 morceaux. À vérifier : les 18 notices antérieures sont rédigées de mémoire et non
  sourcées une à une.
- **Compteur de playlist dynamique** : le bandeau d'accueil annonçait « les 18 morceaux »
  en dur ; il lit désormais `actifs.length` et ne demande plus de correction à chaque ajout.
- Validation headless : 9/9 points d'insertion, contrôle syntaxique du bloc script
  (`node --check`), parsing jsdom, rendu isolé des 20 motifs SVG, dégradation propre sur
  univers inconnu. Contrôle d'unicité en base : 20 morceaux, 20 univers distincts.
