# Nightcall — fiche projet collectif

Page unique et autonome pour organiser une reprise collective de **Nightcall** (Kavinsky, 2010 — La mineur, ≈ 92 BPM, 4/4). One-shot inspiré du modèle Jam Maker +, mais réduit à un seul morceau, sans connexion ni compte.

## Fonctions

- **Fiche morceau** : tonalité, tempo, mesure, durée, contexte (Drive, Lovefoxxx, Guy-Manuel de Homem-Christo).
- **Pupitres** : liste pré-remplie (chant couplets, chant refrain, synthé lead, basse, batterie, guitare) avec niveau de besoin (indispensable / souhaitable / bonus). Chacun prend un pupitre, indique son statut (en apprentissage / jouable / maîtrisé), peut le libérer ou ajouter un pupitre.
- **Vidéos utiles** : dépôt de liens YouTube (original, reprise, tuto, autre) avec vignette et lecture intégrée (youtube-nocookie) au clic. Chacun peut retirer ses propres liens.
- **Messages** : petit fil partagé entre participants, rafraîchi automatiquement.

## Architecture

- `index.html` unique, aucun build, aucune dépendance embarquée (polices Google Fonts uniquement).
- Données partagées via **Supabase** (projet « Music Noel », `hifqtzxhmboxbruraiab`), API REST PostgREST en `fetch` direct — pas de SDK.
- Tables : `nightcall_pupitres`, `nightcall_liens`, `nightcall_messages` (RLS activée, politiques anon ouvertes — app conviviale de classe, comme verrevacances).
- Synchronisation par sondage toutes les 7 s + rafraîchissement au retour d'onglet (même approche que verrevacances).
- Identité légère : prénom en `localStorage`, aucune authentification.
- Échappement HTML systématique de toute donnée affichée.

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
