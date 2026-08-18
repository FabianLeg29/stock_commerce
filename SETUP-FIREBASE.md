# Configuration Firebase — Suivi Stock La Légumière

Projet Firebase **séparé** de celui du plan de stockage paloxs alliums (comptes et données indépendants). Comptez environ 15 minutes, aucune compétence technique requise au-delà du copier-coller.

## 1. Créer le projet Firebase

1. Allez sur [console.firebase.google.com](https://console.firebase.google.com), connecté avec le compte Google de l'entreprise.
2. **Ajouter un projet** > nommez-le par exemple `suivi-stock-legumiere`.
3. Désactivez Google Analytics si proposé (pas nécessaire ici) > **Créer le projet**.

## 2. Activer l'authentification par email/mot de passe

1. Menu de gauche > **Build > Authentication** > **Get started**.
2. Onglet **Sign-in method** > cliquez sur **Email/Password** > activez le premier interrupteur > **Save**.
3. Onglet **Users** > **Add user** : créez un premier compte (celui qui sera administrateur), par exemple `fabian@lalegumiere.com` avec un mot de passe temporaire. Notez-le, vous vous en servirez à la première connexion.
4. Créez ensuite un compte par personne qui utilisera l'application (production, commerce...), avec la même méthode.

## 3. Créer la base de données Firestore

1. Menu de gauche > **Build > Firestore Database** > **Create database**.
2. Choisissez **Production mode** (pas "test mode").
3. Choisissez une région proche (ex : `eur3 (europe-west)`) > **Enable**.

## 4. Publier les règles de sécurité

1. Dans Firestore Database, onglet **Rules**.
2. Supprimez le contenu existant, collez-y **tout le contenu** du fichier `firestore.rules` fourni.
3. **Publish**.

Ces règles définissent les 4 rôles (admin / commerce / production / viewer) et ce que chacun peut lire ou modifier — voir le détail dans `CLAUDE.md`.

## 5. Récupérer la configuration et l'ajouter à l'application

1. Roue crantée en haut à gauche > **Paramètres du projet**.
2. Section **Vos applications** > icône **`</>`** (Web) > donnez un nom (ex: `suivi-stock-web`) > **Enregistrer l'application** (pas besoin de cocher "Firebase Hosting").
3. Firebase affiche un bloc `firebaseConfig = { apiKey: ..., authDomain: ..., ... }`. Copiez ces valeurs.
4. Ouvrez `index.html`, repérez la section tout en haut du `<script>` :
   ```js
   const firebaseConfig = {
     apiKey: "REMPLACER_apiKey",
     authDomain: "REMPLACER.firebaseapp.com",
     projectId: "REMPLACER_projectId",
     storageBucket: "REMPLACER.appspot.com",
     messagingSenderId: "REMPLACER_senderId",
     appId: "REMPLACER_appId"
   };
   ```
5. Remplacez chaque valeur par celle de votre projet Firebase, puis enregistrez le fichier.

## 6. Donner un rôle au premier administrateur

Au premier login, l'application crée automatiquement un document dans `users/{uid}` avec le rôle `aucun` par défaut — ce rôle ne donne accès à rien : la personne voit un écran « Accès non autorisé » à la place de l'application, tant qu'un administrateur ne lui a pas attribué un rôle. Il faut passer votre propre compte en `admin` manuellement une seule fois :

1. Ouvrez l'application (voir `README-DEPLOIEMENT.md` pour la mettre en ligne) et connectez-vous une première fois avec le compte administrateur créé à l'étape 2 — cela crée le document `users/{votre-uid}` avec le rôle `aucun`, et affiche l'écran « Accès non autorisé ».
2. Dans la console Firebase > **Firestore Database** > **Data**, ouvrez la collection `users`, trouvez le document correspondant à votre email, et changez le champ `role` de `aucun` à `admin`.
3. Rechargez l'application : l'onglet **Paramètres** apparaît, et vous pouvez désormais changer le rôle des autres comptes (y compris passer de `aucun` à `production`, `commerce`, `viewer` ou `admin`) directement depuis l'application (plus besoin de repasser par la console Firebase par la suite).

## 7. Créer la liste de produits

Le premier chargement de l'application crée automatiquement les collections nécessaires, mais la liste de produits est vide au départ — c'est volontaire, aucun catalogue n'est pré-chargé.

Dans l'onglet **Paramètres > Produits** de l'application (une fois connecté en admin), ajoutez chaque produit avec le formulaire : Désignation (texte libre), puis Famille / Variété / Conditionnement / Suremballage — quatre listes déroulantes alimentées par les valeurs déjà utilisées, chacune avec une option **« + Nouvelle... »** qui fait apparaître un champ texte pour créer une nouvelle valeur la première fois qu'elle est utilisée (ex : la première fois que vous ajoutez la famille « Mini légumes », vous la tapez ; ensuite elle apparaît dans la liste pour les produits suivants). Un bouton **« Supprimer tous les produits »** permet de vider toute la liste en une fois si besoin de repartir de zéro.

## Sécurité et coût

- Le dépôt GitHub est public (nécessaire pour GitHub Pages gratuit) mais **ce n'est pas un risque** : la config Firebase ci-dessus n'est pas un secret, toute la sécurité repose sur les règles Firestore (`firestore.rules`) et sur l'authentification — exactement comme pour l'application paloxs.
- Le forfait gratuit **Spark** de Firebase est largement suffisant pour ce volume d'usage (quelques dizaines de lectures/écritures par jour).
