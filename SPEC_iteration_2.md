# Spécification – BT Mesh Messenger (v0.2 – Itération 2)

## 1. Objectif et périmètre

Messagerie Android **décentralisée** sur maillage Bluetooth (BLE), sans serveur ni contrôleur central.
Chaque appareil Android est à la fois **terminal utilisateur** et **routeur store-and-forward** (nœud).

* **Cible** : Uniquement Android (API 26 / Android 8.0 min ; API 31 / Android 12+ recommandé).
* **Hors périmètre** : Communications synchrones (appel audio/vidéo, chat temps réel garanti), accès Internet ou passerelle Cloud, récupération de clé administrateur perdue.

### Architecture en 4 couches

```
┌────────────────────────────────────────────┐
│ 4. Application de messagerie (UI Android)  │  Comptes, conversations, lecteur média, modération
├────────────────────────────────────────────┤
│ 3. Middleware (API locale + Stockage SQL)  │  send_*, get_*, URI, historique, suivi ACK, cache
├────────────────────────────────────────────┤
│ 2. Protocole maillage (DTN & Routage)      │  Annuaire CRDT, routage UserID, fragmentation 16b
├────────────────────────────────────────────┤
│ 1. Transport Bluetooth (BLE)               │  Advertising/Scan, GATT (contrôle), L2CAP CoC (data)
└────────────────────────────────────────────┘
```

L'application n'a aucune dépendance envers le protocole maillage ou le Bluetooth : elle interagit exclusivement avec l'API du middleware.

---

## 2. Définitions, identifiants et limites

| Élément | Règle et format |
|---|---|
| **Nœud** | Appareil Android exécutant le service. `NodeID` = entier 8 bits (`1–254`). `0x00` = non attribué, `0xFF` = broadcast lien. **Max 254 nœuds actifs.** |
| **Utilisateur** | Identifié par `UserID` entier 10 bits (`0–1023`), **max 1024 utilisateurs**, unique dans le réseau. |
| **Admin** | `UserID = 0`, unique, racine de confiance cryptographique du réseau, modérateur suprême. |
| **Modérateur** | Utilisateur désigné par l'admin (certificat de délégation). Ajoute/supprime des utilisateurs de façon **autonome** et peut lire/rapatrier tous les messages (y compris privés `UC`/`MC`). |
| **Session** | Un nœud peut héberger $N$ comptes locaux mais **un seul utilisateur est actif/connecté à la fois** par nœud. Si un utilisateur se connecte sur un second nœud, il **prend la main** et un *merge* d'état s'effectue (§5.4). |
| **MsgUID** | Identifiant **global** et immuable d'un message : tuple `(SenderUserID: 10b, OriginNodeID: 8b, UserMsgSeq: 32b)` (codé sur 64 bits). |
| **IDM** | Identifiant de message **local** au nœud : compteur 32 bits auto-incrémenté en base SQLite locale. |
| **Diffusion** | `UNICAST` (1→1 UserID), `MULTICAST` (1→N UserID), `BROADCAST` (1→tous les UserID), codes `UC`, `MC`, `BC`. |

---

## 3. Transport Bluetooth (Couche 1)

* **BLE** : Utilisation conjointe de GATT (signalisation légère) et **L2CAP Connection-Oriented Channels (CoC)** (transfert rapide de trames et médias par flux).
* **Advertising** : Balise périodique contenant :
  * `NetworkID` (hash 4 octets du nom du réseau + clé publique Admin),
  * `NodeID` (1 octet),
  * `Ver` (4 bits) + `Flags` (4 bits : session ouverte, stockage DTN disponible),
  * `dirHash` (hash court 4 octets de l'état de l'annuaire local),
  * `PSM` L2CAP (2 octets, port dynamique du canal L2CAP CoC).
  * *Total payload manufacturier : 12 octets* (compatible BLE Legacy Advertising 31 octets et BLE 5 Extended Advertising).
