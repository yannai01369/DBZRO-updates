# DBZ RO — Updater

Launcher / auto-updater pour le client **Drago Ball Z Resurrection Online** (DBZRO.exe).

- Vérifie au démarrage la dernière version publiée sur GitHub Releases.
- Compare chaque fichier local au manifeste distant (taille puis SHA-256).
- Ne télécharge que ce qui a changé (`update.zip`), avec repli sur `full.zip` (première installation ou fichier corrompu/absent).
- Vérifie les SHA-256 avant d'écrire dans le dossier du jeu.
- Ne touche jamais à `Data Files/config.ini`, aux logs, ni aux fichiers de l'updater.
- Bouton **Jouer** pour lancer `DBZRO.exe`.
- Cible **.NET Framework 4.8**, déjà présent sur Windows 10/11 : aucune installation de runtime pour les joueurs.

---

## Contenu du projet

```
DBZROUpdater/
  DBZROUpdater.csproj     Projet WinForms (net48)
  Program.cs              Point d'entrée + instance unique + TLS 1.2
  MainForm.cs             Interface du launcher
  Core/                   Moteur : config, GitHub, hachage, application des MAJ
  Ui/                     Barre de progression personnalisée
  updater.json            Configuration à côté de l'exe (modèle)
  app.manifest            DPI aware / asInvoker
  icon.ico
tools/
  publish-update.ps1      Génère et publie les releases GitHub
README.md
```

L'exe compilé se trouve dans `DBZROUpdater/bin/Release/net48/` après compilation.

---

## Compiler l'updater

Prérequis : [.NET SDK](https://dotnet.microsoft.com/download) (déjà installé sur cette machine).

```powershell
cd DBZROUpdater
dotnet build -c Release
```

Résultat : `DBZROUpdater\bin\Release\net48\DBZROUpdater.exe`.

---

## Installation côté joueur

1. Compiler l'updater (ou récupérer l'exe déjà compilé).
2. Copier dans le dossier du jeu, **à côté de `DBZRO.exe`** :
   - `DBZROUpdater.exe`
   - `DBZROUpdater.exe.config`
   - `updater.json` (renseigné, voir ci-dessous)
3. Double-cliquer sur `DBZROUpdater.exe` : la vérification se lance automatiquement.
4. Une fois à jour, cliquer sur **Jouer**.

Le dossier ressemble alors à :

```
DBZ-RO\
  DBZRO.exe
  DBZROUpdater.exe
  DBZROUpdater.exe.config
  updater.json
  Data Files\...
  ...
  .updater\state.json        (créé automatiquement, ne pas distribuer)
```

> Un seul updater peut tourner à la fois (mutex). L'updater refuse de s'exécuter si `DBZRO.exe` est déjà lancé, pour ne pas modifier des fichiers en cours d'utilisation.

---

## Configuration : `updater.json`

Fichier JSON placé à côté de l'exe. Le bouton **Réglages** de l'updater l'ouvre dans le Bloc-notes.

```json
{
  "githubOwner": "moncompte",
  "githubRepo": "DBZRO-updates",
  "gameExe": "DBZRO.exe",
  "token": "",
  "apiBase": "https://api.github.com"
}
```

| Champ | Obligatoire | Rôle |
|---|---|---|
| `githubOwner` | oui | Compte ou organisation GitHub qui héberge les releases. |
| `githubRepo` | oui | Nom du dépôt (ex. `DBZRO-updates`). |
| `gameExe` | non | Exécutable lancé par **Jouer**. Défaut : `DBZRO.exe`. |
| `token` | non | Jeton GitHub, uniquement nécessaire si le dépôt est **privé**. |
| `apiBase` | non | Défaut : `https://api.github.com`. À ne changer que pour un GitHub Enterprise / un miroir. |

Le launcher lit la **dernière release publiée** (`/releases/latest`) et cherche les assets `manifest.json`, `update.zip` et `full.zip`. Le numéro de version affiché est celui du tag (le `v` initial est retiré).

### Dépôt privé / jeton

Un dépôt privé exige un jeton dans `updater.json`. Attention :

