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
├── produits/{produitId}     { nom, famille, variete, conditionnement, suremballage, actif, prixAchatDefaut, ordre }
├── dernierStock/main        { [nomProduit]: { valeur, date } }   ← cache du dernier stock connu
├── saisieStock/{date__slug} { date, produit, stockSoir, horodatage, responsable }
├── commerce/{date__slug}    { date, produit, achats, ventes, prixAchat, horodatage, responsable }
├── transactions/{autoId}    { date, type, produit, valeur, responsable, horodatage }   ← journal (Historique)
└── users/{uid}              { email, role }
```

Points importants :
- **`produits/{produitId}`** : un document par produit (`produitId` = `slugify(nom)`), pour permettre des règles de sécurité par champ (voir Rôles). Créé/supprimé uniquement par l'admin (onglet Paramètres) ; le champ `actif`/`prixAchatDefaut` peut aussi être modifié par le rôle `commerce` (onglet Commerce > Gestion des produits). Chargé en entier au démarrage (`onSnapshot` sur la collection, triée côté client par `ordre`) — volume faible, pas de pagination nécessaire.
- **`dernierStock/main`** : cache dénormalisé, mis à jour à chaque saisie de stock du soir (écriture batch en même temps que `saisieStock`), pour que l'onglet Commerce affiche le "dernier stock connu" sans avoir à faire une requête historique par produit (évite un index composé Firestore et des lectures inutiles).
- **`saisieStock` / `commerce`** : une collection à écriture continue, un document par produit et par jour (id = `${date}__${slug(produit)}`), interrogées avec un simple filtre `where('date','==', aujourdhui)` (pas d'index composé nécessaire). C'est l'historique complet ; il n'est jamais rechargé en entier par l'app (seul `transactions` sert de journal consultable, limité aux 300 dernières entrées).
- **`transactions`** : journal des actions (comme l'Historique de paloxs), avec export CSV (point-virgule, BOM UTF-8 — même convention que le CSV de paloxs), suppression réservée à l'admin.

## Rôles et droits (5 niveaux)

Stockés dans Firestore, collection `users`, un document par compte (`id` = UID Firebase Auth), champs `email` + `role`.

- **`admin`** : tout, y compris l'onglet **Paramètres** (création/suppression de produits, familles/variétés/conditionnements/suremballages, gestion des comptes/rôles).
- **`production`** : saisie du stock du soir (onglet Production) uniquement.
- **`commerce`** : achats/ventes/prix d'achat (onglet Commerce), + gestion du prix d'achat par défaut et de l'actif/inactif d'un produit existant (section « Gestion des produits » du même onglet) — pas la création, la suppression, ni les familles/variétés/conditionnements/suremballages, réservés à l'admin. Cette restriction est appliquée par les règles Firestore elles-mêmes (`produits/{produitId}`, champ par champ), pas seulement par l'interface.
- **`viewer`** : lecture seule partout (rôle à assigner explicitement si une personne doit consulter sans saisir).
- **`aucun`** : rôle par défaut si aucun document `users/{uid}` n'existe (auto-créé à la première connexion). Ne donne accès à rien : l'écran « Accès non autorisé » s'affiche à la place de l'application tant que l'administrateur n'a pas changé ce rôle. Seules les personnes avec un rôle explicitement attribué voient l'application.

Un compte ne peut pas changer son propre rôle (sécurité anti-blocage, même règle que paloxs). Création de compte = toujours via la console Firebase (pas de self-signup) ; l'admin assigne ensuite le rôle depuis l'onglet **Paramètres > Comptes** de l'application.

## Onglets de l'application

- **Production** : liste des produits actifs, un champ "stock du soir" par produit, enregistrement automatique au blocage du champ (`change`), pas de bouton "Enregistrer" global. Recherche/filtre en haut.
- **Commerce** : pour chaque produit actif — dernier stock connu (valeur + date), achats du jour, ventes du jour, prix d'achat, et **stock réel** recalculé en direct = dernier stock connu + achats − ventes. En dessous, section **Gestion des produits** (tous les produits, actifs ou non) où le rôle commerce ajuste le prix d'achat par défaut et l'actif/inactif (saisonnalité) — les autres colonnes (famille, variété, conditionnement, suremballage) y sont affichées en lecture seule.
- **Historique** : journal de toutes les actions (stock du soir, achats, ventes, prix, activation/désactivation produit), recherche libre, export CSV, suppression réservée à l'admin.
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

Ouvrir ce dossier avec Claude Code (ou coller ce fichier en contexte) suffit à comprendre l'architecture sans relire tout `index.html`. Toujours valider la syntaxe JS après édition (`node --check` sur le contenu du `<script>` inline) avant de committer/pousser. Deux évolutions déjà anticipées dans le modèle de données :
- Étendre à d'autres commerciaux : ajouter un champ `commercial` sur les produits et/ou sur les rôles, filtrer les vues en conséquence.
- Réimport ponctuel vers Excel si besoin : les collections `saisieStock`/`commerce` s'exportent facilement (requête par plage de dates → CSV), sur le même principe que l'export CSV déjà présent dans l'onglet Historique.
