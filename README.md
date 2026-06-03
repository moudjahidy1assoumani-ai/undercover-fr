```markdown
# 🕵️ UNDERCOVER FR — Jeu Web Multijoueur

Un jeu de déduction sociale et de bluff temps réel optimisé pour smartphones et navigateurs. 
Développé en architecture découplée, sans framework lourd, s'appuyant sur une synchronisation instantanée.

**Version actuelle** : Mars 2026

---

## 🔧 1. Stack Technique

- **Frontend** : Vanilla JavaScript (ES Modules) — architecture modulaire pure, aucun framework (React/Vue/Angular).
- **Build Tool** : Vite — serveur de développement ultra-rapide avec Hot Module Replacement (HMR).
- **Styles & Design** : Tailwind CSS + styles personnalisés (*dark spy aesthetic*).
  - *Titres* : Syne (bold) | *Corps* : DM Sans | *Code/Chiffres* : IBM Plex Mono
  - *Palette* : Dark Navy (`#020817`), Cyan Glow (`#00F5D4`), Amber Warning (`#F59E0B`).
- **Base de données** : Firebase Realtime Database — synchronisation bidirectionnelle instantanée de l'état des salons de jeu.
- **Authentification** : Firebase Anonymous Authentication — permet aux utilisateurs de rejoindre instantanément sans barrière (ni email ni mot de passe).
- **Déploiement** : Vercel (CI/CD automatisé à chaque commit).

---

## 📁 2. Structure des Fichiers

L'intégralité de la logique applicative est centralisée dans le répertoire `undercover-game/` :

```text
undercover-game/
├── index.html
├── vite.config.js
├── tailwind.config.js
├── package.json
├── .env.local
└── src/
    ├── main.js              # Router d'état principal (home ➔ lobby ➔ game)
    ├── firebase.js          # Initialisation du SDK Firebase + authentification anonyme
    ├── utils.js             # Gestionnaire de Toasts UI, états de chargement, avatars
    ├── css/
    │   └── styles.css       # Directives Tailwind et design tokens système
    ├── data/
    │   └── words.js         # Base lexicale : 1 156 paires de mots réparties sur 18 thèmes
    ├── game/
    │   ├── roles.js         # Algorithme de distribution des rôles et règles de victoire
    │   ├── twists.js        # Moteur de modificateurs de rounds (10 twists, 20% de probabilité)
    │   └── roomManager.js   # Abstraction CRUD Firebase (listeners, transactions de vote)
    └── screens/
        ├── HomeScreen.js    # Vue d'aiguillage (Création / Jonction de salon via code)
        ├── LobbyScreen.js   # Hub d'attente en temps réel + configuration multi-thèmes
        └── GameScreen.js    # Cycle de vie complet des phases d'une partie

```

---

## 🎮 3. Cycle de Vie & Architecture Firebase

### États d'une Room Firebase

La machine à états synchrone fait transiter chaque salon (`room`) par 6 phases séquentielles strictes :

1. `waiting` ➔ Configuration initiale, attente des joueurs et sélection des thèmes.
2. `revealing` ➔ Révélation sécurisée et individuelle du mot secret.
3. `playing` ➔ Phase active d'énonciation des indices selon l'ordre généré.
4. `voting` ➔ Ouverture des scrutins en temps réel.
5. `results` ➔ Dépouillement automatisé et affichage de la sentence.
6. `ended` ➔ Fin de la session de jeu et affichage du tableau des scores.

### Schéma de Données (JSON NoSQL)

```json
rooms/
  "{ROOM_CODE}": {
    "code": "A7B4",
    "state": "playing",
    "creatorId": "uid_host",
    "currentRound": 1,
    "wordPair": ["Pastèque", "Melon"],
    "selectedThemes": ["Nourriture", "Manga & Animé"],
    "speakingOrder": ["uid_1", "uid_2", "uid_3"],
    "currentSpeakerIndex": 0,
    "eliminatedThisRound": "",
    "players": {
      "uid_1": {
        "pseudo": "Xénomorphe",
        "role": "civil",
        "secretWord": "Pastèque",
        "isEliminated": false
      }
    },
    "votes": {
      "uid_1": "uid_2"
    }
  }
}

```

---

## 👥 4. Les 5 Rôles Intégrés

| Icône | Rôle | Conditions de distribution | Description / Objectifs |
| --- | --- | --- | --- |
| 👤 | **Civil** | Majorité du salon | Connaît le mot principal. Doit démasquer les intrus sans s'exposer. |
| 🕵️ | **Undercover** | 1 joueur par défaut | Reçoit un mot sémantiquement proche. Doit analyser les indices pour se fondre dans la masse. |
| 🗺️ | **Touriste (Mr White)** | Dès 5 joueurs et plus | Aucun mot attribué. Doit bluffer à l'aveugle en décodant les indices des autres. |
| ⚖️ | **La Balance** | Dès 6 joueurs et plus | Connaît le mot ET l'identité exacte de l'Undercover. Perd immédiatement si l'Undercover parvient à le cibler. |
| 🎭 | **Agent Double** | Dès 7 joueurs et plus | Reçoit le mot de l'Undercover. Son but secret est de saboter la partie pour se faire éliminer en premier. |

---

## ⚙️ 5. Logique Métier & Règles Avancées

