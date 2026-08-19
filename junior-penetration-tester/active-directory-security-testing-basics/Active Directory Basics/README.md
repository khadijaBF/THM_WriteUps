Active Directory= gestion des utilisateurs, machines, groupes, OU ... d'une maniere qui garantit la centralisation de gestion des identites et qui gere aussi les politiques de securite
DC _ serveur qui gere les composants de l'AD


Voici la traduction complète, sans rien rater.

---

## Domaine parent

Nous allons maintenant travailler dans le domaine parent (ou racine) de notre forêt Active Directory. Il s'agit du domaine TryHackMe (THM.LOC). Connecte-toi au contrôleur de domaine (ROOTDC.THM.LOC) via RDP en utilisant les identifiants suivants :

**Identifiants**

<img width="1062" height="276" alt="image" src="https://github.com/user-attachments/assets/6ec5ccbf-e374-4cc4-badf-e3645e91407c" />



## Active Directory Domain Service

Le cœur de tout domaine Windows est le service **Active Directory Domain Service (AD DS)**. Ce service agit comme un catalogue qui stocke les informations de tous les « objets » existant sur votre réseau. Parmi les nombreux types d'objets pris en charge par AD, on trouve les utilisateurs, les groupes, les machines, les imprimantes, les partages, et bien d'autres. Voyons-en quelques-uns :

### Utilisateurs

Les utilisateurs sont l'un des types d'objets les plus courants dans Active Directory. Les utilisateurs font partie des objets appelés **principaux de sécurité** (security principals), ce qui signifie qu'ils peuvent être authentifiés par le domaine et se voir attribuer des privilèges sur des ressources comme des fichiers ou des imprimantes. On pourrait dire qu'un principal de sécurité est un objet capable d'agir sur les ressources réseau.

Les utilisateurs peuvent représenter deux types d'entités :

- **Des personnes** : les utilisateurs représentent généralement des individus de votre organisation ayant besoin d'accéder au réseau, comme des employés.
- **Des services** : vous pouvez aussi définir des utilisateurs pour des services comme IIS ou MSSQL. Chaque service a besoin d'un utilisateur pour s'exécuter, mais ces utilisateurs de service diffèrent des utilisateurs classiques, car ils ne disposent que des privilèges nécessaires à l'exécution de leur service spécifique.

### Machines

Les machines sont un autre type d'objet dans Active Directory ; pour chaque ordinateur qui rejoint le domaine Active Directory, un objet machine est créé. Les machines sont également considérées comme des « principaux de sécurité » et se voient attribuer un compte, tout comme n'importe quel utilisateur classique. Ce compte dispose de droits quelque peu limités au sein du domaine lui-même.

Les comptes machines eux-mêmes sont administrateurs locaux sur l'ordinateur auquel ils sont assignés ; ils ne sont généralement pas censés être utilisés par qui que ce soit d'autre que l'ordinateur lui-même, mais comme pour tout autre compte, si vous possédez le mot de passe, vous pouvez l'utiliser pour vous connecter.

*Remarque : les mots de passe des comptes machines sont automatiquement renouvelés (rotation) et se composent généralement de 120 caractères aléatoires.*

Identifier les comptes machines est relativement facile. Ils suivent un schéma de nommage spécifique. Le nom du compte machine correspond au nom de l'ordinateur suivi d'un symbole dollar. Par exemple, une machine nommée ROOTDC aura un compte machine appelé `ROOTDC$`.

### Groupes de sécurité

Si vous êtes familier avec Windows, vous savez probablement que vous pouvez définir des groupes d'utilisateurs pour attribuer des droits d'accès à des fichiers ou d'autres ressources à des groupes entiers plutôt qu'à des utilisateurs individuels. Cela permet une meilleure gestion, car vous pouvez ajouter des utilisateurs à un groupe existant, et ils hériteront automatiquement de tous les privilèges du groupe. Les groupes de sécurité sont eux aussi considérés comme des principaux de sécurité et peuvent donc disposer de privilèges sur les ressources réseau.

Les groupes peuvent contenir à la fois des utilisateurs et des machines comme membres. Si nécessaire, les groupes peuvent également inclure d'autres groupes.

Plusieurs groupes sont créés par défaut dans un domaine et peuvent être utilisés pour accorder des privilèges spécifiques aux utilisateurs. Voici, à titre d'exemple, quelques-uns des groupes les plus importants d'un domaine :

