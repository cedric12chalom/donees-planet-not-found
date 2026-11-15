# NISAR Vision — Guide complet du projet

> **Pour une présentation orale en français.**
> Chaque affirmation de ce document est tirée du code source lu directement.
> Étiquettes de statut utilisées :
> - **[RÉEL]** — fonctionne avec le vrai granule NISAR ingéré
> - **[SIMULÉ]** — fonctionne uniquement avec les six granules de démonstration synthétiques (`is_demo=True`)
> - **[PARTIEL]** — le code existe mais le pipeline ne s'exécute pas de bout en bout dans la configuration actuelle
> - **[VITRINE]** — visible dans l'interface mais repose sur du texte statique, un modèle illustratif ou un TODO
> - **[NON IMPLÉMENTÉ]** — mentionné dans les commentaires ou le README mais absent du code

---

## 1. Vue d'ensemble de l'architecture

### Schéma général

```
Navigateur (application React monopage)
    │  requêtes REST + tuiles
    ▼
Backend FastAPI  (Python 3.12)
    ├── /api/v1/tiles/*        rio-tiler lit les COG → tuiles PNG
    ├── /api/v1/map/pixel      rasterio lit un seul pixel
    ├── /api/v1/layers         catalogue de couches (SQLite)
    ├── /api/v1/translator/*   classification + détection de changements
    ├── /api/v1/analytics/*    jobs asynchrones (série temporelle, matrice, projection)
    ├── /api/v1/stories        récits guidés
    └── /api/v1/learning/*     quiz, progression, certificat
    │
    ├── Base de données SQLite (nisar_vision.db)
    ├── Fichiers COG  (backend/data/cogs/<source_id>/*.tif)
    └── CLI d'ingestion  (python -m app.cli)
```

