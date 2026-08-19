# Suivi Stock — Production & Commerce (SAS La Légumière)

Application web de suivi du stock quotidien (saisie du stock du soir par la production, achats/ventes/prix par le commerce), pour l'entreprise **la légumière**. Construite sur le même modèle que l'application sœur `Plan_stockage_allium` (paloxs échalotes/oignons), mais avec son propre projet Firebase et son propre dépôt GitHub. Ce fichier résume ce qui a été construit pour qu'une session Claude (ou un autre développeur) reprenne le contexte rapidement.

## Vue d'ensemble

- **Un seul fichier** `index.html` — HTML + CSS + JavaScript vanilla (pas de framework, pas de build). Tout le JS est dans une balise `<script>` en IIFE.
- **Multi-utilisateur en temps réel** via Firebase : Authentication (email/mot de passe) + Firestore. Sans Firebase, pas de partage de données entre les tablettes production et l'ordinateur commerce — contrainte assumée, comme pour l'app paloxs.
- **Hébergement** : GitHub Pages, dépôt dédié (`Suivi_Stock_Legumiere` ou équivalent), séparé du dépôt paloxs.
- **Projet Firebase** : séparé du projet paloxs (`plan-de-stockage-alliums`) — comptes et données indépendants, par choix explicite (voir historique de conversation : l'utilisateur a préféré isoler les deux outils plutôt que mutualiser les comptes).

## Différence de modèle de données par rapport à paloxs

L'app paloxs stocke tout dans **un seul document Firestore** (`planStockage/main`), pertinent pour un état "photo à l'instant T" (plan de cellules). Ici, les données sont un **flux quotidien qui grossit sans fin** (une entrée par produit et par jour, tous les jours) — un document unique grossirait indéfiniment et finirait par dépasser les limites Firestore. Le modèle est donc volontairement différent :

```
Firestore
├── produits/{produitId}     { nom, famille, variete, conditionnement, suremballage, actif, prixAchatDefaut, prixAchatUnite, ordre }
├── dernierStock/main        { [nomProduit]: { valeur, date } }   ← cache du dernier stock connu (alimenté par la validation commerce, pas directement par la production)
├── saisieStock/{date__slug} { date, produit, stockHaut, stockBas, stockSoir, importe, horodatage, responsable }
├── commerce/{date__slug__commercial} { date, produit, commercial, ventes, reservations, horodatage, responsable }
├── achats/{autoId}          { date, fournisseur, responsable, horodatage,
│                              lignes: [ { produit, quantite, prixAchat, heureDepart, transportType, transporteurNom } ] }
├── transactions/{autoId}    { date, type, produit, valeur, responsable, horodatage }   ← journal (Historique)
└── users/{uid}              { email, role }
```

Points importants :
- **`produits/{produitId}`** : un document par produit (`produitId` = `slugify(nom)`), pour permettre des règles de sécurité par champ (voir Rôles). Créé/supprimé uniquement par l'admin (onglet Paramètres) ; les champs `actif`/`prixAchatDefaut` peuvent aussi être modifiés par le rôle `commerce`. Chargé en entier au démarrage (`onSnapshot` sur la collection, triée côté client par `ordre`) — volume faible, pas de pagination nécessaire. Le produit ne porte plus de champ `reservations` persistant (voir `commerce` ci-dessous — les réservations sont désormais quotidiennes, comme les ventes).
- **`dernierStock/main`** : cache dénormalisé, mis à jour **uniquement par le rôle commerce** lors de la validation du stock du soir (onglet Commerce > « Stock du soir à valider »), pour que l'onglet Commerce affiche le "dernier stock connu" sans avoir à faire une requête historique par produit. La production ne l'écrit plus directement — ses saisies transitent d'abord par `saisieStock` (champ `importe: false`) avant d'être validées.
- **`saisieStock`** : un document par produit et par jour (id = `${date}__${slug(produit)}`), écrit par la production avec `stockHaut` et `stockBas` (deux niveaux de comptage physique) et `stockSoir` (leur somme, calculée automatiquement, non modifiable), plus `importe: false`. Le rôle commerce peut uniquement faire passer `importe` à `true` (règle Firestore restreinte à ce seul champ) lors de la validation, ce qui répercute alors la valeur dans `dernierStock`. Interrogée pour l'affichage du jour avec `where('date','==', aujourdhui)`, et pour la file d'attente de validation avec `where('importe','==', false)` — deux filtres simples, aucun index composé nécessaire. Un bouton **« Exporter le stock du soir (.xlsx) »** dans l'onglet Production génère un export SheetJS (Famille/Produit/Haut/Bas/Total) du jour, indépendant de la validation commerce — un enregistrement que la production peut garder de ce qu'elle a saisi le soir.
- **`commerce`** : une collection à écriture continue, un document par produit **et par commercial** et par jour (id = `${date}__${slug(produit)}__${slug(emailCommercial)}`), interrogée avec `where('date','==', aujourdhui)`. Porte `ventes` **et** `reservations` — les deux sont désormais quotidiennes (réinitialisées chaque jour, comme les ventes), saisies individuellement par chaque commercial pour ses propres ventes/réservations du jour, et sommées côté client pour obtenir les totaux "Global". `achats` et `prixAchat` du jour ne sont pas dans ce document et vivent dans la collection `achats`.
- **`achats/{autoId}`** : un document par **livraison** (pas par produit) — un fournisseur, une date, et une ou plusieurs `lignes` (une par produit reçu dans cette livraison), chaque ligne portant sa propre quantité, prix d'achat, heure de départ et transport (`transportType: 'ramasse' | 'transporteur'`, `transporteurNom` si transporteur) — pensé ainsi car une même livraison peut se scinder entre plusieurs moyens de transport selon les lignes. Modifiable/supprimable par le rôle `commerce` (pas réservé à l'admin, contrairement aux produits). L'onglet **Commerce** interroge cette collection avec `where('date','==', aujourdhui)` et additionne les `quantite` par produit pour calculer le stock réel du jour — aucun index composé nécessaire (filtre sur un seul champ). L'onglet **Achats** charge en plus un historique complet (`orderBy('horodatage','desc').limit(200)`, chargement paresseux comme `transactions`) pour le récapitulatif.
- **`transactions`** : journal des actions (comme l'Historique de paloxs), avec export CSV (point-virgule, BOM UTF-8 — même convention que le CSV de paloxs), suppression réservée à l'admin.

## Rôles et droits (5 niveaux)

Stockés dans Firestore, collection `users`, un document par compte (`id` = UID Firebase Auth), champs `email` + `role`.

- **`admin`** : tout, y compris l'onglet **Paramètres** (création/suppression de produits, familles/variétés/conditionnements/suremballages, gestion des comptes/rôles).
- **`production`** : saisie du stock du soir (onglet Production) ; peut aussi réactiver un produit inactif existant depuis cet onglet (modifie uniquement le champ `actif`) — ne peut pas créer de nouveau produit, ni le supprimer, ni modifier ses autres champs (famille, variété...).
- **`commerce`** : onglets **Achats** (entrées de marchandise), **Commerce** (ventes/réservations du jour, par commercial + validation du stock du soir) et **Prix d'achat** (prix par défaut + actif/inactif d'un produit existant) — pas la création/suppression de produit, ni les familles/variétés/conditionnements/suremballages, réservés à l'admin. Cette restriction est appliquée par les règles Firestore elles-mêmes (`produits/{produitId}`, champ par champ), pas seulement par l'interface. Le catalogue produit reste partagé entre tous les commerciaux (pas de filtrage), mais chaque commercial (Nadine, Julien, Flo, Mickael, Stéphanie...) saisit désormais ses propres ventes et réservations du jour dans son propre sous-onglet — un compte `commerce` ne peut écrire que le document `commerce` dont le champ `commercial` correspond à son propre email (règle Firestore), l'admin pouvant écrire n'importe quel document pour correction.
- **`viewer`** : lecture seule partout (rôle à assigner explicitement si une personne doit consulter sans saisir).
- **`aucun`** : rôle par défaut si aucun document `users/{uid}` n'existe (auto-créé à la première connexion). Ne donne accès à rien : l'écran « Accès non autorisé » s'affiche à la place de l'application tant que l'administrateur n'a pas changé ce rôle. Seules les personnes avec un rôle explicitement attribué voient l'application.