- Le jeton est distribué avec le jeu : n'importe quel joueur peut l'extraire de l'exe/config.
- Utiliser un jeton **fine-grained** limité à ce dépôt, avec uniquement **Contents: Read** (fine-grained) — et non un jeton personnel complet.
- Un dépôt **public** est plus simple et plus sûr pour la distribution : pas de jeton à embarquer. Les fichiers du jeu seront toutefois téléchargeables par tout le monde.

---

## Publier une mise à jour

### 1. Préparer le dossier client

Utiliser un dossier propre du client à jour (ex. `D:\DBZ-RO`) : c'est son contenu qui sera publié. Les éléments suivants sont **exclus automatiquement** :

- `Data Files/config.ini` (configuration personnelle du joueur)
- `Data Files/logs`, `Data Files/logs/*`
- `Patcher`, `Patcher/*`
- `.updater`, `.updater/*`
- `*.log`, `*.tmp`, `Thumbs.db`, `desktop.ini`, `.DS_Store`, `.updateignore`
- `DBZROUpdater.exe`, `DBZROUpdater.exe.config`, `DBZROUpdater.pdb`, `updater.json`
- `manifest.json`, `update.zip`, `full.zip`

Pour exclure d'autres fichiers (musique, vidéos, gros fichiers inutiles), créer un `.updateignore` à la racine du dossier client, une règle par ligne (mêmes jokers que PowerShell, `#` pour un commentaire) :

```
# Gros fichiers non nécessaires au jeu
Data Files/music/*.mp3
Data Files/video/*.bik
```

### 2. Générer les fichiers

```powershell
.\tools\publish-update.ps1 -GameDir "D:\DBZ-RO" -Version 2.4.1 -Repo "moncompte/DBZRO-updates" -ApiBase "https://api.github.com"
```

Sans `-Upload`, le script écrit seulement les fichiers dans `dist\` (ou `-OutDir`). Sortie produite :

| Fichier | Contenu |
|---|---|
| `manifest.json` | Version + liste de tous les fichiers (chemin, taille, SHA-256). |
| `update.zip` | Uniquement les fichiers modifiés/ajoutés depuis la release précédente. |
| `full.zip` | Client complet (secours : nouvelle installation, fichier corrompu, fichier absent). |

> `-Repo` est nécessaire pour comparer à la release précédente et produire un vrai `update.zip`. Sans `-Repo` (ou sans release précédente), `update.zip` contient tous les fichiers.

### 3. Publier sur GitHub Releases

Créer un jeton GitHub, puis :

```powershell
$env:GITHUB_TOKEN = "ghp_xxxxxxxxxxxxxxxxx"
.\tools\publish-update.ps1 `
  -GameDir "D:\DBZ-RO" `
  -Version 2.4.1 `
  -Repo "moncompte/DBZRO-updates" `
  -Notes "Correction des quêtes + nouveaux sons" `
  -Upload
```

Le script crée la release (tag `v2.4.1`) et téléverse `manifest.json`, `update.zip` et `full.zip` via l'API REST. Aucune installation de `gh` n'est requise.

> Le jeton doit avoir le droit **Contents: Read and write** sur le dépôt (fine-grained) ou la portée `repo` (jeton classique). Il n'est utilisé que par le script, jamais par l'updater si le dépôt est public.

### Options du script

| Option | Effet |
|---|---|
| `-GameDir <dossier>` | Dossier client à publier (obligatoire). |
| `-Version <x.y.z>` | Version de la release (obligatoire). Le `v` initial est accepté. |
| `-Repo <compte/dépôt>` | Dépôt GitHub. Obligatoire pour lire la release précédente et pour `-Upload`. |
| `-OutDir <dossier>` | Dossier de sortie. Défaut : `dist\` à la racine du projet. |
| `-Notes <texte>` | Description de la release. Défaut : « Mise à jour x.y.z ». |
| `-Upload` | Crée la release et téléverse les assets. Sans cette option, génération locale seulement. |
| `-Replace` | Supprime puis recrée la release si le tag existe déjà. |
| `-Prerelease` | Publie en pré-release (ignorée par `/releases/latest`). |
| `-NoDelta` | Ne génère pas `update.zip`. |
| `-NoFull` | Ne génère pas `full.zip` (déconseillé : plus de réparation possible). |
| `-ApiBase <url>` | API GitHub alternative. Défaut : `https://api.github.com`. |