### Frontend
- **Framework :** React 18, TypeScript 5, bundlé avec Vite 5.
- **Routage :** React Router v6 (`App.tsx`). Six routes de premier niveau ; la racine `/` redirige vers `/carte`.
- **État global :** store Zustand (`store.ts`). Contient : le drapeau `demo`, la `center`/`zoom` de la carte, la `date` et `compareDate` sélectionnées, la liste `activeLayers` (tableau de clés de couches), la map `opacity` par couche, et l'objet `aoi` sauvegardé.
- **Carte :** MapLibre GL JS (`MapView.tsx`). Fond de carte OSM à 55 % d'opacité, désaturé, en contexte. Les tuiles NISAR sont servies par le backend.
- **Client API :** enveloppe typée autour de `fetch` (`api/client.ts`). Tous les appels vont vers `/api/v1`. Les erreurs remontent comme des instances `ApiError` avec un message, un code HTTP et un `request_id`.
- **Graphiques :** Recharts (courbe, aire dans le tableau de bord).
- **Structure des dossiers :**
  - `src/pages/` — un fichier par route
  - `src/components/` — MapView, PixelPanel, Timeline, LegendPanel, DemoBanner, composants UI génériques
  - `src/api/` — `client.ts` (appels typés), `types.ts` (interfaces TypeScript)
  - `src/lib/` — `tiles.ts` (constructeurs d'URL), `format.ts` (formatage date/dB/km²), `async.ts`, `useJob.ts`
  - `src/sim/` — cinq simulateurs conceptuels interactifs pour le module d'apprentissage

### Backend
- **Framework :** FastAPI 0.115+ avec Pydantic v2 settings.
- **Base de données :** ORM SQLAlchemy 2, SQLite en développement local (configurable en PostgreSQL+PostGIS via `NV_DATABASE_URL`). Schéma créé par Alembic (`alembic/versions/0001_initial.py`).
- **Rendu de tuiles :** rio-tiler (sur rasterio / GDAL) lit les Cloud-Optimised GeoTIFFs (COG) stockés dans `backend/data/cogs/`. Les overviews permettent des lectures décimées rapides aux faibles niveaux de zoom. Trois modes de rendu : amplitude, classes sémantiques, delta de changement.
- **CLI d'ingestion :** `python -m app.cli` — sous-commandes `seed-reference`, `seed-demo`, `ingest-local`, `ingest-remote` (cette dernière nécessite un jeton Earthdata).
- **Jobs asynchrones :** tâches Celery (`workers/tasks.py`) en mode `always_eager=True` en développement (exécution synchrone inline). Les jobs sont stockés dans la table `job`.
- **Limitation de débit :** fenêtre glissante en mémoire, 240 requêtes/minute par IP (`main.py`).
- **CORS :** configuré via `NV_CORS_ORIGINS` (défaut : `http://localhost:5173`).

### Flux de données : du fichier .h5 à un pixel à l'écran

1. **Ingestion :** `python -m app.cli ingest-local <fichier.h5> --maturity provisional`
   - `pipeline.py` appelle `hdf5_gcov.py::read_gcov()` qui ouvre le fichier HDF5 et lit les tableaux de coordonnées (`xCoordinates`, `yCoordinates`) puis les tableaux de termes de covariance (`VVVV`, `VHVH`) bande par bande — jamais le fichier entier en mémoire.
   - `validate.py::validate_product()` vérifie la régularité de la grille, la validité de l'EPSG, les limites géographiques, et qu'aucune bande n'est entièrement nodata.
   - `cog.py::write_cog()` convertit chaque bande en Cloud-Optimised GeoTIFF (compression Deflate, overviews avec rééchantillonnage moyen) dans `backend/data/cogs/<source_id>/`.
   - `manifest.py::write_manifest()` écrit un enregistrement de provenance JSON.
   - Les lignes `Dataset` et `RasterAsset` sont commises dans SQLite.

2. **Servir les tuiles :** lorsque le navigateur demande `/api/v1/tiles/nisar-gcov-vv/2026-11-13/{z}/{x}/{y}.png` :
   - `tiles.py` (API) résout la clé de couche → bande (`VVVV`) → trouve les datasets pour cette date → obtient le chemin du COG.
   - `services/tiles.py::render_amplitude()` appelle `rio-tiler Reader.tile()` qui ne lit que les pixels nécessaires à cette tuile depuis le COG (grâce à la structure interne et aux overviews).
   - Les valeurs sont converties en dB (`to_db`), normalisées sur [0, 255], et rendues en PNG niveaux de gris avec alpha=0 pour les pixels nodata.

3. **Affichage :** MapLibre GL ajoute la couche de tuiles comme source raster par-dessus le fond OSM, en respectant l'opacité de la couche.

4. **Infobulle de pixel :** un clic sur la carte déclenche `onMapClick` → état `pixel` mis à jour → `PixelPanel` s'affiche → appel `GET /api/v1/map/pixel?lon=…&lat=…&date=…&layer=…` → `services/pixel.py::pixel_info()` reprojette le point en UTM, appelle `geo.py::sample_pixel()` qui lit le pixel unique dans le COG, le convertit en dB, et retourne le JSON. Le panneau affiche la valeur linéaire, la valeur en dB, le libellé capteur « NISAR L-SAR bande L », le badge de maturité, et (uniquement pour les couches HH/HV) une classe sémantique.

---

## 2. Routes — Page par page

### `/carte` — Page carte (`MapPage.tsx`)

**Objectif :** visualiser les images radar NISAR sur une carte interactive, inspecter les valeurs de pixels, dessiner des zones d'intérêt.

**Ce que l'utilisateur voit :** une barre latérale à gauche avec les contrôles de couches ; la carte pleine largeur à droite ; une barre de frise chronologique en bas quand des données existent.

#### Éléments interactifs

| Libellé (interface) | Action | API / Store | Statut |
|---|---|---|---|
| Afficher les données de démonstration simulées | Bascule `s.demo` ; recharge les couches et efface les dates | `store.demo` | [SIMULÉ] quand coché ; [RÉEL] décoché |
| Case à cocher couche (ex. « NISAR L-band, rétrodiffusion VV ») | `s.toggleLayer(key)` ; désactivée si `coverage.available === false` | `store.activeLayers` | [RÉEL] pour VV/VH ; [SIMULÉ] pour HH/HV sans démo |
| Curseur d'opacité (aria : « Opacité … ») | `s.setOpacity(key, valeur)` — met à jour `raster-opacity` dans MapLibre | `store.opacity` | [RÉEL] |
| Boutons ↑ / ↓ (aria : « Monter / Descendre … ») | `s.moveLayer(key, ±1)` — réordonne les couches MapLibre | `store.activeLayers` | [RÉEL] |
| Bouton ⊕ (title : « Centrer sur la scène ») | `setFit(coverage.bounds)` → MapLibre `fitBounds` avec 40 px de marge | état local `fit` | [RÉEL] |
| Section « Légende » | Rampe noir→blanc avec libellés min/max en dB ; se met à jour avec les couches actives | `LegendPanel` lit `layer.style.rescale` | [RÉEL] |
| Fond de carte OSM (contexte) | `setBasemap(bool)` ; reconstruit le style MapLibre | état local `basemap` | [RÉEL] |
| Champ « Rechercher un lieu » + bouton « Aller » | Essaie d'abord lat/lon ; repli sur le géocodage Nominatim ; appelle `s.setView()` | Nominatim (externe), `store.setView` | [RÉEL] |
| Bouton « Cercle » | Active l'outil cercle ; le prochain clic place le centre | état local `tool` | [RÉEL] |
| Bouton « Rectangle » | Active l'outil rectangle ; deux clics définissent le cadre | état local `tool` | [RÉEL] |
| Curseur « Rayon » (1–25 km) | Ajuste `radius` ; redessine le cercle GeoJSON en temps réel | état local `radius` | [RÉEL] |
| Bouton « Enregistrer la zone » | POST `/api/v1/analytics/aoi` → stocke la zone en base, met à jour `s.aoi` | `api.createAoi()` | [RÉEL] |
| Bouton « Lecture » / « Pause » (Timeline) | Avance automatiquement les dates à la vitesse choisie | composant `Timeline` | [RÉEL] |
| Sélecteur de vitesse (lente / normale / rapide) | Intervalle 2 400 / 1 200 / 500 ms | composant `Timeline` | [RÉEL] |
| Curseur de date | Sélectionne une des dates disponibles ; appelle `s.setDate()` | `store.date` | [RÉEL] |
| Case « Comparer avant / après » | Active le composant `Swipe` ; nécessite ≥ 2 dates | état local `compare` | [PARTIEL] — 1 seule date réelle |
| Sélecteur « Date après » | `s.setCompareDate(d)` | `store.compareDate` | [PARTIEL] — même contrainte |
| Clic sur la carte (outil par défaut) | `setPixel({lon, lat})` ; ouvre `PixelPanel` | état local `pixel` | [RÉEL] |
| Bouton « Fermer » (PixelPanel) | `setPixel(null)` | état local | [RÉEL] |
| Barre d'échelle | Rendu par MapLibre `ScaleControl` (métrique) en bas à gauche | MapLibre | [RÉEL] |
| Contrôles de navigation + / − | `NavigationControl` MapLibre standard (boussole masquée) | MapLibre | [RÉEL] |

**Raccourcis clavier :** aucun raccourci personnalisé dans le code. La navigation MapLibre par défaut (flèches pour déplacer, +/- pour zoomer) fonctionne.

**État vide :** quand aucune date n'est disponible et que la démo est désactivée, un encart affiche : « Aucune acquisition NISAR n'est disponible dans le catalogue. Ingérez des granules réels… ou cochez « données de démonstration simulées ». »

---

### `/traducteur` — Traducteur radar (`TranslatorPage.tsx`)

**Objectif :** afficher la même carte dans trois modes de lecture — signal brut en dB, classes sémantiques, ou détection de changements entre deux dates.

**Ce que l'utilisateur voit :** barre latérale gauche avec les contrôles de mode et de paramètres ; carte pleine largeur ; frise chronologique en bas.

#### Éléments interactifs

| Libellé | Action | API / Store | Statut |
|---|---|---|---|
| Case « Données de démonstration simulées » | Identique à la page carte | `store.demo` | [SIMULÉ] / [RÉEL] |
| Onglet « Signal brut » | Tuiles niveaux de gris en dB pour la bande sélectionnée | `tileUrl(…, "amplitude")` | [RÉEL] |
| Onglet « Lecture sémantique » | Tuiles de classes colorées ; nécessite HH ET HV | `tileUrl(hh.key, …, "classes")` | [SIMULÉ] uniquement (granule réel = VV/VH) |
| Onglet « Changements » | Tuiles delta en dB entre deux dates | `changeTileUrl(…)` | [SIMULÉ] uniquement (1 seule date réelle) |
| Sélecteur « Polarisation » (HH / HV) | Visible en modes brut + changements | état local `band` | [SIMULÉ] uniquement (HH/HV sans données réelles) |
| Curseur « Seuil de changement » (1–10 dB) | Minimum de différence dB considéré comme un changement | état local `threshold` | [PARTIEL] |
| Sélecteur « Date de comparaison » | Sélectionne la date « avant » pour le changement | `store.compareDate` | [PARTIEL] |
| Section « Règles utilisées (version) » | Liste détaillable de tous les seuils HH/HV en dB | `GET /api/v1/translator/ruleset` | [VITRINE] pour VV/VH — règles calibrées HH/HV uniquement |
| Section « Répartition » (parts de classes) | % et km² par classe de couverture terrestre pour le dataset courant | `GET /api/v1/translator/classification?…` | [SIMULÉ] uniquement |
| Clic sur la carte | Ouvre PixelPanel (toujours avec `hh.key`) | `api.pixel(…, layer: hh.key)` | [SIMULÉ] uniquement |
| Frise chronologique | Identique à `/carte` | | [SIMULÉ] uniquement |

**Note sur « Lecture sémantique » :** la fonction `render_classes()` dans `services/tiles.py` récupère explicitement les chemins HHHH et HVHV. Le granule réel contient VVVV/VHVH, pas HHHH/HVHV. Les tuiles sémantiques retournent un PNG transparent pour le granule réel. Le traducteur affiche correctement le message d'erreur « La lecture sémantique demande les bandes HH et HV. Au moins une bande est non disponible. »

---

### `/recits` — Liste des récits (`StoryList` dans `StoryPage.tsx`)

**Objectif :** catalogue des récits guidés.

**Ce que l'utilisateur voit :** liste de cartes avec thème, durée, nombre de scènes et synopsis.

| Libellé | Action | API | Statut |
|---|---|---|---|
| Titre du récit (lien) | Navigue vers `/recits/:slug` | `GET /api/v1/stories` | [RÉEL] (données chargées depuis la base) |

**Récits présents dans la base (d'après `seed/stories.py`) :**
1. « Amazonie : voir la forêt à travers les nuages » (slug : `amazonie-exemple`) — 4 scènes, aucune couche NISAR associée, texte uniquement.
2. « Démonstration : un défrichement simulé » (slug : `demo-defrichement`) — 4 scènes, utilise `nisar-gcov-hh` et `nisar-gcov-hv` avec des dates de démo.

---

### `/recits/:slug` — Récit détaillé (`StoryView` dans `StoryPage.tsx`)

**Objectif :** un récit guidé — le texte défile à gauche et pilote la carte à droite.

**Ce que l'utilisateur voit :** titre, cartes de scènes défilantes ; carte réactive au scroll (via IntersectionObserver) ; animation automatique des dates toutes les 1,4 s pour les récits de démo ; bandeau « DONNÉES SIMULÉES » si `is_demo === true`.

| Libellé | Action | API | Statut |
|---|---|---|---|
| Lien « ← Tous les récits » | Navigue vers `/recits` | (navigation) | [RÉEL] |
| Liens sources dans le pied de page | Liens externes dans un nouvel onglet | statique (depuis la base) | [VITRINE] pour Amazonie (aucune donnée NISAR) |
| Carte (animation auto) | Fait défiler les dates toutes les 1,4 s sauf si `prefers-reduced-motion` | `GET /api/v1/layers/{key}/metadata` | [SIMULÉ] pour demo-defrichement |

---

### `/apprendre` — Accueil apprentissage (`LearnHome` dans `LearnPage.tsx`)

**Objectif :** vue d'ensemble des cinq niveaux d'apprentissage avec XP et badges.

**Ce que l'utilisateur voit :** total XP, niveaux complétés, badges, grille de cartes de niveaux. Les niveaux 2 à 5 sont verrouillés jusqu'à la complétion du niveau précédent.

| Libellé | Action | API | Statut |
|---|---|---|---|
| Bouton « Commencer » / « Revoir » | Navigue vers `/apprendre/niveau/:id` | `GET /api/v1/learning/levels` | [RÉEL] |
| Bouton « Obtenir mon certificat » | Navigue vers `/apprendre/certificat` | `GET /api/v1/learning/progress` | [RÉEL] (vérification d'éligibilité côté serveur) |
| Lien « Glossaire des termes » | Navigue vers `/apprendre/glossaire` | (navigation) | [RÉEL] |
| Badges (jetons) | Affichage uniquement (titre = description au survol) | | [RÉEL] |

---

### `/apprendre/niveau/:id` — Page de niveau (`LevelPage` / `LevelBody`)

**Objectif :** texte de leçon, simulateur conceptuel interactif, et quiz à choix multiples.

**Ce que l'utilisateur voit :** cartes de leçon, le simulateur du niveau, puis le quiz.

**Cinq niveaux et leurs simulateurs :**
1. « Les bases du radar » — simulateur `DayNight` (curseurs jour/nuit/nuages)
2. « Rugosité et rétrodiffusion » — simulateur `Roughness` (rugosité + angle d'incidence, diagramme SVG)
3. « Détecter l'eau et les inondations » — simulateur `Flood` (case inondé, curseur vent, seuil)
4. « La polarimétrie » — simulateur `Polarization` (boutons type de surface, table HH/HV/VV, couleur composite RVB)
5. « InSAR et déplacements millimétriques » — simulateur `Insar` (curseur déplacement, visualisation de franges)

| Libellé | Action | API | Statut |
|---|---|---|---|
| Contrôles des simulateurs (curseurs, boutons, cases) | Mettent à jour des valeurs calculées dans le navigateur ; purement qualitatif | aucune (tout dans `sim/models.ts`) | [VITRINE] — illustration pédagogique, aucune donnée réelle |
| Boutons radio (options du quiz) | Enregistre la réponse dans l'état local `answers` | aucune (avant soumission) | [RÉEL] |
| Bouton « Valider mes réponses » | POST `/api/v1/learning/quiz/:id/attempts` → score, XP côté serveur | `api.attempt()` | [RÉEL] |
| Bouton « Réessayer » | Réinitialise `answers` et `result` | état local | [RÉEL] |
| Bouton « Retour aux niveaux » | Navigue vers `/apprendre` | (navigation) | [RÉEL] |
| Lien « ← Niveaux » | Navigue vers `/apprendre` | (navigation) | [RÉEL] |

---

### `/apprendre/glossaire` — Glossaire (`Glossary`)

**Objectif :** liste consultable de termes techniques avec définitions et exemples.

| Libellé | Action | API | Statut |
|---|---|---|---|
| Champ « Rechercher un terme » | Appelle `GET /api/v1/learning/glossary?q=…` à chaque saisie | `api.glossary(q)` | [RÉEL] |
| Lien « ← Niveaux » | Navigue vers `/apprendre` | (navigation) | [RÉEL] |

---

### `/apprendre/certificat` — Page certificat (`CertificatePage`)

**Objectif :** délivrer un certificat PDF après avoir validé les cinq niveaux.

**Ce que l'utilisateur voit :** soit un formulaire pour saisir un nom (si éligible), soit un message indiquant combien de niveaux restent.

| Libellé | Action | API | Statut |
|---|---|---|---|
| Champ « Nom à inscrire » | Requis (2–80 caractères) | état local | [RÉEL] |
| Bouton « Délivrer le certificat » | POST `/api/v1/learning/certificate` | `api.certificate(name)` | [RÉEL] |
| Lien « Télécharger le PDF » | GET `/api/v1/learning/certificate/{code}/pdf` → PDF généré par ReportLab | Backend | [RÉEL] |

**Note :** le certificat indique explicitement qu'il n'est pas une certification officielle de la NASA.

---

### `/tableau-de-bord` — Tableau de bord analytique (`DashboardPage.tsx`)

**Objectif :** analyse quantitative sur une zone d'intérêt : série temporelle, matrice de transition, projection exploratoire.

**Ce que l'utilisateur voit :** deux cartes de paramétrage (zone + période), puis trois blocs d'analyse, chacun avec un bouton « Lancer l'analyse », une barre de progression, des graphiques de résultat et des boutons d'export.

| Libellé | Action | API | Statut |
|---|---|---|---|
| Champs de coordonnées (lon, lat, rayon) | Crée une zone d'intérêt | POST `/api/v1/analytics/aoi` | [RÉEL] |
| Bouton « Créer la zone » | Soumet le formulaire ci-dessus | | [RÉEL] |
| Bouton « Changer » (zone active) | Efface `s.aoi` | `store.aoi = null` | [RÉEL] |
| Case démo | Identique aux autres pages | `store.demo` | [SIMULÉ] |
| Champs de dates (Début / Fin) | Bornés aux dates disponibles | état local | [RÉEL] |
| Sélecteur « Pas » (jour / semaine / mois) | Bucket d'agrégation pour la série temporelle | état local `freq` | [RÉEL] |
| « Lancer l'analyse » (série temporelle) | POST `/api/v1/analytics/time-series` → job asynchrone | `api.timeSeries()` + `pollJob()` | [PARTIEL] — 1 date réelle ; une série à un point n'a pas de sens |
| Bouton « Annuler » | POST `/api/v1/jobs/:id/cancel` | `api.cancelJob()` | [RÉEL] |
| Sélecteurs Date A / Date B (matrice) | Nécessitent deux dates différentes | état local `da`/`db` | [PARTIEL] — nécessite 2 dates |
| « Lancer l'analyse » (matrice de transition) | POST `/api/v1/analytics/change-matrix` | `api.changeMatrix()` | [PARTIEL] — même contrainte |
| Curseur « Horizon » (1–12 mois) | Pour la projection what-if | état local `horizon` | [PARTIEL] |
| Cases scénarios (3 scénarios) | Filtre les courbes de projection affichées | état local `scenarios` | [VITRINE] — les scénarios multiplient simplement la pente observée par un facteur fixe |
| « Lancer l'analyse » (projection) | POST `/api/v1/analytics/projection` | `api.projection()` | [PARTIEL] |
| Boutons export (CSV / GeoJSON / SVG / PDF) | POST `/api/v1/exports` puis téléchargement | `api.createExport()` + `downloadExport()` | [RÉEL] (le mécanisme de job fonctionne) |
| Section « Données en tableau » | Tableau HTML dépliable des données brutes de la série | état local React | [RÉEL] |

---

### `*` — Page 404

**Ce que l'utilisateur voit :** « Page introuvable. » et un lien « Retour à la carte » vers `/carte`. Défini dans `App.tsx` comme la route attrape-tout `path="*"`.

---

## 3. La page carte en détail

### Case « Afficher les données de démonstration simulées »
Définie dans `store.ts` : `setDemo(v)` positionne `demo`, efface `date` et `compareDate`. Quand `demo=true`, tous les appels API ajoutent `include_demo=true`. Le `DemoBanner` apparaît. Quand `demo=false` et qu'aucune donnée réelle n'existe pour une couche, celle-ci affiche « Aucune donnée ingérée ».

### Activation/désactivation des couches
Chaque carte de couche contient une case à cocher. Elle est `disabled` quand `l.coverage?.available === false`. Le basculement appelle `store.toggleLayer(key)`. Activer une couche l'ajoute à `activeLayers` ; la carte tente immédiatement de charger les tuiles pour la date sélectionnée.

### Curseur d'opacité
Plage 0–1, pas de 0,05. Stocké dans `store.opacity`. À chaque changement, `MapView.sync()` appelle `map.setPaintProperty(layerId, "raster-opacity", valeur)` sans recréer la source.

### Flèches ↑ / ↓ d'ordre des couches
Appellent `store.moveLayer(key, ±1)`. Le store réordonne `activeLayers`. `MapView.sync()` appelle ensuite `map.moveLayer()` pour s'aligner — le premier élément de la liste est rendu le plus bas, le dernier le plus haut.

### Bouton ⊕ « Centrer sur la scène »
Ajouté lors de la dernière session de développement. Visible uniquement quand `l.coverage?.bounds` est non nul (c'est-à-dire quand la couche possède au moins un dataset réel ou de démo). Appelle `setFit(bounds)`. `MapView` réagit aux changements de la prop `fitBounds` avec `map.fitBounds([[w,s],[e,n]], {padding:40, duration:0})`. **[RÉEL]** pour les couches VV et VH.

### Légende
`LegendPanel.tsx` rend un `RampLegend` par couche active. La rampe est un dégradé CSS (noir → blanc). Les libellés min et max proviennent de `layer.style.rescale` (ex. `[-25, 0]` pour VV). La note indique : « Valeurs en décibels (10·log10 de la puissance linéaire). Les pixels sans donnée sont transparents, jamais affichés comme 0. »

### Bascule du fond de carte OSM
Quand décoché, `basemap=false` est passé à `MapView`. Le style MapLibre est reconstruit sans la source/couche OSM, ne laissant que la couleur de fond sombre `#0b1220`. La carte est recréée (le `useEffect` dépend de `basemap`).

### Recherche de lieu (« Rechercher un lieu » + « Aller »)
Deux stratégies dans `doSearch()` :
1. Si la saisie correspond à `latitude, longitude` (virgule, point-virgule ou espace), appelle directement `s.setView([lon, lat], 11)`.
2. Sinon, interroge `https://nominatim.openstreetmap.org/search?format=json&limit=1&q=…`. En cas de succès, `setSearchMsg(displayName)` et `s.setView(...)` au zoom 10.

Le texte indicatif du champ est « Douala ou 4.05, 9.70 » — le centre des données de démo.

### Outils de zone d'intérêt
- **Bouton « Cercle » :** active `tool="circle"`. Le prochain clic place le centre du cercle ; le polygone GeoJSON est dessiné par la fonction `circle()` (64 segments, approximation équirectangulaire). Le curseur de rayon (1–25 km) redessine le cercle en temps réel.
- **Bouton « Rectangle » :** active `tool="rect"`. Le premier clic enregistre `rectStart` ; le second calcule le polygone de la boîte englobante.
- **Bouton « Enregistrer la zone » :** apparaît quand un brouillon existe. Appelle `api.createAoi()` → POST `/api/v1/analytics/aoi`. En cas de succès, le GeoJSON est remplacé par la géométrie validée par le backend (qui utilise un cercle géodésique, non l'approximation du navigateur), et `s.setAoi()` est appelé.

### Frise chronologique (contrôles de date et de comparaison)
`Timeline.tsx` s'affiche sous la carte uniquement quand `dates.length > 0`. Elle contient :
- **Bouton « Lecture » / « Pause » :** avance automatiquement les dates à la vitesse choisie.
- **Sélecteur de vitesse :** lente (2 400 ms), normale (1 200 ms), rapide (500 ms).
- **Curseur de plage :** s'accroche uniquement aux dates disponibles (aucune interpolation entre les trous).
- **Libellé de date :** format court locale française (ex. « 13 nov. 2026 »).
- **Compteur de dates :** « 1 date réelle » pour le granule réel.
- **Case « Comparer avant / après » :** activée uniquement si `dates.length ≥ 2`. Avec les données réelles (1 date), ce contrôle est désactivé.
- **Sélecteur « Date après » :** visible quand la comparaison est activée ; filtre le même tableau `dates`.

### Barre d'échelle
Ajoutée par `ScaleControl({ unit: "metric" })` de MapLibre en bas à gauche de chaque `MapView`. Ce n'est pas un composant personnalisé — c'est une fonctionnalité native de MapLibre.

### Infobulle de pixel (PixelPanel)
Quand aucun outil de zone n'est actif et qu'une date est sélectionnée, cliquer sur la carte positionne `pixel={lon, lat}`, ce qui rend `PixelPanel`. Le panneau appelle `GET /api/v1/map/pixel?lon=…&lat=…&date=…&include_demo=…&layer=…`.

**Signification de chaque valeur :**
- **Coordonnées** (en haut) : latitude N, longitude E, date formatée — l'endroit cliqué.
- **Badge de maturité :** « Provisoire » pour les données réelles, « DÉMO simulée » pour les données de démo.
- **Source ID :** le nom complet du fichier granule NISAR.
- **Tableau des bandes :** colonnes « Bande » (ex. VV, VH), « Linéaire » (valeur float32 brute du COG, en notation scientifique), « Valeur » (dB, calculé comme 10·log₁₀(linéaire)).
- **Ligne unités :** « dB (10·log10 de la puissance linéaire) » — précise la conversion.
- **Bloc interprétation :** affiché UNIQUEMENT pour les couches HH/HV. Pour les couches VV/VH, `interpretation` est `null` et rien n'est affiché — le jeu de règles de classification est calibré HH/HV uniquement.
- **Mini graphique de série temporelle :** affiché si `series.length > 1` ; trace l'évolution en dB sur toutes les dates disponibles pour ce pixel. Avec le granule réel (1 date), aucun graphique n'apparaît.

---

## 4. Éléments de l'en-tête

### Lien d'évitement « Aller au contenu »
Défini dans `App.tsx` : `<a href="#main" className="sr-only focus:not-sr-only …">Aller au contenu</a>`. Visuellement masqué sauf quand focalisé par la touche Tab. Saute vers `<main id="main">`. Fonctionnalité d'accessibilité standard.

### Onglets de navigation
Cinq onglets rendus depuis le tableau `NAV` dans `App.tsx` :
- **Carte** → `/carte`
- **Traducteur radar** → `/traducteur`
- **Récits** → `/recits`
- **Apprendre** → `/apprendre`
- **Tableau de bord** → `/tableau-de-bord`

Chacun est un `NavLink` de React Router. La route active reçoit `bg-ink-600 text-white` ; les autres `text-slate-300 hover:bg-ink-700`. Les onglets débordent horizontalement sur les petits écrans.

### DemoBanner
Défini dans `DemoBanner.tsx`. Lit `useApp(s => s.demo)`. S'affiche **uniquement quand `demo === true`**, comme une barre ambre pleine largeur au-dessus de l'en-tête avec le texte : **« DONNÉES DE DÉMONSTRATION SIMULÉES. Ces images ne sont pas des observations NISAR. »** L'attribut `role="alert"` le fait annoncer par les lecteurs d'écran. Il disparaît immédiatement quand la case de démo est décochée sur n'importe quelle page.

---

## 5. Table de vérité des données

### Couches et leur statut réel

| Clé de couche | Bande | Données réelles ? | Données de démo ? | Notes |
|---|---|---|---|---|
| `nisar-gcov-vv` | VVVV | **[RÉEL]** — 1 date : 2026-11-13 | Non | Ajoutée lors de cette session. Le granule réel contient cette bande. |
| `nisar-gcov-vh` | VHVH | **[RÉEL]** — 1 date : 2026-11-13 | Non | Ajoutée lors de cette session. Le granule réel contient cette bande. |
| `nisar-gcov-hh` | HHHH | Aucune donnée réelle | **[SIMULÉ]** — 6 dates (2026-03-04 à 2026-08-11) | Aucune bande HHHH dans le granule ingéré. Affiche « Aucune donnée ingérée » sans la démo. |
| `nisar-gcov-hv` | HVHV | Aucune donnée réelle | **[SIMULÉ]** — 6 dates | Identique. |

Le store initialise `activeLayers: ["nisar-gcov-hh"]`. Au premier chargement, la couche HH affiche « Aucune donnée ingérée » jusqu'à ce que l'utilisateur active la démo ou bascule sur VV/VH.

### Le granule réel ingéré

| Champ | Valeur |
|---|---|
| Nom du fichier | `NISAR_L2_PR_GCOV_035_120_D_084_2005_DVDV_A_20261113T125528_20261113T125603_P05036_N_F_J_001.h5` |
| Type de produit | GCOV L2 (Ground-projected Covariance) |
| Début d'acquisition | 13 novembre 2026 à 12 h 55 min 28 s UTC |
| Fin d'acquisition | 13 novembre 2026 à 12 h 56 min 04 s UTC (35 secondes) |
| SCR | EPSG:32644 (UTM zone 44N) |
| Taille de pixel | 20 m × 20 m |
| Dimensions de la grille | 16 848 × 16 488 pixels |
| **Polarisations présentes** | **VH et VV** (termes de covariance GCOV : VHVH, VVVV) |
| Polarisations HH/HV ? | **Absentes** de ce granule |
| Boîte englobante (lon/lat) | O=81,447 E=84,527 S=7,366 N=10,362 |
| Couverture géographique | Inde du Sud (Tamil Nadu) et nord-est du Sri Lanka |
| Direction orbitale | Descendante |
| Trace | 120 |
| Version du produit | 1.0.20 |
| Nodata | NaN (45 % des pixels en lecture décimée — bord d'emprise de la fauchée) |
| Médiane VV (scène entière) | −13,44 dB |
| Médiane VH (scène entière) | −26,66 dB |

### Ce que l'application affiche quand aucune donnée réelle n'existe

L'endpoint `/api/v1/layers` appelle `catalog.layer_coverage()` pour chaque couche. Si `datasets_for_layer()` retourne une liste vide, `coverage.available` est `false`. Dans l'interface, la case de la couche est désactivée et le sous-libellé affiche « Aucune donnée ingérée ». Si toutes les couches actives sont sans données, l'encart d'état vide apparaît sur la carte.

---

## 6. API backend — Liste complète des endpoints

Tous les endpoints sont sous `/api/v1`. Méthode, chemin, objectif et quel élément de l'interface les appelle.

### Système
| Méthode | Chemin | Objectif | Appelé par |
|---|---|---|---|
| GET | `/health` | Sonde de liveness : le processus répond | Déploiement / Docker |
| GET | `/health/ready` | Readiness : base joignable, nombre de datasets | Déploiement |

### Catalogue et carte
| Méthode | Chemin | Objectif | Appelé par |
|---|---|---|---|
| GET | `/layers?include_demo=…` | Liste toutes les couches avec leur couverture (dates, emprise, disponibilité) | Chaque page au chargement |
| GET | `/layers/{clé}/metadata?include_demo=…` | Détail d'une couche + liste de datasets | MapPage (barre d'info), TranslatorPage, StoryView |
| GET | `/datasets?…` | Liste paginée de datasets (type produit, maturité, bbox, dates) | Non appelé directement depuis le frontend |
| GET | `/datasets/{id}` | Métadonnées d'un dataset | Non appelé directement depuis le frontend |
| GET | `/map/pixel?lon=…&lat=…&date=…&layer=…&include_demo=…` | Valeur(s) de pixel en dB, interprétation, mini série temporelle | PixelPanel (clic sur /carte et /traducteur) |
| GET | `/events` | Événements datés et sourcés | Non appelé depuis le frontend actuel |

### Tuiles
| Méthode | Chemin | Objectif | Appelé par |
|---|---|---|---|
| GET | `/tiles/{clé_couche}/{date}/{z}/{x}/{y}.png?mode=amplitude\|classes&demo=…` | Rendu PNG d'une tuile (amplitude ou sémantique) | Source raster MapLibre, toutes les pages avec une carte |
| GET | `/tiles/change/{clé_couche}/{date_a}/{date_b}/{z}/{x}/{y}.png?…` | Tuile de détection de changement | MapLibre, mode changements /traducteur |

### Traducteur
| Méthode | Chemin | Objectif | Appelé par |
|---|---|---|---|
| GET | `/translator/ruleset` | Jeu de règles de classification actif (version, seuils, notes) | TranslatorPage, PixelPanel |
| GET | `/translator/classification?dataset_id=…&aoi_id=…` | Proportions de classes pour un dataset (éventuellement découpé sur une AOI) | Barre latérale en mode sémantique du Traducteur |
| POST | `/translator/changes` | Résumé de détection de changements entre deux datasets | Mode changements du Traducteur |

### Analyses (jobs asynchrones)
| Méthode | Chemin | Objectif | Appelé par |
|---|---|---|---|
| POST | `/analytics/aoi` | Crée une zone d'intérêt (cercle, rectangle, GeoJSON) | MapPage « Enregistrer la zone », DashboardPage « Créer la zone » |
| GET | `/analytics/aoi/{id}` | Récupère une AOI | En interne par les tâches de jobs |
| POST | `/analytics/time-series` | Soumet un job de série temporelle (202 Accepté) | DashboardPage « Lancer l'analyse » (série temporelle) |
| POST | `/analytics/change-matrix` | Soumet un job de matrice de transition | DashboardPage « Lancer l'analyse » (matrice) |
| POST | `/analytics/projection` | Soumet un job de projection | DashboardPage « Lancer l'analyse » (projection) |
| GET | `/jobs/{id}` | Interroge le statut et le résultat d'un job | `pollJob()` dans client.ts |
| POST | `/jobs/{id}/cancel` | Annule un job en cours | Bouton « Annuler » dans DashboardPage |
| POST | `/exports` | Crée un job d'export (CSV/GeoJSON/SVG/PDF) | Boutons d'export dans DashboardPage |
| GET | `/exports/{id}/download` | Télécharge un export complété | `downloadExport()` dans client.ts |

### Récits et apprentissage
| Méthode | Chemin | Objectif | Appelé par |
|---|---|---|---|
| GET | `/stories` | Liste les récits publiés | /recits |
| GET | `/stories/{id_ou_slug}` | Récit complet avec toutes ses scènes | /recits/:slug |
| GET | `/learning/levels` | Liste les niveaux avec état déverrouillé/complété | /apprendre |
| GET | `/learning/levels/{id}` | Détail d'un niveau (leçons, clé simulateur, quiz) | /apprendre/niveau/:id |
| POST | `/learning/quiz/{id}/attempts` | Soumet des réponses ; retourne score, XP et badge | Bouton « Valider mes réponses » |
| GET | `/learning/progress` | XP, niveaux complétés, badges, éligibilité certificat | /apprendre accueil, /apprendre/certificat |
| GET | `/learning/glossary?q=…` | Termes du glossaire (filtrés) | /apprendre/glossaire |
| POST | `/learning/certificate` | Délivre le certificat (nécessite les 5 niveaux) | « Délivrer le certificat » |
| GET | `/learning/certificate/{code}` | Vérifie un certificat (nom partiellement masqué) | Lien de vérification public |
| GET | `/learning/certificate/{code}/pdf` | Télécharge le certificat PDF | Lien « Télécharger le PDF » |

---

## 7. Limites connues et problèmes ouverts

1. **Une seule date d'acquisition.** Le granule réel ne couvre qu'une date (13 novembre 2026). La comparaison, la détection de changements, la série temporelle et la projection nécessitent toutes au moins deux dates. Ces fonctionnalités sont architecturalement complètes mais inutilisables avec les données réelles en l'état.

2. **Couche par défaut incorrecte.** Le store initialise `activeLayers: ["nisar-gcov-hh"]`. HH n'a aucune donnée réelle. Au premier chargement, l'utilisateur voit « Aucune donnée ingérée » pour la couche par défaut. Il doit manuellement basculer sur `nisar-gcov-vv` ou `nisar-gcov-vh`, ou activer les données de démo.

3. **Mode sémantique incompatible avec le granule réel.** `render_classes()` dans `services/tiles.py` récupère en dur les chemins HHHH et HVHV. Le granule réel a VVVV et VHVH. Les tuiles sémantiques retournent un PNG transparent pour les données réelles.

4. **Jeu de règles de classification non validé sur VV/VH.** Le fichier JSON `gcov_lband_prototype_v0_1.json` indique explicitement : « Seuils illustratifs pour un prototype pédagogique. Ils n'ont pas été validés par un expert ni calibrés sur des données de référence. » L'endpoint pixel retourne désormais `interpretation: null` pour les couches VV/VH, mais le jeu de règles n'a jamais été calibré sur VV/VH ni sur aucune scène NISAR réelle.

5. **45 % de pixels nodata.** Près de la moitié des pixels du COG sont NaN, correspondant aux parties de la fauchée hors de l'emprise d'acquisition. C'est normal pour un produit GCOV, mais visuellement cela signifie que le raster ne couvre qu'une partie des tuiles de la carte.

6. **La scène est principalement côtière.** L'empreinte couvre le sud de l'Inde et le nord-est du Sri Lanka (côte du golfe du Bengale). La majorité de l'aire visible à faible zoom est l'océan (hors raster, transparent). Les pixels terrestres intéressants se trouvent dans la partie est de la fauchée.

7. **Aucun dépôt git.** Le répertoire de travail ne contient aucun dossier `.git`. Il n'y a pas d'historique de versions. `git status` et `git diff` échouent.

8. **Les données de démo sont centrées sur Douala (Cameroun), les données réelles sur le Tamil Nadu.** Les deux sont géographiquement sans rapport. Les récits référencent Douala (lon=9,70, lat=4,05). Le point de vue par défaut du store est `center: [9.7, 4.05]`. Quand les données VV/VH réelles sont chargées, l'utilisateur doit utiliser le bouton ⊕ « Centrer sur la scène » ou la recherche de lieu pour atteindre la zone Tamil Nadu.

9. **Le récit Amazonie n'a aucune donnée.** Le premier récit (`amazonie-exemple`) indique explicitement dans sa dernière scène : « Cette étape reste à faire. » Les quatre scènes ont `layers: []` et n'affichent que du texte et le fond de carte.

10. **Les scénarios de projection ne sont pas de vrais modèles.** Les trois scénarios (« Tendance actuelle », « Protection renforcée », « Accélération ») multiplient la pente linéaire observée par des facteurs 1,0 / 0,5 / 1,5. Le code l'indique dans un encart `Prototype`. Ils ne reposent sur aucun modèle écologique ou physique.

11. **`ingest-remote` nécessite un jeton Earthdata.** La commande CLI existe mais `NV_EARTHDATA_TOKEN` n'est pas défini. Tenter `ingest-remote` sans jeton échouera à l'authentification ASF.

12. **`NV_USE_POSTGIS=false` dans le `.env` local.** L'installation locale utilise SQLite. La colonne de géométrie `extent` est stockée comme un dict GeoJSON brut via le type `GeoJSONGeometry` (`core/geotypes.py`), non comme une géométrie PostGIS. Les requêtes spatiales utilisent de simples comparaisons de flottants sur les colonnes `west`/`south`/`east`/`north`.

13. **Le certificat n'est pas un titre officiel.** Le PDF et la page de vérification indiquent tous deux : « Ce n'est pas une certification officielle de la NASA ni d'aucun organisme. »

---

## 8. Script de démonstration en direct — 8 minutes

### Préparation avant la présentation
1. Backend lancé : `cd backend && python -m uvicorn app.main:app --reload --host 127.0.0.1 --port 8000`
2. Frontend lancé : `cd frontend && npm run dev`
3. Navigateur ouvert sur `http://localhost:5173`

---

### Étape 1 (0:00–0:45) — Présentation du projet
**Dire :** « NISAR Vision est une plateforme web qui démontre comment visualiser, interpréter et apprendre à partir de données radar NISAR en bande L directement dans un navigateur. Le backend est en Python avec FastAPI, le frontend en React. Tout tourne en local. »

**Montrer :** les cinq onglets de navigation en haut. Pas de clic pour l'instant.

---

### Étape 2 (0:45–2:00) — La carte avec des données réelles
**Cliquer :** l'onglet « Carte » (déjà actif). La carte s'ouvre centrée sur Douala — ignorer cela.

**Regarder :** la liste des couches. La case « NISAR L-band, rétrodiffusion HH » affiche « Aucune donnée ingérée ».

**Dire :** « La couche par défaut, HH, n'a pas de données réelles dans cette installation — le granule NISAR ingéré contient les polarisations VV et VH, pas HH. »

**Cocher :** la case « NISAR L-band, rétrodiffusion VV » pour l'activer.

**Cliquer :** le bouton ⊕ (Centrer sur la scène) à côté de la couche VV.

**Dire :** « La carte se déplace vers le sud de l'Inde et le nord-est du Sri Lanka. C'est l'empreinte réelle d'un produit GCOV L2 NISAR acquis le 13 novembre 2026, passage descendant, trace 120. La résolution est de 20 mètres. »

**Montrer :** les tuiles en niveaux de gris qui se chargent. Pointer les bords transparents (45 % de nodata — limite de fauchée).

---

### Étape 3 (2:00–3:00) — Clic sur un pixel
**Cliquer :** un point sur la terre ferme dans la partie visible du raster.

**Montrer :** le PixelPanel qui apparaît en haut à droite.

**Dire :** « Cliquer sur un pixel interroge le backend, qui lit le Cloud-Optimised GeoTIFF à cet endroit et retourne la puissance linéaire brute et la valeur en dB. Par exemple, au centre de la scène, on obtient VV = −14,0 dB et VH = −28,0 dB. L'écart de 14 dB entre la polarisation co-polariée et croisée est typique d'une terre sèche et modérément rugueuse. »

**Pointer :** le badge « Provisoire », l'identifiant du granule, la note « NISAR L-SAR bande L ».

**Dire :** « Remarquez qu'il n'y a pas d'interprétation thématique ici. Les règles de classification ont été calibrées sur HH et HV uniquement, donc on retourne volontairement rien pour VV/VH plutôt que d'afficher une mauvaise réponse. »

**Cliquer :** « Fermer » pour fermer le panneau.

---

### Étape 4 (3:00–4:00) — Légende et opacité
**Montrer :** la section « Légende ». Pointer la rampe noir-blanc et les libellés « −25 dB » (sombre) et « 0 dB » (clair).

**Dire :** « L'échelle va de −25 dB (retour très faible, sombre, comme l'eau calme) à 0 dB (retour fort, clair, comme les surfaces urbaines). La couche VV utilise cette plage ; VH utilise −30 à −5 dB. »

**Faire glisser :** le curseur d'opacité de la couche VV.

**Cocher/décocher :** « Fond de carte OSM » pour montrer la bascule de fond de carte.

---

### Étape 5 (4:00–5:00) — Module d'apprentissage
**Cliquer :** l'onglet « Apprendre ».

**Montrer :** les cinq niveaux. Le niveau 1 est déverrouillé.

**Cliquer :** « Commencer » sur le niveau 1.

**Montrer :** les cartes de leçon, puis faire défiler jusqu'au simulateur « DayNight ». Déplacer le curseur de couverture nuageuse.

**Dire :** « Chaque niveau contient un simulateur conceptuel — illustratifs uniquement, aucune donnée réelle. Après les leçons, un quiz. Les scores sont stockés côté serveur. Valider les cinq niveaux débloque un certificat PDF téléchargeable. »

**Ne pas soumettre le quiz** (inutile pour la démonstration).

---

### Étape 6 (5:00–6:00) — Récits
**Cliquer :** l'onglet « Récits ».

**Montrer :** les deux récits dans la liste.

**Cliquer :** « Démonstration : un défrichement simulé ».

**Dire :** « Les récits sont des narrations à défilement qui pilotent la carte. Celui-ci utilise des données de démo SIMULÉES — vous voyez le bandeau jaune "DONNÉES SIMULÉES". En faisant défiler, chaque carte de scène déplace la carte et change la date affichée. La démo montre un événement de défrichement simulé sur six acquisitions synthétiques. »

**Faire défiler** lentement une ou deux scènes. Pointer le bandeau jaune sur la carte.

**Dire :** « L'autre récit, sur l'Amazonie, n'a encore aucune donnée satellite — il affiche seulement du texte et un fond de carte. C'est un gabarit pour des contenus futurs. »

---

### Étape 7 (6:00–7:00) — Ce qui est simulé, en toute transparence
**Revenir à Carte.**

**Cocher :** la case « Afficher les données de démonstration simulées ».

**Dire :** « Cocher cette case charge six granules synthétiques. Ils ont été générés par programme — ils simulent un lac, une ville, un sol nu et une forêt avec un défrichement progressif. Ce ne sont PAS des observations NISAR. Le bandeau jaune le rend sans ambiguïté. Les données de démo alimentent le mode sémantique du Traducteur et les analyses du Tableau de bord — ces fonctionnalités nécessitent plusieurs dates, que nous n'avons pas encore depuis le satellite réel. »

**Décocher** la case démo.

---

### Étape 8 (7:00–8:00) — Conclusion
**Dire :** « Pour résumer ce qui est réel aujourd'hui : un granule NISAR, 13 novembre 2026, sud de l'Inde et Sri Lanka, polarisations VV et VH, pixels de 20 m, maturité provisoire. Tout le reste — les couches HH/HV, la classification, la série temporelle, la détection de changements, la projection — nécessite soit des données de démo, soit plusieurs acquisitions réelles, qui arriveront au fur et à mesure de la mission NISAR. »

**Montrer :** le terminal du backend brièvement si visible, pour montrer l'API en fonctionnement.

**Fin.**

### Parties à éviter ou à traiter avec précaution
- Ne PAS démontrer la « Lecture sémantique » sur des données réelles (retourne une tuile transparente).
- Ne PAS démontrer « Comparer avant / après » (1 seule date réelle, le contrôle est désactivé).
- Ne PAS démontrer le Tableau de bord « Lancer l'analyse » sur les données réelles seules (série à un point sans signification).
- Ne PAS présenter les libellés de classes de couverture terrestre comme validés (le JSON du jeu de règles indique qu'ils sont illustratifs et non validés).
- Ne PAS montrer le récit Amazonie comme contenant des images satellitaires (aucune couche associée).

---

## 9. Glossaire des termes techniques

**Rétrodiffusion (backscatter) :** la fraction d'une impulsion radar qui revient vers l'antenne ; un pixel lumineux signifie qu'une plus grande partie de l'énergie est revenue.

**dB (décibels) :** unité logarithmique exprimant l'intensité du signal comme 10 × log₁₀(puissance linéaire) ; chaque +10 dB correspond à 10 fois plus de puissance.

**Polarisation :** orientation du champ électrique de l'onde radar émise ou reçue ; NISAR émet et reçoit en combinaisons horizontal (H) et vertical (V).

**HH / VV (co-polarisé) :** les polarisations d'émission et de réception sont identiques ; sensibles à la rugosité de surface, à l'humidité et au double rebond des structures verticales.

**HV / VH (croisé) :** la polarisation change au retour ; sensible à la diffusion de volume à l'intérieur des couverts végétaux.

**GCOV (Ground-projected Covariance) :** produit NISAR de niveau L2 contenant la matrice de covariance des canaux polarimétriques, projetée sur une grille géographique ; les éléments diagonaux (HHHH, HVHV, VVVV, VHVH) sont les valeurs de puissance par canal utilisées ici.

**COG (Cloud-Optimised GeoTIFF) :** fichier GeoTIFF avec tuilage interne et niveaux d'overviews permettant à un serveur web de ne lire que les pixels nécessaires à une tuile de carte donnée, sans télécharger le fichier entier.

**Bande L :** bande de fréquence radar autour de 1,26 GHz (longueur d'onde ≈ 24 cm) ; pénètre la couverture nuageuse, la végétation légère et le sol sec mieux que les longueurs d'onde plus courtes.

**InSAR (SAR interférométrique) :** technique qui compare la phase de deux acquisitions SAR de la même zone pour mesurer des déformations du sol à l'échelle millimétrique ; illustrée au niveau 5 du module d'apprentissage mais non utilisée sur des données réelles dans cette application.

**rio-tiler :** bibliothèque Python qui enveloppe rasterio et GDAL pour servir des tuiles d'image depuis des COG ou d'autres fichiers raster, optimisée pour les requêtes de tuiles de cartes web.

**Maturité :** étiquette de confiance auto-attribuée de NISAR pour un produit : « beta » (précoce, peut contenir des artefacts), « provisional » (opérationnel standard), « validated » (vérifié indépendamment) ; les données de démo utilisent l'étiquette personnalisée « demo ».

**Fauchée (swath) :** la bande de terrain imagée lors d'un seul passage du satellite ; les bords de la fauchée produisent les pixels NaN (nodata) visibles comme des zones transparentes dans la superposition de tuiles.

**Nodata / NaN :** pixels hors de l'empreinte d'acquisition, ou pour lesquels l'instrument n'a renvoyé aucune mesure valide, stockés comme IEEE 754 NaN dans le COG float32 ; rendus transparents (alpha=0), jamais comme des zéros.

**Jeu de règles (prototype) :** fichier JSON versionné de seuils (`gcov_lband_prototype_v0_1.json`) utilisé pour affecter chaque pixel à une classe de couverture terrestre ; explicitement non validé et calibré uniquement pour HH/HV.

**AOI (Zone d'intérêt) :** polygone géographique dessiné par l'utilisateur, stocké en base de données et utilisé pour découper les analyses dans le Tableau de bord et le Traducteur.

---

## Ce qu'il ne faut PAS affirmer dans la présentation

1. **Ne pas affirmer que la classification thématique fonctionne sur des données réelles.** Le moteur de tuiles sémantiques récupère les chemins HHHH et HVHV. Le granule réel a VVVV/VHVH. Les tuiles de classification sont vides pour les données réelles.

2. **Ne pas affirmer que les résultats de classification sont validés.** Le fichier JSON du jeu de règles dit explicitement qu'il est illustratif et non validé. Aucune métrique de précision n'existe.

3. **Ne pas affirmer que les images HH ou HV proviennent de NISAR.** Ces couches (`nisar-gcov-hh`, `nisar-gcov-hv`) sont alimentées exclusivement par des données générées synthétiquement (`seed_demo_data`). Le granule réel ne contient pas de bandes HH ou HV.

4. **Ne pas affirmer que la détection de changements fonctionne sur des données réelles.** Elle nécessite au moins deux dates. Il n'existe qu'une seule acquisition réelle (13 novembre 2026).

5. **Ne pas affirmer que les graphiques de série temporelle ou de projection reposent sur des tendances réelles.** Avec un seul point de données, toute « tendance » est sans signification. Les scénarios de projection multiplient la pente par des coefficients fixes — ce ne sont pas des modèles écologiques.

6. **Ne pas affirmer que le récit Amazonie montre des images réelles.** Les quatre scènes ont `layers: []` ; la carte ne montre que le fond OSM.

7. **Ne pas affirmer que les données de démo représentent un lieu géographique réel.** La démo est centrée sur Douala, Cameroun, avec des valeurs de pixels entièrement synthétiques.

8. **Ne pas affirmer que le certificat est une qualification officielle.** Le PDF indique explicitement qu'il n'est pas une certification officielle de la NASA ni d'aucun autre organisme.

9. **Ne pas affirmer que les étiquettes d'interprétation de pixels (« Sol nu », « Végétation dense », etc.) constituent une carte de couverture terrestre fiable.** Elles n'apparaissent que sur des données HH/HV simulées et utilisent des seuils non validés.

10. **Ne pas affirmer que la « Projection exploratoire » prévoit l'évolution future du couvert forestier.** Le commentaire dans le code backend précise : « extrapolation linéaire de la tendance observée… Ce n'est pas une prévision. »
