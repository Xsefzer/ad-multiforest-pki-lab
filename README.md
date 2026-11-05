<div align="center">

# 🔐 Active Directory Multi-Forêts & PKI d'entreprise

### Forest Trust bidirectionnel · Authentification sélective · AGDLP · AD CS · Auto-Enrollment

![Windows Server](https://img.shields.io/badge/Windows%20Server-2019-0078D6?logo=windows&logoColor=white)
![Active Directory](https://img.shields.io/badge/Active%20Directory-Multi--Forest-0078D6)
![AD CS](https://img.shields.io/badge/AD%20CS-Enterprise%20Root%20CA-success)
![VMware](https://img.shields.io/badge/Virtualisation-VMware%20Workstation-607078?logo=vmware&logoColor=white)
![Status](https://img.shields.io/badge/Statut-Terminé-brightgreen)

**Auteur : Youssef Lahlali** · Cours *SRV-WIN (III)* · 420-4RS-HY

</div>

---

## 📌 Résumé du projet

Ce laboratoire reproduit un scénario réaliste d'entreprise : **deux organisations indépendantes (Contoso et Fabrikam) doivent collaborer** et partager des ressources, tout en gardant des annuaires séparés. Le projet est divisé en deux grands volets :

1. **Interopérabilité entre forêts Active Directory** : création d'un *forest trust* bidirectionnel avec authentification sélective, gestion des accès selon le modèle **AGDLP**, et partage de fichiers inter-forêts sécurisé (NTFS + partage).
2. **Infrastructure à clés publiques (PKI)** : déploiement d'une **Enterprise Root CA** (AD CS), création de modèles de certificats personnalisés, **auto-enrollment par GPO**, enrollment manuel via Web Enrollment, et validation par authentification client sur IIS (SSL + certificat client).

> 🎯 **Objectif professionnel** : démontrer des compétences concrètes en administration de systèmes Windows Server, en sécurité des identités et en dépannage.

---

## 🧰 Compétences démontrées

| Domaine | Détails |
|---|---|
| **Active Directory** | Installation de 2 forêts, OU, utilisateurs, groupes (Global / Universal / Domain Local) |
| **Relations d'approbation** | Forest trust two-way, authentification sélective, validation `nltest` / `netdom` |
| **Gestion des accès** | Modèle AGDLP/AGUDLP, permissions NTFS vs permissions de partage, héritage |
| **DNS** | Résolution inter-forêts via DNS secondaire |
| **PKI / AD CS** | Enterprise Root CA, RSA 4096 / SHA-256, CRL/AIA, templates, Web Enrollment |
| **GPO** | Auto-enrollment ordinateur + utilisateur |
| **IIS** | Binding HTTPS, Require SSL, certificats clients |
| **Dépannage** | Analyse des *Failed Requests* de la CA, erreurs d'accès réseau, résolution documentée |

---

## 🗺️ Architecture du laboratoire

```
┌─────────────────────────────┐   Forest Trust (two-way)   ┌─────────────────────────────┐
│  Forêt : contosoYL.com      │◄──────────────────────────►│  Forêt : fabrikam.com       │
│  (domaine des RESSOURCES)   │   Selective Authentication │  (domaine des UTILISATEURS) │
│                             │                            │                             │
│  DC-contosoYL  192.168.151.1│                            │  DC-Fabrikam  192.168.151.11│
│  • AD DS + DNS              │                            │  • AD DS + DNS              │
│  • Partage Projets_Communs  │                            │                             │
│  • AD CS (Contoso-Root-CA)  │                            │  OU Ventes → VentesUserF1   │
│  • IIS (Web Enrollment)     │                            │  GG-Ventes-Fabrikam         │
│                             │                            │  UG-Projets-Communs         │
│  OU Ventes → VentesUser1    │                            │                             │
│  OU Utilisateurs-PKI        │                            │  Client-Fabrikam            │
│  OU Ordinateurs-PKI         │                            │                             │
│  DLG-Acces-Modif-Projets    │                            │                             │
│  Client-Contoso             │                            │                             │
└─────────────────────────────┘                            └─────────────────────────────┘
```

| Machine | Rôle | Adresse IP |
|---|---|---|
| `DC-contosoYL` | Contrôleur de domaine, serveur de fichiers, autorité de certification, IIS | `192.168.151.1/24` |
| `DC-Fabrikam` | Contrôleur de domaine de la forêt Fabrikam | `192.168.151.11/24` |
| `Client-Contoso` | Poste Windows 10 joint à `contosoYL.com` | DHCP/statique du réseau lab |
| `Client-Fabrikam` | Poste Windows 10 joint à `fabrikam.com` | DHCP/statique du réseau lab |

---

## 📑 Table des matières

1. [Section 1 – Préparation des deux forêts](#section-1--préparation-des-deux-forêts)
2. [Section 2 – Configuration du forest trust](#section-2--configuration-du-forest-trust)
3. [Section 3 – Groupes et nesting (AGDLP / AGUDLP)](#section-3--groupes-et-nesting-agdlp--agudlp)
4. [Section 4 – Permissions NTFS et partage](#section-4--permissions-ntfs-et-partage)
5. [Section 5 – Tests et dépannage](#section-5--tests-et-dépannage)
6. [Section 6 – Préparation de la CA](#section-6--préparation-de-la-ca)
7. [Section 7 – Installation de l'Enterprise CA](#section-7--installation-et-configuration-de-lenterprise-ca)
8. [Section 8 – Templates et Auto-Enrollment](#section-8--templates--auto-enrollment)
9. [Section 9 – Tests d'enrollment](#section-9--tests-denrollment-depuis-un-client-windows)
10. [Section 10 – Tests d'utilisation et validation](#section-10--tests-dutilisation-et-validation)
11. [Problèmes rencontrés & leçons apprises](#-problèmes-rencontrés--leçons-apprises)
12. [Bonnes pratiques et pistes d'amélioration](#-bonnes-pratiques-et-pistes-damélioration)

---

# PARTIE A — Forêts Active Directory & Trust

## Section 1 – Préparation des deux forêts

### 1.1 Installation des deux domaines/forêts indépendants

**Pourquoi ?** Deux forêts distinctes représentent deux entreprises (ou deux entités) avec chacune sa propre base d'annuaire, ses propres administrateurs et sa propre frontière de sécurité. Dans AD, **la forêt est la vraie frontière de sécurité**, pas le domaine.

**Ce que j'ai fait :** j'ai installé le rôle AD DS et promu deux serveurs en contrôleurs de domaine, chacun créant une **nouvelle forêt** :

- `contosoYL.com`
- `fabrikam.com`

📸 **Capture – Contoso** `screenshots/01-contoso-aduc.png`
La console *Active Directory Users and Computers* (ADUC) affiche le domaine `contosoYL.com` avec ses conteneurs par défaut : `Builtin`, `Computers`, `Domain Controllers`, `ForeignSecurityPrincipals`, `Managed Service Accounts` et `Users`. Cela confirme que la promotion en contrôleur de domaine s'est déroulée correctement.

📸 **Capture – Fabrikam** `screenshots/02-fabrikam-aduc.png`
Même vérification pour la seconde forêt `fabrikam.com` : structure par défaut présente, ce qui valide que les deux forêts sont **créées, indépendantes et opérationnelles**.

### 1.2 Création des OU et des utilisateurs

**Pourquoi ?** Les **unités d'organisation (OU)** structurent l'annuaire et permettent de déléguer l'administration et d'appliquer des GPO de façon ciblée.

| Forêt | OU | Utilisateur |
|---|---|---|
| `contosoYL.com` | `Ventes` | `VentesUser1` |
| `fabrikam.com` | `Ventes` | `VentesUserF1` |

📸 **Capture – Création de l'OU** `screenshots/03-contoso-ou-ventes.png`
Fenêtre *New Object – Organizational Unit* : nom `Ventes`, avec l'option **« Protect container from accidental deletion »** cochée. Cette protection est une bonne pratique : elle empêche la suppression accidentelle de l'OU et de tout son contenu.

📸 **Capture – Utilisateur Contoso** `screenshots/04-contoso-ventesuser1.png`
L'OU `Ventes` apparaît dans l'arborescence et contient bien l'objet utilisateur `VentesUser1`.

📸 **Capture – Utilisateur Fabrikam** `screenshots/05-fabrikam-ventesuserf1.png`
Même structure côté Fabrikam : OU `Ventes` contenant `VentesUserF1`.

### 1.3 Jonction des clients et test de connexion

**Pourquoi ?** Avant de configurer quoi que ce soit de complexe, il faut valider la base : chaque client doit joindre son domaine et joindre son contrôleur de domaine sur le réseau.

📸 **Capture – `ipconfig /all` sur DC-Contoso** `screenshots/06-ipconfig-dc-contoso.png`
Elle montre :
- Nom d'hôte `DC-contosoYL`, suffixe DNS primaire `contosoYL.com`
- Adresse IPv4 **statique** `192.168.151.1`, masque `255.255.255.0`, DHCP désactivé
- Serveur DNS : `127.0.0.1` (le DC se résout lui-même, indispensable pour AD)

📸 **Capture – Ping depuis VentesUser1** `screenshots/07-ping-ventesuser1.png`
Depuis le client Contoso (session `VentesUser1`), `ping DC-contosoYL` résout le nom en `192.168.151.1` (preuve que le **DNS du domaine fonctionne**), puis `ping 192.168.151.1` répond **4/4 paquets, 0 % de perte**. La connexion de l'utilisateur de domaine prouve aussi que la jonction au domaine est réussie.

📸 **Capture – `ipconfig /all` sur DC-Fabrikam** `screenshots/08-ipconfig-dc-fabrikam.png`
Configuration équivalente : `DC-Fabrikam`, `fabrikam.com`, IPv4 `192.168.151.11`, DNS `127.0.0.1`.

📸 **Capture – Ping depuis VentesUserF1** `screenshots/09-ping-ventesuserf1.png`
Depuis le client Fabrikam, `ping DC-fabrikam` résout `dc-fabrikam.fabrikam.com [192.168.151.11]` avec **0 % de perte**, puis ping direct par IP réussi. *(Le titre du document mentionne « vers DC-Contoso » mais le test cible bien DC-Fabrikam, la communication inter-forêts est validée plus loin.)*

---

## Section 2 – Configuration du forest trust

### 2.1 Résolution DNS inter-forêts

**Pourquoi ?** Deux forêts distinctes ne se connaissent pas. Pour qu'un contrôleur de domaine localise l'autre forêt, il doit pouvoir **résoudre son nom DNS**. Sans cela, le trust échoue dès l'assistant.

📸 **Capture – DNS** `screenshots/10-dns-alternate.png`
Dans les propriétés IPv4 de `DC-contosoYL` :
- DNS préféré : `127.0.0.1` (lui-même)
- **DNS secondaire : `192.168.151.11`** (le DC de Fabrikam)

Ainsi, Contoso peut résoudre `fabrikam.com` en interrogeant le DNS de Fabrikam.

> 💡 En production, on préférerait des **conditional forwarders** (redirecteurs conditionnels) ou des zones secondaires plutôt qu'un DNS alternatif, plus propre et plus prévisible.

### 2.2 Création du Forest Trust bidirectionnel

**Ce que j'ai fait :** dans *Active Directory Domains and Trusts*, j'ai lancé le **New Trust Wizard** sur `contosoYL.com` :

📸 **Capture – Nom du trust** `screenshots/11-trust-name.png`
Saisie de la forêt cible : `fabrikam.com`.

📸 **Capture – Type de trust** `screenshots/12-trust-type.png`
Choix **Forest trust** plutôt que *External trust*. Différence clé :
- *External trust* : **non transitif**, limité à deux domaines précis.
- *Forest trust* : **transitif**, tous les domaines d'une forêt peuvent s'authentifier dans tous les domaines de l'autre.

📸 **Capture – Direction** `screenshots/13-trust-direction.png`
**Two-way** : les utilisateurs de chaque forêt peuvent être authentifiés dans l'autre.

📸 **Capture – Côtés du trust** `screenshots/14-trust-sides.png`
**« Both this domain and the specified domain »** : l'assistant crée les deux côtés du trust en une fois (nécessite des droits d'administration dans les deux forêts).

### 2.3 Authentification sélective

📸 **Capture – Niveau d'authentification** `screenshots/15-selective-auth.png`
Choix de **Selective authentication** au lieu de *Forest-wide authentication*.

| Mode | Comportement | Cas d'usage |
|---|---|---|
| Forest-wide | Tous les utilisateurs de la forêt approuvée peuvent s'authentifier sur toutes les ressources | Même organisation |
| **Selective** ✅ | **Aucun accès par défaut** : l'accès doit être accordé explicitement serveur par serveur | **Organisations différentes**, principe du moindre privilège |

### 2.4 Validation du trust

📸 **Capture – `nltest /sc_verify:fabrikam.com`** `screenshots/16-nltest.png`
Résultat : `Trusted DC Name \\DC-Fabrikam.fabrikam.com`, `Trusted DC Connection Status = 0x0 NERR_Success`, `Trust Verification Status = 0x0 NERR_Success`. ✅ **Le secure channel fonctionne.**

**À quoi servent ces commandes ?**

| Commande | Rôle |
|---|---|
| `nltest /sc_verify:<domaine>` | Vérifie le **secure channel**, le canal chiffré entre les deux DC utilisé pour l'authentification (Kerberos/NTLM) entre les forêts |
| `netdom trust <domaine> /domain:<autre> /verify` | Vérifie que le **trust existe** et que son **mot de passe est synchronisé** entre les deux DC |

### 2.5 Authentification sélective sur le trust entrant

📸 **Capture – Propriétés du trust** `screenshots/17-trust-properties.png`
Dans *contosoYL.com → Properties → Trusts*, section **« Domains that trust this domain (incoming trusts) »** : `fabrikam.com`, type *Forest*, transitif *Yes*. Dans *Properties → Authentication*, l'option **Selective authentication** est bien sélectionnée.

---

## Section 3 – Groupes et nesting (AGDLP / AGUDLP)

**Le principe :** pour gérer proprement les accès inter-domaines/forêts, on **imbrique les groupes** selon une chaîne précise :

```
Account  →  Global Group  →  (Universal Group)  →  Domain Local Group  →  Permission
(Utilisateur)   (par rôle)       (inter-domaines)      (par ressource)       (NTFS/partage)
   A      →        G         →        (U)           →         DL         →       P
```

Les **Global Groups** regroupent des utilisateurs (côté compte), les **Domain Local Groups** reçoivent les permissions (côté ressource). On ne donne **jamais** de permission directement à des utilisateurs.

### 3.1 Dans fabrikam.com (domaine des utilisateurs)

📸 **Capture – Global Group** `screenshots/18-gg-ventes-fabrikam.png`
Création de `GG-Ventes-Fabrikam` : étendue **Global**, type **Security**.

📸 **Capture – Ajout de l'utilisateur** `screenshots/19-gg-add-member.png`
`VentesUserF1@fabrikam.com` est ajouté comme membre du groupe global (vérifié avec *Check Names*).

### 3.2 Universal Group

📸 **Capture – Universal Group** `screenshots/20-ug-projets-communs.png`
Création de `UG-Projets-Communs` (étendue **Universal**, type **Security**), puis ajout de `GG-Ventes-Fabrikam` comme membre. Les groupes universels sont répliqués dans le **Global Catalog** et permettent la visibilité inter-domaines.

### 3.3 Dans contoso.com (domaine des ressources)

📸 **Capture – Domain Local Group** `screenshots/21-dlg-creation.png`
Création de `DLG-Acces-Modif-Projets` (étendue **Domain local**, type **Security**).

📸 **Capture – Ajout du groupe étranger** `screenshots/22-dlg-add-ug.png`
Ajout de `UG-Projets-Communs` (provenant de `fabrikam.com`) dans le groupe local de domaine. Cette étape est possible uniquement grâce au **forest trust**, c'est lui qui permet de sélectionner un principal de sécurité d'une autre forêt.

**Chaîne finale obtenue :**
`VentesUserF1 → GG-Ventes-Fabrikam → UG-Projets-Communs → DLG-Acces-Modif-Projets → Modify sur C:\Projets_Communs`

---

## Section 4 – Permissions NTFS et partage

Sur le serveur de fichiers de Contoso.

### 4.1 Création du dossier

📸 **Capture** `screenshots/23-dossier-projets-communs.png`
Création de `C:\Projets_Communs` à la racine du disque.

### 4.2 Partage réseau

📸 **Capture – Permissions de partage** `screenshots/24-share-permissions.png`
Partage `Projets_Communs` avec **Domain Users = Read**.

> ⚠️ **Point important :** les permissions de **partage** et **NTFS** se cumulent, et c'est **la plus restrictive qui s'applique**. Cette configuration a une conséquence visible dans la section 5.

### 4.3 Permissions NTFS

📸 **Capture – Ajout du DLG** `screenshots/25-ntfs-add-dlg.png`
Attribution de **Modify** (Modify, Read & execute, List folder contents, Read, Write) au groupe `DLG-Acces-Modif-Projets`, appliqué à *This folder, subfolders and files*.

📸 **Capture – Désactivation de l'héritage** `screenshots/26-disable-inheritance.png`
Dans les paramètres de sécurité avancés, j'utilise **Disable inheritance** pour couper l'héritage depuis `C:\`. Les entrées héritées (`Users`, `CREATOR OWNER`…) peuvent alors être supprimées.

📸 **Capture – Résultat** `screenshots/27-ntfs-final.png`
ACL finale, propre et minimale :

| Principal | Accès | Appliqué à |
|---|---|---|
| `SYSTEM` | Full control | Dossier, sous-dossiers, fichiers |
| `Administrators` | Full control | Dossier, sous-dossiers, fichiers |
| `DLG-Acces-Modif-Projets` | **Modify** | Dossier, sous-dossiers, fichiers |

*(La capture intermédiaire montre aussi un état temporaire « No groups or users have permission » juste après suppression de toutes les entrées, avant ré-ajout des bons principaux.)*

---

## Section 5 – Tests et dépannage

Depuis le **client Fabrikam**, connecté avec `VentesUserF1`, accès à `\\DC-contosoYL\Projets_Communs`.

| Test | Capture | Résultat |
|---|---|---|
| 1. Lecture | `28-test-lecture.png` | ✅ Le fichier `test.txt` est visible et son contenu (`test`) s'ouvre dans le Bloc-notes |
| 2. Création de fichier | `29-test-creation.png` | ❌ *Destination Folder Access Denied – You need permission to perform this action* |
| 3. Modification | `30-test-modification.png` | ❌ *You do not have permission to open this file* |
| 4. Suppression | `31-test-suppression.png` | ❌ *File Access Denied – You require permission from Administrators* |

**Analyse :** le trust et la chaîne de groupes fonctionnent (l'accès en lecture est établi), mais les écritures sont refusées. La cause la plus probable est la **combinaison des permissions de partage et NTFS** : le partage n'accorde que **Read** à *Domain Users*, et la règle de la permission effective la plus restrictive limite donc les utilisateurs à la lecture, même avec *Modify* côté NTFS.

> 🔧 **Correction recommandée :** ajouter `DLG-Acces-Modif-Projets` au partage avec **Change**, ou passer le partage à *Authenticated Users : Change* et laisser NTFS faire le contrôle fin (bonne pratique courante). À valider ensuite par une **déconnexion/reconnexion** du client (le jeton Kerberos n'est mis à jour qu'à l'ouverture de session).

### 5.3 Authentification sélective : accorder l'accès

📸 **Capture – « Allowed to authenticate »** `screenshots/32-allowed-to-authenticate.png`

Avec l'authentification sélective, **rien n'est autorisé par défaut**. Il faut accorder le droit **« Allowed to authenticate »** sur l'**objet ordinateur** du serveur de ressources, pour les utilisateurs/groupes de la forêt approuvée.

Ce que j'ai fait : dans ADUC → propriétés de `DC-CONTOSOYL` → onglet **Security** → ajout de `VentesUserF1 (VentesUserF1@fabrikam.com)` → case **Allowed to authenticate** cochée.

> 💡 **Mieux en production :** accorder ce droit au **groupe** (`GG-Ventes-Fabrikam`) plutôt qu'à un utilisateur individuel, pour rester cohérent avec AGDLP et faciliter la maintenance.

---

# PARTIE B — Infrastructure PKI (AD CS)

## Section 6 – Préparation de la CA

### 6.1 Prérequis serveur

📸 **Capture – `ipconfig /all`** `screenshots/33-ipconfig-ca.png`
Le serveur qui héberge la CA est **joint au domaine** `contosoYL.com` et possède une **IP statique** `192.168.151.1` (DHCP désactivé). Prérequis indispensables : l'adresse d'une CA ne doit jamais changer, car elle est référencée dans les certificats émis (CDP/AIA).

### 6.2 OU et objets de test

📸 **Capture – OU Utilisateurs-PKI** `screenshots/34-ou-utilisateurs-pki.png`
OU `Utilisateurs-PKI` contenant l'utilisateur de test `TestUserPKI`.

📸 **Capture – OU Ordinateurs-PKI** `screenshots/35-ou-ordinateurs-pki.png`
OU `Ordinateurs-PKI` créée pour accueillir le client. Séparer utilisateurs et ordinateurs permet d'appliquer des **GPO différentes** à chaque catégorie.

---

## Section 7 – Installation et configuration de l'Enterprise CA

### 7.1 Installation du rôle

📸 **Capture – Installation** `screenshots/36-install-adcs.png`
Via *Server Manager → Add Roles and Features*, installation de **Active Directory Certificate Services** avec les services de rôle :
- **Certification Authority**
- **Certification Authority Web Enrollment**

Les dépendances (outils d'administration, **IIS** et sa compatibilité de gestion) sont installées automatiquement.

### 7.2 Configuration de la CA (assistant AD CS Configuration)

| Étape | Choix | Capture | Justification |
|---|---|---|---|
| Type d'installation | **Enterprise CA** | `37-setup-type.png` | S'intègre à AD : publication automatique des modèles et des certificats, authentification des demandeurs |
| Type de CA | **Root CA** | `38-ca-type.png` | Sommet de la hiérarchie PKI, auto-signée. *Acceptable en lab ; en production on utilise une Root CA hors ligne + CA subordonnée émettrice* |
| Cryptographie | **RSA 4096 bits / SHA-256** | `39-cryptography.png` | Clé longue et hachage robuste (SHA-1 et MD5 sont obsolètes) |
| Nom de la CA | `contoso-Root-CA` | `40-ca-name.png` | DN généré : `CN=contoso-Root-CA,DC=contosoYL,DC=com` |
| Validité | **5 ans** (expiration 12/04/2031) | `41-validity.png` | La durée de la CA doit dépasser celle des certificats qu'elle émet |
| Base de données | Emplacements par défaut (`C:\Windows\system32\CertLog`) | `42-database.png` | Base de données et journaux des certificats émis |

### 7.3 IIS pour le Web Enrollment

📸 **Capture – Server Manager › IIS** `screenshots/43-iis-installed.png`
Le rôle **IIS** apparaît dans Server Manager, installé automatiquement, car requis par la page web d'enrollment `/certsrv`.

---

## Section 8 – Templates + Auto-Enrollment

### 8.1 Duplication et configuration des modèles

Dans la console *Certification Authority → Certificate Templates → Manage*, j'ai **dupliqué** deux modèles par défaut (on ne modifie jamais les modèles d'origine).

#### 🖥️ Modèle ordinateur : `Computer-Auth-Contoso`

| Paramètre | Valeur | Capture |
|---|---|---|
| Nom / validité | `Computer-Auth-Contoso`, 1 an, renouvellement 6 semaines | `44-computer-general.png` |
| Subject Name | **Build from Active Directory information** → DNS name | `45-computer-subject.png` |
| Sécurité | *Authenticated Users* → **Read + Enroll + Autoenroll** | `46-computer-security.png` |
| Extensions | **Client Authentication + Server Authentication** | `47-computer-extensions.png` |

> Le sujet construit depuis AD garantit la **cohérence des noms** et évite la saisie manuelle du demandeur.

#### 👤 Modèle utilisateur : `User-Auth-Contoso`

| Paramètre | Valeur | Capture |
|---|---|---|
| Nom / validité | `User-Auth-Contoso`, 1 an, publication dans AD | `48-user-general.png` |
| Subject Name | Build from AD, E-mail + **UPN** dans le SAN | `49-user-subject.png` |
| Sécurité | *Authenticated Users* → **Enroll + Autoenroll** | `50-user-security.png` |
| Extensions | **Client Authentication** (+ EFS, Secure Email) | `51-user-extensions.png` |

### 8.2 Publication des modèles

📸 **Captures** `screenshots/52-publish-computer.png` · `53-publish-user.png`
Clic droit sur *Certificate Templates → New → Certificate Template to Issue* et sélection de `Computer-Auth-Contoso` puis `User-Auth-Contoso`. **Un modèle non publié n'est pas proposé aux demandeurs.**

### 8.3 GPO d'auto-enrollment : `PKI-AutoEnrollment`

📸 **Capture – Emplacement du paramètre** `screenshots/54-gpo-public-key-policies.png`
*Computer Configuration → Policies → Windows Settings → Security Settings → **Public Key Policies*** → **Certificate Services Client – Auto-Enrollment**.

📸 **Capture – Configuration ordinateur** `screenshots/55-gpo-computer.png`
Configuration Model : **Enabled**, avec :
- ☑ Renew expired certificates, update pending certificates and remove revoked certificates
- ☑ Update certificates that use certificate templates

📸 **Capture – Configuration utilisateur** `screenshots/56-gpo-user.png`
Même paramétrage sous **User Configuration** : les deux scopes (machine **et** utilisateur) sont couverts.

---

## Section 9 – Tests d'enrollment depuis un client Windows

### 9.1 Auto-enrollment

📸 **Capture – `gpupdate /force`** `screenshots/57-gpupdate.png`
Application forcée des stratégies : *Computer Policy update has completed successfully.*

📸 **Capture – Console MMC** `screenshots/58-mmc-snapins.png`
Ajout du snap-in **Certificates** pour **Current User** et **Local Computer**, afin d'inspecter les deux magasins.

📸 **Capture – `certutil -pulse`** `screenshots/59-certutil-pulse.png`
Déclenchement immédiat du cycle d'auto-enrollment (plutôt que d'attendre 5 à 10 minutes) : *CertUtil: -pulse command completed successfully.*

### 9.2 🛠️ Dépannage : les certificats n'apparaissent pas

📸 **Capture – Failed Requests** `screenshots/60-failed-requests.png`

**Symptôme :** aucun certificat utilisateur délivré.

**Diagnostic :** dans la console de la CA → **Failed Requests**, plusieurs demandes de `CONTOSOYL\TestUserPKI` pour le modèle `User-Auth-Contoso` sont rejetées avec l'erreur :

```
0x80094812 (-2146875374) CERTSRV_E_SUBJECT_EMAIL_REQUIRED
The EMail name is unavailable and cannot be added to the Subject or Subject Alternate name.
```

**Cause :** le modèle exige l'e-mail dans le sujet, mais le compte `TestUserPKI` **n'avait aucun attribut e-mail** dans AD.

**Deux solutions possibles :**
1. Renseigner l'e-mail de l'utilisateur dans AD ✅ *(choix retenu)*
2. Décocher « Include e-mail name in subject name » dans le modèle

📸 **Capture – Correction** `screenshots/61-user-email.png`
Ajout de `TestUserPKI@contosoYL.com` dans l'onglet *General* de l'utilisateur via ADUC.

### 9.3 Vérification de l'enrollment

📸 **Capture – `certutil -store`** `screenshots/62-certutil-store.png`
Après `gpupdate /force` + `certutil -pulse` :

| Commande | Contexte | Certificat obtenu |
|---|---|---|
| `certutil -store -user My` | Utilisateur | `E=TestUserPKI@contosoYL.com, CN=TestUserPKI…` · Template **User-Auth-Contoso** · Émetteur `contoso-Root-CA` · Valide 1 an |
| `certutil -store My` *(admin)* | Machine | `CN=client-TP1.contosoYL.com` · Template **Computer-Auth-Contoso** |

📸 **Captures MMC** `screenshots/63-mmc-user-cert.png` · `64-mmc-computer-cert.png`
Confirmation visuelle dans les consoles : **TestUserPKI** (modèle `User-Auth-Contoso`) dans *Current User → Personal*, et **client-TP1.contosoYL.com** (modèle `Computer-Auth-Contoso`) dans *Local Computer → Personal*. ✅ **L'auto-enrollment fonctionne pour l'utilisateur et l'ordinateur.**

### 9.4 Enrollment manuel (Web Enrollment)

**Problème :** l'accès à `/certsrv` échouait depuis le client. **Solution :** ajout d'un **binding HTTPS (port 443)** dans IIS sur `DC-contoso`, pour pouvoir accéder à `https://192.168.151.1/certsrv`.

📸 **Capture – Demande avancée** `screenshots/65-advanced-request.png`
*Request a certificate → Advanced certificate request* avec le modèle `User-Auth-Contoso`, CSP *Microsoft Enhanced Cryptographic Provider v1.0*, clé 2048 bits, format PKCS10.

📸 **Captures – Émission et installation** `screenshots/66-certificate-issued.png` · `67-certificate-installed.png`
*« The certificate you requested was issued to you »* → *Install this certificate* → *« Your new certificate has been successfully installed »*.

> L'avertissement *Certificate error* du navigateur s'explique par l'accès via **IP** (le certificat IIS est émis pour un nom DNS) : nom ≠ adresse, comportement normal et sans impact sur le lab.

---

## Section 10 – Tests d'utilisation et validation

### 10.1 Authentification client sur IIS

📸 **Capture – Paramètres SSL** `screenshots/68-iis-ssl-settings.png`
Dans IIS Manager → *Default Web Site → SSL Settings* : **Require SSL** coché et **Client certificates = Require**. Le serveur refuse toute connexion sans certificat client valide.

📸 **Capture – Sélection du certificat** `screenshots/69-select-certificate.png`
Depuis le client, l'accès à `https://192.168.151.1/` déclenche la fenêtre *Windows Security → Select a Certificate* : `TestUserPKI@contosoYL.com`, émis par `contoso-Root-CA`, valide du 13/04/2026 au 13/04/2027.

📸 **Capture – Accès réussi** `screenshots/70-iis-access-ok.png`
La page d'accueil **IIS** s'affiche : **authentification par certificat réussie** ✅.

### 10.2 Vérification CRL / AIA

📸 **Capture – CRL Distribution Points** `screenshots/71-crl-distribution-points.png`
Dans les détails du certificat, le champ **CRL Distribution Points** pointe vers un emplacement LDAP :
`ldap:///CN=contoso-Root-CA,CN=…,CN=CDP,CN=Public Key Services,CN=Services,CN=Configuration,DC=contosoYL,DC=com`
Un client sait ainsi où vérifier si un certificat a été révoqué.

📸 **Captures – Téléchargement du CRL** `screenshots/72-crl-download.png` · `73-crl-open-warning.png` · `74-crl-info.png`
Le fichier `contoso-Root-CA.crl` (1,16 Ko) est téléchargé, puis ouvert : émetteur `contoso-Root-CA`, date d'effet `12/04/2026`, **prochaine mise à jour `20/04/2026`**, signature **sha256RSA**. ✅ La publication du CRL est opérationnelle.

---

## 🧩 Problèmes rencontrés & leçons apprises

| # | Problème | Cause | Solution | Leçon |
|---|---|---|---|---|
| 1 | Écriture refusée sur le partage depuis Fabrikam | Permissions de partage limitées à *Read* | Accorder *Change* au groupe DLG sur le partage | Partage + NTFS = **la plus restrictive gagne** |
| 2 | Accès refusé au serveur depuis la forêt approuvée | Authentification sélective = aucun accès par défaut | Droit **Allowed to authenticate** sur l'objet ordinateur | Toujours autoriser explicitement en mode sélectif |
| 3 | Certificat utilisateur non émis | `CERTSRV_E_SUBJECT_EMAIL_REQUIRED` : pas d'e-mail dans AD | Renseigner l'attribut e-mail | Lire les **Failed Requests** de la CA en premier |
| 4 | `/certsrv` inaccessible en HTTPS | Aucun binding 443 dans IIS | Ajout du binding HTTPS | Le Web Enrollment exige un binding SSL |
| 5 | Les certificats n'arrivent pas tout de suite | Cycle d'auto-enrollment périodique | `gpupdate /force` + `certutil -pulse` | Forcer le rafraîchissement lors des tests |

---

## ✅ Bonnes pratiques et pistes d'amélioration

Ce laboratoire est volontairement simplifié. Voici ce que je ferais en **environnement de production** :

- 🔒 **PKI à deux niveaux** : Root CA **hors ligne** + CA subordonnée émettrice, **sur un serveur dédié**, jamais sur un contrôleur de domaine
- 🌐 **DNS** : utiliser des *conditional forwarders* pour la résolution inter-forêts
- 👥 **Authentification sélective** : accorder *Allowed to authenticate* à des **groupes**, pas à des comptes individuels
- 🔗 **CDP/AIA** : publier aussi en **HTTP** avec un nom DNS dédié, et utiliser un certificat IIS valide au lieu de l'adresse IP
- 🛡️ **Durcissement** : supprimer les modèles inutilisés, restreindre les droits d'enrollment, activer l'audit de la CA
- 💾 **Sauvegarde** : sauvegarde régulière de la clé privée et de la base de la CA
- 🤖 **Automatisation** : scripter la création des OU, utilisateurs et groupes en **PowerShell**

---

## 🛠️ Commandes utiles de ce lab

```powershell
# Vérification du trust
nltest /sc_verify:fabrikam.com
netdom trust contosoYL.com /domain:fabrikam.com /verify

# Configuration réseau
ipconfig /all

# Rafraîchissement des GPO et de l'auto-enrollment
gpupdate /force
certutil -pulse

# Inspection des certificats
certutil -store My          # magasin ordinateur (en admin)
certutil -store -user My    # magasin utilisateur
```

---

## 📂 Structure du dépôt

```
📦 ad-multiforest-pki-lab
 ┣ 📂 screenshots        # Captures numérotées citées dans ce README
 ┣ 📄 README.md          # Documentation complète du projet
 ┗ 📄 TP-SRV-WIN-III.pdf # Rapport original du travail pratique
```

---

## 👤 Auteur

**Youssef Lahlali** — Étudiant en administration des réseaux et systèmes (420-4RS-HY)
📍 Québec, Canada · 🔎 À la recherche d'un **stage** en administration systèmes / réseaux / cybersécurité

📧 *yssflhl.777@proton.me* · 💼 *linkedin.com/(en préparation)*

---

<div align="center">

⭐ *Si ce projet t'a été utile, n'hésite pas à laisser une étoile !*

</div>