`-NoDelta` et `-NoFull` sont incompatibles : l'updater n'aurait plus rien à télécharger.

### Première release

Il n'y a pas encore de release précédente : `update.zip` et `full.zip` contiennent tous les deux le client complet, et les joueurs téléchargent l'ensemble des fichiers lors de la première mise à jour. Les releases suivantes ne transporteront que le delta.

---

## Fonctionnement de la vérification

1. L'updater lit `updater.json` et vérifie la présence de `DBZRO.exe`.
2. Il demande la dernière release GitHub et récupère `manifest.json`.
3. Pour chaque fichier du manifeste : comparaison de la taille, puis calcul du SHA-256 si la taille correspond.
4. Si tout correspond, il enregistre la version dans `.updater/state.json` et affiche « À jour ».
5. Sinon il télécharge `update.zip`, vérifie chaque fichier (taille + SHA-256), puis copie dans le dossier du jeu. Les fichiers absents de l'archive sont cherchés dans `full.zip`, téléchargé seulement si nécessaire.
6. En cas de fichier corrompu ou manquant, relancer la vérification : `full.zip` sert de réparation.

La première vérification peut prendre un peu de temps (environ 3 000 fichiers, ~340 Mo) : seules les métadonnées et les empreintes sont lues.

---

## Dépannage

| Message / symptôme | Cause probable | Solution |
|---|---|---|
| « 404 » / release introuvable | `githubOwner`/`githubRepo` erronés, aucune release publiée, ou dépôt privé sans jeton. | Corriger `updater.json` (bouton **Réglages**) ou publier une release. |
| « Limite de requêtes GitHub atteinte » | 60 requêtes/heure par IP sans authentification. | Attendre l'heure indiquée ou ajouter un jeton dans `updater.json`. |
| « Connexion à GitHub impossible » | Pas d'accès réseau, proxy, pare-feu. | Vérifier la connexion. Les téléchargements ont 60 min de délai maximum. |
| « Ferme DBZRO.exe avant de lancer la mise à jour » | Le jeu tourne. | Fermer le jeu puis relancer l'updater. |
| « updater.json est introuvable » | L'exe n'est pas dans le dossier du jeu ou le fichier a été supprimé. | Placer `updater.json` à côté de `DBZROUpdater.exe`. |
| « Remplace VOTRE_COMPTE… » | `updater.json` est encore le modèle. | Renseigner `githubOwner` et `githubRepo`. |
| « full.zip dépasse 2 Go » | Limite d'un asset GitHub (2 Go). | Exclure des fichiers via `.updateignore`, ou publier sans `full.zip`. |
| Le jeu ne se lance pas avec **Jouer** | `gameExe` incorrect. | Vérifier le nom exact (défaut `DBZRO.exe`). |
| Un fichier reste corrompu après MAJ | Fichier inchangé entre deux releases et absent de `update.zip`. | Relancer la vérification : `full.zip` est utilisé en secours. |
| La release existe déjà | Tag `v<version>` déjà présent. | Incrémenter `-Version` ou ajouter `-Replace` (destructif pour cette release). |

### Limites GitHub

- 2 Go maximum par asset : `full.zip` du client complet doit rester sous cette limite.
- 1 000 assets maximum par release (le projet n'en utilise que 3).
- 60 requêtes/heure en anonyme, 5 000/heure avec jeton.

---

## Sécurité

- Chaque fichier est vérifié (taille + SHA-256) avant d'être copié dans le dossier du jeu.
- Les chemins des archives sont validés : aucune extraction hors du dossier du jeu (protection Zip-Slip).
- Le jeton n'est **jamais** envoyé aux serveurs de téléchargement (CDN) : l'en-tête `Authorization` est retiré après la redirection GitHub.
- Ne pas publier `updater.json` s'il contient un jeton, ni `.updater/state.json`.