Un compte ne peut pas changer son propre rôle (sécurité anti-blocage, même règle que paloxs). Création de compte = toujours via la console Firebase (pas de self-signup) ; l'admin assigne ensuite le rôle depuis l'onglet **Paramètres > Comptes** de l'application.

## Onglets de l'application

- **Production** : produits actifs groupés par **famille** (sous-titres), avec deux champs de saisie **Haut** et **Bas** par produit et un **Total** calculé automatiquement (non modifiable), enregistrement automatique au blocage du champ (`change`). Un bouton « Ajouter » ouvre un panneau listant les produits **inactifs** (avec recherche) pour en réactiver un en un clic — la création d'un nouveau produit reste réservée à l'onglet Paramètres (admin). Un bouton **« Exporter le stock du soir (.xlsx) »** génère un export SheetJS du jour (Famille/Produit/Haut/Bas/Total), indépendant de la validation commerce. Recherche/filtre en haut. Important : cette saisie n'impacte jamais directement le "dernier stock connu" utilisé par le Commerce — elle passe par la validation (voir onglet Commerce) avant de compter dans le stock réel.
- **Achats** : formulaire de saisie d'une livraison — fournisseur (liste déroulante auto-alimentée + « Nouveau... »), date, puis une ou plusieurs lignes produit (produit, quantité, prix d'achat, heure de départ, transport = Ramasse ou nom de transporteur — liste déroulante auto-alimentée + « Nouveau... », par ligne pour permettre à une même livraison de se scinder entre plusieurs transports). En dessous, récapitulatif de tous les achats (recherche, modification en rechargeant la livraison dans le formulaire, suppression). Réservé en écriture au rôle `commerce`, visible en lecture par tous.
- **Commerce** : sous-onglets **Global** (agrégat en lecture seule, tous commerciaux confondus) et un sous-onglet par commercial (comptes ayant le rôle `commerce`, listés depuis `users`, libellé dérivé du préfixe de leur email). Un compte `commerce` arrive par défaut sur son propre sous-onglet ; un admin peut consulter/corriger n'importe quel sous-onglet. Pour chaque produit actif : **prix d'achat** par défaut (lecture seule, `produits.prixAchatDefaut`), **stock initial** = dernier stock connu (valeur en évidence + date sur une ligne séparée, plus discrète), achats du jour (calculés depuis la collection `achats`, lecture seule ici), ventes du jour et réservations (quantité promise à un client, pas encore livrée — **réinitialisées chaque jour comme les ventes**, saisies par commercial), **stock réel** = stock initial + achats − Σventes(tous commerciaux), et **stock disponible** = stock réel − Σréservations(tous commerciaux) (recalculés en direct, toujours sur les totaux tous commerciaux confondus, y compris dans les sous-onglets individuels — les ventes/réservations d'un commercial déduisent donc bien le stock réel/disponible vu par tous les autres). Dans le sous-onglet Global les champs ventes/réservations sont en lecture seule (totaux) ; dans un sous-onglet commercial ils sont éditables (uniquement par ce commercial ou par l'admin) et ne portent que la valeur de ce commercial pour la journée. Un bouton **▸ Détail** par produit déplie une ligne listant les achats du jour (fournisseur, prix, heure, transport) ainsi que la ventilation ventes/réservations par commercial. En dessous, section **Stock du soir à valider** : liste des saisies de production pas encore importées (`saisieStock.importe == false`), avec Haut/Bas/Total, un bouton de validation par ligne et un bouton « Tout valider » — la validation copie la valeur dans `dernierStock` et marque l'entrée comme importée.
- **Prix d'achat** : tous les produits (actifs ou non) avec leur prix d'achat par défaut (affiché/saisi en €) et son unité (**Colis** ou **Kilo**, liste déroulante — `produits.prixAchatUnite`), plus leur statut actif/inactif (saisonnalité) ; le prix et l'unité sont aussi modifiables depuis l'onglet Paramètres (admin). Modifiables par le rôle `commerce` — les autres colonnes (famille, variété, conditionnement, suremballage) y sont affichées en lecture seule. Dans l'onglet Commerce, la colonne Prix d'achat affiche la même valeur avec son unité (ex : « 2 € / colis »).
- **Historique** : journal de toutes les actions (stock du soir, achats, ventes, prix, activation/désactivation produit), recherche libre, export CSV, suppression réservée à l'admin. Un bouton **« Exporter sauvegarde Excel (.xlsx) »** génère en plus un classeur complet (un onglet par collection : Produits, Stock du soir, Ventes, Achats, Historique) via la bibliothèque SheetJS chargée en CDN — pensé comme sauvegarde locale, accessible à tous les rôles.
- **Paramètres** (admin uniquement) : création/suppression de produits (désignation, famille, variété, conditionnement, suremballage), gestion des rôles par compte.

