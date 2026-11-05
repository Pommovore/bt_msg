# Spécification – BT Mesh Messenger (v0.1 brouillon)

## 1. Objectif et périmètre

Messagerie Android **décentralisée** sur maillage Bluetooth, sans serveur ni contrôleur central.
Chaque appareil est à la fois **terminal** et **routeur** (nœud).

Hors périmètre : communications synchrones (appel, visio, chat temps réel garanti), accès Internet.

### Architecture en 4 couches

```
┌──────────────────────────────┐
│ Application de messagerie    │  UI, comptes, modération
├──────────────────────────────┤
│ Middleware (API + stockage)  │  send_*, get_*, URI, files d'attente
├──────────────────────────────┤
│ Protocole maillage           │  découverte, annuaire, routage, fragmentation
├──────────────────────────────┤
│ Transport Bluetooth (BLE)    │  advertising/scan, GATT, L2CAP CoC
└──────────────────────────────┘
```

L'application n'a aucune dépendance envers le protocole : elle ne parle qu'à l'API du middleware.

## 2. Définitions et limites

| Élément | Règle |
|---|---|
| Nœud | Un appareil Android. `NodeID` = entier 8 bits (0–255), **max 256 nœuds** |
| Utilisateur | Identifié par `UserID` entier 10 bits (0–1023), **max 1024**, unique dans le réseau |
| Admin | `UserID = 0`, unique, modérateur suprême |
| Modérateur | Utilisateur désigné par l'admin |
| Session | Un nœud héberge N comptes mais **un seul utilisateur connecté à la fois** ; un utilisateur ne peut être connecté que sur un nœud à la fois (voir §5.4) |
| IDM | Identifiant de message local au nœud récepteur : compteur circulaire 32 bits |
| Diffusion | `UNICAST` (1→1), `MULTICAST` (1→N), `BROADCAST` (1→tous), codes `UC`, `MC`, `BC` |

Hypothèse : `NodeID` attribué à l'enregistrement du nœud (§4.2), jamais réutilisé tant que le nœud n'est pas révoqué.

## 3. Transport Bluetooth

- **BLE** (Android 8+, API 26 min ; recommandé API 31+ pour les permissions `BLUETOOTH_SCAN/ADVERTISE/CONNECT`).
- **Advertising** : balise périodique contenant `NodeID`, version de protocole, hash court de l'annuaire (`dirHash`), flag « session ouverte ».
- **Scan** : détection des voisins ; connexion GATT/L2CAP CoC pour échanges de données.
- Service de premier plan Android (*foreground service*) obligatoire pour rester actif en arrière-plan.
- Contraintes à respecter : nombre limité de connexions simultanées (typ. 5–7), débit BLE faible (~10–100 kB/s en L2CAP/2M PHY) → la vidéo est limitée en taille (§7.3).

## 4. Protocole maillage

### 4.1 Format de trame

```
| Ver(4b) | Type(4b) | Flags(8b) | TTL(8b) | Src NodeID(8b) | Dst NodeID(8b) | MsgSeq(16b) | FragIdx(8b) | FragCnt(8b) | Payload | MAC/Sig |
```

Types : `HELLO`, `DIR_SYNC`, `REGISTER`, `ROUTE`, `DATA`, `ACK`, `ADMIN`, `MODO_FETCH`.
`Dst = 0xFF` désigne une diffusion nœud-à-tous.

### 4.2 Autodécouverte et enregistrement

1. Un nœud allume le service, émet des `HELLO` et scanne.
2. Au premier contact avec un réseau, il envoie `REGISTER` (clé publique du nœud, nom) à un voisin.
3. Le voisin lui attribue un `NodeID` libre (voir §4.6 sur les conflits) et diffuse l'annonce.
4. L'annonce se **propage de proche en proche** (inondation contrôlée avec TTL et déduplication par `(Src,MsgSeq)`).
5. Chaque nœud tient un **annuaire local** : `NodeID → {clé publique, dernier contact, voisins}` pour **tous** les nœuds découverts (≤256 entrées, taille négligeable).
6. `DIR_SYNC` : à chaque rencontre, les nœuds comparent `dirHash` et échangent les deltas (anti-entropie) ; fusion par numéro de version par entrée.

### 4.3 Routage

- Routage par table de voisinage à vecteur de distance, repli sur **inondation contrôlée** (TTL max 16).
- **Store-and-forward** (DTN) : un message sans route est conservé et retransmis à chaque nouvelle rencontre jusqu'à expiration (TTL de livraison, défaut 7 jours) ; le maillage peut être partitionné.
- Accusé de réception de bout en bout (`ACK`) par fragment/message.

