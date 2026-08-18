# Mettre l'application en ligne (GitHub Pages)

Même principe que l'application paloxs alliums : un dépôt GitHub public dédié, avec GitHub Pages activé dessus. Gratuit, aucun serveur à gérer.

## 1. Créer le dépôt GitHub

1. Sur [github.com](https://github.com) (connecté avec votre compte, ou celui utilisé pour `Plan_stockage_allium`) > **New repository**.
2. Nom : `Suivi_Stock_Legumiere` (ou autre nom de votre choix).
3. Laissez-le **Public** (nécessaire pour GitHub Pages gratuit — voir la remarque sécurité dans `SETUP-FIREBASE.md`).
4. **Create repository**.

## 2. Déposer les fichiers

Deux façons de faire, selon votre aisance avec Git :

**Sans ligne de commande (le plus simple)** :
1. Sur la page du nouveau dépôt, cliquez sur **uploading an existing file** (ou **Add file > Upload files**).
2. Glissez-y **`index.html`** (celui que vous avez déjà complété avec votre `firebaseConfig`, voir `SETUP-FIREBASE.md` étape 5) et **`firestore.rules`**.
3. **Commit changes**.

**Avec Git**, si vous préférez :
```bash
git clone https://github.com/<votre-compte>/Suivi_Stock_Legumiere.git
cd Suivi_Stock_Legumiere
# copiez-y index.html et firestore.rules
git add index.html firestore.rules
git commit -m "Première version"
git push
```

## 3. Activer GitHub Pages

1. Dans le dépôt > **Settings** > **Pages** (menu de gauche).
2. **Source** : Deploy from a branch.
3. **Branch** : `main`, dossier `/ (root)` > **Save**.
4. Au bout d'une à deux minutes, l'URL apparaît en haut de cette même page, du type :
   `https://<votre-compte>.github.io/Suivi_Stock_Legumiere/`

C'est ce lien qu'il faut mettre en raccourci sur l'écran d'accueil des tablettes (menu du navigateur > **Ajouter à l'écran d'accueil**).

## 4. Mettre à jour l'application plus tard

Toute modification de `index.html` (nouvelle fonctionnalité, correction) : remplacez le fichier dans le dépôt (upload à nouveau, ou `git push`) — GitHub Pages republie automatiquement en une à deux minutes, sans rien à faire côté tablette ou ordinateur (il suffit de recharger la page).

Si vous rouvrez ce projet avec Claude Code (ou un développeur) pour ajouter une fonctionnalité, faites-lui vérifier la syntaxe du script avant de pousser (`node --check` sur le contenu du dernier `<script>...</script>` du fichier) — c'est ce qui a été fait à chaque étape de sa construction.