* **Double passe au Tour 1** : Pour équilibrer l'avantage informationnel, le Tour 1 impose à chaque joueur de fournir deux indices distincts (Passe 1, puis Passe 2 selon un ordre d'énonciation remélangé). Les tours suivants reprennent une passe standard.
* **Limite de tours dynamique** : Afin d'éviter les situations d'impasse, la durée de la partie est indexée sur le nombre de participants :
* `3 joueurs` ➔ Maximum 1 tour de table.
* `4 joueurs` ➔ Maximum 2 tours de table.
* `5+ joueurs` ➔ Maximum 3 tours de table.
*Si la limite est atteinte sans l'élimination de l'intrus, l'Undercover l'emporte automatiquement.*


* **Gestion fine du système de vote** :
* Les votes sont modifiables en direct tant que le scrutin est ouvert.
* Dès le dernier vote enregistré, une finalisation automatique se déclenche (délai de grâce de `800ms`).
* L'hôte possède un privilège d'administration pour forcer le passage au tour suivant si un joueur est AFK.
* Le système verrouille le bouton de validation si aucun vote n'est présent dans l'arbre Firebase.


* **Inversion Civil / Undercover** : À chaque initialisation de round, l'algorithme applique une probabilité de 50% d'inverser l'affectation de la paire de mots (ex: Partie 1 : Civil=Pastèque / Undercover=Melon ➔ Partie 2 : Civil=Melon / Undercover=Pastèque). Cela double mathématiquement les combinaisons effectives du dictionnaire.

---

## 📚 6. Base Lexicale Multi-thèmes

Le jeu intègre un dictionnaire robuste de **1 156 paires de mots** qualifiées, réparties sur **18 thèmes**. Le Lobby gère la multi-sélection : l'hôte peut fusionner plusieurs catégories pour générer un pool de mots sur-mesure.

```text
🍽️ Nourriture     ⚽ Sport         📱 Tech           🎬 Cinéma
🌍 Voyage         🎵 Musique       🏠 Maison         🐾 Animaux
🎮 Jeux           🎓 École/Travail 👗 Mode           🍹 Boissons
💪 Santé          🌿 Nature        🤣 Culture        ⚽ Football
🎌 Manga & Animé  🌙 Islam

```

---

## 🚀 7. Déploiement & Lancement Local

### Prérequis

* **Node.js** (Version LTS recommandée)
* Un projet actif sur **Firebase Console** avec *Authentication Anonyme* et *Realtime Database* activées.

### Installation

1. Cloner le dépôt et se positionner dans le dossier :

```bash
   git clone <url-du-repo>
   cd undercover-game
   npm install

```

2. Configurer les variables d'environnement en créant un fichier `.env.local` à la racine :

```env
   VITE_FIREBASE_API_KEY=AIzaSy...
   VITE_FIREBASE_AUTH_DOMAIN=ton-projet.firebaseapp.com
   VITE_FIREBASE_DATABASE_URL=[https://ton-projet-default-rtdb.europe-west1.firebasedatabase.app](https://ton-projet-default-rtdb.europe-west1.firebasedatabase.app)
   VITE_FIREBASE_PROJECT_ID=ton-projet
   VITE_FIREBASE_STORAGE_BUCKET=ton-projet.appspot.com
   VITE_FIREBASE_MESSAGING_SENDER_ID=123456789
   VITE_FIREBASE_APP_ID=1:123456789:web:abc123

```

3. Configurer les Règles de Sécurité de la Realtime Database dans Firebase :

```json
   {
     "rules": {
       "rooms": {
         "$roomCode": {
           ".read": true,
           ".write": "auth != null",
           "players": {
             "$uid": {
               ".write": "auth != null && (auth.uid == $uid || data.parent().parent().child('creatorId').val() == auth.uid)"
             }
           }
         }
       }
     }
   }

```

4. Lancer le serveur d'infrastructure de développement :

```bash
   npm run dev

```

*Note : Pour tester le comportement multi-joueurs en local, connectez votre smartphone sur le même réseau Wi-Fi et flashez ou ouvrez l'adresse IP locale (`http://192.168.1.X:3000`) affichée dans votre terminal.*

---

## 🛠️ 8. Historique de Débug (Rétrospective technique)

* **🐛 Persistance anormale du Toast UI** : Correction de l'instabilité du cycle d'extinction liée à l'écouteur `animationend` CSS. Remplacement par une transition d'opacité native couplée à un `setTimeout` asynchrone géré par le thread applicatif.
* **🐛 Blocage du bouton "Nouvelle Partie"** : Résolution d'une rupture du routeur dans `GameScreen.js` provoquée par l'absence du bloc d'évaluation `case 'waiting'` dans la structure de contrôle globale.
* **🐛 Biais d'attribution des rôles (Hôte systématiquement Undercover)** : Remplacement de l'ancien algorithme de mélange de tableau non uniforme par une implémentation stricte du principe de permutation de **Fisher-Yates**.
* **🐛 Erreurs de parsing de chaînes (Caractères d'apostrophes)** : Nettoyage intégral de la base de données `data/words.js`. Sécurisation des chaînes de caractères en remplaçant les quotes simples par des doubles quotes strictes.
* **🐛 Validation de scrutin à blanc (0 vote)** : Sécurisation de l'interface graphique. Désactivation complète des fonctions d'envoi et du bouton de validation si l'objet de transaction Firebase ne contient aucune clé de vote active.
* **🐛 Exception fatale `selectedTheme is not defined**` : Correction d'une régression de scope. Refactorisation globale et renommage des pointeurs en `selectedThemes` (Array) pour s'aligner sur la gestion multi-thèmes du moteur.

---

Développé avec exigence en Vanilla JS • Propulsé par Firebase & Vite • Déployé sur Vercel

```

```