| Groupe de sécurité | Description |
|---|---|
| **Domain Admins** | Les utilisateurs de ce groupe disposent de privilèges administratifs sur l'ensemble du domaine. Par défaut, ils peuvent administrer n'importe quel ordinateur du domaine, y compris les contrôleurs de domaine. |
| **Server Operators** | Les utilisateurs de ce groupe peuvent administrer les contrôleurs de domaine. Ils ne peuvent pas modifier l'appartenance à des groupes administratifs. |
| **Backup Operators** | Les utilisateurs de ce groupe sont autorisés à accéder à n'importe quel fichier, en ignorant leurs permissions. Ils sont utilisés pour effectuer des sauvegardes de données sur les ordinateurs. |
| **Account Operators** | Les utilisateurs de ce groupe peuvent créer ou modifier d'autres comptes dans le domaine. |
| **Domain Users** | Inclut tous les comptes utilisateurs existants dans le domaine. |
| **Domain Computers** | Inclut tous les ordinateurs existants dans le domaine. |
| **Domain Controllers** | Inclut tous les contrôleurs de domaine existants dans le domaine. |

Vous pouvez obtenir la liste complète des groupes de sécurité par défaut dans la documentation Microsoft.

## Active Directory Users and Computers

Pour configurer les utilisateurs, groupes ou machines dans Active Directory, nous devons nous connecter au contrôleur de domaine et exécuter « Active Directory Users and Computers » depuis le menu Démarrer.
<img width="415" height="675" alt="image" src="https://github.com/user-attachments/assets/d022ddeb-57ef-428b-9f9e-3913053bdef9" />


Cela ouvrira une fenêtre montrant la hiérarchie des utilisateurs, ordinateurs et groupes du domaine. Ces objets sont organisés en **unités d'organisation** (Organizational Units, OU), qui sont des objets conteneurs permettant de classer les utilisateurs et les machines. Les OU servent principalement à définir des ensembles d'utilisateurs ayant des exigences de gestion des politiques similaires. Par exemple, les personnes du service commercial de votre organisation sont susceptibles d'avoir un ensemble de politiques différent de celles du service informatique. Gardez à l'esprit qu'un utilisateur ne peut appartenir qu'à une seule OU à la fois.

En examinant notre machine, on peut voir qu'il existe déjà une OU appelée THM avec cinq OU enfants pour les départements IT, Management, Marketing, R&D, et Sales. Il est très courant de voir les OU refléter la structure de l'entreprise, car cela permet de déployer efficacement des politiques de base s'appliquant à des départements entiers. Rappelez-vous que même si ce serait le modèle attendu la plupart du temps, vous pouvez définir des OU de manière arbitraire. N'hésitez pas à créer une nouvelle OU appelée Students pour le plaisir, en faisant un clic droit sur l'OU THM, puis en sélectionnant Nouveau, puis Unité d'organisation.
<img width="915" height="451" alt="image" src="https://github.com/user-attachments/assets/4f4a6ffd-38e2-4aa9-8b3c-951ebe9d1c64" />


Si vous ouvrez n'importe quelle OU, vous pouvez voir les utilisateurs qu'elle contient et effectuer des tâches simples comme les créer, les supprimer, ou les modifier selon les besoins. Vous pouvez également réinitialiser les mots de passe si nécessaire (assez utile pour le support technique/helpdesk) :
<img width="913" height="451" alt="image" src="https://github.com/user-attachments/assets/3757c291-a843-4da2-961e-c79debcf93f0" />


Vous avez probablement déjà remarqué qu'il existe d'autres conteneurs par défaut en plus de l'OU THM. Ces conteneurs sont créés automatiquement par Windows et contiennent les éléments suivants :

- **Builtin** : contient les groupes par défaut disponibles sur n'importe quel hôte Windows.
- **Computers** : toute machine rejoignant le réseau sera placée ici par défaut. Vous pouvez les déplacer si nécessaire.
- **Domain Controllers** : OU par défaut contenant les contrôleurs de domaine de votre réseau.
- **Users** : utilisateurs et groupes par défaut s'appliquant à un contexte global du domaine.
- **Managed Service Accounts** : contient les comptes utilisés par les services de votre domaine Windows.

## Groupes de sécurité vs OU

Vous vous demandez probablement pourquoi on dispose à la fois de groupes et d'OU. Bien que les deux servent à classer les utilisateurs et les ordinateurs, leurs objectifs sont complètement différents :

- Les **OU** sont pratiques pour appliquer des politiques aux utilisateurs et ordinateurs, y compris des configurations spécifiques s'appliquant à des ensembles d'utilisateurs en fonction de leur rôle dans l'entreprise. Rappelez-vous qu'un utilisateur ne peut être membre que d'une seule OU à la fois, car il ne serait pas logique d'appliquer deux ensembles de politiques différents à un même utilisateur.
- Les **groupes de sécurité**, en revanche, servent à accorder des permissions sur des ressources. Par exemple, vous pouvez utiliser des groupes pour permettre à certains utilisateurs d'accéder à un dossier partagé ou à une imprimante réseau. Un utilisateur peut faire partie de nombreux groupes, ce qui est nécessaire pour accorder l'accès à plusieurs ressources.

