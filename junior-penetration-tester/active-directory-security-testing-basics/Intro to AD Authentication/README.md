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



