# Arrive Connect — Plan de projet

> Plateforme web interne permettant aux ~4 000 collaborateurs d'Arrive de déclarer leurs déplacements professionnels, de se découvrir mutuellement sur une même destination, de confirmer leur présence sur place et de partager des bons plans géolocalisés.

**Hypothèses cadrantes (validées)**
- Participation **100 % volontaire (opt-in)**.
- Vérification de présence par **check-in manuel** (« je suis là »), pas de GPS continu.
- Effectifs **mondiaux / multi-régions** → RGPD applicabgit@github.com:pierregier/arrive-travel-match-meet.gitle + autres régimes (CCPA, etc.) à anticiper.
- Hébergement **cloud**, sans préférence forte → on optimise pour la résidence des données et le coût.

---

## 1. Contexte et objectifs

### Contexte
Arrive est une organisation d'environ 4 000 personnes réparties sur plusieurs régions. Les collaborateurs se déplacent régulièrement pour raisons professionnelles, mais ces déplacements sont peu visibles : deux personnes peuvent se trouver dans la même ville le même jour sans le savoir, alors qu'une rencontre en présentiel aurait de la valeur (réseau, collaboration, culture d'entreprise).

### Problème
Aucune visibilité transversale sur « qui est où, quand ». Les occasions de rencontre informelle sont perdues, et la connaissance locale (bons restaurants, lieux de réunion, astuces pratiques) reste cloisonnée.

### Objectifs
1. **Donner de la visibilité** sur les déplacements professionnels déclarés.
2. **Provoquer des rencontres** en détectant automatiquement les chevauchements ville + dates.
3. **Confirmer une présence réelle** de façon légère et respectueuse de la vie privée.
4. **Capitaliser la connaissance locale** via une carte collaborative de bons plans.

### Indicateurs de succès (KPIs)
- Taux d'adoption : % d'employés ayant déclaré ≥ 1 trajet sur 3 mois.
- Nombre de « matches » générés / nombre de matches ayant donné lieu à une mise en relation acceptée.
- Nombre de check-ins effectués.
- Nombre de bons plans ajoutés et taux de consultation de la carte.
- Rétention : utilisateurs actifs mensuels.

### Hors périmètre (non-objectifs)
- Réservation de voyages / billetterie (pas un outil de travel booking).
- Note de frais / comptabilité.
- Surveillance ou contrôle du temps de travail (à exclure explicitement, voir §10).

---

## 2. Fonctionnalités — MVP puis itérations

### MVP (v1)
- Authentification via le SSO d'entreprise (SAML/OIDC).
- Profil minimal (nom, équipe, photo issue de l'annuaire).
- **Déclaration de trajet** : date(s), ville de destination, durée, objet.
- **Liste / calendrier** de mes propres trajets (édition, suppression).
- **Matching automatique** ville + dates avec notification.
- **Opt-in de visibilité** : un trajet n'est « visible/matchable » que si l'employé le décide.
- **Check-in manuel** sur un trajet (« je suis bien sur place »).
- Consentement et page de confidentialité claire au premier login.

### Itération 2 (v1.5)
- **Carte des bons plans** : ajout de points géolocalisés (resto, lieu utile), catégories, vote/upvote.
- Filtres de recherche : par ville, par période, par équipe.
- Préférences de mise en relation (qui peut me voir : tout le monde / mon BU / personne).
- Notifications configurables (email + in-app).

### Itération 3 (v2+)
- Suggestions intelligentes (« 3 collègues seront à Berlin la semaine prochaine »).
- Agrégation par destination (« événements / hubs » : 12 personnes à Lisbonne en mars).
- Intégration calendrier (Google/Microsoft) pour pré-remplir les trajets — **opt-in strict**.
- Modération communautaire des bons plans, signalement.
- Tableau de bord anonymisé / agrégé pour les RH ou la direction Mobilité (jamais nominatif).
- Application mobile (ou PWA) pour faciliter le check-in en déplacement.

---

## 3. User stories / cas d'usage

**Déclaration**
- En tant qu'employé, je déclare un déplacement (ville, dates, durée, objet) afin que le système puisse me proposer des rencontres.
- En tant qu'employé, je choisis si un trajet est privé ou visible/matchable.
- En tant qu'employé, je modifie ou supprime un trajet à tout moment.

**Matching**
- En tant qu'employé, je suis notifié quand un·e collègue sera dans la même ville aux mêmes dates.
- En tant qu'employé, j'accepte ou j'ignore une suggestion de mise en relation avant tout partage de coordonnées.
- En tant qu'employé, je peux masquer mes trajets à certaines personnes / périmètres.

**Présence**
- En tant qu'employé sur place, je fais un check-in en un tap pour confirmer ma présence.
- En tant que collègue, je vois que la présence est « confirmée » et non plus seulement « déclarée ».

**Bons plans**
- En tant qu'employé, j'ajoute un bon plan sur la carte d'une ville.
- En tant qu'employé en déplacement, je consulte les bons plans des collègues pour ma destination.
- En tant qu'employé, je vote pour les bons plans utiles et je signale les contenus inappropriés.

**Confidentialité / contrôle**
- En tant qu'employé, je consulte et exporte toutes mes données.
- En tant qu'employé, je révoque mon consentement et supprime mon compte/données en self-service.

---

## 4. Logique de matching des trajets

### Définition d'un match
Deux trajets « matchent » si :
1. **Même ville** de destination (normalisée), et
2. **Chevauchement de dates** : `trajetA.dateDébut ≤ trajetB.dateFin` ET `trajetB.dateDébut ≤ trajetA.dateFin`, et
3. Les **deux** trajets sont en visibilité « matchable », et
4. Les périmètres de visibilité des deux personnes sont mutuellement compatibles.

### Normalisation des villes (point critique)
Le piège classique : « Paris », « paris », « Paris, FR », « PAR » ne matchent pas en comparaison textuelle naïve.
**Recommandation** : ne pas stocker une chaîne libre. Utiliser un **autocomplete géographique** à la saisie (par ex. service de geocoding) qui résout vers un identifiant stable :
- un `city_id` canonique (référentiel type GeoNames), **ou**
- au minimum un couple `(latitude, longitude)` + nom normalisé.

Le matching se fait alors sur le `city_id` (exact, indexable, rapide), pas sur le texte.

### Approche technique du calcul
- **Déclencheur** : à chaque création/modification de trajet (visibilité = matchable), lancer une requête de recherche des trajets se chevauchant sur le même `city_id`.
- **Requête** : index composite sur `(city_id, date_start, date_end)`. Le chevauchement de dates est une condition SQL simple et indexable.
- À l'échelle de 4 000 personnes, le volume de trajets est faible (quelques dizaines de milliers/an). Pas besoin de moteur complexe : **PostgreSQL suffit largement**. Un job asynchrone (queue) gère la notification pour ne pas bloquer la requête utilisateur.

### Pseudo-code
```sql
SELECT t2.*
FROM trips t2
WHERE t2.city_id = :new_city_id
  AND t2.user_id <> :new_user_id
  AND t2.visibility = 'matchable'
  AND t2.date_start <= :new_date_end
  AND t2.date_end   >= :new_date_start
  -- + filtre de compatibilité des périmètres de visibilité
;
```

### Mise en relation à double consentement
Un match **ne révèle pas** immédiatement les coordonnées. Flux recommandé :
1. Les deux personnes reçoivent « un·e collègue sera là aux mêmes dates ».
2. Chacune peut accepter la mise en relation.
3. Les coordonnées (ou un canal de chat / lien e-mail) ne sont partagées **qu'après acceptation des deux côtés** (ou au moins de la personne contactée).

Ceci protège la vie privée et évite le sentiment de « surveillance ».

---

## 5. Mécanisme de vérification de présence + choix de géolocalisation

### Décision : check-in manuel (retenu)
Conformément à ta réponse, la présence est confirmée par une **action volontaire de l'employé** : un bouton « Je suis sur place » sur le trajet concerné. C'est le bon choix — il est :
- léger techniquement,
- très peu intrusif,
- nettement plus simple à justifier au regard du RGPD (pas de collecte de localisation continue).

### Niveaux d'assurance possibles (du plus simple au plus contraignant)
| Niveau | Mécanisme | Intrusion | Recommandation |
|---|---|---|---|
| 0 | Présence simplement *déclarée* (le trajet existe) | Nulle | Base |
| 1 | **Check-in manuel** en un tap | Très faible | **Retenu (MVP)** |
| 2 | Check-in **avec capture GPS ponctuelle** au moment du tap (one-shot, opt-in) | Faible/modérée | Option future, opt-in explicite |
| 3 | Suivi GPS continu / arrière-plan | Élevée | **Écarté** (disproportionné) |

### Renforcement optionnel (sans GPS continu)
Si un jour Arrive veut une preuve plus forte que le seul tap, sans basculer dans la surveillance :
- **Géolocalisation ponctuelle one-shot** au moment du check-in via l'API standard du navigateur (`navigator.geolocation.getCurrentPosition`), **opt-in à chaque usage**, avec contrôle « distance < N km de la ville déclarée ». On ne stocke alors **pas** la position brute, seulement un booléen « cohérent oui/non » + la ville.
- Alternative non-GPS : **check-in par QR code / Wi-Fi du site** sur les bureaux Arrive (scan d'un QR affiché dans le bureau de la ville). Confirme la présence dans un lieu Arrive sans données de localisation personnelle.

**Choix recommandé pour le MVP** : niveau 1 (tap), avec architecture prête à accueillir le niveau 2 en opt-in plus tard. On ne collecte aucune coordonnée GPS au lancement.

---

## 6. Système de bons plans + intégration cartographique

### Modèle fonctionnel
- Un bon plan = point géolocalisé `(lat, lng)` + titre + catégorie (restaurant, café, salle de réunion, transport, autre) + description courte + auteur + votes.
- Saisie : recherche d'adresse (autocomplete/geocoding) ou clic sur la carte.
- Consultation : carte filtrable par ville et catégorie ; vue liste également.
- Communauté : upvotes, signalement, modération.

### Choix de la solution cartographique
| Solution | Modèle | Avantages | Limites |
|---|---|---|---|
| **MapLibre GL JS + tuiles OpenStreetMap** (ex. via un fournisseur de tuiles type MapTiler/Protomaps) | Open-source, tuiles vectorielles | Pas de lock-in, coût maîtrisé, contrôle des données, pas de transfert vers Google | Geocoding à brancher séparément |
| Mapbox GL JS | SaaS | Très abouti, beau rendu | Coût à l'échelle, dépendance fournisseur |
| Google Maps JS API | SaaS | Familier, geocoding excellent | Coût, transfert de données vers Google (sujet RGPD), lock-in |
| Leaflet + OSM | Open-source léger | Simple, mature | Tuiles raster, rendu moins fluide que vectoriel |

**Recommandation : MapLibre GL JS** avec tuiles OSM servies par un fournisseur compatible UE (MapTiler ou Protomaps auto-hébergé). Raison : open-source, pas de verrouillage, coût prévisible à l'échelle de 4 000 utilisateurs, et meilleur contrôle sur où transitent les données — argument important vu le contexte multi-régions/RGPD. Pour le **geocoding/autocomplete** (saisie de villes et d'adresses), utiliser un service distinct (Nominatim auto-hébergé pour le souverain, ou une API geocoding compatible UE).

---

## 7. Choix techniques (dimensionné pour 4 000 utilisateurs)

> 4 000 utilisateurs internes, c'est une charge **modérée** : quelques centaines d'utilisateurs actifs simultanés en pic. Inutile de sur-architecturer (pas de microservices, pas de Kubernetes obligatoire). On vise simplicité, robustesse et conformité.

### Frontend
- **React** (ou Vue) + TypeScript.
- **MapLibre GL JS** pour la carte.
- PWA pour faciliter le check-in mobile sans app store (itération mobile).

### Backend
- **API REST/GraphQL** en **Node.js (NestJS)** ou **Python (FastAPI/Django)** — choisir selon les compétences de l'équipe. Django offre l'admin et l'ORM « batteries included », utile pour démarrer vite.
- **Authentification** : SSO d'entreprise via **OIDC/SAML** (pas de mot de passe géré en interne).
- **File de tâches** (Redis + worker) pour les notifications de matching en asynchrone.

### Base de données
- **PostgreSQL** + extension **PostGIS** (requêtes géospatiales pour la carte et les distances de check-in). PostGIS est l'outil de référence et couvre tous les besoins géo du projet.

### Hébergement
- Cloud managé (au choix : AWS / GCP / Azure / Scaleway / OVHcloud).
- **Résidence des données** : compte tenu du contexte mondial avec collaborateurs UE, héberger la base principale **dans l'UE** et gérer les autres régions par configuration. Privilégier une base managée (RDS/Cloud SQL/équivalent) avec sauvegardes chiffrées.
- Conteneurs (Docker) sur un service managé simple (ECS/Cloud Run/App Service) — suffisant à cette échelle.

### Observabilité / sécurité
- Chiffrement en transit (TLS) et au repos.
- Logs d'accès, journalisation des consentements.
- CI/CD, environnements séparés (dev/staging/prod).

---

## 8. Modèle de données

```
User
  id (PK)
  sso_subject_id        -- identifiant SSO, pas de mot de passe stocké
  display_name
  team / business_unit
  home_region
  default_visibility    -- 'private' | 'bu' | 'all'
  created_at

ConsentRecord
  id (PK)
  user_id (FK -> User)
  consent_type          -- 'core', 'matching', 'location_oneshot', etc.
  granted (bool)
  version               -- version de la politique acceptée
  timestamp

City                    -- référentiel normalisé
  id (PK)               -- city_id canonique (ex. GeoNames)
  name
  country_code
  latitude
  longitude

Trip
  id (PK)
  user_id (FK -> User)
  city_id (FK -> City)
  date_start
  date_end
  purpose               -- objet du déplacement (texte court ou enum)
  visibility            -- 'private' | 'matchable'
  presence_status       -- 'declared' | 'checked_in'
  created_at, updated_at
  -- index: (city_id, date_start, date_end), (user_id)

Match
  id (PK)
  trip_a_id (FK -> Trip)
  trip_b_id (FK -> Trip)
  city_id
  overlap_start, overlap_end
  status                -- 'suggested' | 'a_accepted' | 'both_accepted' | 'declined'
  created_at

CheckIn
  id (PK)
  trip_id (FK -> Trip)
  user_id (FK -> User)
  method                -- 'manual' | 'geo_oneshot' | 'qr'
  is_location_consistent (bool, nullable)  -- résultat du contrôle, PAS la position brute
  timestamp

Tip
  id (PK)
  author_id (FK -> User)
  city_id (FK -> City)
  geom (POINT, PostGIS)
  title
  category              -- 'restaurant' | 'cafe' | 'meeting' | 'transport' | 'other'
  description
  upvotes (compteur, ou table dédiée)
  status                -- 'active' | 'flagged' | 'removed'
  created_at

TipVote
  id (PK)
  tip_id (FK -> Tip)
  user_id (FK -> User)
  UNIQUE(tip_id, user_id)
```

**Principes du modèle**
- On ne stocke **jamais** de trace GPS continue.
- Pour le check-in géo optionnel, on ne conserve que le **résultat** (`is_location_consistent`), pas la latitude/longitude de l'employé.
- Le consentement est versionné et historisé (`ConsentRecord`).

---

## 9. RGPD / vie privée

> Données de déplacement + localisation = **données personnelles**. Le contexte employeur/employé impose une vigilance forte : un salarié peut difficilement « consentir librement » à son employeur. Le consentement n'est donc pas toujours la meilleure base légale ; pour un outil **purement volontaire** comme celui-ci, il reste défendable, mais doit être réellement libre (aucune conséquence à refuser).

### Bases légales
- Outil **opt-in** : le **consentement** explicite est la base retenue, renforcé par le caractère facultatif (pas de pénalité à ne pas participer).
- Documenter pourquoi ce n'est pas du contrôle du temps de travail (finalité = mise en relation, pas surveillance).

### Principes appliqués
- **Minimisation** : on ne collecte que ville/dates/objet. Pas de GPS continu. Le check-in géo optionnel ne stocke qu'un booléen.
- **Limitation de finalité** : interdiction d'usage RH/disciplinaire. À écrire noir sur blanc dans la politique et à séparer techniquement (pas d'export nominatif vers les RH).
- **Visibilité contrôlée par l'utilisateur** : privé par défaut, matchable sur choix explicite, périmètres réglables.
- **Double consentement à la mise en relation** : aucune coordonnée partagée sans accord.
- **Durée de conservation** : purge automatique des trajets passés (ex. après X mois) ; suppression des matches obsolètes.
- **Droits des personnes** : accès, export (portabilité), rectification, effacement et **révocation du consentement** en self-service.
- **Transferts internationaux** : contexte mondial → cartographier les flux, héberger en UE par défaut, encadrer les transferts hors UE (clauses contractuelles types) ; attention au choix du fournisseur de cartes/geocoding (éviter les transferts implicites vers des tiers).
- **Sécurité** : chiffrement repos/transit, contrôle d'accès, journalisation.

### Démarches formelles
- **AIPD / DPIA** : une analyse d'impact est recommandée (traitement de données de localisation à grande échelle dans un contexte employeur). À mener avant le lancement.
- Information transparente (politique de confidentialité dédiée, lisible au 1er login).
- Consultation des **représentants du personnel** selon les juridictions.
- Impliquer le **DPO** dès la conception (privacy by design).

> ⚠️ Ce plan fournit des repères de conformité, pas un avis juridique. Fais valider le dispositif (bases légales, durées, DPIA, transferts) par le DPO et le service juridique d'Arrive avant la mise en production.

---

## 10. Garde-fous anti-dérive (important)
- **Aucune fonction de contrôle ou de surveillance** : pas de reporting nominatif vers le management/RH, pas de classement, pas d'alerte « absent ».
- Tableaux de bord uniquement **agrégés et anonymisés** (seuils d'agrégation pour éviter la ré-identification).
- Séparation stricte des finalités dans le code et les accès.

---

## 11. Roadmap par phases

### Phase 0 — Cadrage (2–3 semaines)
- Validation DPO / juridique, lancement de la DPIA.
- Choix définitif de la stack et du cloud (résidence UE).
- Maquettes UX, parcours de consentement.

### Phase 1 — MVP (6–10 semaines)
- SSO, profil, déclaration de trajets, référentiel villes normalisé.
- Matching ville+dates + notifications asynchrones.
- Double consentement de mise en relation.
- Check-in manuel.
- Centre de confidentialité (export, suppression, révocation).
- **Pilote** sur 1–2 business units.

### Phase 2 — Bons plans & carte (4–6 semaines)
- Carte MapLibre + ajout/consultation de bons plans, votes, signalement.
- Filtres (ville, période, catégorie), préférences de visibilité fines.

### Phase 3 — Enrichissement (en continu)
- Suggestions intelligentes, agrégation par destination.
- Intégration calendrier (opt-in), PWA mobile.
- Modération communautaire, tableaux de bord anonymisés.

### Jalons transverses
- Revue sécurité avant chaque mise en prod.
- Mesure des KPIs et boucle de feedback après le pilote.

---

## Récapitulatif des recommandations clés
- **Matching** : normaliser les villes via geocoding → `city_id`, requête de chevauchement indexée dans PostgreSQL. Inutile de sur-architecturer à cette échelle.
- **Présence** : check-in manuel (retenu) ; prévoir l'option géo one-shot opt-in et le QR/Wi-Fi sans jamais stocker de position brute.
- **Carte** : MapLibre GL JS + tuiles OSM (fournisseur compatible UE), geocoding séparé.
- **Stack** : React/TS + Node ou Django + PostgreSQL/PostGIS + cloud managé en UE.
- **RGPD** : opt-in réellement libre, minimisation, double consentement, séparation des finalités, DPIA et validation DPO/juridique avant lancement.