TASK5: Trees, Forests and Trusts


Le problème de base

Jusqu'ici on a vu un seul domaine. Mais quand une entreprise grandit, avoir un seul gros domaine devient compliqué à gérer.

Trees (Arbres)

Exemple concret : imagine que TryHackMe se lance dans le secteur bancaire. Plutôt que de tout entasser dans le même domaine existant, on crée un sous-domaine dédié à cette nouvelle activité.

thm.loc = le domaine parent (l'entreprise principale)
tbm.thm.loc = le nouveau sous-domaine (la division bancaire)
<img width="1218" height="963" alt="image" src="https://github.com/user-attachments/assets/460d93ab-7578-4418-8912-ee5508e0fbdc" />


Pourquoi faire ça plutôt qu'un seul gros domaine ? Chaque division a sa propre équipe IT qui gère ses propres ressources. Un seul énorme AD serait difficile à gérer et plus sujet aux erreurs de configuration et aux failles de sécurité.

Analogie simple : c'est comme une grande entreprise avec plusieurs filiales — chaque filiale a son propre service informatique qui gère ses propres employés et équipements, plutôt qu'un seul service IT géant qui gère tout le monde en même temps.

On peut aller encore plus loin en créant des sous-domaines régionaux, par exemple us.tbm.thm.loc et uk.tbm.thm.loc, chacun avec son propre AD, ses propres ordinateurs et utilisateurs.

L'avantage clé : un meilleur contrôle de qui peut accéder à quoi. Par exemple, un employé de la division bancaire ne peut pas modifier les permissions du domaine parent — c'est cloisonné.

Nouveau groupe important : Enterprise Admins

Domain Admins = privilèges d'administrateur sur un seul domaine (comme on l'a vu avant).
Enterprise Admins = privilèges d'administrateur sur tous les domaines de l'entreprise, à travers tout l'arbre (tree).



Forests (Forêts)

Les domaines qu'on gère peuvent aussi être dans des espaces de noms totalement différents — pas juste des sous-domaines les uns des autres.

Exemple concret : l'entreprise continue de grandir et doit maintenant s'intégrer avec un nouveau fournisseur, "TryVendorMe". Les deux entreprises ont chacune leur propre domaine AD (leur propre "tree" complet), mais elles doivent parfois partager des ressources entre elles. Quand on relie plusieurs "trees" indépendants (avec des noms différents) ensemble, on obtient une forêt (forest).
<img width="963" height="386" alt="image" src="https://github.com/user-attachments/assets/f88673db-9b18-46a9-87d1-e4a83241a43e" />


Analogie simple : un "tree", c'est une entreprise avec ses filiales internes. Une "forest", c'est un partenariat entre plusieurs entreprises différentes, chacune gardant sa propre structure interne, mais reliées entre elles pour collaborer.

Trust Relationships (Relations de confiance)

Le problème : un employé de TryVendorMe doit parfois accéder à des ressources chez TryBankMe (par exemple pour du support technique). Comment permettre ça entre deux domaines complètement séparés ?

La réponse : les relations de confiance (trusts), qui relient les domaines organisés en trees et forests.

Types de confiance :

Confiance à sens unique (one-way trust) :
Si Domaine AAA fait confiance à Domaine BBB, cela signifie qu'un utilisateur de BBB peut être autorisé à accéder aux ressources de AAA.
Point important à bien retenir : la direction de la confiance est l'inverse de la direction d'accès. AAA "fait confiance" à BBB, mais c'est BBB qui peut accéder aux ressources de AAA (pas l'inverse).

Confiance à double sens (two-way trust) :
Les deux domaines s'autorisent mutuellement l'accès à leurs ressources respectives.
Par défaut, quand plusieurs domaines rejoignent un même tree ou forest, une confiance à double sens est automatiquement créée entre eux.

Point très important à ne pas oublier :

Une relation de confiance entre deux domaines ne donne pas automatiquement accès à toutes les ressources de l'autre domaine. Ça ouvre juste la possibilité d'autoriser des utilisateurs à travers les domaines — mais c'est ensuite à l'administrateur de décider précisément quoi autoriser et quoi refuser.