### 4.4 Annuaire utilisateurs

Chaque nœud synchronise aussi la table `UserID → {nom, rôle, NodeID courant, certificat}` (≤1024 entrées). La localisation d'un utilisateur (`NodeID courant`) est mise à jour à chaque login/logout et propagée.

### 4.5 Sécurité (proposition – absente de la demande)

- Pas d'autorité centrale mais **l'admin (user 0) est la racine de confiance** : sa clé publique est fixée à la création du réseau (QR code / provisioning).
- Chaque ajout d'utilisateur et chaque rôle de modérateur est un **certificat signé** par l'admin (ou par un modérateur pour l'ajout/suppression d'utilisateurs, avec délégation signée par l'admin).
- Messages signés par l'expéditeur ; chiffrement de bout en bout pour `UC`/`MC` (clés publiques de l'annuaire). `BC` signé, non chiffré (ou chiffré par clé réseau).
- Liste de révocation (utilisateurs/nœuds) propagée via `DIR_SYNC`.

### 4.6 Conflits et cas limites

- Deux nœuds obtiennent le même `NodeID` pendant une partition → résolution déterministe : le plus ancien `REGISTER` (horodatage + hash de clé) conserve l'ID, l'autre en réattribue un.
- Réseau plein (256 nœuds) : `REGISTER` refusé (code `NETWORK_FULL`).
- Horloges non fiables → utiliser compteurs/versions logiques plutôt que l'heure absolue.

## 5. Utilisateurs et rôles

### 5.1 Rôles

| Rôle | Droits |
|---|---|
| Admin (0) | Tout ; seul à ajouter/supprimer des modérateurs ; seul à pouvoir se démettre/transférer (hors périmètre v1) |
| Modérateur | Ajouter/supprimer des utilisateurs ; demander le rapatriement des messages d'un utilisateur |
| Utilisateur | Envoyer/recevoir des messages |

### 5.2 Gestion des utilisateurs

- `add_user(userID?, nom, credentials)` : attribue le plus petit `UserID` libre (≥1) ; certificat signé propagé.
- `remove_user(userID)` : révocation propagée ; l'ID n'est pas réutilisé avant purge confirmée.
- `add_moderator(userID)` / `remove_moderator(userID)` : **admin uniquement**.
- L'admin ne peut pas être supprimé.

### 5.3 Rapatriement de messages (modération)

- `modo_fetch(userID)` : un modérateur émet `MODO_FETCH` ; chaque nœud ayant stocké des messages envoyés **ou** reçus par cet utilisateur (à préciser) les transfère au demandeur.
- Stockage chez le modérateur : `modo/<UserID>/` (un fichier par message + index).
- Opération journalisée (audit) et visible de l'admin ; transfert chiffré vers la clé du modérateur.
- Les messages `UC`/`MC` chiffrés de bout en bout ne sont lisibles par un modérateur que si l'expéditeur/destinataire a chiffré aussi pour la clé de modération — **décision à prendre** (§10).

### 5.4 Session

- Login sur un nœud : le nœud annonce `user→node` ; si l'utilisateur est déjà connecté ailleurs, le login est refusé ou l'ancienne session est expirée (à trancher).
- Les messages pour un utilisateur sont stockés jusqu'à son prochain login et livrés au nœud où il se connecte.

## 6. Middleware – API

Toutes les opérations sont asynchrones et locales à l'utilisateur connecté.

### 6.1 Objets

```
Destinataires : { mode: UC | MC | BC, users: [UserID] }   // users vide pour BC
TextMessage   : { texte: UTF-8, max 1024 octets }
AudioMessage  : { codec: "opus", durée_ms, données }   // max proposé 2 min / ~1 Mo
VideoMessage  : { codec: "h264|av1", durée_ms, résolution, données } // max proposé 30 s / ~5 Mo
```

### 6.2 Primitives

| Primitive | Description |
|---|---|
| `send_text(texte, destinataires) → msgRef` | Envoi de texte court |
| `send_audio(audio, destinataires) → msgRef` | Envoi audio |
| `send_video(video, destinataires) → msgRef` | Envoi vidéo |
| `has_new_message() → bool` | `True` si au moins un message reçu n'a pas été consulté via `get_message` |
| `get_new_messages() → [IDM]` | Liste des IDM non consultés (vide si aucun). Ne les marque pas comme consultés |
| `get_message(IDM) → URI` | Renvoie l'URI du message et le marque comme consulté |