## Style / thème

Même identité que l'app paloxs (cohérence entre les deux outils) : palette `--copper: #963E88`, vague dégradée (`--wave-1: #B23C92` → `--wave-2: #5B2A66`) sous l'en-tête et la carte de connexion, polices Fraunces (titres) / Inter (texte) / IBM Plex Mono (chiffres). Icône caisse/palox (SVG inline) sur l'écran de connexion, plutôt qu'oignon/échalote (déjà utilisé par l'app paloxs) — pour distinguer visuellement les deux outils tout en gardant la même famille graphique.

## Origine des données produits

Le schéma produit garde en tête un export possible du catalogue de l'entreprise (fiche article : Code, EAN13, Désignation, Famille, Variété, Origine, Conditionnement, Calibre, Catégorie, PCB Vte, Suremballage, Label, Actif...), mais l'app ne conserve volontairement qu'un sous-ensemble de ces champs, jugés pertinents pour l'usage quotidien et pour la création manuelle depuis l'onglet **Paramètres** (pas de catalogue pré-rempli — chaque produit est créé par l'admin) :

- `nom` (= Désignation complète de l'article, saisie libre — pas encore décidé si elle sera composée automatiquement à partir des autres champs, voir historique de conversation)
- `famille` (ex : Oignon, Mini légumes)
- `variete` (ex : Oignon de Roscoff, Mini carotte)
- `conditionnement` (ex : Sac 5kg, Barquette 250g, Tresse fourreau 1kg)
- `suremballage` (ex : Palox, Carton, Caisse, Meuble, Coffret, Barquette, Sachet, Sans suremballage)
- `actif`, `prixAchatDefaut`, `ordre`

Champs volontairement **non conservés** : Code article, EAN13, Origine, Calibre, Catégorie, PCB Vte, Label, Préparation, Complément, Controle légumière, Marque, Poids U. Vte. Le champ `nom` (désignation) sert de clé d'unicité fonctionnelle.

`famille`, `variete`, `conditionnement` et `suremballage` sont **entièrement gérés par l'admin, sans liste figée dans le code** : dans l'onglet Paramètres, chacun des quatre champs est une liste déroulante alimentée par les valeurs déjà utilisées dans les produits existants, plus une option **« + Nouvelle... »** qui révèle un champ texte pour créer une valeur la première fois qu'elle est utilisée — elle réapparaît ensuite dans la liste déroulante pour les produits suivants. Aucun catalogue n'est pré-chargé (`Produits_initial.json` est un tableau vide, gardé comme documentation du format) ; l'admin construit sa liste produit par produit depuis l'app. Premières familles prévues : Oignon (Roscoff AOP), puis Mini légumes. Extension prévue à d'autres familles et, à terme, aux autres commerciaux (Nadine, Julien, Flo, Mickael, Stéphanie).

## Fichiers du dépôt

- `index.html` — l'application (production, à connecter à votre propre projet Firebase — voir `SETUP-FIREBASE.md`).
- `firestore.rules` — règles de sécurité Firestore (rôles, lecture/écriture par collection).
- `SETUP-FIREBASE.md` — guide pas-à-pas de configuration Firebase (projet, Auth, Firestore, règles, comptes, import produits).
- `README-DEPLOIEMENT.md` — guide pas-à-pas pour créer le dépôt GitHub et activer GitHub Pages.
- `Produits_initial.json` — catalogue produits prêt à coller dans `produits/main.liste` (voir « Origine des données produits » ci-dessus pour le schéma des champs).

## Pour continuer le développement

Ouvrir ce dossier avec Claude Code (ou coller ce fichier en contexte) suffit à comprendre l'architecture sans relire tout `index.html`. Toujours valider la syntaxe JS après édition (`node --check` sur le contenu du `<script>` inline) avant de committer/pousser. Évolution déjà anticipée dans le modèle de données : réimport ponctuel vers Excel si besoin — les collections `saisieStock`/`commerce` s'exportent facilement (requête par plage de dates → CSV), sur le même principe que l'export CSV déjà présent dans l'onglet Historique.

Note sur la migration `commerce` (doc par produit+jour → doc par produit+jour+commercial) : les anciens documents `commerce` écrits avant ce changement n'ont pas de champ `commercial` — ils ne remontent donc dans aucun sous-onglet individuel (seulement dans un total Global s'ils sont un jour ré-agrégés manuellement) et n'ont pas de `reservations` (ce champ vivait auparavant sur `produits`). Aucune migration automatique n'a été faite ; à surveiller si un historique `commerce` antérieur à ce changement doit être ré-exploité.