Analogie finale : avoir une relation de confiance, c'est comme avoir un accord de passage entre deux pays voisins — ça ne veut pas dire que n'importe quel citoyen peut aller n'importe où sans visa ; ça veut juste dire qu'un cadre existe pour autoriser certains passages spécifiques, que chaque pays configure à sa manière.



Task 6 — Créer un nouveau domaine
Domaines enfants (Child Domains)

Comme vu dans la tâche précédente, l'objectif est de créer un nouveau "tree" dans la forêt pour la division bancaire. On a déjà le domaine principal TryHackMe (thm.loc), et on veut maintenant créer une filiale, TryBankMe (TBM). Ce nouveau domaine enfant s'appellera tbm.thm.loc.

Première étape : se connecter en RDP au serveur qui va devenir le nouveau contrôleur de domaine (avec les identifiants Administrateur fournis, à l'adresse 192.168.10.110).
<img width="888" height="223" alt="image" src="https://github.com/user-attachments/assets/b8e12368-3ef7-41e9-b7c1-e8cb64004774" />


Promotion du domaine (Domain Promotion)

Pour transformer ce serveur en contrôleur de domaine, on utilise un script PowerShell (C:\install-domain.ps1). Le document explique ensuite ce que fait ce script, étape par étape :

```powershell
#DNS
$parentdcIPAddress = "192.168.10.100"

echo 'Configuring DNS...'
$adapters = Get-WmiObject Win32_NetworkAdapterConfiguration
if ($adapters) {
    $adapters | ForEach-Object {$_.SetDNSServerSearchOrder($parentdcIPAddress)}
}

#Password Reset
$domainAdministratorPassword = "childdomainftw1!"
$domainSafeModeAdministratorPassword = "childdomainftw1!"

echo 'Resetting the Administrator account password and settings...'
$localAdminPassword = ConvertTo-SecureString $domainAdministratorPassword -AsPlainText -Force
Set-LocalUser `
    -Name Administrator `
    -AccountNeverExpires `
    -Password $localAdminPassword `
    -PasswordNeverExpires:$true `
    -UserMayChangePassword:$true

$safeModePassword = ConvertTo-SecureString $domainSafeModeAdministratorPassword -AsPlainText -Force

#Parent Domain Credentials
$parentAdministratorPassword = "learningadisfun1!"
$parentName = "THM.LOC"

$parentPassword = ConvertTo-SecureString $parentAdministratorPassword -AsPlainText -Force
$parentDA =  $parentName + "\Administrator" 

echo 'Configuring the parent domain credentials'
$parentCredentials = New-Object System.Management.Automation.PSCredential($parentDA, $parentPassword)

#Tool Installation
echo 'Installing the AD services and administration tools...'
#Install-WindowsFeature AD-Domain-Services,RSAT-AD-AdminCenter,RSAT-ADDS-Tools

#Promoting to DC
$domainName = "tbm"
$domainNetbiosName = "TBM"
$parentFqdn = "thm.loc"

echo 'Installing the AD domain (be patient, this will take more than 5m to install)...'

Import-Module ADDSDeployment
Install-ADDSDomain `
    -Credential $parentCredentials `
    -NewDomainName $domainName `
    -SafeModeAdministratorPassword $safeModePassword `
    -CreateDnsDelegation:$true `
    -DatabasePath "C:\Windows\NTDS" `
    -DomainMode "6" `
    -NewDomainNetbiosName $domainNetbiosName `
    -InstallDns:$true `
    -NoRebootOnCompletion:$true `
    -Force:$true `
    -ParentDomainName $parentFqdn

echo 'Promotion complete! Restart required!'
```

1. DNS

Ce nouveau serveur doit pouvoir communiquer avec le contrôleur de domaine existant (le parent, ROOTDC). Pour ça, on configure l'IP de ROOTDC (192.168.10.100) comme serveur DNS de ce nouveau serveur — un peu comme donner l'adresse du "bureau central" pour que le nouveau serveur sache où chercher les informations du domaine parent.

Analogie simple : avant de devenir toi-même une filiale reconnue, tu as besoin de connaître l'adresse du siège social pour t'enregistrer auprès de lui.

2. Réinitialisation du mot de passe (Password Reset)

Quand on promeut ce serveur en contrôleur de domaine, son compte Administrateur local actuel devient le compte Administrateur du nouveau domaine. Il faut donc définir ce mot de passe avant la promotion, puisque ce même compte va continuer d'exister, mais avec un rôle différent (administrateur de tout un domaine, plutôt que juste de cette machine).

3. Identifiants du domaine parent (Parent Domain Credentials)

Pour pouvoir créer un domaine enfant sous thm.loc, il faut des privilèges provenant du domaine parent lui-même — on ne peut pas juste décider unilatéralement de devenir un sous-domaine, il faut l'autorisation du domaine parent. Le script crée donc un objet d'identifiants (PSCredential) avec le compte Administrateur du domaine parent (THM.LOC\Administrator), pour prouver cette autorisation pendant l'installation.

Analogie simple : pour ouvrir une filiale officielle sous le nom d'une grande entreprise, il faut l'accord signé de la direction générale — pas juste ta propre décision.

4. Installation des outils (Tool Installation)

Normalement, il faut installer des rôles et fonctionnalités Windows supplémentaires (les services AD) pour pouvoir promouvoir un serveur en DC. Ici, cette étape est déjà faite à l'avance dans le lab (la commande est mise en commentaire, donc elle ne s'exécute pas).