> La primitive `get_message(IDM)` figurait deux fois dans la demande ; elle est supposée unique.

### 6.3 Format d'URI

```
bt-msg://<IDM>:<scope>:<UserID_expéditeur>[:<NodeID>]
```

Exemples : `2:BC:290` = message n°2, diffusion, expéditeur utilisateur 290 ;
`17:UC:5`, `4:MC:12`.

Résolution via l'API de requête de l'URI :

| Requête | Retour |
|---|---|
| `uri.sender()` | `UserID` |
| `uri.kind()` | `TEXT | AUDIO | VIDEO` |
| `uri.scope()` | `UC | MC | BC` |
| `uri.timestamp()` | horodatage d'émission |
| `uri.media()` | pointeur/chemin vers le média (ou texte inline) |
| `uri.status()` | `COMPLETE | PARTIAL(n/m)` |

Le chemin du média est local au stockage du middleware (`content://` ou fichier privé de l'appli).
**IDM** : compteur 32 bits circulaire, incrémenté à chaque réception complète sur ce nœud ; unique localement, non global (un même message a des IDM différents selon le récepteur).

### 6.4 Erreurs

`NOT_LOGGED_IN`, `UNKNOWN_USER`, `TOO_LARGE`, `NO_ROUTE_YET` (message mis en file), `PERMISSION_DENIED`, `NETWORK_FULL`, `UNKNOWN_IDM`.

### 6.5 API d'administration (middleware)

`add_user`, `remove_user`, `add_moderator`, `remove_moderator`, `modo_fetch(userID)`, `list_users()`, `list_nodes()`.

## 7. Messagerie

### 7.1 Fonctions

- Texte court, audio, vidéo, en 1→1, 1→N, 1→tous.
- Messages asynchrones uniquement (pas de présence, pas de flux temps réel).
- Réception : notification locale quand `has_new_message()` passe à `True`.

### 7.2 Stockage

- Base locale (SQLite) + fichiers médias sur le nœud récepteur.
- Quota local configurable ; politique d'éviction (les plus anciens consultés d'abord).
- Les nœuds routeurs conservent temporairement (cache DTN) les messages en transit, chiffrés, avec TTL.

### 7.3 Médias et fragmentation

- Fragments de ≤ 200 octets (advertising/GATT sans MTU étendue) ou ≤ MTU L2CAP négociée.
- Reprise sur interruption (bitmap de fragments reçus) ; transmission en tâche de fond.
- Limites de taille (§6.1) imposées à l'envoi. Compression obligatoire côté expéditeur.
- Le `BC` vidéo est coûteux (inondation) : **quota/limitation par utilisateur** (ex. 1 BC média/heure).

## 8. Exigences non fonctionnelles

- Android 8+ ; permissions Bluetooth, notifications, service de premier plan.
- Consommation : scan en duty-cycle adaptatif.
- Pas de dépendance cloud ; fonctionne hors ligne.
- Interopérabilité : champ `Ver` pour évolutions.
- Tests : simulateur de maillage (≥256 nœuds virtuels) avant tests terrain.

## 9. Jalons proposés

1. Transport BLE + HELLO + découverte de voisins.
2. Annuaire nœuds, enregistrement, propagation.
3. Routage + DTN + fragmentation + ACK.
4. Middleware (API, URI, stockage) avec texte.
5. Audio/vidéo.
6. Utilisateurs, rôles, certificats, modération.
7. Simulateur et tests à grande échelle.

## 10. Points à trancher

1. **Modération vs chiffrement** : les modérateurs peuvent-ils lire les messages privés (`UC`/`MC`) ? Sinon, que rapatrie-t-on ?
2. `modo_fetch` : messages **envoyés**, **reçus**, ou les deux ?
3. Un utilisateur connecté ailleurs : refuser le login ou prendre la main ?
4. Les modérateurs ajoutent-ils des utilisateurs de façon autonome (signature déléguée) ou avec contre-signature admin ?
5. Disparition de l'admin/perte de clé : prévoir un transfert ou une récupération ?
6. Limites de taille audio/vidéo acceptables vu le débit BLE ?
7. L'IDM étant local, faut-il aussi un identifiant global `(UserID émetteur, seq)` pour déduplication ? (proposé en interne : oui)
8. Durée de rétention des messages en transit (7 jours proposé).
9. Authentification utilisateur : mot de passe local, PIN, biométrie ?
10. Cible : iOS à terme ? (impacte le choix du transport BLE).
