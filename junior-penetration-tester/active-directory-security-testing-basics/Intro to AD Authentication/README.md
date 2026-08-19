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