5. Promotion en DC (Promoting to DC)

Une fois toutes les étapes précédentes prêtes, le script exécute la vraie promotion avec Install-ADDSDomain, en lui donnant :

les identifiants du domaine parent (pour l'autorisation),
le nom du nouveau domaine (tbm),
le nom NetBIOS (TBM),
et le nom du domaine parent complet (thm.loc), pour bien indiquer que c'est un domaine enfant de celui-ci, pas un domaine indépendant.
Exécution du script

On lance simplement le script depuis PowerShell :

C:\install-domain.ps1
Redémarrage

Une fois le script terminé, il faut redémarrer le serveur pour que la promotion en contrôleur de domaine prenne pleinement effet :

Restart-Computer -Force

Remarque importante : comme ce serveur vient tout juste d'être promu, la configuration met un peu de temps à se mettre en place — le document recommande d'attendre au moins 5 minutes après le redémarrage avant de tenter de se reconnecter.

Nouveaux identifiants

Maintenant que le nouveau domaine et son contrôleur existent, les identifiants de connexion changent — ce n'est plus le mot de passe original du serveur, mais celui défini pendant le script (childdomainftw1!), et on se connecte maintenant en précisant le nouveau domaine TBM.THM.LOC :

<img width="858" height="217" alt="image" src="https://github.com/user-attachments/assets/7581f3a1-2653-498b-a42d-b2d19f28adfc" />


Dernier point : même après la reconnexion, le contrôleur de domaine peut encore être en train d'appliquer les stratégies (Group Policies) pour les ordinateurs et utilisateurs — il faut encore patienter environ 5 minutes avant que tout soit pleinement opérationnel. Une fois prêt, un script automatique aura déjà pré-rempli le domaine avec des objets qu'on va ensuite pouvoir explorer.



## Task 7 — Gérer les utilisateurs dans AD

Ta première mission en tant que nouvel administrateur de domaine pour l'organisation TryBankMe est de passer en revue les OU et utilisateurs existants dans AD, suite à des changements récents survenus dans l'entreprise. Tout comme tu l'as fait sur le contrôleur de domaine parent, ouvre le panneau **Active Directory Users and Computers**, et mettons-nous au travail ! Lorsque tu as reçu ton flag dans la tâche précédente, le domaine aurait dû être automatiquement peuplé pour toi. Quand tu ouvres Active Directory Users and Computers, ça devrait maintenant ressembler à ceci.
<img width="560" height="353" alt="image" src="https://github.com/user-attachments/assets/a04374d9-ca8f-4899-8b10-adf02213bed0" />

Avant de continuer, active les **Advanced Features** (Fonctionnalités avancées) dans le menu View (Affichage).
<img width="754" height="417" alt="image" src="https://github.com/user-attachments/assets/366bdcfa-dc6f-4442-9a79-7696c6fc9c12" />

with no advanced features: <img width="744" height="515" alt="image" src="https://github.com/user-attachments/assets/7135836e-7be1-4fc1-a905-37103b4b6fbd" />
with advanced feature: <img width="745" height="505" alt="image" src="https://github.com/user-attachments/assets/fdd8fffe-0ddf-4f3d-9960-0790e6bb33d4" />

Cela affichera des conteneurs supplémentaires et activera davantage d'options, comme la possibilité de supprimer des objets. Familiarise-toi avec la structure de ce domaine en examinant de plus près les OU **Groups** et **People**.


### Délégation

L'un des avantages d'AD est de pouvoir donner à des utilisateurs spécifiques un certain contrôle sur certaines OU. Ce processus, appelé **délégation**, permet d'accorder à des utilisateurs des privilèges spécifiques pour effectuer des tâches avancées sur des OU, sans qu'un administrateur de domaine ait besoin d'intervenir.

L'un des cas d'usage les plus courants consiste à accorder au **support IT** le privilège de réinitialiser les mots de passe d'autres utilisateurs à faibles privilèges. Dans le domaine TryBankMe, les membres du groupe **Tech Support** fournissent un support IT aux Bankers, donc on souhaite déléguer le contrôle de réinitialisation des mots de passe pour l'OU **Bankers** à tout membre de ce groupe.

Pour cet exemple, on va déléguer le contrôle de l'OU **Bankers** au groupe **Tech Support**. Pour déléguer le contrôle d'une OU, tu peux faire un clic droit dessus et sélectionner **Delegate Control** (Déléguer le contrôle).
<img width="247" height="115" alt="image" src="https://github.com/user-attachments/assets/6e45460a-1587-492e-831b-8c7d1fd19c09" />


Cela devrait ouvrir une nouvelle fenêtre où on te demandera d'abord à quels utilisateurs tu souhaites déléguer le contrôle.

*Remarque : pour éviter de mal orthographier le nom du groupe, tape "tech" puis clique sur le bouton Check Names (Vérifier les noms). Windows complétera automatiquement l'utilisateur pour toi.*
<img width="719" height="586" alt="image" src="https://github.com/user-attachments/assets/bd7f270d-39bb-4f04-9227-b3bcccba9fe5" />


Clique sur **OK**, puis à l'étape suivante, sélectionne l'option **Reset user passwords and force password change at next logon** (Réinitialiser les mots de passe des utilisateurs et forcer le changement de mot de passe à la prochaine connexion).
<img width="531" height="414" alt="image" src="https://github.com/user-attachments/assets/5a33c8ca-12e1-4e96-a641-15c59c55a3ee" />


Clique quelques fois sur **Next** (Suivant), puis sur **Finish** (Terminer). Désormais, tout utilisateur du groupe **Tech Support** devrait pouvoir réinitialiser les mots de passe de n'importe quel utilisateur du département bancaire. Il existe diverses permissions qui peuvent être déléguées. N'hésite pas à explorer davantage ces options. Tu peux même créer une tâche personnalisée à déléguer, ce qui permet un contrôle extrêmement granulaire sur la configuration de la délégation et sur les objets auxquels elle s'applique au sein de l'OU.

Attention : 
<img width="1020" height="513" alt="image" src="https://github.com/user-attachments/assets/f09f32e3-005d-4dc9-8856-c68f4195237e" />
<img width="1021" height="366" alt="image" src="https://github.com/user-attachments/assets/06a3d6b3-7d8f-49fa-95be-50f01d6f6553" />



## Task 8 — Joindre un ordinateur au domaine

### Ordinateurs du domaine

TryBankMe a acheté un nouveau serveur (Server1) et souhaite le joindre au domaine. Une fois qu'une machine est jointe au domaine, les services Active Directory peuvent être utilisés pour gérer cette machine et fournir un accès à celle-ci. En gardant ta session RDP ouverte sur le contrôleur de domaine TBM, créons une nouvelle session RDP vers ce serveur :


<img width="819" height="214" alt="image" src="https://github.com/user-attachments/assets/b33ec847-98ab-4037-b5cf-a1a7f9010dda" />


Une fois de plus, nous allons utiliser un script situé dans `C:\join-domain.ps1` pour joindre ce serveur à notre domaine, comme montré ci-dessous :

```powershell
#DNS
$dcIPAddress = "192.168.10.110"
echo "Pointing DNS"
$adapters = Get-WmiObject Win32_NetworkAdapterConfiguration
if ($adapters) {
    $adapters | ForEach-Object {$_.SetDNSServerSearchOrder($dcIPAddress)}
}
#Credentials
$domainAdministratorPassword = "childdomainftw1!"
$domainNetbiosName = "tbm.thm.loc"
$securePassword = ConvertTo-SecureString $domainAdministratorPassword -AsPlainText -Force
$username = $domainNetbiosName + "\Administrator" 
$domainAdminCredentials = New-Object System.Management.Automation.PSCredential($username, $securePassword)
#Domain Joining
echo "Joining computer"
Add-Computer -DomainName $domainNetbiosName -Credential $domainAdminCredentials
echo "Computer Joined"
```

Passons en revue les trois étapes principales du script :

- **DNS** : de manière similaire à la promotion du serveur en contrôleur de domaine, nous devons indiquer au serveur où trouver le contrôleur de domaine. En pointant son DNS vers notre nouveau DC, il pourra établir le contact avec notre domaine.
- **Credentials (Identifiants)** : tout le monde n'est pas autorisé à intégrer ce nouveau serveur. Si n'importe qui pouvait intégrer une machine à votre domaine, ce serait un risque de sécurité majeur. Nous devons donc fournir au script les identifiants d'un utilisateur AD ayant la permission de joindre cet ordinateur au domaine. Il ne s'agit pas nécessairement de l'utilisateur Administrateur, mais c'est généralement un compte utilisateur assez privilégié qui est utilisé.
- **Domain Joining (Jonction au domaine)** : en utilisant les identifiants fournis, nous pouvons demander au serveur de contacter le contrôleur de domaine et de rejoindre le domaine.

Maintenant que nous comprenons ce que fait le script, exécutons-le via PowerShell :

```
PS C:\Users\Administrator> C:\join-domain.ps1
```

Une fois le script terminé, nous devons redémarrer le serveur pour finaliser le processus, comme montré ci-dessous :

```
PS C:\Users\Administrator> Restart-Computer -Force
```

Puisque l'hôte a désormais été joint au domaine AD, nous n'avons plus besoin d'utiliser le compte local pour l'authentification. Nous pouvons à la place utiliser les identifiants de notre domaine :

<img width="827" height="204" alt="image" src="https://github.com/user-attachments/assets/721368e3-07a3-4023-bc0e-7c80407b9374" />


Dans la prochaine tâche, nous verrons comment gérer ces ordinateurs dans AD.



## Task 9 — Gérer les ordinateurs dans AD

Retournons au contrôleur de domaine de TryBankMe. Par défaut, toutes les machines qui rejoignent un domaine (à l'exception des contrôleurs de domaine) sont placées dans le conteneur appelé **Computers**. Si nous vérifions notre DC, nous verrons que certains appareils s'y trouvent déjà :
<img width="490" height="448" alt="image" src="https://github.com/user-attachments/assets/849b9f5f-5c85-411f-b325-e63bb6546415" />

Nous pouvons voir des serveurs ainsi que des ordinateurs portables correspondant aux utilisateurs de notre réseau. Avoir tous nos appareils au même endroit n'est pas la meilleure idée, car il est probable que vous souhaitiez appliquer des politiques différentes à vos serveurs et aux machines utilisées quotidiennement par les utilisateurs classiques.

Bien qu'il n'existe pas de règle absolue pour organiser vos machines, un excellent point de départ consiste à séparer les appareils selon leur usage. En général, on s'attend à voir les appareils répartis en au moins les trois catégories suivantes :

### Workstations (Postes de travail)

Les postes de travail sont l'un des types d'appareils les plus courants dans un domaine Active Directory. Chaque utilisateur du domaine se connectera probablement à un poste de travail. C'est l'appareil qu'il utilisera pour son travail ou pour la navigation courante. Ces appareils ne devraient jamais avoir d'utilisateur privilégié connecté dessus.

### Servers (Serveurs)

Les serveurs sont le deuxième type d'appareil le plus courant dans un domaine Active Directory. Les serveurs sont généralement utilisés pour fournir des services aux utilisateurs ou à d'autres serveurs.

### Domain Controllers (Contrôleurs de domaine)

Les contrôleurs de domaine sont le troisième type d'appareil le plus courant dans un domaine Active Directory. Les contrôleurs de domaine permettent de gérer le domaine Active Directory. Ces appareils sont souvent considérés comme les appareils les plus sensibles du réseau, car ils stockent les mots de passe hachés de tous les comptes utilisateurs de l'environnement.

Puisque nous sommes en train de faire le ménage dans notre AD, utilisons des OU distinctes pour les Workstations et les Servers (les Domain Controllers ont déjà leur propre OU créée par Windows). Fais glisser-déposer les différents serveurs depuis l'OU **Computers** vers l'OU **Servers**, jusqu'à obtenir cette structure :
<img width="487" height="347" alt="image" src="https://github.com/user-attachments/assets/8385dfd6-cf2c-4f09-8a27-10054c4e5749" />

Maintenant, déplace les ordinateurs portables vers l'OU **Workstations**. Faire cela nous permettra de configurer des politiques pour chaque OU par la suite.





## Task 10 — Group Policies (Stratégies de groupe)

Jusqu'à présent, nous avons organisé les utilisateurs et ordinateurs en OU juste pour le principe, mais l'idée principale est de pouvoir déployer différentes politiques à chaque OU individuellement. De cette façon, nous pouvons pousser différentes configurations et bases de référence de sécurité aux utilisateurs selon leur département.

Windows gère ce type de politiques via les **Group Policy Objects (GPO)**. Les GPO sont simplement un ensemble de paramètres pouvant être appliqués aux OU. Les GPO peuvent contenir des politiques destinées soit aux utilisateurs, soit aux ordinateurs, ce qui permet de définir une base de référence sur des machines et des identités spécifiques.

Pour configurer les GPO, tu peux utiliser l'outil **Group Policy Management**, disponible depuis le menu Démarrer.

La première chose que tu verras en l'ouvrant est ta hiérarchie complète d'OU, telle que définie précédemment. Pour configurer des stratégies de groupe, tu crées d'abord une GPO sous **Group Policy Objects**, puis tu la lies à l'OU où tu souhaites que les politiques s'appliquent. Par exemple, tu peux voir qu'il existe déjà des GPO sur ta machine.

L'image ci-dessus montre que 2 GPO ont été créées. Parmi celles-ci, la **Default Domain Policy** est liée au domaine `tbm.thm.loc` dans son ensemble, et la **Default Domain Controllers Policy** est liée uniquement à l'OU **Domain Controllers**. Un point important à garder à l'esprit : toute GPO s'appliquera à l'OU à laquelle elle est liée, ainsi qu'à toutes les sous-OU en dessous. Par exemple, l'OU **Bankers** sera quand même affectée par la Default Domain Policy.

Examinons la **Default Domain Policy** pour voir ce que contient une GPO. Le premier onglet que tu verras en sélectionnant une GPO affiche sa **portée (scope)**, c'est-à-dire l'endroit où la GPO est liée dans l'AD. Pour la politique actuelle, on peut voir qu'elle n'a été liée qu'au domaine `tbm.thm.loc`.

Comme tu peux le voir, tu peux aussi appliquer un **filtrage de sécurité (Security Filtering)** aux GPO, afin qu'elles ne s'appliquent qu'à des utilisateurs/ordinateurs spécifiques au sein d'une OU. Par défaut, elles s'appliquent au groupe **Authenticated Users**, qui inclut tous les utilisateurs/PC.

L'onglet **Settings** contient le contenu réel de la GPO et nous montre les configurations spécifiques qu'elle applique. Comme mentionné précédemment, chaque GPO possède des configurations qui s'appliquent uniquement aux ordinateurs, et d'autres uniquement aux utilisateurs.

### Création d'administrateurs locaux

TryBankMe souhaite s'assurer que les membres du groupe **Product Admins** puissent effectuer des tâches administratives sur tous les serveurs. Cela signifie que nous devons nous assurer que le groupe Product Admins est ajouté au groupe **Administrators** local sur tous les serveurs de l'OU **Servers**. Créons une nouvelle GPO pour appliquer cette configuration. Fais un clic droit sur l'OU **Servers**, puis clique sur **Create a GPO in this domain, and Link it here**.

Donne un nom à la GPO, clique sur OK, puis fais un clic droit sur la GPO nouvellement créée pour l'**Edit** (Modifier).

Le changement que nous voulons effectuer doit s'appliquer aux objets **Computer**. Nous devons donc modifier la **Computer Configuration** dans la GPO. Navigue vers **Computer Configuration → Policies → Windows Settings → Security Settings → Restricted Groups**. Ici, nous pouvons faire un clic droit pour ajouter un nouveau groupe.

En utilisant **Browse**, tape "Product", puis clique sur **Check Names** pour rechercher le groupe **Product Admins**.

Clique sur OK. Nous pouvons maintenant préciser que Product Admins doit être ajouté au groupe **Administrators**, comme montré ci-dessous.

Clique sur OK, et ta GPO devrait être configurée.

Testons pour voir si notre GPO a bien été appliquée !

### Distribution des GPO

Les GPO sont distribuées sur le réseau via un partage réseau appelé **SYSVOL**, stocké sur le DC. Tous les utilisateurs d'un domaine devraient normalement avoir un accès réseau pour synchroniser leurs GPO périodiquement. Le partage SYSVOL pointe par défaut vers le répertoire `C:\Windows\SYSVOL\sysvol\` sur chaque DC de notre réseau.

Une fois qu'une modification a été apportée à une GPO, il peut falloir jusqu'à **2 heures** pour que les ordinateurs se mettent à jour. Si tu souhaites forcer un ordinateur en particulier à synchroniser immédiatement ses GPO, tu peux toujours exécuter la commande suivante sur l'ordinateur concerné :

```
PS C:\> gpupdate /force
```

Pour vérifier si notre GPO a été appliquée, exécute cette commande sur **Server1**. Une fois terminé, tu peux utiliser les identifiants suivants pour te connecter en RDP à Server1 en tant que membre du groupe Product Admins :

<img width="838" height="210" alt="image" src="https://github.com/user-attachments/assets/4a6b014f-5ee1-4fa7-9789-93c0dbecfcf6" />


Si la GPO a bien été appliquée, le compte `terry.fox` pourra se connecter en RDP à Server1.