* **Scan & Connexion** :
  * Duty-cycle adaptatif (scan accru lors d'une activité utilisateur ou d'une file d'envoi non vide, réduit en veille).
  * Limite matérielle de connexions simultanées : typiquement **5 connexions concurrentes max** gérées en pool tournant (round-robin sur les voisins).
* **Exécution en arrière-plan** : Service Android de premier plan (*Foreground Service* de type `connectedDevice`) obligatoire avec notification persistante.

---

## 4. Protocole maillage et adressage (Couche 2)

### 4.1 Séparation En-tête de Trame (Nœud) et En-tête de Message (Utilisateur)

Les destinataires d'un message sont des **`UserID`** (et non des nœuds). Le protocole sépare donc le routage saut-par-saut (niveau Nœud) de l'adressage de bout en bout (niveau Utilisateur).

#### A. En-tête de trame (Liaison saut-par-saut – 8 octets)
```
| Ver(4b) | Type(4b) | Flags(8b) | TTL(8b) | HopSrcNode(8b) | HopDstNode(8b) | FrameLen(16b) | HeaderCRC(16b) |
```
* **Types (`Type`)** : `HELLO` (0x1), `DIR_SYNC` (0x2), `REGISTER` (0x3), `DATA` (0x4), `ACK` (0x5), `SESSION_SYNC` (0x6), `MODO_FETCH` (0x7), `ADMIN` (0x8).
* `HopSrcNode` / `HopDstNode` : `NodeID` du nœud émetteur du saut courant et du voisin immédiat (`0xFF` = diffusion à tous les voisins directs).

#### B. En-tête de message `DATA` (Bout-en-bout Utilisateur)
```
| Scope(2b) | Kind(2b) | Priorité(4b) | SenderUserID(16b) | OriginNodeID(8b) | UserMsgSeq(32b) |
| Timestamp(32b) | FragIdx(16b) | FragCnt(16b) | DstCount(8b) | DstUserIDs(DstCount × 16b) |
| PayloadChiffré (≤ 1024 octets) | SignatureÉmetteur (64 octets, sur FragIdx=0) |
```
* *Note d'alignement* : `SenderUserID` et chaque `DstUserID` sont stockés sur 16 bits (10 bits utiles `0–1023`, 6 bits réservés).
* **`MsgUID` (Identifiant unique global)** : Constitué de `(SenderUserID, OriginNodeID, UserMsgSeq)`. Il garantit l'unicité globale, la déduplication sur tout le maillage et l'estampillage exigé même si l'utilisateur change de nœud.
* **`Scope`** : `01 = UC` (`DstCount = 1`), `10 = MC` (`DstCount = N`, max 32 destinataires par message), `11 = BC` (`DstCount = 0`, destiné à tous les utilisateurs).
* **`Kind`** : `00 = TEXT`, `01 = AUDIO`, `10 = VIDEO`, `11 = SYSTEM/SYNC`.

### 4.2 Dimensionnement de la fragmentation

Pour supporter des vidéos de 5 s et de l'audio de 30 s sans blocage :
* **Taille d'un fragment (`PayloadChiffré`)** : **1024 octets** sur canal L2CAP CoC (repli automatique en sous-paquets GATT de 240 octets au niveau couche 1 si L2CAP échoue).
* **Compteurs `FragIdx` et `FragCnt` sur 16 bits** (jusqu'à 65 535 fragments).
  * *Texte (≤ 1 Ko)* : **1 fragment**.
  * *Audio 30 s (Opus 20 kbps ≈ 75 Ko, max 128 Ko)* : **~75 à 128 fragments** (~1 à 3 s de transfert en BLE 2M PHY).
  * *Vidéo 5 s (H.264/AV1 ~800 kbps ≈ 500 Ko, max 750 Ko)* : **~500 à 750 fragments** (~8 à 15 s de transfert par saut).
* **Reprise sur interruption** : Les fragments sont stockés individuellement en cache DTN par `(MsgUID, FragIdx)`. En cas de coupure radio à 60% d'une vidéo, la rencontre suivante (avec le même nœud ou un autre porteur) ne transmet que les `FragIdx` manquants via un bitmap d'état (`ACK` partiel).

### 4.3 Attribution des identifiants et gestion pragmatique des collisions

Dans un maillage potentiellement partitionné, attribuer séquentiellement le « plus petit ID libre (1, 2, 3...) » provoquerait des collisions systématiques dès que deux zones fonctionnent isolément.

#### Solution retenue : Allocation par hachage déterministe (*Consistent Probing*) + Résolution CRDT
1. **Identité cryptographique immuable** : Chaque nœud et chaque utilisateur possède une paire de clés Ed25519/X25519 générée à la création. Sa clé publique (`PubKey`, 32 octets) est son identité absolue.
2. **Attribution sans conflit probable (`NodeID` et `UserID`)** :
   * Lorsqu'un nœud ou un modérateur attribue un nouvel ID, il ne part **pas** de 1.
   * Il calcule un index de départ pseudo-aléatoire déterministe à partir de la clé publique :
     * Pour un nœud : `startNodeID = (CRC16(NodePubKey) % 254) + 1`
     * Pour un utilisateur : `startUserID = (CRC16(UserPubKey) % 1023) + 1`
   * Il choisit le premier ID libre dans son annuaire local à partir de `startID` (recherche circulaire).
   * *Avantage* : Deux ajouts simultanés dans deux partitions séparées ont moins de **0,1 %** de risque de tomber sur le même `UserID` (et < 0,4 % pour `NodeID`).
3. **Résolution déterministe en cas de collision rare** :
   * Lors d'un `DIR_SYNC` entre deux partitions qui auraient malgré tout attribué le même ID `X` à deux clés publiques différentes $K_A$ et $K_B$ :
   * **Règle de victoire** : L'entrée ayant le couple `(TimestampCréation, SHA256(PubKey))` le plus petit conserve l'ID `X`.
   * **Réattribution automatique** : Tous les nœuds du réseau déplacent de manière déterministe l'entrée perdante vers le prochain slot libre après `X`. Une table d'alias locale `OldID + OriginPubKey → NewID` assure la livraison transparente des messages déjà en vol.

### 4.4 Annuaire distribué (`DIR_SYNC`)

Chaque nœud maintient deux tables synchronisées par anti-entropie (CRDT *Last-Write-Wins* par numéro de séquence logique d'entrée) :
1. **Table des Nœuds (`≤ 254 entrées`)** :
   `NodeID → { NodePubKey, NomAppareil, VersionEntrée, État (ACTIVE | REVOKED), DernierContact, TableVoisins }`
2. **Table des Utilisateurs (`≤ 1024 entrées`)** :
   `UserID → { UserPubKey, Nom, Rôle (ADMIN | MOD | USER), ActiveNodeID, SessionEpoch, État (ACTIVE | REVOKED), CertificatSigné }`

### 4.5 Routage orienté `UserID`

1. **Résolution `UserID → NodeID`** :
   * Pour acheminer un message destiné à `DstUserID`, un nœud consulte sa Table des Utilisateurs pour connaître son `ActiveNodeID` courant.
2. **Acheminement Unicast (`UC`)** :
   * Si `ActiveNodeID` est connu et accessible via la table de routage à vecteur de distance (TTL max 16 sauts), les fragments sont envoyés vers ce nœud.
   * Si l'utilisateur a changé de nœud pendant le transit (`ActiveNodeID` mis à jour), le nœud intermédiaire redirige dynamiquement le message vers le nouveau `ActiveNodeID`.
   * Si aucune route n'existe ou si l'utilisateur est hors-ligne, le message reste en **Store-and-Forward (DTN)** (rétention par défaut : **7 jours**, paramétrable par l'admin à la création du réseau) et est transmis par portage épidémique maîtrisé (*Spray-and-Wait* : distribution à $K=3$ nœuds relais mobiles maximum pour éviter de saturer les 254 nœuds).
3. **Acheminement Multicast (`MC`)** :
   * Le message transporte la liste des `DstUserIDs` restants à livrer. Lorsqu'un nœud livre le message à l'un des destinataires (ou bifurque vers deux routes distinctes), il met à jour la liste des destinataires restants sur la branche concernée.
4. **Acheminement Broadcast (`BC`)** :
   * Inondation contrôlée à tous les nœuds avec déduplication par `MsgUID`.

---

## 5. Sécurité, Utilisateurs, Sessions et Modération

### 5.1 Création et jonction d'un réseau (Provisioning)

À l'installation du nœud (couvrant toutes les couches) :
* **Mode « Créer un réseau » (Admin, `UserID = 0`)** :
  * L'administrateur saisit : le **Nom du réseau**, son **PIN/mot de passe** administrateur, et la **durée de rétention DTN** (défaut : 7 jours).
  * Le nœud génère :
    1. La paire de clés maîtresse Admin (`AdminPubKey`, `AdminPrivKey`),
    2. La **paire de clés de Modération du réseau** (`ModNetPubKey`, `ModNetPrivKey`).
  * `AdminPrivKey` et `ModNetPrivKey` sont chiffrées localement avec une clé dérivée du PIN (Argon2id + Android Keystore).
* **Mode « Rejoindre un réseau » (Modérateur ou Utilisateur)** :
  * L'utilisateur sélectionne le nom du réseau (découvert par scan BLE ou communiqué par l'admin) et définit son **PIN/mot de passe** personnel.
  * Son appareil génère sa paire de clés (`UserPubKey`, `UserPrivKey` chiffrée par son PIN) et émet une requête d'enregistrement vers un nœud du réseau.
  * Un **Admin ou un Modérateur** valide l'inscription (via code de couplage court / validation UI) et signe le certificat de l'utilisateur.

### 5.2 Rôles et autonomie des modérateurs

| Rôle | Droits et capacités cryptographiques |
|---|---|
| **Admin (`UserID = 0`)** | Tous les droits. Seul habilité à promouvoir/révoquer des modérateurs (`add_moderator`, `remove_moderator`). Irrévocable. Aucune procédure de récupération en cas de perte (§10.5). |
| **Modérateur** | **Ajoute et supprime des utilisateurs de façon totalement autonome** sans nécessiter la présence de l'admin. Peut lire tous les messages (`UC`, `MC`, `BC`) et déclencher `modo_fetch(userID)`. |
| **Utilisateur** | Envoie et reçoit des messages (`UC`, `MC`, `BC`). |

* **Certificat d'autonomie Modérateur** : Lorsqu'un admin nomme un modérateur, il signe un certificat `DelegateCert(AdminPrivKey → ModUserPubKey, Role=MOD)`. Tout nœud du réseau accepte un certificat utilisateur s'il est signé soit par `AdminPrivKey`, soit par un `ModUserPubKey` disposant d'un `DelegateCert` valide et non révoqué.

### 5.3 Chiffrement et lecture par les modérateurs (`UC` / `MC`)

Les modérateurs devant pouvoir lire les messages privés (`UC` et `MC`), le chiffrement de bout en bout intègre un **séquestre cryptographique de modération (Key Escrow)** transparent :
1. Chaque message `UC` ou `MC` est chiffré symétriquement en **ChaCha20-Poly1305** (ou AES-256-GCM) avec une clé éphémère unique `MsgKey` (32 octets).
2. L'en-tête du payload contient la clé `MsgKey` chiffrée (via X25519 ECIES) :
   * Une fois pour la clé publique de chaque destinataire (`DstUserPubKey`),
   * **ET une fois pour la clé publique de modération du réseau (`ModNetPubKey`)** (+48 octets d'empreinte fixe seulement, quel que soit le nombre de modérateurs).
3. Lorsqu'un utilisateur est promu Modérateur, l'Admin lui transmet `ModNetPrivKey` chiffrée avec la clé publique de ce modérateur.
4. Ainsi :
   * Les simples utilisateurs et nœuds routeurs ne peuvent **pas** lire les messages privés d'autrui.
   * L'Admin et tous les Modérateurs détiennent `ModNetPrivKey` et peuvent déchiffrer tout message `UC`/`MC` rapatrié via `modo_fetch(userID)`.
   * Les messages `BC` sont chiffrés avec une clé de réseau partagée (`NetworkKey` distribuée dans le certificat de chaque membre inscrit) et signés par l'émetteur.

### 5.4 Rapatriement de modération (`modo_fetch`)

* `modo_fetch(userID)` : Le modérateur diffuse une trame `MODO_FETCH(TargetUserID)` signée.
* Tout nœud détenant en base locale ou en cache DTN des messages **envoyés OU reçus** par `TargetUserID` les transfère (chiffrés) vers le nœud du modérateur demandeur.
* Les messages rapatriés sont déchiffrés grâce à `ModNetPrivKey` et stockés dans un espace isolé `modo/<UserID>/` consultable via l'API de modération.
* Toute requête `MODO_FETCH` est enregistrée dans le journal d'audit synchronisé avec l'Admin.

### 5.5 Gestion de session et prise de main multi-nœuds (*Merge*)

Un utilisateur ne peut être actif que sur un seul nœud à la fois, mais peut changer d'appareil :
1. **Dérivation / Import de compte** : Pour se connecter sur un nœud B où son compte n'existe pas encore, l'utilisateur s'authentifie avec son `(UserID / Nom, PIN)`. Sa paire de clés utilisateur est déterministe (dérivée via `Argon2id(PIN, SaltRéseau + UserID)`) afin de pouvoir reconstruire sa clé privée sur n'importe quel nœud sans transfert manuel de fichier, tandis que le certificat signé correspondant est déjà présent dans l'annuaire distribué (`DIR_SYNC`).
2. **Prise de main (`SessionEpoch`)** :
   * En se connectant sur le nœud B, le nœud B incrémente `SessionEpoch = ancien_epoch + 1`, signe l'annonce `(UserID, ActiveNodeID = B, SessionEpoch)` et la propage.
   * Dès que l'ancien nœud A reçoit cette mise à jour, il ferme la session locale de l'utilisateur.
3. **Synchronisation et Merge d'état (`SESSION_SYNC`)** :
   * Le nœud A génère un paquet système `SESSION_SYNC` destiné au nœud B contenant :
     * La liste des `MsgUID` reçus/envoyés et leur état (`READ` / `UNREAD`),
     * Les messages complets présents sur A que B ne possède pas encore.
   * À réception sur B, le middleware fusionne (union des messages par `MsgUID`, état `READ` = `READ_A OR READ_B`) afin que l'utilisateur retrouve exactement le même état de messagerie.

---

## 6. Middleware – API et Stockage (Couche 3)

Toutes les opérations du middleware s'exécutent dans le contexte de l'utilisateur connecté.

### 6.1 Objets et contraintes de taille

```
Destinataires : {
    mode  : "UC" | "MC" | "BC",
    users : [UserID]                  // 1 ID pour UC, 2..32 IDs pour MC, vide pour BC
}

TextMessage : {
    texte : String UTF-8              // Max 1024 octets
}

AudioMessage : {
    codec    : "opus",
    duree_ms : Int,                   // Max 30 000 ms (30 secondes)
    donnees  : ByteArray / FileURI    // Max 128 Ko (encodé ~20-24 kbps)
}

VideoMessage : {
    codec      : "h264" | "av1",
    duree_ms   : Int,                 // Max 5 000 ms (5 secondes)
    resolution : "360p" | "480p",
    donnees    : ByteArray / FileURI  // Max 750 Ko (compression obligatoire côté émetteur)
}

MsgRef : {
    idm     : Int32,                  // IDM local attribué immédiatement à l'envoi
    msg_uid : String                  // Identifiant global "<SenderUserID>:<OriginNodeID>:<UserMsgSeq>"
}

DeliveryStatus : {
    state     : "QUEUED" | "IN_TRANSIT" | "DELIVERED" | "PARTIAL" | "EXPIRED" | "FAILED",
    acked_by  : [UserID],             // Destinataires ayant envoyé un ACK de bout en bout
    pending   : [UserID]              // Destinataires n'ayant pas encore acquitté
}
```

### 6.2 Primitives de messagerie (Cœur de spécification)

| Primitive | Description |
|---|---|
| `send_text(texte, destinataires) → MsgRef` | Valide la taille (≤ 1024 o), chiffre, fragmente et met en file d'envoi. |
| `send_audio(audio, destinataires) → MsgRef` | Valide la durée (≤ 30 s / ≤ 128 Ko), chiffre, fragmente et met en file. |
| `send_video(video, destinataires) → MsgRef` | Valide la durée (≤ 5 s / ≤ 750 Ko), chiffre, fragmente et met en file. |
| `has_new_message() → bool` | `True` si au moins un message reçu complet a le statut `UNREAD`. |
| `get_new_messages() → [IDM]` | Liste les `IDM` des messages reçus complets non consultés (sans les marquer lus). |
| `get_message(IDM) → URI` | Renvoie l'objet `URI` du message et le marque comme consulté (`READ`). |

### 6.3 Primitives complémentaires (Cycle de vie, Historique et Événements)

Pour permettre à l'interface Android d'afficher les conversations passées, de suivre les accusés de réception et d'éviter le polling actif :

| Primitive | Description |
|---|---|
| `list_messages(filter?) → [IDM]` | Retourne les `IDM` stockés localement selon un filtre optionnel : `{ peerUserID?, scope?, direction: IN\|OUT\|ALL, unreadOnly?, limit, offset }`. Ne modifie pas l'état lu/non-lu. |
| `delete_message(IDM) → bool` | Supprime le message et son fichier média du stockage local de l'utilisateur. |
| `get_delivery_status(msgRef) → DeliveryStatus` | Retourne l'état d'acheminement et la liste des `ACK` reçus pour un message émis. |
| `register_listener(callback)` | Abonne l'UI aux événements asynchrones : `onNewMessage(IDM)`, `onMessageProgress(IDM, receivedFrags, totalFrags)`, `onDeliveryStatusChanged(MsgRef, DeliveryStatus)`. |

### 6.4 Format d'URI et méthodes de résolution

Format canonique (compatible avec le format compact exigé) :
```
bt-msg://<IDM>:<scope>:<UserID_expéditeur>[:<NodeID_origine>]
```
Exemples : `bt-msg://2:BC:290`, `bt-msg://17:UC:5:12`, `bt-msg://4:MC:12:3`.

Méthodes exposées par l'objet `URI` retourné par `get_message(IDM)` :

| Requête sur `uri` | Retour | Description |
|---|---|---|
| `uri.idm()` | `Int32` | Identifiant local `IDM`. |
| `uri.global_uid()` | `String` | `MsgUID` global (`"<SenderUserID>:<OriginNodeID>:<UserMsgSeq>"`). |
| `uri.sender()` | `UserID` | Identifiant de l'utilisateur expéditeur. |
| `uri.recipients()` | `[UserID]` | Liste des destinataires (`[]` si `BC`). |
| `uri.kind()` | `TEXT \| AUDIO \| VIDEO` | Type du contenu. |
| `uri.scope()` | `UC \| MC \| BC` | Mode de diffusion. |
| `uri.timestamp()` | `Long` | Horodatage d'émission (epoch ms). |
| `uri.text()` | `String?` | Texte décodé UTF-8 si `kind() == TEXT`, sinon `null`. |
| `uri.media()` | `String` | Chemin local `content://...` ou `file://...` vers le fichier audio/vidéo déchiffré (ou texte inline si `TEXT`). |
| `uri.status()` | `COMPLETE \| PARTIAL(n/m)` | État de réception des fragments. |

### 6.5 API de session et d'administration (Middleware)

* **Session & Réseau** :
  * `create_network(networkName, adminName, pin, retentionDays = 7) → NodeID`
  * `discover_networks() → [{ networkName, networkId, neighborNodeId }]`
  * `join_network(networkId, userName, pin) → RegistrationStatus`
  * `login(userID, pin) → SessionInfo`
  * `logout() → void`
* **Administration & Modération** :
  * `add_user(nom, userPubKey / codeCouplage) → UserID` *(Admin ou Modérateur)*
  * `remove_user(userID) → void` *(Admin ou Modérateur)*
  * `add_moderator(userID) → void` *(Admin uniquement)*
  * `remove_moderator(userID) → void` *(Admin uniquement)*
  * `modo_fetch(userID) → FetchJobID` *(Admin ou Modérateur)*
  * `get_modo_messages(userID) → [URI]` *(Liste les messages rapatriés dans `modo/<UserID>/`)*
  * `list_users() → [UserInfo]`
  * `list_nodes() → [NodeInfo]`

### 6.6 Codes d'erreurs standardisés

* `NOT_LOGGED_IN` : Aucune session utilisateur active sur le nœud.
* `INVALID_PIN` : Échec de déverrouillage de la clé privée locale.
* `UNKNOWN_USER` : Le `UserID` destinataire n'existe pas ou est révoqué.
* `TOO_LARGE` : Dépassement des limites (Texte > 1 Ko, Audio > 30 s / 128 Ko, Vidéo > 5 s / 750 Ko).
* `QUOTA_EXCEEDED` : Dépassement du quota d'inondation Broadcast média.
* `NO_ROUTE_YET` : Avertissement non bloquant (message accepté et placé en file DTN store-and-forward).
* `PERMISSION_DENIED` : Opération réservée à un Modérateur ou à l'Admin.
* `NETWORK_FULL` : Limite de 254 nœuds ou 1024 utilisateurs atteinte.
* `UNKNOWN_IDM` : Le message demandé n'existe pas ou a été purgé.

---

## 7. Stockage, Quotas et Protection du Maillage

### 7.1 Stockage local et cache DTN
* **Base de données** : SQLite locale chiffrée (SQLCipher) séparant :
  1. Le magasin applicatif des comptes locaux (`messages`, `media_files`, `conversations`),
  2. Le magasin de transit DTN (`dtn_packets`, `dtn_fragments`),
  3. Le magasin d'audit de modération (`modo/<UserID>/`).
* **Politique d'éviction** :
  * Cache DTN : Plafond par défaut à **100 Mo** par nœud. Éviction prioritaire des messages expirés (TTL dépassé), puis des messages `BC` anciens, puis FIFO.
  * Stockage utilisateur : Quota configurable, éviction des médias les plus anciens déjà consultés (`READ`).

### 7.2 Contrôle de congestion sur les médias Broadcast (`BC`)
Une vidéo de 5 s diffusée en `BC` sur 254 nœuds génère un trafic radio important :
* **Limitation stricte au niveau protocole** : Maximum **1 `BC` vidéo toutes les 15 minutes** et **2 `BC` audio toutes les 5 minutes** par `SenderUserID`. Tout dépassement est rejeté par les nœuds voisins (`QUOTA_EXCEEDED`).
* **Priorité d'ordonnancement radio** :
  1. `HELLO`, `DIR_SYNC`, `REGISTER`, `ADMIN`, `ACK` (Priorité Haute – contrôle),
  2. `DATA (TEXT)` et `MODO_FETCH` (Priorité Normale),
  3. `DATA (AUDIO)` (Priorité Basse),
  4. `DATA (VIDEO)` (Priorité Arrière-plan – transmis par salves pour ne jamais bloquer les messages texte).

---

## 8. Récapitulatif des décisions d'architecture (Points tranchés v0.2)

1. **Modération vs Chiffrement** : Les modérateurs **peuvent lire les messages privés (`UC`/`MC`)** grâce au chiffrement multi-destinataires incluant systématiquement la clé publique de modération du réseau (`ModNetPubKey`).
2. **`modo_fetch(userID)`** : Rapatrie l'intégralité des messages **envoyés ET reçus** par `userID` détenus par les nœuds du maillage, stockés sous `modo/<UserID>/`.
3. **Mobilité utilisateur et prise de main** : Se connecter sur un nouveau nœud incrémente le `SessionEpoch` de l'utilisateur, déconnecte l'ancien nœud et déclenche un protocole `SESSION_SYNC` qui fusionne (*merge*) les messages et statuts de lecture entre les deux appareils.
4. **Autonomie des modérateurs** : **Oui**, les modérateurs ajoutent et révoquent des utilisateurs de manière 100 % autonome grâce à un certificat de délégation signé par l'Admin.
5. **Perte de la clé Admin** : **Aucune récupération ni transfert**. L'admin est unique (`UserID = 0`) et irrévocable.
6. **Limites Audio / Vidéo** : **Audio = 30 s max** (≤ 128 Ko, Opus) ; **Vidéo = 5 s max** (≤ 750 Ko, H.264/AV1).
7. **Identifiant unique global de message (`MsgUID`)** : Chaque message est estampillé de façon unique par `(SenderUserID, OriginNodeID, UserMsgSeq)` en plus de l'`IDM` local au récepteur.
8. **Rétention DTN** : **7 jours par défaut**, configurable par l'administrateur lors de la création du réseau (`create_network`).
9. **Authentification** : Par **PIN / mot de passe** fourni à la création/jonction du réseau, servant aussi à dériver et protéger les clés cryptographiques de l'utilisateur.
10. **Cible matérielle** : **Android uniquement**.

---

## 9. Jalons d'implémentation

1. **Jalon 1 – Transport BLE** : Service de premier plan, Advertising compact 12 octets, Scan adaptatif, pool de connexions L2CAP CoC.
2. **Jalon 2 – Annuaire & Cryptographie** : Création/Jonction de réseau, attribution d'ID par hachage déterministe (`Consistent Probing`), synchronisation CRDT `DIR_SYNC`, certificats Admin/Modérateur/Utilisateur.
3. **Jalon 3 – Routage UserID & DTN** : En-têtes Trame/Message séparés, fragmentation 16 bits (1024 octets/fragment), store-and-forward *Spray-and-Wait*, `ACK` de bout en bout et reprise sur bitmap.
4. **Jalon 4 – Middleware & API Texte** : Base SQLite, primitives `send_text`, `get_new_messages`, `get_message`, résolution `URI`, historique et suivi de livraison.
5. **Jalon 5 – Audio, Vidéo & QoS** : Compression Opus et H.264/AV1, transfert fragmenté en arrière-plan, priorisation des files d'attente.
6. **Jalon 6 – Modération & Multi-nœuds** : Chiffrement avec séquestre `ModNetPubKey`, commande `modo_fetch`, prise de main `SessionEpoch` et fusion `SESSION_SYNC`.
7. **Jalon 7 – Simulateur & Validation** : Simulateur réseau de 254 nœuds virtuels (partitions, collisions, perte de paquets) et tests sur flotte Android physique.
