# Construction d'un Datacenter complet avec vSphere, vCenter Server et PowerCLI

Guide de référence et support de cours pour monter, configurer, exploiter et sécuriser une infrastructure virtualisée centralisée, dans le lab VMware Hands-on Labs (HOL).

Ce document suit le manuel officiel **HOL-2535-01-VCF-L : Virtualization 101** (édition 2024), module par module. Chaque module commence par un **cours complet** (ce que l'étudiant doit connaître), puis déroule les **leçons pratiques** avec, pour chaque série de commandes, un titre, un objectif, une définition et une explication après chaque commande.

Méthode de rédaction : les définitions et procédures viennent du manuel. Les passages marqués **Complément** ajoutent des notions standard de vSphere qui ne figurent pas dans le manuel mais qui sont attendues d'un étudiant (architecture interne, mécanismes, bonnes pratiques).

---

## Table des matières

* Module 0 : Fondamentaux, environnement du lab et connexion
* Module 1 : Introduction à la gestion avec vCenter Server
* Module 2 : Introduction au réseau et à la sécurité vSphere
* Module 3 : Introduction au stockage vSphere
* Annexe A : Nettoyage du lab
* Annexe B : Journal des erreurs connues
* Annexe C : Checklist finale de validation
* Annexe D : Ressources complémentaires du manuel
* Annexe E : Interface Hands-on Labs
* Annexe F : Tableau de couverture du manuel
* Annexe G : Sources

---

# MODULE 0 : Fondamentaux, environnement du lab et connexion

## 0.1 Cours : la virtualisation

### Le problème de départ

Le matériel x86 a été conçu pour exécuter un seul système d'exploitation et une seule application. Résultat : la plupart des serveurs physiques sont très peu utilisés (souvent moins de 15 % de CPU), tout en consommant de l'espace, de l'énergie et du refroidissement.

### La solution

La virtualisation consiste à ajouter une couche logicielle appelée **hyperviseur** au-dessus du serveur physique. L'hyperviseur permet d'exécuter plusieurs machines virtuelles (VM) sur une même machine, chacune avec son système d'exploitation et ses applications, en partageant les ressources physiques (CPU, mémoire, stockage, réseau).

```
Sans virtualisation               Avec virtualisation

+-----------+                     +------+ +------+ +------+
| Application |                   | App1 | | App2 | | App3 |
+-----------+                     +------+ +------+ +------+
| Système     |                   | OS 1 | | OS 2 | | OS 3 |
+-----------+                     +------+ +------+ +------+
| Matériel    |                   |     Hyperviseur (ESXi)  |
+-----------+                     +-------------------------+
                                  |        Matériel         |
                                  +-------------------------+
```

### Les quatre propriétés d'une machine virtuelle

| Propriété | Définition | Exemple concret |
| --- | --- | --- |
| Partitionnement | Plusieurs systèmes sur un seul serveur physique, avec répartition des ressources. | Un serveur héberge 20 VM Linux et Windows. |
| Isolation | Les VM sont isolées les unes des autres, au niveau des pannes et de la sécurité. | Une VM qui plante n'affecte pas ses voisines. |
| Encapsulation | Une VM est un ensemble de fichiers, donc copiable, déplaçable, restaurable. | Copier un dossier de VM sur un autre datastore. |
| Indépendance matérielle | Une VM n'est liée à aucun matériel ou constructeur précis. | Migrer une VM d'un serveur Dell vers un serveur HPE. |

### Bénéfices à connaître

* Consolidation des serveurs et meilleure utilisation du matériel.
* Haute disponibilité et évolutivité des applications critiques.
* Reprise simplifiée : plus de reconfiguration matérielle, de réinstallation du système ni d'agents de sauvegarde par serveur. Une simple restauration récupère une VM entière.
* Réduction des coûts de stockage grâce au **thin provisioning** : l'espace n'est consommé qu'au fur et à mesure des besoins.
* Évitement de coûts (matériel, énergie, refroidissement, espace en baie).

### Complément : types d'hyperviseurs

| Type | Principe | Exemples |
| --- | --- | --- |
| Type 1 (bare metal) | S'installe directement sur le matériel. Performances élevées, usage serveur. | VMware ESXi, Hyper-V, KVM |
| Type 2 (hébergé) | S'exécute comme une application dans un système hôte. Usage poste de travail. | VMware Workstation, VirtualBox |

ESXi est un hyperviseur de type 1.

## 0.2 Glossaire général

| Terme | Définition |
| --- | --- |
| Hyperviseur | Couche logicielle posée sur le serveur physique qui permet d'exécuter plusieurs systèmes et applications. |
| ESXi | Hyperviseur de VMware, fondation de vSphere. Aussi appelé hôte. |
| vCenter Server | Plateforme de gestion centralisée de plusieurs hôtes et VM. Requis pour HA, DRS, vMotion, VDS. |
| VCSA | vCenter Server Appliance : VM préconfigurée (Photon OS) qui exécute vCenter. |
| vSphere Client | Interface web de gestion de vCenter. |
| ESXi Host Client | Interface web qui gère un seul hôte ESXi. |
| SSO | Single Sign-On : service d'authentification de vCenter (domaine `vsphere.local`). |
| Datacenter | Conteneur logique de premier niveau de l'inventaire vCenter. |
| Cluster | Groupe d'hôtes dont les ressources sont mutualisées (HA, DRS). |
| VM | Ordinateur logiciel défini par des fichiers (configuration, disques). |
| Template | Copie maître d'une VM pour déployer rapidement de nouvelles VM. |
| Content Library | Dépôt centralisé d'ISO, templates et autres contenus. |
| VSS / VDS | vSphere Standard Switch (par hôte) / Distributed Switch (centralisé). |
| Port group | Regroupement de ports d'un switch virtuel, souvent associé à un VLAN. |
| VMkernel | Interface réseau de l'hôte pour ses propres services (gestion, vMotion, stockage IP, FT). |
| Datastore | Conteneur de stockage logique pour les fichiers des VM. |
| VMFS / NFS / iSCSI | Système de fichiers vSphere en mode bloc / stockage fichier réseau / stockage bloc sur IP. |
| vMotion / Storage vMotion | Migration à chaud du calcul / du stockage d'une VM. |
| HA / DRS / FT | Redémarrage automatique / équilibrage de charge / disponibilité continue. |
| Snapshot | Capture de l'état d'une VM (paramètres, disques, mémoire en option). |
| DPU | Data Processing Unit : carte qui décharge des services d'infrastructure du CPU de l'hôte. |

## 0.3 Environnement du lab

| Élément | Valeur |
| --- | --- |
| vCenter Server | `vcsa-01a.vcf.sddc.lab` (ou IP : `10.0.0.22`) |
| Identifiants SSO | `administrator@vsphere.local` / `VMware123!` |
| Hôtes ESXi | `esx-01a.vcf.sddc.lab` et `esx-02a.vcf.sddc.lab` |
| Poste de travail | Terminal Linux (`/home/holuser`) avec PowerShell et PowerCLI |
| Proxy | Doit être désactivé pour PowerCLI dans ce lab |
| Plateforme | VMware Hands-on Labs (le lab ne peut pas être sauvegardé et s'arrête à la fin du minuteur) |

Remarques :

* Le manuel HOL-2535 utilise d'autres noms et mots de passe (par exemple `esx-01a.corp.vmbeans.com`, `VMware1!`, cluster `RegionA01-COMP01`). Les concepts et commandes sont identiques. Adaptez les valeurs à votre lab.
* Les adresses IP des sections réseau et stockage (`10.0.50.x`, `10.10.20.60`) sont des exemples. Vérifiez-les avant exécution.
* Les modules du manuel sont indépendants : chaque module de ce README commence par un bloc de variables à rejouer si vous démarrez à cet endroit.
* Si une cmdlet est inconnue de votre version de PowerCLI, vérifiez avec `Get-Command -Name *motclé*` et `Get-Help NomCmdlet -Full`.

## 0.4 Connexion au vCenter

**Objectif** : ouvrir une session PowerCLI authentifiée sur vCenter, point d'entrée de toute l'infrastructure.

**Définition** : PowerCLI est un module PowerShell qui pilote vSphere via l'API. Se connecter à vCenter (et non à un ESXi seul) donne accès aux fonctions centralisées : clusters, vMotion, VDS, Content Library.

### Fermer les sessions précédentes

```powershell
Disconnect-VIServer -Server * -Confirm:$false -ErrorAction SilentlyContinue
```

Explication : ferme toute connexion ouverte (par exemple vers un ESXi autonome) pour éviter qu'une commande s'exécute sur le mauvais serveur. `-ErrorAction SilentlyContinue` évite une erreur s'il n'y a aucune session.

### Désactiver le proxy et gérer les certificats

```powershell
Set-PowerCLIConfiguration -ProxyPolicy NoProxy -Scope Session -Confirm:$false
```

Explication : le proxy du lab bloque PowerCLI (erreur `proxy tunnel request failed`). `-Scope Session` limite le changement à la session courante.

```powershell
Set-PowerCLIConfiguration -InvalidCertificateAction Ignore -Scope Session -Confirm:$false
```

Explication : le lab utilise des certificats auto-signés. Cette option évite l'erreur de certificat non approuvé. À proscrire en production.

### Créer les identifiants SSO

```powershell
$ssoCred = New-Object System.Management.Automation.PSCredential ("administrator@vsphere.local", (ConvertTo-SecureString "VMware123!" -AsPlainText -Force))
```

Explication : construit un objet d'identifiants pour le compte administrateur SSO. Le mot de passe en clair n'est acceptable que dans un lab.

### Se connecter par adresse IP

```powershell
Connect-VIServer -Server "10.0.0.22" -Credential $ssoCred
```

Explication : on utilise l'IP, car la résolution DNS de `vcsa-01a.vcf.sddc.lab` peut échouer dans le lab (erreur `No route to host`).

### Vérifier la connexion

```powershell
$global:DefaultVIServers
```

Explication : affiche les serveurs connectés. Vous devez voir `10.0.0.22`. Une liste vide signifie que la connexion a échoué.

## 0.5 Architecture cible

```
vCenter Server (10.0.0.22)
|
+-- RegionA01 (datacenter existant du lab, réutilisé)
    |
    +-- Cluster-CLI (DRS + HA)
    |   +-- esx-01a.vcf.sddc.lab
    |   +-- esx-02a.vcf.sddc.lab
    |   +-- Resource Pool : RP-Production
    |
    +-- VDS-CLI (4 uplinks)
    |   +-- PG-Serveurs-CLI  (VLAN 100) : trafic des VM
    |   +-- PG-vMotion-CLI   (VLAN 200) : trafic vMotion (VMkernel)
    |
    +-- Stockage
    |   +-- ds-nfs-cli, ds-iscsi-cli
    |   +-- DatastoreCluster-CLI (Storage DRS)
    |
    +-- Workloads
        +-- VM-vCenter-CLI, VM-Clone-01, VM-Deploy-01, template TPL-Photon-CLI
```

## 0.6 Ordre de réalisation recommandé

Le manuel présente les modules dans l'ordre vCenter, réseau, stockage. Pour **construire** un datacenter de bout en bout, certaines étapes en exigent d'autres (le vMotion a besoin du réseau et du stockage partagé). Suivez cet ordre :

| Ordre | Section | Raison |
| --- | --- | --- |
| 1 | 0.4 Connexion | Accès à vCenter |
| 2 | 1.3.1 Cluster et hôtes (dans le datacenter existant) | Base de l'inventaire |
| 3 | 2.3 VDS, port groups, VMkernel vMotion | Réseau requis par les VM et le vMotion |
| 4 | 3.2 et 3.3 Datastores NFS et iSCSI | Stockage partagé requis par HA, DRS, vMotion |
| 5 | 1.6 HA et DRS | Services de cluster |
| 6 | 1.3.2 à 1.5 VM, ISO, template, clone, tags | Charges de travail |
| 7 | 1.9, 1.10, 1.11, 3.5, 3.6, 3.7 | Opérations : ressources, vMotion, monitoring, svMotion, disques, snapshots |
| 8 | 2.5 à 2.8 | Sécurité : Lockdown, services, rôles, SSO |

## 0.7 Correspondance avec le manuel

| Module du manuel | Contenu | Module de ce README |
| --- | --- | --- |
| Lab Overview et Virtualization 101 | Notions de virtualisation | Module 0 |
| Module 1 | ESXi, Host Client, vCenter 8, VM, templates, tags, HA, DRS, FT, alarmes, shares, vMotion, monitoring, Tanzu | Module 1 |
| Module 2 | VSS, VDS, Lockdown, services, pare-feu, rôles, SSO | Module 2 |
| Module 3 | Datastores, Storage vMotion, disques, snapshots, datastore cluster | Module 3 |
| Additional Information et Appendix | Ressources, interface HOL | Annexes D et E |

---

# MODULE 1 : Introduction à la gestion avec vCenter Server

## 1.0 Cours : ce que l'étudiant doit connaître

### 1.0.1 Architecture de vSphere

vSphere est la plateforme de virtualisation de VMware. Elle repose sur deux piliers :

| Composant | Rôle |
| --- | --- |
| ESXi (hyperviseur) | Installé sur chaque serveur physique. Exécute les VM, gère CPU, mémoire, stockage et réseau de l'hôte. |
| vCenter Server | Plan de gestion central. Regroupe les ressources de plusieurs hôtes et gère le placement des VM, l'attribution des ressources et les fonctions avancées (HA, DRS, vMotion, VDS). |

```
Administrateur
     |
     v
vSphere Client (navigateur web)
     |
     v
vCenter Server Appliance  <--- SSO, VMCA, tags, licences, base PostgreSQL
     |
     +-----------+-----------+
     v           v           v
   ESXi 1      ESXi 2      ESXi 3        (agents hostd / vpxa sur chaque hôte)
     |           |           |
   VM VM       VM VM       VM VM
```

**Complément : composants internes d'un hôte ESXi**

| Composant | Rôle |
| --- | --- |
| VMkernel | Noyau de l'hyperviseur : ordonnancement CPU, gestion mémoire, pile de stockage et de réseau. |
| VMM (Virtual Machine Monitor) | Processus qui exécute le code de chaque VM, un par VM. |
| hostd | Service de gestion de l'hôte (utilisé par l'Host Client et par vCenter). |
| vpxa | Agent qui relaie les ordres de vCenter vers hostd. |
| DCUI | Console locale en mode texte pour la configuration initiale (mot de passe root, réseau de gestion). |

Un hôte peut fonctionner sans vCenter : HA continue de redémarrer les VM car c'est une fonction des hôtes. En revanche, vMotion, DRS et le VDS nécessitent vCenter.

### 1.0.2 ESXi Host Client et vSphere Client

| Interface | Portée | Usage typique |
| --- | --- | --- |
| ESXi Host Client | Un seul hôte | Environnements de test, ou production quand vCenter est injoignable |
| vSphere Client | Tout l'inventaire via vCenter | Usage normal. Seul à donner accès à HA, DRS, vMotion, VDS |

Quand vCenter est présent, c'est l'option préférée car elle offre de meilleurs outils pour gérer l'infrastructure dans son ensemble. L'Host Client permet de gérer l'heure et la synchronisation NTP, l'autostart, la gestion de l'énergie, les services (SSH, DCUI), la sécurité (Active Directory, certificats, comptes locaux à l'hôte), la surveillance (graphiques, matériel, journaux, support bundle), les VM, le stockage et le réseau de l'hôte.

### 1.0.3 vCenter Server Appliance (VCSA) et SSO

* Depuis vSphere 7.0, vCenter se déploie uniquement sous forme d'**appliance**, VM préconfigurée.
* L'appliance contient Photon OS, les services d'authentification vSphere, PostgreSQL, vSphere Lifecycle Manager et vCenter Lifecycle Manager.
* Tous les services de l'ancien Platform Services Controller (authentification, certificats, tags, licences) sont intégrés à vCenter. Un PSC externe n'est plus possible ni nécessaire.
* À l'installation, on choisit la taille de l'appliance selon la taille de l'environnement et le volume de base de données voulu.
* **SSO** : les composants vSphere communiquent par échange de jetons sécurisés (SAML) au lieu de réauthentifier l'utilisateur auprès de l'annuaire à chaque fois. Les **solution users** (services vSphere) reçoivent un certificat de la VMCA. Les autres utilisateurs saisissent identifiant et mot de passe, puis reçoivent un jeton SAML. Les droits dépendent ensuite des permissions vCenter.

### 1.0.4 vSphere Distributed Services Engine et DPU

Un **DPU** (Data Processing Unit) est une carte matérielle, comparable à une carte réseau ou GPU, qui exécute des services d'infrastructure. Avec vSphere 8, une seconde instance d'ESXi s'installe directement sur le DPU. Les services réseau (offload avec NSX) y sont déchargés du CPU de l'hôte, ce qui améliore performances et sécurité. Le cycle de vie est géré par vSphere Lifecycle Manager, et la version ESXi du DPU reste synchronisée avec celle de l'hôte parent.

### 1.0.5 Les objets de l'inventaire et les quatre vues

| Vue | Contenu |
| --- | --- |
| Hosts and Clusters | Datacenters, clusters, hôtes, resource pools, VM |
| VMs and Templates | VM et templates organisés en dossiers |
| Storage | Datastores et datastore clusters |
| Networking | Switchs standard et distribués, port groups |

Hiérarchie : vCenter, dossier racine, **Datacenter**, **Cluster**, **Hôte**, **VM**. Les permissions, alarmes et tags s'héritent du haut vers le bas.

### 1.0.6 Anatomie d'une machine virtuelle

Une VM est un ensemble de fichiers dans un dossier du datastore (principe d'encapsulation).

| Fichier | Rôle |
| --- | --- |
| `.vmx` | Fichier de configuration de la VM |
| `.vmdk` | Description du disque virtuel (et `-flat.vmdk` : données) |
| `.vmsd` | Métadonnées des snapshots |
| `-000001.vmdk` | Disque delta créé par un snapshot |
| `.nvram` | État du BIOS ou UEFI |
| `.vswp` | Fichier d'échange mémoire de la VM |
| `.vmss` | État de suspension |
| `vmware.log` | Journal de la VM |

**Matériel virtuel** : vCPU, mémoire, contrôleurs (SCSI : LSI Logic, LSI Logic SAS, BusLogic, VMware Paravirtual), disques, cartes réseau (E1000, VMXNET3), lecteur CD/DVD, carte vidéo.

**Version matérielle virtuelle** : détermine les fonctions disponibles pour la VM. L'appliance vCenter 8.0 utilise une version matérielle qui supporte 64 vCPU par VM dans ESXi.

**VMware Tools** : pilotes et services installés dans le système invité (arrêt propre, synchronisation de l'heure, pilotes optimisés, statistiques). Indispensables en production.

**États d'alimentation** : sous tension, hors tension, suspendue. L'arrêt propre (Shut Down Guest) passe par VMware Tools, la mise hors tension (Power Off) équivaut à débrancher le câble.

### 1.0.7 Les méthodes de provisionnement

| Méthode | Principe |
| --- | --- |
| Création from scratch | Assistant New Virtual Machine, puis installation du système depuis un ISO |
| Clone | Copie indépendante d'une VM existante |
| Template | Copie maître non démarrable, base de déploiements identiques |
| Content Library | Stockage centralisé d'ISO, templates et OVF partagés entre vCenter |
| OVF / OVA | Format standard d'import et d'export de VM |

**Assistant de création de VM** : nom et dossier, ressource de calcul, stockage, compatibilité, système invité, personnalisation du matériel, récapitulatif.

### 1.0.8 vMotion : mécanisme et prérequis

vMotion déplace une VM allumée d'un hôte à un autre sans interruption perceptible. L'état mémoire est copié par itérations pendant que la VM tourne. Quand il reste très peu de pages modifiées, la VM est brièvement suspendue (moins d'une seconde), l'état final est transféré, puis la VM reprend sur l'hôte cible.

Prérequis :

* Même vCenter, hôtes d'un même datacenter.
* Stockage partagé accessible par les deux hôtes (sauf vMotion sans stockage partagé).
* Interface VMkernel activée pour vMotion sur chaque hôte, joignable entre hôtes.
* Réseau de VM disponible (même nom de port group) sur les deux hôtes.
* Processeurs compatibles (ou **EVC**, Enhanced vMotion Compatibility, qui masque les différences de jeu d'instructions CPU).

Types de migration : calcul seul (vMotion), stockage seul (Storage vMotion, voir Module 3), calcul et stockage ensemble.

La **priorité vMotion** (standard ou haute) influence les ressources allouées à la migration.

### 1.0.9 vSphere HA en détail

vSphere HA regroupe VM et hôtes dans un cluster. Les hôtes sont surveillés, et les VM d'un hôte défaillant sont redémarrées sur d'autres hôtes. Un redémarrage implique une courte interruption (la VM démarre de zéro), contrairement à FT.

| Notion | Explication |
| --- | --- |
| Hôte primaire | Élu automatiquement à la création du cluster. Communique avec vCenter, surveille les VM protégées et les hôtes secondaires. |
| Hôtes secondaires | Les autres hôtes. Si le primaire tombe, une nouvelle élection a lieu. |
| Heartbeat réseau | Échanges sur le réseau de gestion entre hôtes. |
| Heartbeat datastore | Seconde couche de protection : permet de distinguer un hôte en panne d'un hôte isolé ou en partition réseau. |
| Hôte isolé | Hôte qui ne joint plus les autres ni son adresse d'isolement. Une **réponse à l'isolement** (ne rien faire, arrêter, éteindre) s'applique à ses VM. |
| VM Monitoring | Redémarre une VM si ses heartbeats VMware Tools cessent pendant un délai (30 secondes par défaut dans le lab). Option VM and Application Monitoring pour inclure les applications. |
| Admission Control | Réserve de capacité pour garantir le redémarrage après panne : nombre d'hôtes tolérés, pourcentage de ressources du cluster (25 % CPU et mémoire dans le manuel), ou hôtes de basculement dédiés. |
| Priorité de redémarrage | Ordre dans lequel les VM sont redémarrées (haute, moyenne, basse, désactivée). |

HA est une fonction des hôtes : il n'y a pas de dépendance à vCenter pour réaliser le basculement.

### 1.0.10 vSphere DRS en détail

DRS garantit les performances en équilibrant la charge des VM entre les hôtes d'un cluster. Il agit à deux moments : **placement initial** à la mise sous tension d'une VM, et **équilibrage** en cours de fonctionnement par vMotion.

| Niveau d'automatisation | Placement initial | Migrations |
| --- | --- | --- |
| Manuel | Recommandation | Recommandations à valider |
| Partiellement automatisé | Automatique | Recommandations à valider |
| Entièrement automatisé | Automatique | Automatiques |

**Complément** : le **seuil de migration** règle l'agressivité de l'équilibrage. Les **règles DRS** contrôlent le placement : affinité de VM (garder ensemble), anti-affinité de VM (séparer, par exemple deux serveurs d'un même service), règles VM vers hôtes. Depuis vSphere 7, DRS calcule aussi un score de santé par VM (VM DRS Score).

HA et DRS fonctionnent ensemble : HA redémarre les VM après une panne, DRS rééquilibre ensuite.

### 1.0.11 vSphere Fault Tolerance (FT)

FT assure une **disponibilité continue**. Une VM primaire et une VM secondaire identiques tournent sur deux hôtes différents. L'état de la primaire est répliqué en continu vers la secondaire. En cas de défaillance, la secondaire prend le relais sans perte de connexion ni de données en mémoire, et une nouvelle secondaire est recréée automatiquement.

* FT évite le **split-brain** (deux copies actives après une panne) par verrouillage atomique de fichiers sur stockage partagé.
* Cas d'usage : applications qui doivent rester disponibles avec des connexions clientes longues, applications sans clustering possible, remplacement de clusters applicatifs trop complexes.
* **On-Demand Fault Tolerance** : protéger une VM par HA en temps normal, et activer FT seulement pendant une période critique (par exemple un rapport de fin de trimestre), puis la désactiver.

| | HA | FT |
| --- | --- | --- |
| Principe | Redémarrage sur un autre hôte | Deux VM identiques en parallèle |
| Interruption | Courte (redémarrage) | Aucune |
| Coût en ressources | Faible | Double |
| Perte mémoire | Oui | Non |

### 1.0.12 Gestion des ressources : shares, limites, réservations

| Paramètre | Rôle | Effet |
| --- | --- | --- |
| Réservation | Quantité de ressource garantie | Réservée même si la VM est inactive. Compte dans l'admission control. |
| Limite | Plafond de consommation | La VM ne peut pas dépasser cette valeur, même si l'hôte est libre. À utiliser avec prudence. |
| Shares | Poids relatif | N'agit qu'en cas de **contention** (ressource saturée). |

Valeurs de shares : Low, Normal, High, Custom. Par vCPU : 500, 1000, 2000. Exemple du manuel : deux VM, développement et production. À 1000 shares chacune, elles se partagent le CPU équitablement. En passant la production à 2000 shares, elle reçoit deux fois plus de CPU en cas de contention.

**Resource Pool** : conteneur de ressources CPU et mémoire dans un cluster, qui applique shares, limites et réservations à un groupe de VM.

**Complément : techniques de récupération mémoire** (par ordre d'intervention croissante) : partage transparent de pages, ballooning (pilote dans VMware Tools qui récupère de la mémoire), compression, échange sur disque (swap). Plus on descend dans la liste, plus la performance se dégrade.

**CPU Ready** : temps pendant lequel une VM est prête à s'exécuter mais attend un cœur physique libre. Un CPU Ready élevé révèle une contention CPU, indicateur classique de surdimensionnement en vCPU.

### 1.0.13 Événements et alarmes

* **Événement** : enregistrement d'une action utilisateur ou système (licence expirée, VM allumée, perte de connexion d'un hôte). Il contient qui, quand et quel type.
* **Alarme** : notification déclenchée par un événement, un ensemble de conditions ou un état d'objet.

Définition d'une alarme : nom et description, type d'objet surveillé, **déclencheurs** (événement, condition ou état, avec gravité), **seuils de tolérance**, **actions** (e-mail, SNMP, exécution d'une commande, entrée en maintenance...).

Gravités : Normal (vert), Warning (jaune), Alert (rouge). Une alarme définie à un niveau supérieur (vCenter, dossier, cluster) est **héritée** par les objets en dessous.

### 1.0.14 Mode maintenance

Le mode maintenance met un hôte hors service pour intervention (ajout de mémoire, mise à jour). Les VM doivent être migrées ou arrêtées. Avec DRS entièrement automatisé, les VM sont migrées automatiquement. Un hôte nouvellement ajouté peut être mis en maintenance le temps de vérifier sa configuration avant de l'intégrer au cluster.

### 1.0.15 Surveillance et performances

Quatre ressources principales à surveiller : CPU, mémoire, disque, réseau. Les graphiques temps réel se rafraîchissent toutes les 20 secondes. Les graphiques avancés permettent de choisir objets, métriques et compteurs, et d'exporter en image ou CSV. Pour une approche proactive et plus avancée, VMware propose Aria Operations.

### 1.0.16 vSphere with Tanzu

Kubernetes est intégré à vSphere : les développeurs gardent leurs outils standard, les administrateurs gèrent les charges avec les objets vSphere habituels et appliquent des politiques sur des groupes d'objets. Les services de développement exposés : service de stockage (disques persistants), service réseau (routeurs virtuels, load balancers, pare-feu), service de registre (images Docker et OCI avec Harbor).

### 1.0.17 Points à retenir

* vCenter est indispensable pour HA, DRS, vMotion et le VDS.
* Les VM sont des fichiers : c'est ce qui rend clone, template, vMotion et snapshot possibles.
* vMotion exige stockage partagé et VMkernel vMotion.
* HA redémarre, FT duplique, DRS équilibre.
* Les shares n'agissent qu'en contention.

---

## 1.1 ESXi : installation et Host Client

### 1.1.1 Installation d'ESXi (simulation et vidéos)

**Objectif** : comprendre le déroulement d'une installation d'ESXi.

**Définition** : ESXi est la fondation de vSphere. Dans HOL, l'installation n'est pas réalisée en direct (trop d'entrées-sorties) : le manuel fournit une **simulation interactive** (HOL-2410-01-ISM_Mod1) et deux vidéos : « Installing and Configuring vSphere » (4 min 36) et « Overview of the DCUI » (4 min 58).

Étapes classiques (à connaître) :

| Étape | Description |
| --- | --- |
| 1. Démarrage | Boot sur l'ISO ou la clé d'installation ESXi |
| 2. Contrat de licence | Acceptation avec F11 |
| 3. Disque | Choix du disque d'installation |
| 4. Clavier | Choix de la disposition |
| 5. Mot de passe root | Définition du mot de passe administrateur local |
| 6. Confirmation | Lancement de l'installation (F11) |
| 7. Redémarrage | Retrait du média puis reboot |
| 8. DCUI | Configuration du réseau de gestion : IP, masque, passerelle, DNS, nom d'hôte, VLAN |

**Complément** : ESXi 8 demande au minimum 2 cœurs CPU et 8 Go de RAM, un processeur 64 bits avec virtualisation matérielle activée dans le BIOS, et au moins une carte réseau Gigabit. Les hôtes se déploient aussi par PXE, par script (kickstart) ou avec vSphere Lifecycle Manager.

### 1.1.2 Se connecter directement à un hôte (équivalent de l'Host Client)

**Objectif** : administrer un seul hôte, sans vCenter, comme avec l'ESXi Host Client.

**Définition** : l'Host Client est l'interface web d'un hôte. Elle sert quand aucun vCenter n'existe ou n'est joignable. Les comptes utilisés sont **locaux** à l'hôte (par exemple `root`), non partagés avec vCenter.

```powershell
$hostCred = New-Object System.Management.Automation.PSCredential ("root", (ConvertTo-SecureString "VMware123!" -AsPlainText -Force))
$esx = Connect-VIServer -Server "esx-02a.vcf.sddc.lab" -Credential $hostCred
```

Explication : crée les identifiants `root` de l'hôte puis ouvre une session directe sur `esx-02a`. Remplacez le nom par l'IP de l'hôte si le DNS échoue. L'utilisateur est local, pas un compte SSO. Gardez à l'esprit que ces commandes ouvrent une session distincte de celle de vCenter : précisez `-Server $esx` si deux sessions sont ouvertes.

```powershell
Get-VMHost -Server $esx | Select-Object Name, ConnectionState, PowerState, Version, Build
```

Explication : écran d'accueil de l'Host Client : état de connexion, alimentation, version ESXi et build.

### 1.1.3 Onglet System : date, heure et autostart

```powershell
$vmh = Get-VMHost -Server $esx
Get-VMHostNtpServer -VMHost $vmh
```

Explication : affiche les serveurs NTP configurés. Un hôte doit avoir une heure précise (journaux, certificats, authentification).

```powershell
Add-VMHostNtpServer -VMHost $vmh -NtpServer "pool.ntp.org"
Get-VMHostService -VMHost $vmh | Where-Object { $_.Key -eq "ntpd" } | Set-VMHostService -Policy On | Restart-VMHostService -Confirm:$false
```

Explication : ajoute un serveur NTP, règle le démarrage du service `ntpd` avec l'hôte, puis le redémarre. Dans le lab, remplacez `pool.ntp.org` par le serveur de temps accessible depuis votre réseau.

```powershell
Get-VMHostStartPolicy -VMHost $vmh
```

Explication : affiche la politique d'autostart de l'hôte, qui définit quelles VM démarrent automatiquement au démarrage de l'hôte.

### 1.1.4 Onglet Hardware : gestion de l'énergie

```powershell
$ps = Get-View $vmh.ExtensionData.ConfigManager.PowerSystem
$ps.Info.CurrentPolicy.ShortName
$ps.Capability.AvailablePolicy | Select-Object Key, ShortName
```

Explication : affiche la politique énergétique actuelle et les politiques disponibles (haute performance, équilibrée, économie, personnalisée). Dans le lab imbriqué, des valeurs comme « Unknown » sous Technology sont normales et peuvent être ignorées. Pour changer de politique, utilisez `$ps.ConfigurePowerPolicy(<Key>)` avec la clé affichée.

### 1.1.5 Onglet Services

```powershell
Get-VMHostService -VMHost $vmh | Select-Object Key, Label, Running, Policy
```

Explication : liste les services de l'hôte (SSH, DCUI, ESXi Shell, NTP) avec leur état. L'Host Client permet de les démarrer et arrêter.

```powershell
Get-VMHostService -VMHost $vmh | Where-Object { $_.Key -eq "TSM-SSH" } | Start-VMHostService -Confirm:$false
```

Explication : démarre le service SSH (clé `TSM-SSH`). À arrêter après usage, SSH augmente la surface d'attaque.

### 1.1.6 Onglet Security and Users

```powershell
Get-VMHostAccount -Server $esx
```

Explication : liste les comptes locaux de l'hôte. Ils ne sont partagés avec aucun autre hôte ni vCenter, ce qui rend la gestion lourde à grande échelle et justifie SSO.

```powershell
Get-VMHostAuthentication -VMHost $vmh
```

Explication : affiche l'état de l'authentification Active Directory de l'hôte. Les certificats et la jonction au domaine se configurent dans l'onglet Security and Users.

### 1.1.7 Onglet Monitor : support bundle

**Définition** : un support bundle est une archive de journaux et d'informations système utile au diagnostic et demandée par le support VMware.

```powershell
Get-Log -VMHost $vmh -Bundle -DestinationPath "/home/holuser/"
```

Explication : génère et télécharge le bundle (équivalent de Generate support bundle, quelques minutes). Le fichier est enregistré dans le dossier indiqué.

```powershell
Get-Stat -Entity $vmh -Stat cpu.usage.average -Realtime -MaxSamples 5
```

Explication : lit les performances temps réel de l'hôte, comme les graphiques de l'onglet Monitor.

### 1.1.8 Fermer la session directe

```powershell
Disconnect-VIServer -Server $esx -Confirm:$false
```

Explication : ferme la session sur l'hôte. Reconnectez-vous à vCenter (section 0.4) pour la suite. Les VM, le stockage et le réseau peuvent aussi se gérer à ce niveau, mais comme les actions sont identiques à celles de vCenter, elles sont traitées ensuite à l'échelle vCenter.

---

## 1.2 Aperçu de vCenter 8

**Objectif** : repérer la version, les composants et l'inventaire de vCenter.

**Définition** : vCenter regroupe les ressources des hôtes pour les partager entre les VM du datacenter, en gérant l'affectation des VM aux hôtes et celle des ressources aux VM selon les politiques de l'administrateur.

```powershell
$global:DefaultVIServer | Select-Object Name, Version, Build, ProductLine
```

Explication : affiche la version et le build du vCenter connecté.

```powershell
(Get-View ServiceInstance).Content.About | Select-Object FullName, Version, Build, OsType
```

Explication : interroge l'API pour obtenir le nom complet du produit, sa version et le système sous-jacent.

```powershell
Get-VMHost | Select-Object Name, Version, Build, Model, ProcessorType, NumCpu, MemoryTotalGB, ConnectionState
```

Explication : liste les hôtes gérés avec leur matériel et leur version ESXi.

```powershell
Get-Folder | Select-Object Name, Type
```

Explication : montre les dossiers de l'inventaire (racine, dossiers de datacenters, de VM, de réseaux, de datastores).

---

## 1.3 vCenter et création d'une machine virtuelle

Variables de session à rejouer si vous démarrez ici :

```powershell
$dc      = Get-Datacenter -Name "RegionA01"
$cluster = Get-Cluster -Name "Cluster-CLI" -ErrorAction SilentlyContinue
$h1 = Get-VMHost -Name 'esx-01a.vcf.sddc.lab'
$h2 = Get-VMHost -Name 'esx-02a.vcf.sddc.lab'
```

Explication : récupère le datacenter existant du lab et le cluster créé en 1.3.1. La variable `$cluster` reste vide tant que 1.3.1 n'est pas faite.

### 1.3.1 Datacenter, cluster et hôtes

**Objectif** : créer un cluster dans le datacenter existant du lab et y regrouper les deux hôtes, prérequis de vMotion, HA et DRS.

**Définition** : l'inventaire vCenter est hiérarchique : vCenter, Datacenter, Cluster, Hôte, VM. Dans l'environnement VCF du lab, un datacenter existe déjà (`RegionA01`) et contient les deux hôtes. On y ajoute notre propre cluster `Cluster-CLI` pour comprendre la hiérarchie.

#### Pourquoi on ne crée pas son propre datacenter (erreur rencontrée et corrigée)

Première approche tentée : créer `Datacenter-CLI`, y créer `Cluster-CLI`, puis déplacer les hôtes avec `Move-VMHost`. La commande échoue avec ce message :

```
Move-VMHost: The operation for the entity "Cluster-CLI" failed with the following message: "The operation is not supported on the object."
```

Explication de la cause :

* Chaque hôte appartient à un datacenter, ici `RegionA01`. `Move-VMHost` déplace un hôte entre les clusters et dossiers **d'un même datacenter**, mais pas d'un datacenter à un autre. Vérification dans le lab : les deux hôtes sont bien dans `RegionA01`, et `Datacenter-CLI` n'en contenait aucun.
* Pour changer un hôte de datacenter, il faudrait le **retirer de l'inventaire** (`Remove-VMHost`) puis le **rajouter** (`Add-VMHost`). C'est déconseillé dans ce lab.

Pourquoi il ne faut pas retirer et rajouter les hôtes ici :

| Raison | Détail |
| --- | --- |
| Le VDS est lié au datacenter | Un VDS appartient à un datacenter et ne peut pas en couvrir plusieurs. Les hôtes sont membres du VDS d'origine, et celui-ci porte leur réseau de gestion. Les détacher fait perdre la connexion à l'hôte. |
| Les datastores et réseaux existants | Datastores partagés, port groups et VMkernel (gestion, stockage, vMotion) sont rattachés au datacenter d'origine. Dans un nouveau datacenter, il faudrait tout recréer. |
| Services et VM déjà présents | Les VM de service `vCLS` et les VM du lab dépendent du cluster et du datacenter d'origine. |
| Pas de secours | Le lab est imbriqué et sans console physique : une erreur de réseau de gestion n'est pas récupérable. |

Conséquence pour la suite du guide : **on réutilise le datacenter `RegionA01`**. Le cluster, le VDS et le datastore cluster sont créés dans `$dc = RegionA01`. N'utilisez pas `New-Datacenter` dans ce lab. En production, on crée les datacenters avant d'y ajouter les hôtes (`Add-VMHost -Location`), de sorte qu'aucun déplacement entre datacenters ne soit nécessaire.

```powershell
Get-Datacenter | ForEach-Object { $d = $_; Get-VMHost -Location $d | Select-Object @{N='Datacenter';E={$d.Name}}, Name }
```

Explication : affiche, pour chaque datacenter, les hôtes qu'il contient. Résultat attendu : `RegionA01` avec `esx-01a` et `esx-02a`. À lancer en cas d'erreur « operation is not supported on the object » pour confirmer que source et destination sont dans le même datacenter.

```powershell
$dc = Get-Datacenter -Name "RegionA01"
```

Explication : récupère le datacenter existant, qui sert de conteneur à tous les objets de ce guide. La variable `$dc` est réutilisée dans les sections suivantes.

```powershell
$cluster = New-Cluster -Name "Cluster-CLI" -Location $dc -DrsEnabled -HAEnabled
```

Explication : crée le cluster `Cluster-CLI` dans `RegionA01`, avec DRS (équilibrage de charge) et HA (redémarrage automatique) activés. Les réglages fins sont faits en 1.6. Si un ancien `Datacenter-CLI` a été créé par erreur, supprimez-le d'abord avec `Get-Datacenter -Name "Datacenter-CLI" | Remove-Datacenter -Confirm:$false`, sinon deux clusters portent le nom `Cluster-CLI` et les commandes deviennent ambiguës.

```powershell
Set-Cluster -Cluster $cluster -HAAdmissionControlEnabled:$false -Confirm:$false
```

Explication : désactive **temporairement** le contrôle d'admission HA. Par défaut, il réserve la capacité d'un hôte (niveau de basculement 1). Tant que le cluster ne contient qu'un seul hôte, il refuse toutes les VM avec `Insufficient resources to satisfy configured failover level for vSphere HA`, et le déplacement des hôtes reste bloqué. Il est réactivé en 1.6.1, une fois les deux hôtes dans le cluster.

```powershell
Get-Datacenter
Get-Cluster
```

Explication : vérifie que `RegionA01` existe et que `Cluster-CLI` est créé à côté de `RegionA01-COMP01`.


**Avertissement (testé dans le lab)** : la commande de mise en maintenance reste bloquée à `percent complete: 0` si l'hôte héberge encore des VM et que DRS est désactivé. Dans le lab, les hôtes sont déjà dans le cluster `RegionA01-COMP01`, où DRS et HA sont désactivés : aucune VM n'est évacuée automatiquement et la commande attend indéfiniment. Videz donc l'hôte avant de le passer en maintenance.

```powershell
Get-VMHost $h1 | Get-VM | Select-Object Name, PowerState, VMHost
Get-Cluster -VMHost $h1 | Select-Object Name, HAEnabled, DrsEnabled, DrsAutomationLevel
```

Explication : liste les VM présentes sur `esx-01a` et l'état de HA et DRS du cluster actuel de l'hôte. Dans le lab, on trouve une VM de service `vCLS-...` et la VM `TinyLinux2`. La VM `vCLS` est gérée par vCenter et s'arrête ou se déplace toute seule à l'entrée en maintenance. Seules les autres VM doivent être déplacées à la main.

```powershell
Get-VM -Name "TinyLinux2" | Move-VM -Destination $h2 -Confirm:$false
```

Explication : migre la VM vers `esx-02a` par vMotion (VM allumée, sans coupure). Si le vMotion échoue (datastore non partagé, par exemple), éteignez la VM avec `Stop-VM -VM "TinyLinux2" -Confirm:$false`. Si une tentative précédente est restée bloquée, annulez-la d'abord avec `Get-Task -Status Running | Stop-Task -Confirm:$false`.

```powershell
Set-VMHost -VMHost $h1 -State Maintenance -Confirm:$false
```

Explication : place `esx-01a` en mode maintenance, état exigé pour déplacer un hôte déjà géré. Une fois l'hôte vidé, la commande se termine en quelques secondes et affiche `ConnectionState : Maintenance`. Si elle reste à 0 pendant plus d'une minute, revérifiez avec `Get-VMHost $h1 | Get-VM` ce qu'il reste sur l'hôte (ajoutez `-RunAsync` pour ne pas bloquer le terminal).

```powershell
Move-VMHost -VMHost $h1 -Destination $cluster
```

Explication : déplace l'hôte dans `Cluster-CLI`.

```powershell
Set-VMHost -VMHost $h1 -State Connected -Confirm:$false
```

Explication : sort l'hôte du mode maintenance, il peut accueillir des VM.

```powershell
Get-VMHost $h2 | Get-VM | Where-Object { $_.Name -notlike "vCLS*" } | Move-VM -Destination $h1 -Confirm:$false
```

Explication : avant de vider `esx-02a`, déplace ses VM (sauf les VM `vCLS`, gérées par vCenter) vers `esx-01a`, qui est maintenant dans `Cluster-CLI`. Cet ordre est obligatoire : si on traitait `esx-02a` en premier, il n'aurait aucun hôte où envoyer ses VM. Le vMotion entre deux clusters du même vCenter fonctionne si le stockage est partagé et si le port group de la VM existe sur les deux hôtes. Sinon, éteignez les VM avec `Stop-VM`.

```powershell
Set-VMHost -VMHost $h2 -State Maintenance -Confirm:$false
Move-VMHost -VMHost $h2 -Destination $cluster
Set-VMHost -VMHost $h2 -State Connected -Confirm:$false
```

Explication : même séquence pour `esx-02a`. Deux hôtes sont le minimum pour démontrer vMotion et HA.

```powershell
Get-Cluster -Name "Cluster-CLI" | Get-VMHost | Select-Object Name, ConnectionState
```

Explication : les deux hôtes doivent être `Connected`.

### 1.3.2 Créer une machine virtuelle

**Objectif** : créer une VM sur le cluster avec l'équivalent de l'assistant du vSphere Client.

**Définition** : l'assistant New Virtual Machine enchaîne : nom et dossier, ressource de calcul, stockage, compatibilité, système invité, personnalisation du matériel, récapitulatif. Sur vCenter, `New-VM` accepte `-NetworkName`, y compris pour un port group distribué (ce qui n'était pas le cas sur un ESXi autonome).

Correspondance assistant et paramètres :

| Étape de l'assistant | Paramètre `New-VM` |
| --- | --- |
| Name and folder | `-Name`, `-Location` |
| Compute resource | `-VMHost`, `-ResourcePool` |
| Storage | `-Datastore`, `-DiskStorageFormat` |
| Compatibility | `-HardwareVersion` |
| Guest OS | `-GuestId` |
| Customize hardware | `-NumCpu`, `-MemoryGB`, `-DiskGB`, `-NetworkName` |

Prérequis : le réseau (module 2.3) et le stockage partagé (module 3.2) doivent exister. Sans eux, utilisez temporairement un port group standard (`VM Network`) et un datastore existant.

```powershell
Get-Datastore | Select-Object Name, Type, CapacityGB, FreeSpaceGB
```

Explication : choisissez un datastore **partagé** par les deux hôtes (NFS, VMFS iSCSI ou vSAN). Évitez un datastore local, qui bloquerait le vMotion.

Dans le lab HOL, seuls les datastores préconfigurés existent à ce stade : `ds-iscsi01` (VMFS, environ 85 Go dont 55 Go libres) et `ds-nfs01` (NFS, environ 5,8 Go libres seulement). Les datastores `ds-nfs-cli` et `ds-iscsi-cli` ne sont créés qu'au module 3 (3.2 et 3.3). On utilise donc **`ds-iscsi01`**, le seul assez grand pour héberger les VM de ce module. Ne choisissez pas `ds-nfs01` : trop petit pour la VM, ses fichiers de swap et les clones.

```powershell
$ds = Get-Datastore -Name "ds-iscsi01"
$vmPortGroup = Get-VDPortgroup -Name "PG-Serveurs-CLI" -ErrorAction SilentlyContinue
if (-not $vmPortGroup) { $vmPortGroup = Get-VirtualPortGroup -VMHost $h1 -Name "VM Network" }
```

Explication : sélectionne le datastore partagé et le port group. Le port group distribué `PG-Serveurs-CLI` n'existe qu'après le module 2.3 : s'il est absent (module 1 fait en premier), la deuxième ligne se rabat sur le port group standard `VM Network`, présent par défaut sur les hôtes. Dans les deux cas `$vmPortGroup.Name` fonctionne pour `-NetworkName`. La VM sera basculée sur le port group distribué plus tard (2.4). Si vous avez déjà fait le module 3, vous pouvez utiliser `ds-nfs-cli` ou `ds-iscsi-cli` à la place, à condition de vérifier l'espace libre avec la commande précédente.

```powershell
New-VM -Name "VM-vCenter-CLI" -VMHost $h1 -Datastore $ds -NumCpu 2 -MemoryGB 2 -DiskGB 20 -DiskStorageFormat Thin -NetworkName $vmPortGroup.Name -GuestId 'ubuntu64Guest'
```

Explication : crée une VM de 2 vCPU, 2 Go de RAM, disque de 20 Go en Thin Provision. Le format Thin est important dans le lab : un disque Thick pourrait saturer l'espace et empêcher la VM de démarrer. `-GuestId` indique le système invité pour optimiser les paramètres.

```powershell
Get-VM -Name "VM-vCenter-CLI" | Select-Object Name, PowerState, NumCpu, MemoryGB, VMHost, HardwareVersion
```

Explication : équivalent de l'onglet Summary de la VM : état, dimensionnement, hôte et version matérielle.

### 1.3.3 Modifier les paramètres de la VM (Edit Settings)

**Objectif** : ajouter une seconde carte réseau et modifier le matériel.

```powershell
New-NetworkAdapter -VM "VM-vCenter-CLI" -NetworkName $vmPortGroup.Name -Type Vmxnet3 -StartConnected
```

Explication : ajoute une seconde carte réseau de type VMXNET3 (carte paravirtualisée performante, nécessite VMware Tools), connectée au démarrage. Le cours la connecte ensuite à un autre réseau et la configure dans le système invité.

```powershell
Get-NetworkAdapter -VM "VM-vCenter-CLI" | Select-Object Name, Type, NetworkName, ConnectionState
```

Explication : liste les cartes réseau de la VM et leur port group.

```powershell
Set-VM -VM "VM-vCenter-CLI" -NumCpu 2 -MemoryGB 4 -Confirm:$false
```

Explication : modifie vCPU et mémoire. La VM doit être éteinte, sauf si l'ajout à chaud est activé.

### 1.3.4 Monter un ISO depuis la Content Library et installer Photon OS

**Objectif** : démarrer la VM sur un ISO de la Content Library (Photon OS) et installer le système.

**Définition** : la Content Library centralise ISO, templates et OVF. vCenter gère nativement leur montage, plus simplement que sur un ESXi autonome.

```powershell
Stop-VM -VM "VM-vCenter-CLI" -Confirm:$false -ErrorAction SilentlyContinue
```

Explication : éteint la VM avant de modifier lecteur CD et ordre de démarrage.

```powershell
$isoItem = Get-ContentLibraryItem -ItemType iso | Where-Object { $_.Name -match "photon" } | Select-Object -First 1
$isoItem | Select-Object Name, ContentLibrary
```

Explication : cherche l'ISO Photon OS dans la Content Library. Vérifiez qu'un élément est retourné.

```powershell
$cd = Get-CDDrive -VM "VM-vCenter-CLI"
if (-not $cd) { $cd = New-CDDrive -VM "VM-vCenter-CLI" }
```

Explication : récupère le lecteur CD de la VM, ou le crée s'il n'existe pas.

Le paramètre `Set-CDDrive -ContentLibraryIso` n'existe que dans les versions récentes de PowerCLI. Si vous obtenez `A parameter cannot be found that matches parameter name 'ContentLibraryIso'`, on contourne : une Content Library est stockée sous forme de fichiers sur un datastore, on retrouve donc le fichier ISO sur le datastore et on le monte avec `-IsoPath`.

```powershell
$isoFile = $null
foreach ($d in (Get-Datastore)) {
    New-PSDrive -Name "dsb" -PSProvider VimDatastore -Root "\" -Location $d | Out-Null
    $isoFile = Get-ChildItem -Path "dsb:\" -Recurse -Include "*photon*.iso" -ErrorAction SilentlyContinue | Select-Object -First 1
    Remove-PSDrive -Name "dsb" -Force
    if ($isoFile) { break }
}
$isoFile | Select-Object Name, DatastoreFullPath
```

Explication : parcourt chaque datastore comme un lecteur de fichiers (`dsb:`) et s'arrête au premier ISO dont le nom contient `photon`. `DatastoreFullPath` est le chemin au format `[datastore] dossier/fichier.iso`. Le parcours peut prendre quelques dizaines de secondes. Vérifiez qu'un fichier est retourné avant de continuer.

```powershell
Set-CDDrive -CD $cd -IsoPath $isoFile.DatastoreFullPath -StartConnected $true -Confirm:$false
```

Explication : monte l'ISO sur le lecteur CD, connecté au démarrage (la VM est éteinte, donc seul `-StartConnected` s'applique). Si votre version de PowerCLI accepte `-ContentLibraryIso`, vous pouvez à la place utiliser `Set-CDDrive -CD $cd -ContentLibraryIso $isoItem -StartConnected $true -Confirm:$false`. Dans le vSphere Client, le même résultat s'obtient par Edit Settings, CD/DVD drive, **Content Library ISO File**.

```powershell
Get-CDDrive -VM "VM-vCenter-CLI" | Select-Object IsoPath, ConnectionState, StartConnected
```

Explication : vérifie le montage.

```powershell
$vm = Get-VM -Name "VM-vCenter-CLI"
$spec = New-Object VMware.Vim.VirtualMachineConfigSpec
$spec.BootOptions = New-Object VMware.Vim.VirtualMachineBootOptions
$cdBoot = New-Object VMware.Vim.VirtualMachineBootOptionsBootableCdromDevice
$spec.BootOptions.BootOrder = @($cdBoot)
$vm.ExtensionData.ReconfigVM($spec)
```

Explication : définit via l'API l'ordre de démarrage sur le CD. Les cmdlets standard ne couvrent pas ce réglage, d'où l'objet `ConfigSpec`.

```powershell
Start-VM -VM "VM-vCenter-CLI" -Confirm:$false
Open-VMConsoleWindow -VM "VM-vCenter-CLI"
```

Explication : démarre la VM et ouvre sa console. Le manuel propose aussi la console web ou VMware Remote Console (VMRC), application séparée à installer. Si `Open-VMConsoleWindow` n'est pas disponible dans le terminal Linux, utilisez la console du vSphere Client.

Déroulement de l'installation de Photon OS (console) :

| Écran | Action |
| --- | --- |
| Boot screen | Choisir Install |
| License agreement | Accepter |
| Select disk | Sélectionner le disque virtuel et accepter le partitionnement automatique |
| Network configuration | Configuration réseau DHCP ou statique |
| Linux kernel | Choisir le noyau adapté à une VM (Linux for VMware) |
| Rename host | Définir le nom d'hôte |
| Password / Confirm | Définir et confirmer le mot de passe root |
| Confirm installation | Valider, l'installation se termine, redémarrer |

Après l'installation, déconnectez le CD pour éviter de redémarrer sur l'ISO. Le lecteur CD de la VM est sur un contrôleur **IDE** (`ide0:0`), dont les paramètres ne peuvent pas être modifiés VM allumée : il faut d'abord **éteindre la VM**.

```powershell
Stop-VM -VM "VM-vCenter-CLI" -Confirm:$false
```

Explication : éteint la VM (arrêt brutal, sans risque ici car l'installation est terminée). Si VMware Tools est installé, `Shutdown-VMGuest -VM "VM-vCenter-CLI" -Confirm:$false` fait un arrêt propre.

```powershell
Get-CDDrive -VM "VM-vCenter-CLI" | Set-CDDrive -NoMedia -Confirm:$false
```

Explication : retire le média du lecteur (équivalent de Disconnect CD/DVD dans Edit Settings). Le lecteur est éteint avec la VM, donc la modification est acceptée.

```powershell
Start-VM -VM "VM-vCenter-CLI"
```

Explication : redémarre la VM, qui démarre maintenant sur son disque et plus sur l'ISO.

```powershell
Get-VM "VM-vCenter-CLI" | Select-Object Name, @{N='Tools';E={$_.ExtensionData.Guest.ToolsStatus}}, @{N='OS';E={$_.ExtensionData.Guest.GuestFullName}}
```

Explication : vérifie l'état de VMware Tools et le système invité détecté. Un état `toolsNotInstalled` indique que les Tools sont absents.

---

## 1.4 Cloner des VM et utiliser des templates

**Objectif** : produire rapidement plusieurs VM identiques.

**Définition (manuel)** : VMware propose plusieurs méthodes de provisionnement. Un template est une copie maître d'une VM. Le **clone vers template** crée le modèle, le **déploiement depuis template** crée une VM à partir de lui. L'assistant de déploiement demande : nom et dossier, ressource de calcul, stockage, options de clonage (personnalisation du système, du matériel, mise sous tension), puis un récapitulatif.

```powershell
Stop-VM -VM "VM-vCenter-CLI" -Confirm:$false -ErrorAction SilentlyContinue
$vmFolder = Get-Folder -Name "vm" -Location $dc
New-Template -VM "VM-vCenter-CLI" -Name "TPL-Photon-CLI" -Location $vmFolder
```

Explication : arrête la VM source puis la clone en template dans le dossier `vm` du datacenter (équivalent de Clone to Template). Le template ne peut pas être démarré, c'est un modèle.

```powershell
Get-Template | Select-Object Name, Id
```

Explication : vérifie la présence du template dans la vue VMs and Templates.

```powershell
New-VM -Name "VM-Deploy-01" -Template (Get-Template -Name "TPL-Photon-CLI") -VMHost $h2 -Datastore $ds
```

Explication : déploie une VM à partir du template sur `esx-02a` (Deploy from Template).

```powershell
New-VM -Name "VM-Clone-01" -VM "VM-vCenter-CLI" -VMHost $h2 -Datastore $ds
```

Explication : clone directement une VM en une copie indépendante, sans passer par un template.

**Complément** : on peut aussi convertir une VM en template sans copie avec `Set-VM -VM <vm> -ToTemplate`, et revenir avec `Set-Template -Template <tpl> -ToVM`. La **personnalisation du système invité** (nom, réseau, domaine) utilise des spécifications de personnalisation (`Get-OSCustomizationSpec`) pour éviter des doublons de nom ou d'IP entre les clones.

```powershell
Get-Task | Select-Object -First 5 | Select-Object Name, State, PercentComplete, StartTime
```

Explication : suit l'avancement des tâches (équivalent de Recent Tasks).

---

## 1.5 Tags et recherche pour retrouver des objets

**Objectif** : organiser et retrouver rapidement les objets de l'inventaire.

**Définition (manuel)** : la barre de recherche du vSphere Client trouve un objet par son nom ou par des critères, filtre les résultats et permet d'enregistrer une recherche fréquente (par exemple « Not-Powered-On-VMs »). Les **tags**, classés par **catégories**, sont des étiquettes personnalisées qui complètent les attributs personnalisés. Une catégorie définit la cardinalité (une seule ou plusieurs valeurs par objet) et les types d'objets associables.

```powershell
Get-VM | Where-Object { $_.PowerState -ne "PoweredOn" } | Select-Object Name, PowerState
```

Explication : équivalent en ligne de commande de la recherche enregistrée « VM non allumées ».

```powershell
Get-VM -Name "VM-*" | Select-Object Name, PowerState, VMHost
```

Explication : recherche par nom avec joker, comme la barre de recherche.

```powershell
$cat = New-TagCategory -Name "Environnement" -Cardinality Single -EntityType VirtualMachine
$tag = New-Tag -Name "Production" -Category $cat
```

Explication : crée la catégorie (une seule valeur par VM, applicable aux VM) puis le tag `Production`.

```powershell
New-TagAssignment -Tag $tag -Entity (Get-VM -Name "VM-Deploy-01")
```

Explication : attribue le tag à une VM (Assign Tag dans l'interface).

```powershell
Get-VM -Tag $tag | Select-Object Name, PowerState, VMHost
```

Explication : retrouve toutes les VM portant ce tag, équivalent de la recherche par tag.

---

## 1.6 vSphere HA et DRS

**Objectif** : protéger les VM contre la panne d'un hôte et équilibrer automatiquement leur charge.

**Définition** : voir 1.0.9 et 1.0.10. Dans l'interface : Cluster, Configure, Services, vSphere Availability puis vSphere DRS.

### 1.6.1 Activer et configurer HA

```powershell
Set-Cluster -Cluster $cluster -HAEnabled:$true -HAAdmissionControlEnabled:$true -HAFailoverLevel 1 -Confirm:$false
```

Explication : active HA avec contrôle d'admission et tolérance d'un hôte en panne. L'interface du manuel propose aussi la définition de capacité par pourcentage de ressources du cluster (25 % CPU et mémoire). Avec deux hôtes, tolérer une panne revient à réserver environ 50 %.

```powershell
Set-Cluster -Cluster $cluster -HAIsolationResponse PowerOff -HARestartPriority Medium -Confirm:$false
```

Explication : règle la réponse à l'isolement (éteindre les VM d'un hôte isolé pour qu'elles redémarrent ailleurs) et la priorité de redémarrage par défaut.

Dans le vSphere Client, deux réglages sont à connaître : **VM Monitoring** sur « VM and Application Monitoring » (redémarrage si les heartbeats cessent pendant le délai fixé, 30 secondes par défaut) et **Heartbeat Datastores** sur « Automatically select datastores accessible from the hosts » (couche de protection supplémentaire en cas de partition du réseau de gestion). Trois tâches de configuration apparaissent dans Recent Tasks, puis le panneau vSphere HA de l'onglet Summary doit afficher **Protected**.

### 1.6.2 Activer et configurer DRS

```powershell
Set-Cluster -Cluster $cluster -DrsEnabled:$true -DrsAutomationLevel FullyAutomated -Confirm:$false
```

Explication : active DRS en mode entièrement automatisé. Valeurs possibles : `Manual`, `PartiallyAutomated`, `FullyAutomated`.

```powershell
Get-DrsRecommendation -Cluster $cluster
```

Explication : liste les recommandations DRS en attente (utile en mode manuel ou partiellement automatisé). Appliquez-en une avec `Apply-DrsRecommendation`.

Une règle DRS n'accepte que des VM **situées dans le cluster**. Vérifiez d'abord où elles se trouvent :

```powershell
Get-VM "VM-Deploy-01", "VM-Clone-01" | Select-Object Name, VMHost, @{N='Cluster';E={(Get-Cluster -VM $_).Name}}
Get-VMHost | Select-Object Name, @{N='Cluster';E={$_.Parent.Name}}
```

Explication : affiche le cluster et l'hôte de chaque VM, puis le cluster de chaque hôte. La colonne `Cluster` des deux VM doit afficher `Cluster-CLI`. Sinon, c'est que leur hôte est resté dans `RegionA01-COMP01` (1.3.1 non terminée pour cet hôte), ou que `$cluster` désigne un autre cluster du même nom (`(Get-Cluster -Name "Cluster-CLI").Count` doit valoir 1).

```powershell
Get-VM "VM-Deploy-01", "VM-Clone-01" | Where-Object { (Get-Cluster -VM $_).Name -ne "Cluster-CLI" } | Move-VM -Destination $h1 -Confirm:$false
```

Explication : si une VM est hors du cluster, la déplace sur `esx-01a`, à condition que cet hôte soit déjà dans `Cluster-CLI` (vérifié par la commande précédente). Sinon, terminez d'abord 1.3.1 : le second hôte (`esx-02a`) doit aussi être déplacé dans `Cluster-CLI`, après avoir migré ses VM vers `esx-01a`.

```powershell
New-DrsRule -Cluster $cluster -Name "Separer-VM" -KeepTogether $false -VM (Get-VM "VM-Deploy-01"), (Get-VM "VM-Clone-01") -Enabled $true
```

Explication : crée une règle d'anti-affinité : DRS maintiendra les deux VM sur des hôtes différents (utile pour deux serveurs redondants).

### 1.6.3 Vérifier l'état du cluster

```powershell
Get-Cluster -Name "Cluster-CLI" | Select-Object Name, HAEnabled, HAAdmissionControlEnabled, HAFailoverLevel, DrsEnabled, DrsAutomationLevel
```

Explication : confirme les réglages HA et DRS.

```powershell
Get-Cluster -Name "Cluster-CLI" | Get-VMHost | Select-Object Name, CpuUsageMhz, CpuTotalMhz, MemoryUsageGB, MemoryTotalGB
```

Explication : montre la charge de chaque hôte, équivalent du widget d'équilibre du cluster dans l'onglet Summary. Un écart important suggère un cluster déséquilibré.

---

## 1.7 vSphere Fault Tolerance

**Objectif** : comprendre la disponibilité continue (voir 1.0.11). Cette fonction dépend du matériel et peut ne pas être disponible dans un lab imbriqué.

Prérequis typiques : stockage partagé, disques en Thick Eager Zeroed, interface VMkernel dédiée au trafic de journalisation FT, hôtes compatibles. L'activation se fait dans le vSphere Client : clic droit sur la VM, Fault Tolerance, Turn On.

```powershell
Get-VM | Select-Object Name, @{N='FT';E={$_.ExtensionData.Runtime.FaultToleranceState}}
```

Explication : affiche l'état FT de chaque VM (`notConfigured`, `enabled`, `needSecondary`, etc.).

---

## 1.8 Surveiller les événements et créer des alarmes

**Objectif** : être informé ou agir automatiquement quand une condition survient.

**Définition** : voir 1.0.13. Dans le vSphere Client : menu, Events pour la console d'événements (trier par gravité, développer un événement pour ses détails), puis vCenter, Configure, Alarm Definitions.

```powershell
Get-VIEvent -MaxSamples 20 | Select-Object CreatedTime, UserName, FullFormattedMessage
```

Explication : affiche les 20 derniers événements, équivalent de la console Events.

```powershell
Get-VIEvent -Entity $cluster -MaxSamples 10 | Select-Object CreatedTime, FullFormattedMessage
```

Explication : limite les événements à un objet, ici le cluster.

```powershell
Get-AlarmDefinition | Select-Object Name, Enabled, Entity | Select-Object -First 15
```

Explication : liste les définitions d'alarmes par défaut. Celles définies au niveau vCenter sont héritées par tous les objets.

```powershell
Get-AlarmDefinition -Name "Host CPU usage" | Select-Object Name, Description, Enabled
```

Explication : retrouve l'alarme « Host CPU usage » utilisée dans l'exemple du manuel.

Configurer les notifications e-mail : vCenter doit connaître un serveur SMTP.

```powershell
Get-AdvancedSetting -Entity $global:DefaultVIServer -Name "mail.smtp.server" | Set-AdvancedSetting -Value "smtp.sddc.lab" -Confirm:$false
Get-AdvancedSetting -Entity $global:DefaultVIServer -Name "mail.sender"      | Set-AdvancedSetting -Value "vcenter@sddc.lab" -Confirm:$false
```

Explication : renseigne le serveur SMTP et l'expéditeur des e-mails d'alarme. Valeurs à adapter à votre lab.

Procédures du manuel (interface vSphere Client) :

**Modifier l'alarme « Host CPU usage »**

1. vCenter, Configure, Alarm Definitions, filtrer sur « CPU », sélectionner Host CPU usage, Edit.
2. Name and Targets : nom de l'alarme, objet concerné (Hosts) et portée.
3. Alarm Rule 1 : passer le seuil de 75 % à 80 % (déclenche un Warning).
4. Add Advanced Actions : choisir par exemple **Enter maintenance mode** pour mettre l'hôte en maintenance au déclenchement.
5. Vérifier le récapitulatif et valider.

**Créer une alarme « CPU Ready Time »**

1. Add Alarm Definition, nom et cible (Virtual Machines).
2. Trigger : métrique **CPU Ready** avec seuils Warning et Critical.
3. Add Advanced Actions (notification ou script) puis Review.
4. L'alarme apparaît dans la liste des définitions.

---

## 1.9 Configurer shares, limites, réservations et resource pools

**Objectif** : donner la priorité aux VM importantes en cas de contention.

**Définition** : voir 1.0.12.

```powershell
Get-VMResourceConfiguration -VM "VM-Deploy-01" | Select-Object VM, CpuSharesLevel, NumCpuShares, CpuReservationMhz, CpuLimitMhz, MemSharesLevel, MemReservationMB, MemLimitMB
```

Explication : affiche la configuration actuelle (équivalent de Edit Settings, section CPU). Les shares CPU sont par défaut à 1000 pour une VM à un vCPU (Normal).

```powershell
Get-VMResourceConfiguration -VM "VM-Deploy-01" | Set-VMResourceConfiguration -CpuSharesLevel High -MemSharesLevel High
```

Explication : passe les shares CPU et mémoire à High, comme dans l'exemple du manuel (la VM de production passe de 1000 à 2000 shares). L'effet n'existe qu'en cas de contention.

```powershell
Get-VMResourceConfiguration -VM "VM-Deploy-01" | Set-VMResourceConfiguration -CpuReservationMhz 500 -MemReservationMB 512
```

Explication : garantit 500 MHz de CPU et 512 Mo de mémoire à la VM. Ces ressources sont réservées même si la VM est inactive.

```powershell
Get-VMResourceConfiguration -VM "VM-Clone-01" | Set-VMResourceConfiguration -CpuLimitMhz 1000
```

Explication : plafonne la VM à 1000 MHz. À utiliser rarement : une limite peut brider la VM même si l'hôte est libre.

```powershell
$rp = New-ResourcePool -Location $cluster -Name "RP-Production" -CpuSharesLevel High -MemSharesLevel High
Move-VM -VM "VM-Deploy-01" -Destination $rp -Confirm:$false
```

Explication : crée un resource pool prioritaire et y place une VM, pour gérer les ressources par groupe plutôt que VM par VM.

---

## 1.10 Migrer des VM avec vMotion

**Objectif** : déplacer des VM allumées d'un hôte à l'autre sans coupure.

**Définition (manuel)** : vMotion et Storage vMotion permettent de déplacer dynamiquement des charges vers d'autres serveurs ou d'autres stockages sans interruption, ce qui supprime les fenêtres de maintenance. Prérequis : voir 1.0.8 (réseau 2.3, stockage 3.2).

Le manuel désactive DRS pour empêcher que les VM reviennent sur l'hôte d'origine pendant l'exercice, puis migre toutes les VM de `esx-01a` vers `esx-02a`.

```powershell
Set-Cluster -Cluster $cluster -DrsEnabled:$false -Confirm:$false
```

Explication : désactive DRS pour que l'équilibrage automatique n'annule pas la migration manuelle.

```powershell
Get-VMHost $h1 | Get-VM | Where-Object { $_.PowerState -eq "PoweredOff" } | Start-VM -Confirm:$false
```

Explication : met sous tension les VM éteintes, sinon l'étape suivante échouerait (le manuel l'indique).

```powershell
Get-VM -Name "VM-vCenter-CLI" | Select-Object Name, VMHost
```

Explication : affiche l'hôte actuel (attendu : `esx-01a`).

```powershell
Move-VM -VM "VM-vCenter-CLI" -Destination $h2 -VMotionPriority High -Confirm:$false
```

Explication : déclenche le vMotion vers `esx-02a`. Les types de migration du manuel (calcul, stockage, ou les deux) correspondent ici à `-Destination` (hôte ou cluster, DRS choisissant alors l'hôte), `-Datastore` (stockage) ou les deux. `-VMotionPriority` correspond à la priorité vMotion de l'assistant (la valeur par défaut convient la plupart du temps). Le réseau de la VM ne change pas dans un cas courant.

```powershell
Get-VMHost $h1 | Get-VM | Move-VM -Destination $h2 -Confirm:$false
```

Explication : migre en une commande toutes les VM de `esx-01a` vers `esx-02a`.

```powershell
Get-VM | Select-Object Name, VMHost, PowerState
```

Explication : confirme le changement d'hôte et que les VM sont toujours `PoweredOn`. Pour prouver la continuité, lancez un `ping` vers la VM pendant la migration.

```powershell
Set-Cluster -Cluster $cluster -DrsEnabled:$true -Confirm:$false
```

Explication : réactive DRS après l'exercice.

---

## 1.11 Surveillance et performances

**Objectif** : localiser la source d'un problème de performance et préparer des graphiques.

**Définition** : vSphere fournit des tableaux de bord et des graphiques de performance. Les quatre composantes de base sont CPU, mémoire, disque et réseau. Les graphiques temps réel se rafraîchissent toutes les 20 secondes.

Procédure du manuel (vSphere Client) : sélectionner l'hôte `esx-02a`, onglet Monitor, Performance Overview, plage Real-time. L'affichage Hôte puis Virtual Machines donne l'usage CPU par VM (une couleur par VM). Défiler pour voir mémoire, réseau (Mbps) et disque (Ko/s). Pour générer de l'activité, redémarrer toutes les VM de l'hôte, puis utiliser **Performance, Advanced**, **Chart Options**, type **Stacked Graph per VM**, sélection des objets, légende cliquable pour surligner un objet, bouton Export (image ou CSV) et, dans Chart metrics, ajouter pour la mémoire les compteurs **Active**, **Ballooned memory** et **Compressed**.

```powershell
Get-VMHost $h2 | Get-VM | Where-Object { $_.Name -notlike "vCLS*" -and $_.PowerState -eq "PoweredOn" } | ForEach-Object {
    if ($_.ExtensionData.Guest.ToolsRunningStatus -eq "guestToolsRunning") {
        Restart-VMGuest -VM $_ -Confirm:$false
    } else {
        Restart-VM -VM $_ -Confirm:$false
    }
}
```

Explication : redémarre toutes les VM allumées de l'hôte pour produire de la charge, en évitant les deux erreurs courantes. Le filtre `-notlike "vCLS*"` exclut les VM de service `vCLS`, gérées par vCenter : l'utilisateur n'a pas le droit de les redémarrer (`VirtualMachine.Interact.Reset` refusé). Ensuite, `ToolsRunningStatus` indique si VMware Tools tourne dans la VM. Si oui, `Restart-VMGuest` fait un redémarrage propre du système invité. Sinon, `Restart-VM` fait un redémarrage forcé (équivalent d'un reset), acceptable dans le lab pour générer de l'activité mais à éviter en production. Une VM qui s'éteint au lieu de redémarrer doit être rallumée à la main (`Start-VM`).

```powershell
Get-Stat -Entity $h2 -Stat cpu.usage.average -Realtime -MaxSamples 10 | Select-Object Timestamp, Value, Unit
```

Explication : relève 10 échantillons temps réel de l'usage CPU de l'hôte.

```powershell
Get-Stat -Entity (Get-VM "VM-vCenter-CLI") -Stat "mem.active.average","mem.vmmemctl.average","mem.compressed.average" -Realtime -MaxSamples 5 | Select-Object Timestamp, MetricId, Value, Unit
```

Explication : relève mémoire active, ballonnée (`vmmemctl`) et compressée d'une VM, les compteurs cités par le manuel. Une mémoire ballonnée ou compressée non nulle révèle une pression mémoire sur l'hôte.

```powershell
Get-Stat -Entity (Get-VM "VM-vCenter-CLI") -Stat "cpu.ready.summation" -Realtime -MaxSamples 5 | Select-Object Timestamp, Value, Unit
```

Explication : mesure le CPU Ready de la VM, indicateur de contention CPU.

```powershell
Get-Stat -Entity $h2 -Stat cpu.usage.average -Realtime -MaxSamples 30 | Export-Csv -Path "/home/holuser/cpu-esx02a.csv" -NoTypeInformation
```

Explication : exporte les mesures en CSV, équivalent du bouton Export du graphique.

---

## 1.12 Introduction à vSphere with Tanzu

Voir 1.0.16. Cette partie est présentée à titre de culture générale : aucune commande dans ce lab. À retenir : Kubernetes intégré à vSphere, administration via les objets vSphere, trois services développeurs (stockage, réseau, registre Harbor).

## 1.13 Parcours de certification

Les Hands-on Labs aident à préparer plusieurs certifications VMware. Ce lab prépare notamment la certification **VCAP-DCV Deploy 2023**, qui valide la capacité à déployer et optimiser des infrastructures VMware vSphere (informations sur le site VMware Learning).

---

## 1.14 Questions de révision du Module 1

1. **Quelle différence entre l'ESXi Host Client et le vSphere Client ?** L'Host Client gère un seul hôte. Le vSphere Client gère tout l'inventaire via vCenter et donne accès à HA, DRS, vMotion et VDS.
2. **Pourquoi un template est-il non démarrable ?** C'est une copie maître. On le déploie pour créer des VM, ce qui garantit des déploiements identiques.
3. **Quelle est la différence entre HA et FT ?** HA redémarre les VM sur un autre hôte après une panne (courte interruption). FT maintient une VM secondaire identique en parallèle (aucune interruption).
4. **Que fait DRS et quels sont ses trois niveaux d'automatisation ?** Il équilibre la charge et place les VM. Manuel, partiellement automatisé, entièrement automatisé.
5. **Quels sont les prérequis d'un vMotion ?** Stockage partagé, VMkernel vMotion sur chaque hôte, réseau de VM identique, CPU compatibles, même vCenter.
6. **Quand les shares s'appliquent-ils ?** Uniquement en cas de contention de la ressource.
7. **Qu'est-ce qu'une alarme et de quoi se compose-t-elle ?** Une notification déclenchée par un événement, une condition ou un état. Elle comporte nom, type d'objet, déclencheurs, seuils et actions.
8. **Que signifie un CPU Ready élevé ?** Les VM attendent un cœur physique libre : contention CPU.
9. **À quoi sert le mode maintenance ?** Mettre un hôte hors service pour intervention après migration ou arrêt de ses VM.
10. **Comment SSO évite-t-il de réauthentifier l'utilisateur à chaque service ?** Par un jeton SAML émis après la première authentification.

---

# MODULE 2 : Introduction au réseau et à la sécurité vSphere

## 2.0 Cours : ce que l'étudiant doit connaître

### 2.0.1 Pourquoi un réseau virtuel

Les VM doivent communiquer entre elles et avec le réseau physique. Cette connectivité passe par un **commutateur logique intégré à l'hyperviseur**. Historiquement, on utilisait un vSwitch standard configuré hôte par hôte. Le **vSphere Distributed Switch (VDS)** est devenu le type recommandé pour la plupart, voire tous les types de trafic entrant et sortant de l'hôte, car il se crée et se gère de façon centralisée dans vCenter et offre des fonctions avancées.

### 2.0.2 Vocabulaire réseau

| Terme | Définition |
| --- | --- |
| vNIC | Carte réseau virtuelle d'une VM (E1000, VMXNET3). |
| vmnic | Carte réseau physique de l'hôte (vmnic0, vmnic1, ...). |
| Uplink | Carte physique reliant un switch virtuel au réseau physique. |
| Port group | Ensemble de ports d'un switch avec une même politique, souvent un VLAN (Production, Développement, DMZ). Les vNIC se connectent aux port groups. |
| VMkernel adapter (vmk) | Interface IP de l'hôte pour ses services. |
| VLAN | Segmentation logique de niveau 2 par étiquette 802.1q, ID de 1 à 4094. |
| MTU | Taille maximale d'une trame. 1500 par défaut, plus de 1500 pour les jumbo frames. |

**Types d'interfaces VMkernel** (services activables) : gestion, vMotion, provisioning, FT logging, vSAN, réplication, stockage IP (NFS, iSCSI). Un VMkernel vMotion est un prérequis du vMotion. Une bonne pratique est de séparer ces trafics sur des VLAN ou des cartes distincts.

### 2.0.3 vSphere Standard Switch (VSS)

* Fonctionne comme un switch Ethernet physique : il sait quelles VM sont connectées à chaque port virtuel et transmet le trafic au bon port.
* Se connecte au réseau physique par des cartes physiques appelées **uplinks**.
* Sa configuration réside sur **chaque hôte**. L'administrateur doit maintenir manuellement la cohérence entre hôtes pour permettre des opérations comme vMotion.
* Sans uplink sur un port group, les VM du même port group communiquent entre elles sur cet hôte, mais pas avec le réseau externe.
* Chaque port logique appartient à un seul port group.

```
VM1 VM2         VMkernel           Hôte ESXi
 |   |             |
[Port group VM] [Port group Mgmt]
        \         /
        vSwitch0 (VSS)
            |
         vmnic0 (uplink)  ----->  switch physique
```

### 2.0.4 vSphere Distributed Switch (VDS)

* Se comporte comme **un seul switch** sur tous les hôtes associés d'un datacenter.
* Se configure dans vCenter et la configuration est **propagée** aux hôtes. Les VM conservent une configuration réseau cohérente en migrant.
* Un switch comporte deux plans : le **plan de données** (commutation, filtrage, étiquetage) et le **plan de gestion** (structure de contrôle pour configurer le plan de données).
* Sur un VSS, les deux plans résident sur chaque hôte. Sur un VDS, le plan de gestion est dans vCenter et le plan de données reste sur chaque hôte sous la forme d'un **host proxy switch**. La configuration créée dans vCenter est poussée automatiquement vers tous les host proxy switches.
* Deux abstractions assurent une configuration cohérente : les **uplink port groups** (pour les cartes physiques) et les **distributed port groups** (pour les VM et les VMkernel).

```
                vCenter (plan de gestion)
                       |
          +------------+------------+
          v                         v
   Host proxy switch         Host proxy switch   (plan de données)
      ESXi 1                    ESXi 2
```

### 2.0.5 Comparaison VSS et VDS

| Fonction | VSS | VDS |
| --- | --- | --- |
| Transfert de trames L2 | Oui | Oui |
| VLAN avec encapsulation 802.1q | Oui | Oui |
| Plusieurs uplinks (NIC teaming) | Oui | Oui |
| Traffic shaping sortant (TX) | Oui | Oui |
| Traffic shaping entrant (RX) | Non | Oui |
| Gestion centralisée par vCenter | Non | Oui |
| Private VLAN (PVLAN) | Non | Oui |
| Personnalisation des plans de données et de contrôle | Non | Oui |
| Network I/O Control, LACP, port mirroring, NetFlow | Non | Oui |

Améliorations de vSphere 6.x sur le VDS : Network I/O Control avec réservations de bande passante par VM, multicast snooping (IGMP pour IPv4, MLD pour IPv6), pile TCP/IP dédiée au vMotion avec sa propre passerelle.

Avec vSphere 8 et les DPU (vSphere Distributed Services Engine), les fonctions suivantes sont prises en charge par DPU : création et suppression du VDS, gestion de configuration, health check, LACP, port mirroring, VLAN privés, LLDP.

### 2.0.6 Politiques réseau à connaître

**Sécurité (switch ou port group)**

| Politique | Reject | Accept |
| --- | --- | --- |
| Promiscuous Mode | Une carte invitée en mode promiscuité ne voit pas de trames supplémentaires | La carte voit toutes les trames du switch autorisées par la politique VLAN du port group |
| MAC Address Changes | Si l'OS invité change la MAC par rapport au `.vmx`, les trames entrantes sont abandonnées | Le changement est pris en compte |
| Forged Transmits | Toute trame sortante dont la MAC source diffère est abandonnée | Aucun filtrage |

Bonne pratique : laisser les trois sur Reject sauf besoin précis (certains logiciels de réseau virtualisé).

**Traffic shaping** : défini par trois valeurs.

| Paramètre | Définition |
| --- | --- |
| Average bandwidth | Débit moyen autorisé (bits par seconde) |
| Peak bandwidth | Débit maximal lors d'une rafale |
| Burst size | Volume maximal d'une rafale (octets) |

Sur un VSS, ESXi ne façonne que le trafic **sortant**. Le VDS gère aussi l'entrant.

**Teaming et failover**

| Politique de répartition | Fonctionnement |
| --- | --- |
| Route based on the originating virtual port | Uplink choisi d'après l'ID du port virtuel. Un VM ou VMkernel garde toujours le même uplink. Par défaut. |
| Route based on IP hash | Uplink choisi d'après le hash des IP source et destination. Requiert EtherChannel (agrégation statique) sur le switch physique. |
| Route based on source MAC hash | Uplink choisi d'après le hash de la MAC source. |
| Route based on physical NIC load | VDS uniquement. Si un uplink reste à 75 % ou plus pendant 30 secondes, une partie du trafic est déplacée vers une carte moins chargée. |
| Use explicit failover order | Utilise le premier uplink actif qui répond aux critères de détection. Aucune répartition de charge. |

Les switchs virtuels ne répartissent que le trafic **sortant**. Le trafic entrant dépend de la politique du switch physique.

| Réglage | Définition |
| --- | --- |
| Network failure detection | **Link status only** : détecte câble débranché ou switch éteint. **Beacon probing** : envoie des paquets balises chaque seconde sur toutes les cartes de l'équipe, en plus du lien. Les cartes doivent être active ou standby. |
| Notify switches | Prévenir le switch physique en cas de basculement. |
| Failback | Si Oui (défaut), une carte réparée redevient active immédiatement et remplace la carte standby. Si Non, elle reste inactive jusqu'à la panne d'une carte active. |
| Failover order | Adaptateurs **actifs**, **en veille** (standby) ou **inutilisés** (unused). Les inutilisés servent à isoler certains trafics sur certains uplinks. |

**Options de port group distribué**

| Option | Valeurs |
| --- | --- |
| Port binding | **Static** : un port est attribué à la VM à sa connexion au port group. **Ephemeral** : pas de liaison de port, la VM peut se connecter même hôte déconnecté de vCenter. |
| Port allocation | **Elastic** (défaut) : 8 ports par défaut, un nouveau lot de 8 est créé quand ils sont tous attribués. **Fixed** : 8 ports, pas de création supplémentaire. |
| VLAN | None, VLAN (ID 1 à 4094), VLAN Trunking (plage), Private VLAN |
| Network resource pool | Pool utilisé avec Network I/O Control |

**Paramètres avancés du VDS** : MTU (au-delà de 1500 pour les jumbo frames, à valider avec l'équipe réseau, car toute la chaîne doit être compatible), mode de filtrage multicast (Basic ou IGMP/MLD snooping), protocole de découverte (CDP ou LLDP, en écoute, annonce ou les deux), contact de l'administrateur, nombre d'uplinks.

**Health Check du VDS** : surveille les changements de configuration (cohérence VLAN et MTU entre VDS et switch physique, et cohérence du teaming et du failover). Disponible depuis les VDS ESXi 5.1, il doit être activé explicitement.

### 2.0.7 Bonnes pratiques de migration vSS vers VDS

1. Créer le VDS et ses distributed port groups avec les mêmes VLAN que les port groups standards.
2. Ajouter les hôtes au VDS en assignant des uplinks libres.
3. Migrer les VM avec l'assistant **Migrate VMs to Another Network**, méthode recommandée par la documentation, plutôt que par l'assistant d'ajout d'hôtes.
4. Migrer ensuite les VMkernel (gestion en dernier).
5. Retirer les anciens switchs standards.

Ne jamais déplacer la carte qui porte la gestion tant que le VMkernel de gestion n'est pas migré, sous peine de perdre la connexion à l'hôte.

### 2.0.8 Sécurité de l'hôte

| Mesure | Description |
| --- | --- |
| **Lockdown Mode** | Restreint l'accès direct à l'hôte. Disponible uniquement pour les hôtes gérés par vCenter. |
| Mode Normal | L'hôte reste accessible via vCenter et la console locale (DCUI), pas en SSH. |
| Mode Strict | Seul vCenter accède à l'hôte. SSH et DCUI sont désactivés. |
| **Exception Users** | Comptes autorisés malgré le Lockdown (par exemple des applications tierces qui utilisent un accès distant). Ne pas en faire un moyen d'accès pour des utilisateurs. |
| Privilège DCUI Access | Les utilisateurs qui l'ont peuvent ouvrir la DCUI quand le Lockdown est actif. À la désactivation du Lockdown par la DCUI, ils reçoivent le rôle Administrator sur l'hôte. |
| Services et pare-feu | Les services (SSH, ESXi Shell, DCUI, NTP) et le pare-feu ESXi limitent les communications entrantes et sortantes. Désactiver tout ce qui est inutile. |

Précisions du manuel :

* Activer ou désactiver le Lockdown change **qui** peut accéder aux services, pas leur **disponibilité** : si SSH, ESXi Shell ou DCUI sont démarrés, ils continuent de tourner.
* Activer ou désactiver le Lockdown depuis la **DCUI** supprime les permissions attribuées aux utilisateurs et groupes sur l'hôte. Pour les conserver, utiliser le vSphere Client connecté à vCenter.
* On peut activer le Lockdown dans l'assistant d'ajout d'hôte, via le vSphere Client, ou via la DCUI.

### 2.0.9 Contrôle d'accès : rôles et permissions

| Notion | Définition |
| --- | --- |
| Privilège | Droit d'effectuer une action précise (par exemple Network.Assign). |
| Rôle | Ensemble de privilèges. |
| Permission | Triplet : utilisateur ou groupe, rôle, objet de l'inventaire (avec option de propagation aux enfants). |

Rôles système courants : **No Access**, **Read-only**, **Administrator**. Les autres sont des rôles d'exemple (par exemple Virtual Machine Power User, Resource Pool Administrator, Network Administrator, Datastore Consumer). Le manuel recommande de **créer ses propres rôles** adaptés aux besoins, en partant d'un rôle fourni ou d'un rôle vide auquel on ajoute les privilèges nécessaires.

Règles à connaître (complément) :

* Une permission définie sur un objet remplace la permission héritée d'un objet parent.
* Pour un même objet, une permission attribuée directement à un utilisateur l'emporte sur celles qui viennent de ses groupes.
* Les permissions de plusieurs groupes sur le même objet se cumulent.
* Principe du **moindre privilège** : n'accorder que le nécessaire.

Modifier un rôle change les privilèges de tous les utilisateurs et groupes qui le portent. En **Linked Mode** (plusieurs vCenter reliés), la création et la modification des rôles sont propagées à tous les vCenter du groupe, mais les **attributions** de rôles à des utilisateurs et objets ne le sont pas.

### 2.0.10 Single Sign-On en détail

* Le service SSO gère l'authentification et l'administration des utilisateurs. Son interface d'administration fait partie du vSphere Client. Il faut y être connecté avec un utilisateur qui a des privilèges d'administrateur SSO.
* Domaine par défaut : `vsphere.local`. Administrateur par défaut : `administrator@vsphere.local` (depuis vSphere 5.5, `admin@System-Domain` en 5.1).
* **Identity sources** : en général SSO utilise une source externe pour l'authentification primaire. Si la machine qui exécute SSO est jointe à un domaine Active Directory, la source de ce domaine est ajoutée automatiquement. Sources possibles : Active Directory (authentification Windows intégrée ou LDAP), OpenLDAP, et la source système `vsphere.local`. Une installation SSO a exactement **une** source d'identité système.
* Les utilisateurs et groupes de `vsphere.local` sont **internes à SSO**, distincts des comptes du système d'exploitation. Le nom d'un utilisateur ne peut pas être modifié après création. Prénom et nom sont facultatifs.
* **Groupe** : conteneur de membres appelés **principals**. Les membres peuvent être des utilisateurs ou d'autres groupes, issus de plusieurs sources, et on peut imbriquer des groupes. Les permissions appliquées à un groupe sont héritées par ses membres.
* **Verrouillage** : par défaut, un compte est verrouillé après trois tentatives de connexion échouées (la politique est désactivée dans le lab pour éviter les problèmes de claviers non américains). Un membre du groupe SSO Administrators peut le déverrouiller manuellement.
* **Expiration des mots de passe** : par défaut 90 jours (vSphere 6). Le mot de passe de `administrator@vsphere.local` doit être changé tous les 90 jours, sinon le compte est bloqué au 91e jour.
* Tout utilisateur peut changer son propre mot de passe SSO (menu du nom d'utilisateur, Change Password), même sans droits d'édition de son profil.

### 2.0.11 Points à retenir

* VSS : par hôte. VDS : centralisé, plan de gestion dans vCenter, plan de données sur chaque hôte.
* Port group = politique appliquée aux vNIC. Uplink = lien physique. VMkernel = services de l'hôte.
* Le VDS ajoute traffic shaping entrant, PVLAN, NIOC, LACP, health check.
* Lockdown Normal : vCenter et DCUI. Strict : vCenter seulement.
* Rôle = privilèges, permission = utilisateur + rôle + objet.
* SSO : domaine `vsphere.local`, jetons SAML, sources d'identité, groupes imbriquables.

---

## 2.1 Ajouter et configurer un vSwitch standard

Variables de session à rejouer si vous démarrez ici :

```powershell
$dc = Get-Datacenter -Name "RegionA01"
$h1 = Get-VMHost -Name 'esx-01a.vcf.sddc.lab'
$h2 = Get-VMHost -Name 'esx-02a.vcf.sddc.lab'
```

Explication : récupère le datacenter et les deux hôtes utilisés dans ce module.

### 2.1.1 Créer un vSwitch standard avec un port group de VM

**Objectif** : créer un switch standard sur un hôte, avec un port group de VM et une carte physique en uplink.

**Définition** : sur vCenter, Add Networking sur l'hôte propose trois types de connexion (VMkernel, port group de VM, adaptateur physique). Pour un port group de VM, on choisit un nouveau switch standard, on lui associe une carte physique en **Active adapters**, on nomme le port group (`VM Network 2`) et on laisse le VLAN ID sur None (0).

```powershell
Get-VMHostNetworkAdapter -VMHost $h2 -Physical | Select-Object Name, BitRatePerSec, Mac
```

Explication : liste les cartes physiques de l'hôte. Repérez une carte **libre** (le manuel utilise `vmnic3`). Dans un lab, les cartes disponibles peuvent différer.

```powershell
$vss = New-VirtualSwitch -VMHost $h2 -Name "vSwitch1" -Nic "vmnic3" -Mtu 1500
```

Explication : crée `vSwitch1` sur `esx-02a` avec `vmnic3` comme uplink actif et un MTU standard de 1500. Pour un switch sans lien physique, omettez `-Nic`.

```powershell
New-VirtualPortGroup -VirtualSwitch $vss -Name "VM Network 2" -VLanId 0
```

Explication : crée le port group de VM `VM Network 2`. `-VLanId 0` correspond à None. Les VM s'y connectent par leur vNIC.

```powershell
Get-VirtualSwitch -VMHost $h2 -Standard | Select-Object Name, Mtu, NumPorts, Nic
Get-VirtualPortGroup -VMHost $h2 -Standard | Select-Object Name, VLanId, VirtualSwitchName
```

Explication : vérifie la création, équivalent de l'affichage « Standard Switch: vSwitch1 » avec son port group et son uplink dans l'onglet Virtual switches.

### 2.1.2 Éditer les propriétés d'un switch standard

**Définition** : les paramètres du switch fixent les valeurs **par défaut** de tout le switch (MTU, sécurité, traffic shaping, teaming). Chaque port group peut les remplacer avec l'option Override.

```powershell
Set-VirtualSwitch -VirtualSwitch (Get-VirtualSwitch -VMHost $h2 -Name "vSwitch1") -Mtu 1500 -Confirm:$false
```

Explication : règle le MTU du switch. Pour utiliser des jumbo frames (par exemple 9000), il faut un MTU compatible sur tous les switchs virtuels et physiques et les équipements de bout en bout (hôtes, baies de stockage), sinon des problèmes de performance apparaissent. Consultez l'équipe réseau.

```powershell
Get-VirtualSwitch -VMHost $h2 -Name "vSwitch1" | Get-SecurityPolicy
```

Explication : affiche la politique de sécurité du switch (promiscuous, MAC changes, forged transmits).

```powershell
Get-VirtualSwitch -VMHost $h2 -Name "vSwitch1" | Get-SecurityPolicy | Set-SecurityPolicy -AllowPromiscuous $false -MacChanges $false -ForgedTransmits $false
```

Explication : applique la configuration la plus stricte (Reject partout) au niveau du switch.

```powershell
Get-VirtualPortGroup -VMHost $h2 -Name "VM Network 2" | Get-SecurityPolicy | Set-SecurityPolicy -ForgedTransmitsInherited $false -ForgedTransmits $true
```

Explication : remplace la politique du switch pour ce seul port group (équivalent de cocher Override dans l'interface), ici pour accepter les Forged Transmits.

Le **traffic shaping** (Average, Peak, Burst) se règle dans le vSphere Client, onglet Configure, Virtual switches, Edit settings du switch ou du port group, section Traffic shaping. Aucune modification n'est nécessaire dans cet exercice du manuel.

### 2.1.3 Ajouter une carte physique (NIC Teaming)

**Définition** : associer plusieurs cartes physiques à un switch augmente le débit et assure la redondance en cas de panne d'un lien. C'est le **NIC Teaming**.

```powershell
$nic = Get-VMHostNetworkAdapter -VMHost $h1 -Physical -Name "vmnic3"
Add-VirtualSwitchPhysicalNetworkAdapter -VirtualSwitch (Get-VirtualSwitch -VMHost $h1 -Name "vSwitch0") -VMHostPhysicalNic $nic -Confirm:$false
```

Explication : ajoute `vmnic3` comme uplink de `vSwitch0` sur `esx-01a`. Dans l'assistant, on la place sous **Active adapters** avec Move Down. Une carte peut aussi être mise en **Standby** (secours) ou **Unused** (réservée à d'autres port groups).

```powershell
Get-VirtualSwitch -VMHost $h1 -Name "vSwitch0" | Get-NicTeamingPolicy
```

Explication : affiche la politique de teaming (répartition de charge, détection de panne, failback, ordre des cartes).

```powershell
Get-VirtualSwitch -VMHost $h1 -Name "vSwitch0" | Get-NicTeamingPolicy | Set-NicTeamingPolicy -LoadBalancingPolicy LoadBalanceSrcId -NetworkFailoverDetectionPolicy LinkStatus -NotifySwitches $true -FailbackEnabled $true
```

Explication : règle les valeurs par défaut : répartition selon le port virtuel d'origine, détection par état du lien, notification des switchs physiques et failback activé. Autres valeurs de répartition : `LoadBalanceIP` (IP hash, exige EtherChannel), `LoadBalanceSrcMac`, `ExplicitFailover`.

```powershell
Get-VirtualPortGroup -VMHost $h1 -Name "VM Network" | Get-NicTeamingPolicy | Set-NicTeamingPolicy -InheritFailoverOrder $false -MakeNicActive "vmnic0" -MakeNicStandby "vmnic3"
```

Explication : remplace l'ordre de failover pour un port group : `vmnic0` actif, `vmnic3` en veille.

### 2.1.4 Retirer une carte et supprimer un switch

```powershell
Get-VMHostNetworkAdapter -VMHost $h1 -Physical -Name "vmnic3" | Remove-VirtualSwitchPhysicalNetworkAdapter -Confirm:$false
```

Explication : retire `vmnic3` de `vSwitch0` ; la carte repasse en **Unclaimed adapters**.

```powershell
Get-VirtualPortGroup -VMHost $h2 -Name "VM Network 2" | Remove-VirtualPortGroup -Confirm:$false
Get-VirtualSwitch -VMHost $h2 -Name "vSwitch1" | Remove-VirtualSwitch -Confirm:$false
```

Explication : supprime le port group puis le switch `vSwitch1` créé plus haut, en préparation de la leçon suivante, comme le fait le manuel.

**Conclusion** : le VSS est un switch simple, configuré et géré au niveau de l'hôte. Il assure l'accès, l'agrégation du trafic et la tolérance de panne via plusieurs cartes. Le VDS étend ces capacités en apparaissant comme un seul switch sur plusieurs hôtes : on configure une fois, la configuration se propage à tous les hôtes membres.

---

## 2.2 Explorer un vSphere Distributed Switch existant

**Objectif** : lire la configuration d'un VDS avant d'en construire un.

**Définition** : dans le vSphere Client, onglet Networking, un VDS expose Properties, Topology, Health check et ses port groups (dont les VMkernel gestion, stockage, vMotion, et les uplinks).

```powershell
Get-VDSwitch | Select-Object Name, Version, Mtu, NumUplinkPorts, LinkDiscoveryProtocol, LinkDiscoveryProtocolOperation, ContactName
```

Explication : affiche les propriétés de base du VDS : version, MTU, nombre d'uplinks, protocole de découverte (CDP ou LLDP) et contact.

```powershell
Get-VDSwitch | Get-VMHost | Select-Object Name, ConnectionState
```

Explication : liste les hôtes membres du VDS (onglet Hosts).

```powershell
Get-VDPortgroup | Select-Object Name, VDSwitch, VlanConfiguration, PortBinding, NumPorts
```

Explication : liste les port groups distribués et leur configuration (VLAN, liaison, nombre de ports).

```powershell
Get-VDPortgroup -Name "VM-RegionA01-vDS-COMP" -ErrorAction SilentlyContinue | Get-VM | Select-Object Name, VMHost
```

Explication : montre les VM connectées à un port group (adaptez le nom à votre lab), équivalent de la vue Topology où un chemin vers l'uplink est surligné pour une VM.

```powershell
Get-VDPortgroup | Get-VDUplinkTeamingPolicy | Select-Object VDPortgroup, LoadBalancingPolicy, FailoverDetectionPolicy
```

Explication : affiche la politique de teaming des port groups.

La vue **Topology** (Configure, Topology) du vSphere Client montre à gauche les port groups (VM et VMkernel gestion, stockage, vMotion) et à droite les uplinks reliés aux cartes physiques des hôtes. Elle ressemble à ce qu'on verrait sur un switch standard, mais définie une seule fois.

### Propriétés avancées et Health Check

**Définition** : l'édition des propriétés (Edit Settings) permet de modifier le nom, d'activer **Network I/O Control**, la description, le MTU, le filtrage multicast, le protocole de découverte, le contact administrateur et le nombre d'uplinks (avec renommage possible).

**Prérequis** : les trois blocs de cette sous-section (LLDP, Network I/O Control, Health Check) s'appliquent au VDS `VDS-CLI`, qui n'est créé qu'en **2.3.1**. Si vous suivez le README dans l'ordre, exécutez-les **après la création du VDS** (ou revenez-y une fois la 2.3.1 faite). Pour les tester avant, remplacez `VDS-CLI` par le nom d'un VDS existant (`Get-VDSwitch | Select-Object Name`).

```powershell
$vds = Get-VDSwitch -Name "VDS-CLI" -ErrorAction SilentlyContinue
if (-not $vds) { throw "VDS-CLI n'existe pas encore : créez-le d'abord (2.3.1), ou changez le nom ci-dessus." }
Set-VDSwitch -VDSwitch $vds -LinkDiscoveryProtocol LLDP -LinkDiscoveryProtocolOperation Both -ContactName "admin@sddc.lab" -Confirm:$false
```

Explication : récupère le VDS, s'arrête avec un message clair s'il est introuvable (sans ce test, `-ErrorAction SilentlyContinue` masque l'échec et `Set-VDSwitch` plante avec `The argument is null or empty`), puis active LLDP en écoute et annonce et renseigne le contact. Utile pour que le switch physique voisin identifie l'hôte. Saisissez l'adresse de contact sans espace (`admin@sddc.lab`).

```powershell
$vds.ExtensionData.EnableNetworkResourceManagement($true)
```

Explication : active **Network I/O Control** sur le VDS (réservations et limites de bande passante par type de trafic ou par VM).

Activer le Health Check (onglet Health check, Edit, Enabled pour VLAN et MTU, et pour Teaming and failover) :

```powershell
$hcVlan = New-Object VMware.Vim.VMwareDVSVlanMtuHealthCheckConfig -Property @{ Enable = $true; Interval = 1 }
$hcTeam = New-Object VMware.Vim.VMwareDVSTeamingHealthCheckConfig -Property @{ Enable = $true; Interval = 1 }
$vds.ExtensionData.UpdateDVSHealthCheckConfig_Task(@($hcVlan, $hcTeam))
```

Explication : active le Health Check pour VLAN et MTU et pour teaming et failover, avec un intervalle d'une minute. Cette fonction détecte des incohérences de configuration entre le VDS et le réseau physique. Le manuel constate qu'il est désactivé par défaut. Si l'appel échoue dans votre lab, utilisez l'interface.

---

## 2.3 Créer un vSphere Distributed Switch

**Objectif** : fournir une connectivité cohérente à tous les hôtes avec un VDS, un port group pour les VM et un VMkernel dédié au vMotion.

**Définition** : dans le vSphere Client, clic droit sur le datacenter, Distributed Switch, New Distributed Switch. L'assistant demande le nom, la **version** (8.0.0, ESXi 8.0 et ultérieur), les paramètres (nombre d'uplinks, port group par défaut, Network I/O Control), puis un récapitulatif. Ensuite, **Add and Manage Hosts** permet d'ajouter des hôtes, d'affecter des cartes physiques à des uplinks, de gérer les VMkernel et de migrer le réseau des VM.

### 2.3.1 Créer le VDS

```powershell
$vds = New-VDSwitch -Name "VDS-CLI" -Location $dc -NumUplinkPorts 4 -Version "8.0.0"
```

Explication : crée un Distributed Switch dans le datacenter `RegionA01` avec quatre ports d'uplink logiques, version 8.0.0. Adaptez la version à votre vCenter (`Get-VDSwitch` l'affiche).

### 2.3.2 Ajouter les hôtes

```powershell
Add-VDSwitchVMHost -VDSwitch $vds -VMHost $h1, $h2
```

Explication : associe les deux hôtes au VDS. La configuration leur est propagée. À ce stade, aucune carte physique n'est encore utilisée.

### 2.3.3 Affecter les cartes physiques aux uplinks

**Attention (lab VCF, à vérifier)** : les hôtes sont déjà reliés au VDS d'origine du lab, qui utilise probablement toutes leurs cartes physiques. Lancez la commande ci-dessous et ne raccordez que des cartes réellement libres. S'il n'y en a pas, passez cette étape : `VDS-CLI` fonctionnera pour des VM d'un même hôte, mais sans lien vers le réseau physique.

```powershell
Get-VMHostNetworkAdapter -VMHost $h1, $h2 -Physical | Select-Object VMHost, Name, BitRatePerSec
```

Explication : liste les cartes physiques. Repérez des cartes **non utilisées** par la gestion (le manuel assigne `vmnic3` à `Uplink1`).

```powershell
foreach ($h in @($h1, $h2)) {
  $n = Get-VMHostNetworkAdapter -VMHost $h -Physical -Name "vmnic3"
  Add-VDSwitchPhysicalNetworkAdapter -DistributedSwitch $vds -VMHostPhysicalNic $n -Confirm:$false
}
```

Explication : raccorde `vmnic3` de chaque hôte au VDS. Ne déplacez jamais la carte qui porte la gestion tant que le VMkernel de gestion n'est pas migré, vous perdriez la connexion à l'hôte. Passez cette étape si aucune carte libre n'est disponible dans votre lab imbriqué.

### 2.3.4 Créer les port groups distribués

```powershell
New-VDPortgroup -VDSwitch $vds -Name "PG-Serveurs-CLI" -VlanId 100 -PortBinding Static -NumPorts 16
```

Explication : crée le port group des VM, VLAN 100, liaison statique (un port est attribué à la VM à sa connexion). Ne jamais connecter une VM à un port group « Uplink » : c'est un groupe de liens physiques, pas un réseau de VM.

```powershell
New-VDPortgroup -VDSwitch $vds -Name "PG-vMotion-CLI" -VlanId 200 -PortBinding Static
```

Explication : port group réservé au trafic vMotion, isolé dans le VLAN 200.

```powershell
New-VDPortgroup -VDSwitch $vds -Name "PG-Trunk-CLI" -VlanTrunkRange "100-110"
```

Explication : exemple de port group en **VLAN Trunking** : il transporte une plage de VLAN (utile pour une VM de routage ou de pare-feu).

```powershell
New-VDPortgroup -VDSwitch $vds -Name "PG-Ephemeral-CLI" -VlanId 0 -PortBinding Ephemeral
```

Explication : exemple de liaison **Ephemeral** (sans liaison de port préalable).

### 2.3.5 Private VLAN (option)

```powershell
New-VDSwitchPrivateVlan -VDSwitch $vds -PrimaryVlanId 300 -SecondaryVlanId 300 -PrivateVlanType Promiscuous
New-VDSwitchPrivateVlan -VDSwitch $vds -PrimaryVlanId 300 -SecondaryVlanId 301 -PrivateVlanType Community
```

Explication : définit un VLAN privé : un VLAN primaire en mode promiscuité et un VLAN secondaire de type communauté. Seul le VDS supporte les PVLAN. Les VM d'un même VLAN communauté se parlent entre elles et avec le promiscuous, pas avec les autres communautés.

### 2.3.6 Créer les VMkernel vMotion

**Attention (lab VCF, à vérifier)** : le vMotion de `TinyLinux2` entre les deux hôtes a fonctionné sans configuration, ce qui indique que le VDS d'origine porte déjà des VMkernel vMotion. Vérifiez avec `Get-VMHostNetworkAdapter -VMHost $h1, $h2 -VMKernel | Select VMHost, Name, IP, VMotionEnabled`. Si `VMotionEnabled` vaut `True`, la création ci-dessous est facultative : un VMkernel sur `VDS-CLI` sans carte physique en uplink (2.3.3) ne pourrait de toute façon pas transporter de trafic entre hôtes.

Dans ce lab, les hôtes ont déjà une **pile TCP/IP vMotion dédiée** (netstack `vmotion`), utilisée par le VDS d'origine. Quand cette pile existe, un VMkernel de vMotion doit être créé **directement sur cette pile** : ni `-VMotionEnabled` ni l'ajout du tag `VMotion` ne fonctionnent sur la pile par défaut (erreurs `The VirtualNic capability 'vmotion' is not supported` puis `Failed to add VMotion tag when dedicated stack is instantiated`).

Si vous avez déjà créé un VMkernel sans cette pile (cas de `vmk3`), supprimez-le d'abord :

```powershell
foreach ($h in $h1, $h2) {
    Get-VMHostNetworkAdapter -VMHost $h -VMKernel | Where-Object { $_.PortGroupName -eq "PG-vMotion-CLI" } | Remove-VMHostNetworkAdapter -Confirm:$false
}
```

Explication : retrouve sur chaque hôte le VMkernel rattaché à `PG-vMotion-CLI` et le supprime pour repartir proprement. Sans effet si aucun n'existe.

Puis créez-le sur la pile `vmotion` :

```powershell
$stack1 = Get-VMHostNetworkStack -VMHost $h1 -Name "vmotion"
$stack2 = Get-VMHostNetworkStack -VMHost $h2 -Name "vmotion"
New-VMHostNetworkAdapter -VMHost $h1 -VirtualSwitch $vds -PortGroup "PG-vMotion-CLI" -IP 10.0.50.11 -SubnetMask 255.255.255.0 -NetworkStack $stack1
New-VMHostNetworkAdapter -VMHost $h2 -VirtualSwitch $vds -PortGroup "PG-vMotion-CLI" -IP 10.0.50.12 -SubnetMask 255.255.255.0 -NetworkStack $stack2
```

Explication : récupère la pile TCP/IP `vmotion` de chaque hôte, puis crée un VMkernel dessus. Une interface placée sur cette pile transporte le trafic vMotion sans autre réglage (pas de `-VMotionEnabled`). Les IP sont des exemples : dans un même sous-réseau, joignables entre elles, et **différentes de celles déjà utilisées** par les VMkernel vMotion existants du lab. Le manuel ne se sert pas de ce VDS (« we will not be using this distributed switch for any VMkernel functions »), mais le vMotion en a besoin. Si `-NetworkStack` est refusé par votre version de PowerCLI, créez le VMkernel dans le vSphere Client (hôte, Configure, VMkernel adapters, Add Networking, VMkernel Network Adapter, port group `PG-vMotion-CLI`, **TCP/IP stack : vMotion**).

Vérification de la pile utilisée :

```powershell
foreach ($h in $h1, $h2) {
    (Get-EsxCli -VMHost $h -V2).network.ip.interface.list.Invoke() | Select-Object Name, NetstackInstance, PortgroupName
}
```

Explication : liste les VMkernel de l'hôte avec la pile TCP/IP à laquelle ils appartiennent. Le nouveau VMkernel doit apparaître avec `NetstackInstance = vmotion`.

### 2.3.7 Vérifier

```powershell
Get-VDSwitch
Get-VDPortgroup -VDSwitch $vds | Select-Object Name, VlanConfiguration, PortBinding
Get-VMHostNetworkAdapter -VMHost $h1, $h2 -VMKernel | Select-Object VMHost, Name, IP, VMotionEnabled, ManagementTrafficEnabled
```

Explication : affiche le VDS, ses port groups avec VLAN et liaison, et les interfaces VMkernel avec leur usage. Dans le vSphere Client, l'onglet Hosts et la Topology confirment les hôtes et les uplinks.

---

## 2.4 Migrer des VM d'un switch standard vers un VDS

**Objectif** : déplacer une VM en fonctionnement d'un port group standard vers un port group distribué.

**Définition** : le manuel utilise l'assistant **Migrate VMs to Another Network** (clic droit sur le datacenter) : choisir le réseau source (`VM Network`), le réseau destination (port group distribué), les VM et leur adaptateur (si une VM a plusieurs cartes, on choisit laquelle migrer), puis Finish. C'est la méthode recommandée par la documentation.

```powershell
Get-VM | Get-NetworkAdapter | Select-Object @{N='VM';E={$_.Parent.Name}}, Name, NetworkName
```

Explication : liste les cartes réseau des VM et leur réseau actuel.

```powershell
$pg = Get-VDPortgroup -Name "PG-Serveurs-CLI"
Get-VM -Name "VM-Clone-01" | Get-NetworkAdapter | Where-Object { $_.NetworkName -eq "VM Network" } | Set-NetworkAdapter -Portgroup $pg -Confirm:$false
```

Explication : bascule la carte réseau de la VM du port group standard `VM Network` vers le port group distribué, VM allumée. La connectivité doit être maintenue : lancez un `ping` continu pendant l'opération.

```powershell
Get-VDPortgroup -Name "PG-Serveurs-CLI" | Get-VM | Select-Object Name, VMHost
```

Explication : vérifie que la VM apparaît sur le port group distribué (visible aussi dans la Topology du VDS).

---

## 2.5 Utiliser le Lockdown Mode

**Objectif** : réduire la surface d'attaque des hôtes en interdisant l'accès direct.

**Définition** : voir 2.0.8. Dans l'interface : hôte, Configure, Security Profile, Lockdown Mode, Edit. Le manuel active d'abord le mode **Normal** sur `esx-01a` (accès via vCenter et DCUI, plus de SSH), vérifie qu'un SSH direct est refusé, puis désactive, puis refait avec le mode **Strict** (vCenter seul, SSH et DCUI coupés).

Avant d'activer : vérifier que le service SSH tourne.

```powershell
Get-VMHostService -VMHost $h1 | Where-Object { $_.Key -in @("TSM-SSH","TSM") } | Select-Object Key, Label, Running
```

Explication : affiche l'état de SSH (`TSM-SSH`) et d'ESXi Shell (`TSM`). Cela confirme que le Lockdown change qui peut se connecter, pas l'état des services.

```powershell
$am = Get-View ($h1.ExtensionData.ConfigManager.HostAccessManager)
$am.LockdownMode
```

Explication : récupère le gestionnaire d'accès de l'hôte et affiche le mode courant (`lockdownDisabled` par défaut).

```powershell
$am.ChangeLockdownMode("lockdownNormal")
```

Explication : active le mode **Normal**. Un SSH direct vers `esx-01a` échoue, vCenter continue à gérer l'hôte. Activer depuis vCenter préserve les permissions locales. La commande n'affiche rien quand elle réussit.

```powershell
$am.UpdateViewData()
$am.LockdownMode
```

Explication : l'objet `$am` est une copie prise avant le changement, donc `$am.LockdownMode` afficherait encore `lockdownDisabled`. `UpdateViewData()` le rafraîchit : on doit lire `lockdownNormal`.

**Prérequis** : un Exception User doit **déjà exister** comme compte local de l'hôte (ou utilisateur Active Directory). Un nom inventé provoque `The user or group named '...' does not exist`. Dans le lab, il n'y a pas de compte `svc-sauvegarde` : utilisez un compte existant comme `root`. Pour créer un vrai compte de service, faites-le **avant** d'activer le Lockdown, dans l'Host Client (Manage, Security & users, Users) ou par connexion directe à l'hôte (`Connect-VIServer esx-01a... -User root`, puis `New-VMHostAccount -Id svc-sauvegarde -Password <mot de passe> -UserAccount`). Une fois le Lockdown actif, la connexion directe à l'hôte n'est plus possible.

```powershell
$am.UpdateLockdownExceptions(@("root"))
$am.QueryLockdownExceptions()
```

Explication : définit les **Exception Users**, comptes autorisés malgré le Lockdown (par exemple pour une application de sauvegarde tierce), puis les affiche : `root` doit apparaître. Ici `root` sert d'exemple de compte existant. En production, on y met un compte de service dédié, jamais `root`. Cette liste ne doit pas servir à donner un accès à des utilisateurs. Pour la vider ensuite : `$am.UpdateLockdownExceptions(@())`.

```powershell
Get-AdvancedSetting -Entity $h1 -Name "DCUI.Access"
```

Explication : affiche la liste des utilisateurs avec le privilège **DCUI Access**, autorisés à ouvrir la console locale quand le Lockdown est actif.

```powershell
$am.ChangeLockdownMode("lockdownDisabled")
```

Explication : désactive le mode et revient à la normale.

```powershell
$am.ChangeLockdownMode("lockdownStrict")
$am.ChangeLockdownMode("lockdownDisabled")
```

Explication : active le mode **Strict** (seul vCenter accède à l'hôte), puis le désactive immédiatement. Dans le lab, n'attendez pas : une erreur en Strict peut vous priver de tout accès direct si vCenter devient indisponible.

---

## 2.6 Services de l'hôte et pare-feu

**Objectif** : limiter les services et ports exposés par l'hôte.

**Définition** : les services de l'hôte (SSH, ESXi Shell, DCUI, NTP) se règlent dans Configure, Services. Le pare-feu ESXi bloque ou autorise les communications entrantes et sortantes par règle (service). Pour permettre à l'hôte de s'authentifier avec des identifiants Active Directory, on joint l'hôte au domaine (vidéo facultative du manuel).

```powershell
Get-VMHostService -VMHost $h1 | Select-Object Key, Label, Running, Policy
```

Explication : liste services, état et politique de démarrage (On, Off, Automatic).

```powershell
Get-VMHostService -VMHost $h1 | Where-Object { $_.Key -eq "TSM-SSH" } | Set-VMHostService -Policy Off | Stop-VMHostService -Confirm:$false
```

Explication : coupe SSH et empêche son démarrage automatique. Bonne pratique : SSH désactivé hors besoin.

```powershell
Get-VMHostFirewallException -VMHost $h1 | Where-Object { $_.Enabled } | Select-Object Name, Enabled, IncomingPorts, OutgoingPorts, Protocols
```

Explication : liste les règles actives avec leurs ports et protocoles.

```powershell
Get-VMHostService -VMHost $h1 | Where-Object { $_.Key -eq "TSM-SSH" } | Select-Object Key, Running, Policy
Get-VMHostFirewallException -VMHost $h1 -Name "SSH Server" | Set-VMHostFirewallException -Enabled $false
```

Explication : la première ligne vérifie que le service SSH est bien arrêté (`Running = False`). C'est un prérequis : tant que le service `TSM-SSH` tourne, ESXi considère la règle de pare-feu du serveur SSH comme requise et refuse de la désactiver (`Cannot change the host configuration. Required ruleset cannot be disabled`). Si `Running` vaut `True`, relancez d'abord la commande d'arrêt du service ci-dessus. La seconde ligne désactive alors la règle de pare-feu du serveur SSH. Si l'erreur persiste malgré le service arrêté, ce n'est pas bloquant : un service SSH arrêté avec une politique `Off` suffit à fermer l'accès, la règle de pare-feu n'est qu'une couche supplémentaire.

```powershell
Get-VMHostFirewallDefaultPolicy -VMHost $h1
```

Explication : affiche la politique par défaut du pare-feu pour le trafic entrant et sortant (non autorisé par défaut).

```powershell
Get-VMHostAuthentication -VMHost $h1 | Set-VMHostAuthentication -JoinDomain -Domain "corp.local" -Username "svc-join" -Password "MotDePasse!" -Confirm:$false
```

Explication : joint l'hôte à un domaine Active Directory pour authentifier les administrateurs avec leurs identifiants AD. Exemple facultatif : il exige un domaine disponible, absent du lab par défaut.

---

## 2.7 Accès utilisateur : rôles et permissions

**Objectif** : appliquer le moindre privilège, par exemple pour un prestataire qui ne fait que du réseau (exemple du manuel).

**Définition** : voir 2.0.9. Interface : Menu, Administration, Access Control, Roles. Le manuel montre qu'on peut partir d'un rôle fourni ou créer un rôle sans permission et n'y ajouter que le nécessaire. Exemple : nouveau rôle `Role1`, catégorie **Network**, case Select all, Create. Puis **Edit** pour ajouter les privilèges d'accès à d'autres objets quand le rôle doit être étendu.

```powershell
Get-VIRole | Select-Object Name, IsSystem
```

Explication : liste les rôles existants, système (`IsSystem = True`) et d'exemple.

```powershell
$netPrivs = Get-VIPrivilege | Where-Object { $_.Id -like "Network.*" }
$role = New-VIRole -Name "Role-Reseau-CLI" -Privilege $netPrivs
```

Explication : récupère tous les privilèges de la catégorie Network et crée un rôle qui les contient (équivalent de cocher Select all sous Network).

```powershell
Set-VIRole -Role $role -AddPrivilege (Get-VIPrivilege -Id "Datastore.Browse")
```

Explication : étend le rôle avec un privilège supplémentaire (Edit Role). Les modifications s'appliquent à tous les utilisateurs et groupes portant ce rôle. En Linked Mode, elles se propagent aux autres vCenter.

```powershell
Get-VIPrivilege -Role $role | Select-Object Id | Select-Object -First 10
```

Explication : affiche les privilèges contenus dans le rôle.

**Prérequis** : l'utilisateur `prestataire-reseau` doit exister **avant** de lui attribuer une permission. Il est créé en 2.8.2, **dans le vSphere Client** (Administration, Single Sign On, Users and Groups, Add). Si vous suivez le document dans l'ordre, faites d'abord 2.8.2 puis revenez ici. Sans cela, `New-VIPermission` échoue avec `Could not find VIAccount with name ...` (voir Annexe B).

```powershell
$dc = Get-Datacenter -Name "RegionA01"
Get-VIAccount -User | Where-Object { $_.Name -like "*prestataire*" }
```

Explication : définit `$dc` (le datacenter du lab) et vérifie que vCenter voit bien le compte. On filtre avec `Where-Object` plutôt que d'utiliser `-Name` et `-Domain` ensemble, car cette combinaison provoque `Parameter set cannot be resolved` (voir Annexe B). Si la seconde commande ne renvoie rien, le compte n'existe pas encore (ou pas sous ce nom) : créez-le en 2.8.2. Cette vérification est facultative : `New-VIPermission` échoue de toute façon proprement si le compte est introuvable.

```powershell
New-VIPermission -Entity $dc -Principal "VSPHERE.LOCAL\prestataire-reseau" -Role $role -Propagate:$true
```

Explication : attribue le rôle sur le datacenter `RegionA01` et ses enfants (`-Propagate`). Attention à la syntaxe : `-Entity` et `-Principal` prennent un tiret, le nom du domaine est écrit sans espace (`VSPHERE.LOCAL\prestataire-reseau`) et le `$` de `$true` se colle aux deux-points (`-Propagate:$true`). Il peut gérer le réseau et rien d'autre.

```powershell
Get-VIPermission -Entity $dc | Select-Object Principal, Role, Propagate, IsGroup
```

Explication : vérifie les permissions posées sur le datacenter.

```powershell
Get-VIPermission -Entity $dc | Where-Object { $_.Principal -like "*prestataire*" } | Remove-VIPermission -Confirm:$false
```

Explication : supprime la permission (nettoyage ou changement d'équipe).

---

## 2.8 Single Sign-On

**Méthode retenue dans le lab : l'interface graphique.** Le module PowerShell `VMware.vSphere.SsoAdmin` n'est pas disponible dans le lab HOL (pas d'accès à la PowerShell Gallery). La création des utilisateurs et des groupes SSO se fait donc dans le **vSphere Client** (Menu, Administration, Single Sign On, Users and Groups), comme dans le manuel. Les commandes PowerShell ci-dessous sont données en **variante**, pour un environnement où le module est installé.

**Prérequis pour la variante PowerShell : le module SSO.** Les cmdlets `*-Sso*` ne sont pas dans PowerCLI de base : elles viennent du module séparé `VMware.vSphere.SsoAdmin`. Vérifiez d'abord s'il est présent, puis importez-le :

```powershell
Get-Module -ListAvailable VMware.vSphere.SsoAdmin
Import-Module VMware.vSphere.SsoAdmin
```

Explication : la première commande liste le module s'il est installé (aucun résultat = absent). La seconde le charge dans la session. Si le module est absent, essayez `Install-Module VMware.vSphere.SsoAdmin -Scope CurrentUser`. Cela demande un accès à la PowerShell Gallery, **souvent bloqué dans le lab HOL** (pas d'accès Internet). Dans ce cas, créez l'utilisateur et le groupe avec le vSphere Client (Menu, Administration, Single Sign On, Users and Groups, onglet Users, Add), comme dans le manuel, puis reprenez les commandes PowerCLI qui ne dépendent pas du module (2.7).

Variables de session : ouvrez d'abord la session d'administration SSO.

```powershell
Connect-SsoAdminServer -Server "10.0.0.22" -User "administrator@vsphere.local" -Password "VMware123!"
```

Explication : ouvre une session sur l'administration SSO, nécessaire pour gérer les utilisateurs du domaine. Équivalent de Menu, Administration, Single Sign On, Users and Groups. Le compte doit être administrateur SSO.

### 2.8.1 Voir les sources d'identité

**Définition** : SSO s'appuie en général sur une source externe. Dans le lab, la source Active Directory affiche « The node didn't join any Active Directory » car aucune source AD n'est incluse.

```powershell
Get-IdentitySource
```

Explication : liste les sources d'identité configurées (au minimum `vsphere.local`). Pour en ajouter, voir `Add-ActiveDirectoryIdentitySource` et `Add-LDAPIdentitySource`.

### 2.8.2 Créer un utilisateur SSO

**Définition** : les utilisateurs de l'onglet Users sont internes à SSO, dans le domaine `vsphere.local`, et diffèrent des comptes du système d'exploitation.

**Méthode dans le lab : vSphere Client (interface graphique)**

1. Connectez-vous au vSphere Client en `administrator@vsphere.local`.
2. Menu, **Administration**, **Single Sign On**, **Users and Groups**.
3. Choisissez le domaine `vsphere.local`, onglet **Users**, puis **Add**.
4. Renseignez : Username `prestataire-reseau`, Password `VMware123!VMware123!` (à confirmer), First Name `Prestataire`, Last Name `Reseau`, Email `prestataire@vsphere.local`, Description `Prestataire reseau (lab)`.
5. Cliquez sur **Add** pour valider. L'utilisateur apparaît dans la liste.

Explication : le mot de passe doit respecter la politique de complexité. Le nom d'utilisateur ne pourra plus être modifié après création. Le manuel crée un utilisateur `holadmin` de la même façon.

**Vérification dans PowerCLI** (fonctionne sans le module SSO, avec une session `Connect-VIServer` ouverte) :

```powershell
Get-VIAccount -User | Where-Object { $_.Name -like "*prestataire*" }
```

Explication : si le compte est renvoyé, vCenter le voit et vous pouvez attribuer le rôle avec `New-VIPermission` (section 2.7). Si rien n'est renvoyé, vérifiez l'orthographe du nom et le domaine.

**Variante PowerShell** (uniquement si le module `VMware.vSphere.SsoAdmin` est disponible) :

```powershell
New-SsoPersonUser -UserName "prestataire-reseau" -Password "VMware123!VMware123!" -FirstName "Prestataire" -LastName "Reseau" -EmailAddress "prestataire@vsphere.local" -Description "Prestataire reseau (lab)"
```

Explication : crée l'utilisateur. Le mot de passe doit respecter la politique de complexité. Le nom d'utilisateur ne pourra plus être modifié. Le manuel crée un utilisateur `holadmin` de la même façon.

```powershell
Get-SsoPersonUser -Domain "vsphere.local" | Select-Object Name, FirstName, LastName, Disabled, Locked
```

Explication : liste les utilisateurs du domaine, avec leur état désactivé ou verrouillé.

### 2.8.3 Créer un groupe et y ajouter un membre

**Définition** : un groupe contient des principals (utilisateurs ou groupes). Les membres héritent des permissions du groupe.

**Méthode dans le lab : vSphere Client (interface graphique).** Les cmdlets de groupe SSO (`New-SsoGroup`, `Add-UserToSsoGroup`...) viennent du module `VMware.vSphere.SsoAdmin`, absent du lab : tout se fait donc dans l'interface.

**Créer le groupe**

1. Menu, **Administration**, **Single Sign On**, **Users and Groups**.
2. Domaine `vsphere.local`, onglet **Groups**, puis **Add Group**.
3. Renseignez : Group Name `Groupe-Reseau`, Description `Equipe reseau`.
4. Cliquez sur **Add** pour valider. Le groupe apparaît dans la liste.

**Y ajouter un membre**

1. Dans l'onglet Groups, cliquez sur `Groupe-Reseau`.
2. Dans la zone **Group Members**, cliquez sur **Add Members**.
3. Domaine `vsphere.local`, recherchez `prestataire-reseau`, sélectionnez-le.
4. Cliquez sur **Save**. L'utilisateur est maintenant membre du groupe et hérite de ses permissions.

Explication : équivalent de saisir `holadmin` dans le champ membre du manuel.

**Ajouter un groupe dans le groupe Administrators (exemple du manuel)**

**Définition** : le groupe SSO **Administrators** donne les droits d'administration sur SSO (gérer les utilisateurs, groupes, politiques). Tout groupe placé dedans, et donc tous ses membres, devient administrateur SSO. Le manuel y ajoute le groupe `HOL Group`.

**Prérequis** : le groupe à ajouter doit déjà exister (le manuel crée `HOL Group` comme `Groupe-Reseau` ci-dessus, avec l'utilisateur `holadmin` comme membre).

1. Connectez-vous au vSphere Client en `administrator@vsphere.local` (ou avec un compte déjà membre d'Administrators).
2. Menu, **Administration**, **Single Sign On**, **Users and Groups**.
3. Domaine `vsphere.local`, onglet **Groups**.
4. Cliquez sur le groupe **Administrators** pour l'ouvrir.
5. Dans la zone **Group Members**, cliquez sur **Add Members**.
6. Dans la fenêtre qui s'ouvre, choisissez le domaine `vsphere.local`.
7. Saisissez le nom du groupe dans la zone de recherche (par exemple `HOL Group`), puis sélectionnez-le dans la liste.
8. Cliquez sur **Save**. Le groupe apparaît dans les membres d'Administrators.

Explication : le groupe imbriqué hérite des droits d'administration SSO, et ses membres aussi. Ne le faites pas pour `Groupe-Reseau` : il est prévu pour un prestataire qui ne doit gérer que le réseau (moindre privilège, voir 2.7).

**Retirer un groupe d'Administrators (nettoyage)** : ouvrez Administrators, cochez le groupe dans Group Members, **Remove Member**, confirmez.

**Variante PowerShell** : l'imbrication de groupes passe par `Add-GroupToSsoGroup`, du module `VMware.vSphere.SsoAdmin` (absent du lab). Dans le lab, utilisez l'interface.

**Donner le rôle réseau au groupe plutôt qu'à l'utilisateur (facultatif)** : cette commande est du PowerCLI standard, elle ne demande pas le module SSO. Elle suppose que `$dc` et `$role` sont définis (voir 2.7).

```powershell
New-VIPermission -Entity $dc -Principal "VSPHERE.LOCAL\Groupe-Reseau" -Role $role -Propagate:$true
Get-VIPermission -Entity $dc | Select-Object Principal, Role, Propagate, IsGroup
```

Explication : attribue le rôle au groupe, donc à tous ses membres, puis vérifie. `IsGroup` vaut `True` pour cette ligne. Cette commande n'a pas été testée dans le lab.

**Variante PowerShell** (uniquement si le module `VMware.vSphere.SsoAdmin` est installé) :

```powershell
$grp = New-SsoGroup -GroupName "Groupe-Reseau" -Description "Equipe reseau"
$usr = Get-SsoPersonUser -Name "prestataire-reseau" -Domain "vsphere.local"
Add-UserToSsoGroup -TargetGroup $grp -User $usr
```

Explication : crée le groupe puis y ajoute l'utilisateur. Selon la version du module, la syntaxe de `Add-UserToSsoGroup` peut différer : vérifiez avec `Get-Help Add-UserToSsoGroup -Full`.

### 2.8.4 Verrouillage, déverrouillage et mots de passe

**Méthode dans le lab : vSphere Client (interface graphique).** Les cmdlets `Get-SsoLockoutPolicy`, `Get-SsoPasswordPolicy`, `Unlock-SsoPersonUser`, `Set-SsoPersonUser` et `Disconnect-SsoAdminServer` viennent du module `VMware.vSphere.SsoAdmin`, absent du lab (erreur `not recognized as a name of a cmdlet`, voir Annexe B). Tout se fait dans l'interface.

**Voir les politiques de verrouillage et de mots de passe**

1. Menu, **Administration**, **Single Sign On**, **Configuration**.
2. Onglet **Local Accounts**, puis les sous-onglets **Password Policy** et **Lockout Policy**.
3. Cliquez sur **Edit** pour modifier une valeur (le libellé exact des onglets peut varier légèrement selon la version).

Explication : la politique de verrouillage verrouille par défaut un compte après trois échecs de connexion (elle est désactivée dans le lab). La politique de mots de passe prévoit une expiration par défaut de 90 jours.

**Déverrouiller un compte bloqué**

1. Menu, Administration, Single Sign On, **Users and Groups**, onglet **Users**, domaine `vsphere.local`.
2. Repérez le compte dont la colonne **Locked** affiche `Yes`.
3. Cliquez sur le menu des trois points à gauche de la ligne, puis **Unlock**.

Explication : il faut être membre du groupe SSO **Administrators**.

**Réinitialiser le mot de passe d'un utilisateur (par un administrateur)**

1. Users and Groups, onglet Users, domaine `vsphere.local`.
2. Menu des trois points du compte, puis **Edit**.
3. Saisissez le nouveau mot de passe (par exemple `NouveauMdp!2026Lab`), confirmez, **Save**.

Explication : un utilisateur change son propre mot de passe dans le vSphere Client : menu de son nom, Change Password, mot de passe actuel, nouveau, confirmation, OK. Pensez à changer le mot de passe de `administrator@vsphere.local` avant 90 jours, sinon il est bloqué au 91e jour. Si vous le changez dans le lab, notez-le pour les autres exercices.

**Variante PowerShell** (uniquement si le module `VMware.vSphere.SsoAdmin` est installé et importé) :

```powershell
Get-SsoLockoutPolicy
Get-SsoPasswordPolicy
Get-SsoPersonUser -Name "prestataire-reseau" -Domain "vsphere.local" | Unlock-SsoPersonUser
Get-SsoPersonUser -Name "prestataire-reseau" -Domain "vsphere.local" | Set-SsoPersonUser -NewPassword "NouveauMdp!2026Lab"
Disconnect-SsoAdminServer
```

Explication : dans l'ordre, affiche la politique de verrouillage, affiche la politique de mots de passe, déverrouille un compte, réinitialise un mot de passe, puis ferme la session d'administration SSO.

---

## 2.9 Parcours de certification

Les Hands-on Labs aident à préparer les certifications VMware. Ce lab prépare notamment la certification **VCAP-DCV Deploy 2023**.

---

## 2.10 Questions de révision du Module 2

1. **Quelle différence entre VSS et VDS ?** Le VSS est configuré sur chaque hôte. Le VDS est configuré une fois dans vCenter et propagé aux hôtes (plan de gestion centralisé, plan de données local).
2. **Qu'est-ce qu'un host proxy switch ?** Le plan de données local d'un VDS sur chaque hôte.
3. **Qu'est-ce qu'un VMkernel adapter ?** Une interface IP de l'hôte pour ses services (gestion, vMotion, stockage IP, FT).
4. **Que font Promiscuous Mode, MAC Address Changes et Forged Transmits ?** Ils contrôlent l'écoute du trafic, le changement de MAC et l'usurpation de MAC source. Reject est le choix sûr.
5. **Quelle politique de teaming exige un EtherChannel ?** Route based on IP hash.
6. **Quelle politique n'existe que sur le VDS ?** Route based on physical NIC load.
7. **Qu'est-ce que le Lockdown Mode Normal et Strict ?** Normal : vCenter et DCUI. Strict : vCenter seulement.
8. **Quel effet a le Lockdown sur les services SSH ou DCUI ?** Il limite qui peut s'y connecter sans arrêter les services.
9. **Rôle, permission, privilège : définissez.** Privilège : droit élémentaire. Rôle : ensemble de privilèges. Permission : utilisateur ou groupe, plus rôle, plus objet.
10. **Quel est le domaine SSO par défaut et son administrateur ?** `vsphere.local` et `administrator@vsphere.local`.
11. **Pourquoi migrer des VM d'un VSS vers un VDS avec l'assistant Migrate VMs to Another Network ?** C'est la méthode recommandée par la documentation vSphere.

---

# MODULE 3 : Introduction au stockage vSphere

## 3.0 Cours : ce que l'étudiant doit connaître

### 3.0.1 Virtualisation du stockage

L'hyperviseur ESXi assure une **virtualisation du stockage au niveau de l'hôte** : il abstrait la couche de stockage physique vis-à-vis des VM.

* Une VM stocke son système, ses programmes et ses données dans un **disque virtuel**, qui est en réalité un gros fichier (ou un ensemble de fichiers) que l'on copie, déplace, archive et sauvegarde comme n'importe quel fichier. Une VM peut avoir plusieurs disques virtuels.
* Pour accéder à ses disques, la VM utilise des **contrôleurs SCSI virtuels** : BusLogic Parallel, LSI Logic Parallel, LSI Logic SAS et VMware Paravirtual. Ce sont les seuls types de contrôleurs que la VM voit.
* Chaque disque virtuel réside sur un **datastore** : VMFS ou NFS, déployé sur du stockage physique. Du point de vue de la VM, le disque apparaît comme un disque SCSI relié à un contrôleur SCSI.
* Que le stockage physique soit accessible par SCSI parallèle, iSCSI, réseau, Fibre Channel ou FCoE est **transparent** pour le système invité et les applications.
* La gestion commence par l'espace alloué par l'administrateur du stockage sur les baies, avant son affectation aux hôtes ESXi. vSphere prend en charge deux familles : stockage **local** et stockage **réseau**.

```
VM  --->  Contrôleur SCSI virtuel  --->  Disque virtuel (.vmdk)
                                              |
                                      Datastore (VMFS ou NFS)
                                              |
                       Stockage physique (local, FC, iSCSI, NFS, vSAN)
```

### 3.0.2 Stockage local et stockage réseau

| Type | Description | Conséquences |
| --- | --- | --- |
| Local | Disques internes de l'hôte, ou systèmes externes branchés directement par SAS ou SATA. | Visible par un seul hôte. Pas de HA, DRS, vMotion classique ni FT. |
| Réseau | Systèmes externes accédés par un réseau de stockage rapide : Fibre Channel, iSCSI, NFS, SAS partagé. | Partagé. Les datastores sont accessibles simultanément par plusieurs hôtes : HA, DRS, vMotion et FT deviennent possibles. |

### 3.0.3 Protocoles et types de datastore

| Technologie | Mode | Particularités |
| --- | --- | --- |
| Fibre Channel (FC) | Bloc | Réseau dédié (SAN), très performant, matériel spécifique (HBA, switchs FC). |
| FCoE | Bloc | FC encapsulé dans Ethernet. |
| iSCSI | Bloc | SCSI sur IP. Initiateur logiciel ou matériel, cible (target) qui présente des LUN. |
| NFS | Fichier | Partage exporté par un serveur NFS, monté par l'hôte. NAS. Versions 3 et 4.1. |
| vSAN | Bloc distribué | Stockage défini par logiciel qui agrège les disques locaux des hôtes d'un cluster. |
| Shared SAS | Bloc | Baie SAS partagée par quelques hôtes. |

**VMFS** (Virtual Machine File System) : système de fichiers en cluster de VMware pour le stockage en mode bloc. Plusieurs hôtes lisent et écrivent en même temps sur le même volume grâce à un verrouillage fin. **Complément** : la version actuelle est VMFS 6, adaptée aux disques de grande capacité.

**NFS** : le datastore est un partage fichier. **Complément** : NFS 3 est simple mais sans multipathing natif ni authentification Kerberos. NFS 4.1 apporte le multipathing et Kerberos.

### 3.0.4 iSCSI en détail

| Terme | Définition |
| --- | --- |
| Initiateur (initiator) | Côté hôte ESXi : adaptateur iSCSI **logiciel** (à activer, apparaît comme `vmhba65` par exemple) ou matériel. |
| Cible (target) | Serveur de stockage qui présente les LUN. L'hôte doit savoir où la chercher. |
| LUN | Volume bloc présenté à l'hôte. Le manuel présente deux disques iSCSI de 45 Go. |
| Découverte | **Dynamic Discovery** (Send Targets) : on donne l'adresse du serveur, l'hôte découvre ses cibles. **Static** : on saisit les cibles. |
| IQN | Nom qualifié iSCSI qui identifie initiateur et cible. |
| Rescan | Nouvelle interrogation des adaptateurs pour découvrir les LUN. |
| CHAP | Authentification facultative entre initiateur et cible. |

**Complément** : on dédie une interface VMkernel au trafic iSCSI et on peut lier (port binding) plusieurs VMkernel à l'adaptateur pour le multipathing.

**Complément : multipathing.** Le Pluggable Storage Architecture (PSA) de vSphere gère plusieurs chemins vers un même LUN. Le Native Multipathing Plugin (NMP) choisit une politique de chemins : **MRU** (Most Recently Used), **Fixed** ou **Round Robin** (répartit les I/O entre chemins actifs).

### 3.0.5 Formats de disque virtuel

Lors de la création d'un disque, d'un clone en template ou d'une migration, on choisit une politique d'allocation.

| Format | Allocation | Remise à zéro | Usage |
| --- | --- | --- | --- |
| **Thin Provision** | L'espace est consommé au fur et à mesure. La taille provisionnée est celle saisie, mais le disque démarre petit. | À la demande | Économie d'espace. Surveiller la sur-allocation du datastore. |
| **Thick Provision Lazy Zeroed** | Espace alloué entièrement à la création (format thick par défaut). | À la demande, à la première écriture de la VM | Performance correcte, création rapide. Ne supprime pas les anciennes données présentes sur le support. Impossible à convertir en Thin. |
| **Thick Provision Eager Zeroed** | Espace alloué entièrement à la création. | À la création | Requis pour la **Fault Tolerance** et le clustering. Création nettement plus longue. |

Dans le lab, il faut choisir Thin pour éviter de manquer d'espace : sinon la VM peut ne pas démarrer.

**Complément : modes de disque** : dependent (inclus dans les snapshots), independent persistent (ignoré par les snapshots, modifications conservées), independent nonpersistent (modifications perdues à l'extinction).

### 3.0.6 Storage vMotion

Plus de 80 % des arrêts d'un datacenter sont des arrêts **planifiés** (maintenance matérielle, migration de serveurs, mises à jour de firmware). vMotion et Storage vMotion les réduisent : les charges se déplacent dynamiquement vers d'autres serveurs ou d'autres stockages sans interruption de service. On peut :

* supprimer l'arrêt pour les opérations de maintenance courantes ;
* supprimer les fenêtres de maintenance planifiées ;
* intervenir à tout moment sans perturber les utilisateurs.

**Storage vMotion** déplace les fichiers d'une VM en fonctionnement d'un datastore vers un autre. Cas d'usage : vider un datastore avant maintenance, migrer entre baies, changer de type de disque (Thin vers Thick), rééquilibrer l'espace.

### 3.0.7 Snapshots

Un snapshot préserve l'**état et les données** d'une VM à un instant donné. Il sert à revenir plusieurs fois à un même état sans créer plusieurs VM, ou à créer plusieurs positions de restauration en séquence (arbre de snapshots géré par le **Snapshot Manager**).

Un snapshot préserve :

| Élément | Contenu |
| --- | --- |
| Paramètres de la VM | Le dossier de la VM, y compris les disques ajoutés ou modifiés depuis le snapshot |
| État d'alimentation | Sous tension, hors tension ou suspendue |
| État des disques | Tous les disques virtuels |
| État de la mémoire (facultatif) | Contenu de la mémoire |

**Complément : fonctionnement.** À la prise d'un snapshot, le disque d'origine est figé en lecture seule et un **disque delta** (`-000001.vmdk`) reçoit les nouvelles écritures. Plus le snapshot vit longtemps, plus le delta grandit et plus les performances baissent. **Quiesce** demande à VMware Tools de mettre le système de fichiers invité dans un état cohérent.

Bonnes pratiques : un snapshot n'est **pas une sauvegarde**, il est conservé quelques heures ou jours, pas indéfiniment, on évite de multiplier les niveaux et on le supprime (consolidation) après validation. La suppression (Delete) fusionne les données du delta dans le disque d'origine, alors que **Revert** abandonne les changements faits depuis le snapshot.

Capturer la mémoire permet de revenir à une VM allumée dans le même état, au prix d'entrées-sorties supplémentaires (le manuel la décoche dans le lab pour éviter cette charge).

### 3.0.8 Datastore cluster et Storage DRS

Un **datastore cluster** équilibre les I/O et la capacité d'un groupe de datastores. Selon le niveau d'automatisation, **Storage DRS (SDRS)** place et migre les VM pour équilibrer l'utilisation des datastores.

| Fonction | Description |
| --- | --- |
| Placement initial | Choix du meilleur datastore à la création ou au clonage d'une VM |
| Équilibrage de capacité | Migrations quand l'espace utilisé dépasse un seuil |
| Équilibrage des I/O | Migrations quand la latence dépasse un seuil |
| Niveau d'automatisation | Manuel (recommandations) ou entièrement automatisé |
| Règles d'affinité | Garder les disques d'une VM ensemble (défaut) ou séparer des VM |
| Mode maintenance du datastore | Évacue toutes les VM d'un datastore |

**Complément** : les seuils par défaut sont environ 80 % d'espace utilisé et 15 ms de latence. Un datastore cluster ne mélange pas des datastores NFS et VMFS. Storage DRS fonctionne avec **Storage I/O Control (SIOC)**, qui répartit le débit d'un datastore selon des shares quand la latence monte.

### 3.0.9 Ajouter un hôte à vCenter et mode maintenance

L'assistant Add Host demande : nom ou adresse, identifiants (root), résumé de l'hôte, licence, **Lockdown Mode** (voir Module 2), emplacement des VM présentes sur l'hôte importé, récapitulatif. Un nouvel hôte peut arriver en **mode maintenance** pour vérifier sa configuration avant de l'activer et éviter des conflits avec les hôtes existants. Pour les datastores, on monte ensuite les NFS existants, on ajoute la cible iSCSI et on rescanne, puis on déplace l'hôte dans le cluster et on sort du mode maintenance. Si HA est actif, l'agent HA est configuré et démarré avant que l'hôte affiche l'état Normal. Idéalement, on ajoute l'hôte au VDS avant de le sortir de maintenance.

### 3.0.10 Points à retenir

* Le stockage **partagé** conditionne HA, DRS, vMotion et FT.
* VMFS et NFS sont les deux types de datastore. Le protocole sous-jacent est transparent pour la VM.
* Thin économise de l'espace, Eager Zeroed est requis pour FT.
* Un snapshot n'est pas une sauvegarde.
* Storage vMotion déplace le stockage sans interruption, Storage DRS l'automatise.

---

## 3.1 Vue d'ensemble du stockage vSphere

Variables de session à rejouer si vous démarrez ici :

```powershell
$dc      = Get-Datacenter -Name "RegionA01"
$cluster = Get-Cluster -Name "Cluster-CLI" -ErrorAction SilentlyContinue
$h1 = Get-VMHost -Name 'esx-01a.vcf.sddc.lab'
$h2 = Get-VMHost -Name 'esx-02a.vcf.sddc.lab'
```

Explication : récupère les objets d'inventaire utilisés dans ce module.

**Objectif** : repérer ce que l'hôte voit en stockage avant de créer des datastores.

```powershell
Get-VMHostHba -VMHost $h1 | Select-Object Device, Type, Model, Status
```

Explication : liste les adaptateurs de stockage de l'hôte (adaptateurs bloc, iSCSI, Fibre Channel).

```powershell
Get-ScsiLun -VMHost $h1 -LunType disk | Select-Object CanonicalName, CapacityGB, Vendor, Model, MultipathPolicy
```

Explication : liste les disques et LUN visibles avec leur capacité et leur politique de chemins.

```powershell
Get-Datastore | Select-Object Name, Type, CapacityGB, FreeSpaceGB, @{N='Hôtes';E={($_ | Get-VMHost).Count}}
```

Explication : liste les datastores, leur type (VMFS, NFS, vSAN), leur capacité et le nombre d'hôtes qui les voient. Un datastore vu par un seul hôte est local.

Le manuel présente deux datastores préconfigurés, un iSCSI (`ds-iscsi01`) et un NFS (`ds-nfs01`) : dans le vSphere Client, onglet Summary pour la capacité, Files pour le contenu, Hosts pour les hôtes connectés, VMs pour les machines hébergées.

```powershell
Get-Datastore -Name "ds-nfs01" -ErrorAction SilentlyContinue | Get-VM | Select-Object Name, PowerState
```

Explication : liste les VM hébergées sur un datastore (adaptez le nom à votre lab).

---

## 3.2 Créer un datastore NFS

**Objectif** : monter un partage NFS en datastore partagé par les deux hôtes.

**Définition** : le manuel crée un datastore à partir d'un montage NFS préparé : type **NFS**, version **NFS 3**, nom `ds-nfs02`, dossier `/mnt/NFS02`, serveur `10.10.20.60`, sélection des hôtes, puis Finish. Un nouveau datastore apparaît dans l'inventaire.

> Les valeurs de serveur et de partage proviennent du manuel HOL-2535. Adaptez-les à votre lab.

```powershell
New-Datastore -Nfs -VMHost $h1 -Name "ds-nfs-cli" -Path "/mnt/NFS02" -NfsHost "10.10.20.60"
```

Explication : monte le partage sur `esx-01a` et crée le datastore. `-Path` est le dossier exporté, `-NfsHost` le serveur NFS. Par défaut, c'est NFS 3.

```powershell
New-Datastore -Nfs -VMHost $h2 -Name "ds-nfs-cli" -Path "/mnt/NFS02" -NfsHost "10.10.20.60"
```

Explication : monte le même partage sur le second hôte avec le **même nom**, ce qui rend le datastore partagé par le cluster. Dans le vSphere Client, un seul assistant sélectionne plusieurs hôtes d'un coup.

```powershell
New-Datastore -Nfs -VMHost $h1 -Name "ds-nfs41-cli" -Path "/mnt/NFS41" -NfsHost "10.10.20.60" -FileSystemVersion "4.1"
```

Explication : exemple de montage en **NFS 4.1**, qui apporte multipathing et Kerberos. Facultatif et dépendant de votre serveur NFS.

```powershell
Get-Datastore -Name "ds-nfs-cli" | Select-Object Name, Type, CapacityGB, FreeSpaceGB, RemoteHost, RemotePath
```

Explication : vérifie le datastore NFS créé (serveur et chemin distants inclus).

---

## 3.3 Créer un datastore iSCSI

**Objectif** : présenter une LUN iSCSI aux deux hôtes et la formater en VMFS.

**Définition** : le manuel ajoute un adaptateur iSCSI logiciel, déclare l'adresse de la cible, rescanne, puis crée un datastore VMFS (`ds-iscsi02`) sur une LUN préprovisionnée.

### 3.3.1 Activer l'adaptateur iSCSI logiciel

```powershell
Get-VMHostStorage -VMHost $h1, $h2 | Set-VMHostStorage -SoftwareIScsiEnabled $true
```

Explication : active l'initiateur iSCSI logiciel sur les deux hôtes. Sans adaptateur, l'hôte ne peut pas contacter de cible.

```powershell
Get-VMHostHba -VMHost $h1, $h2 -Type iScsi | Select-Object VMHost, Device, Model, IScsiName
```

Explication : vérifie la présence de l'adaptateur (par exemple `vmhba65`) et son nom IQN.

### 3.3.2 Déclarer la cible et rescanner

```powershell
foreach ($h in @($h1, $h2)) {
  $hba = Get-VMHostHba -VMHost $h -Type iScsi
  New-IScsiHbaTarget -IScsiHba $hba -Address "10.10.20.60" -Type Send
}
```

Explication : ajoute l'adresse du serveur iSCSI en **Dynamic Discovery** (Send Targets) sur chaque hôte. La liste des serveurs iSCSI était vide avant. L'hôte sait maintenant où chercher les LUN.

```powershell
Get-VMHostStorage -VMHost $h1, $h2 -RescanAllHba
```

Explication : rescanne tous les adaptateurs pour interroger la cible et découvrir les LUN.

```powershell
Get-ScsiLun -VMHost $h1 -LunType disk | Where-Object { [math]::Round($_.CapacityGB) -eq 45 } | Select-Object CanonicalName, CapacityGB
```

Explication : vérifie que les disques iSCSI sont visibles. Le manuel affiche deux disques de 45 Go. Adaptez la taille filtrée à votre lab.

### 3.3.3 Créer le datastore VMFS

```powershell
$lun = Get-ScsiLun -VMHost $h1 -LunType disk | Where-Object { [math]::Round($_.CapacityGB) -eq 45 } | Select-Object -First 1
New-Datastore -Vmfs -VMHost $h1 -Name "ds-iscsi-cli" -Path $lun.CanonicalName -FileSystemVersion 6
```

Explication : formate la LUN choisie en VMFS 6 et crée le datastore. Vérifiez que la LUN n'est pas déjà utilisée.

```powershell
Get-VMHostStorage -VMHost $h2 -RescanVmfs
```

Explication : rescanne les volumes VMFS sur `esx-02a` pour qu'il voie le nouveau datastore.

```powershell
Get-Datastore -Name "ds-iscsi-cli" | Get-VMHost | Select-Object Name
```

Explication : confirme que les deux hôtes voient le datastore.

---

## 3.4 Ajouter un hôte au datacenter et au cluster (exemple du manuel)

**Objectif** : intégrer un nouvel hôte (`esx-03a` dans le manuel) en lui présentant le stockage existant, puis l'activer dans le cluster. Si votre lab ne compte que deux hôtes, lisez cette section comme un exemple et adaptez le nom.

```powershell
$h3cred = New-Object System.Management.Automation.PSCredential ("root", (ConvertTo-SecureString "VMware123!" -AsPlainText -Force))
$h3 = Add-VMHost -Name "esx-03a.vcf.sddc.lab" -Location $dc -Credential $h3cred -Force
```

Explication : ajoute l'hôte au datacenter (équivalent de l'assistant Add Host : nom, identifiants, résumé, licence, Lockdown, emplacement des VM). `-Force` accepte l'empreinte du certificat de l'hôte. Le Lockdown Mode peut être choisi dans l'assistant, laissé par défaut ici.

```powershell
Set-VMHost -VMHost $h3 -State Maintenance -Confirm:$false
```

Explication : met l'hôte en maintenance pour vérifier sa configuration avant de l'intégrer au cluster, comme dans le manuel. Il n'accepte alors aucune VM.

```powershell
New-Datastore -Nfs -VMHost $h3 -Name "ds-nfs-cli" -Path "/mnt/NFS02" -NfsHost "10.10.20.60"
```

Explication : monte le datastore NFS existant sur le nouvel hôte. Répétez pour chaque datastore NFS (le manuel monte `ds-nfs01` et `ds-nfs02`).

```powershell
Get-VMHostStorage -VMHost $h3 | Set-VMHostStorage -SoftwareIScsiEnabled $true
New-IScsiHbaTarget -IScsiHba (Get-VMHostHba -VMHost $h3 -Type iScsi) -Address "10.10.20.60" -Type Send
Get-VMHostStorage -VMHost $h3 -RescanAllHba
```

Explication : active l'iSCSI logiciel, ajoute la cible (manuel : serveur `10.10.20.60`) puis rescanne. Les deux disques iSCSI de 45 Go doivent apparaître, puis les datastores iSCSI deviennent visibles.

```powershell
Get-VMHost -Name $h3.Name | Get-Datastore | Select-Object Name, Type
```

Explication : vérifie que les datastores NFS et iSCSI sont visibles sur le nouvel hôte.

```powershell
Move-VMHost -VMHost $h3 -Destination $cluster
Set-VMHost -VMHost $h3 -State Connected -Confirm:$false
```

Explication : déplace l'hôte dans le cluster puis le sort du mode maintenance. Si HA est activé, l'agent HA est configuré et démarré avant que l'hôte affiche l'état Normal (un rafraîchissement peut être nécessaire). L'hôte peut alors accueillir des VM. Dans le manuel, le réseau de VM, le vMotion et le stockage IP sont préconfigurés sur cet hôte. Idéalement, on l'ajoute au VDS **avant** de le sortir de maintenance (`Add-VDSwitchVMHost`).

---

## 3.5 Storage vMotion

**Objectif** : déplacer à chaud les fichiers d'une VM vers un autre datastore (exemple du manuel : iSCSI vers NFS, VM `TinyLinux`).

**Définition** : voir 3.0.6. Dans le vSphere Client : clic droit sur la VM, Migrate, **Change storage only**, choix du datastore et du format de disque.

```powershell
Get-VM -Name "VM-vCenter-CLI" | Get-Datastore | Select-Object Name, Type
```

Explication : indique le datastore actuel de la VM.

```powershell
Move-VM -VM "VM-vCenter-CLI" -Datastore (Get-Datastore -Name "ds-iscsi-cli") -Confirm:$false
```

Explication : migre disques et fichiers de la VM vers le datastore iSCSI, VM allumée, sans interruption.

```powershell
Move-VM -VM "VM-vCenter-CLI" -Datastore (Get-Datastore -Name "ds-nfs-cli") -DiskStorageFormat Thin -Confirm:$false
```

Explication : retour vers le datastore NFS en imposant le format de disque Thin (le format peut être changé pendant la migration).

```powershell
Get-Datastore -Name "ds-nfs-cli" | Get-VM | Select-Object Name, PowerState
```

Explication : confirme que la VM est listée sur le datastore NFS et toujours allumée : le stockage a été migré sans mettre la VM hors ligne.

---

## 3.6 Gérer les disques de machines virtuelles

**Objectif** : ajouter un disque et agrandir un disque existant.

**Définition** : un disque virtuel est composé d'un ou plusieurs fichiers qui apparaissent comme un seul disque au système invité. Les disques sont **portables entre hôtes**. L'assistant de création peut ajouter des disques ; ici on travaille sur une VM existante (le manuel utilise `Windows10`, disque unique de 30 Go).

```powershell
Get-HardDisk -VM "VM-vCenter-CLI" | Select-Object Name, CapacityGB, StorageFormat, Filename
```

Explication : vérifie la configuration initiale (le manuel note un seul disque de 30 Go).

```powershell
New-HardDisk -VM "VM-vCenter-CLI" -CapacityGB 5 -StorageFormat Thin
```

Explication : ajoute un second disque de 5 Go en Thin (équivalent de Edit Settings, Add New Device, Hard Disk). Pour la Fault Tolerance, utilisez `-StorageFormat EagerZeroedThick`.

```powershell
Get-HardDisk -VM "VM-vCenter-CLI" | Where-Object { $_.Name -eq "Hard disk 1" } | Set-HardDisk -CapacityGB 22 -Confirm:$false
```

Explication : agrandit le disque 1 (le manuel passe de 30 à 32 Go). On ne peut que **augmenter** la taille, puis il faut étendre la partition et le système de fichiers dans le système invité.

```powershell
Get-VM -Name "VM-vCenter-CLI" | Select-Object Name, ProvisionedSpaceGB, UsedSpaceGB
```

Explication : montre l'effet du Thin Provisioning : l'espace réellement consommé (`UsedSpaceGB`) est inférieur à l'espace provisionné, comme l'indique le manuel pour le disque de 32 Go.

---

## 3.7 Travailler avec les snapshots

**Objectif** : sauvegarder l'état d'une VM, la modifier, puis revenir à l'état d'origine.

**Définition** : voir 3.0.7. Déroulé du manuel : prendre le snapshot `Snapshot #1` sans mémoire, éteindre la VM, passer la mémoire à 4 Go, vérifier dans Summary, puis **Revert** depuis le Snapshot Manager.

```powershell
New-Snapshot -VM "VM-vCenter-CLI" -Name "Snapshot #1" -Description "Snapshot taken prior to VM settings change" -Memory:$false -Quiesce:$false
```

Explication : prend un snapshot sans mémoire ni quiescence (le manuel décoche la mémoire pour éviter des entrées-sorties inutiles). Ajoutez `-Memory:$true` pour inclure la mémoire.

```powershell
Get-Snapshot -VM "VM-vCenter-CLI" | Select-Object Name, Description, Created, SizeGB, IsCurrent
```

Explication : affiche les snapshots de la VM, équivalent de Manage Snapshots et de la fenêtre de détails.

```powershell
Stop-VM -VM "VM-vCenter-CLI" -Confirm:$false
Set-VM -VM "VM-vCenter-CLI" -MemoryGB 4 -Confirm:$false
Get-VM "VM-vCenter-CLI" | Select-Object Name, MemoryGB
```

Explication : éteint brutalement la VM (rapide pour le lab, pas la bonne pratique, qui est un arrêt propre avec `Stop-VMGuest`), passe la mémoire à 4 Go et vérifie.

```powershell
Set-VM -VM "VM-vCenter-CLI" -Snapshot (Get-Snapshot -VM "VM-vCenter-CLI" -Name "Snapshot #1") -Confirm:$false
Get-VM "VM-vCenter-CLI" | Select-Object Name, MemoryGB, PowerState
```

Explication : restaure l'état du snapshot (**Revert**). La mémoire doit revenir à sa valeur d'origine. La VM prend l'état d'alimentation du snapshot.

```powershell
Get-Snapshot -VM "VM-vCenter-CLI" | Remove-Snapshot -Confirm:$false
```

Explication : supprime le snapshot (Delete) et consolide les disques. À faire dès qu'il n'est plus utile.

---

## 3.8 Datastore cluster et Storage DRS

**Objectif** : regrouper des datastores et équilibrer automatiquement capacité et I/O.

**Définition** : voir 3.0.8. L'assistant du vSphere Client (Storage, clic droit sur le datacenter, Storage, New Datastore Cluster) enchaîne : nom et emplacement, **automatisation Storage DRS**, **paramètres d'exécution** (sensibilité de l'équilibrage), sélection des clusters et hôtes (les hôtes sans cluster sont à choisir à part), sélection des datastores, puis récapitulatif. Le manuel crée `DatastoreCluster-01` avec deux datastores iSCSI et laisse les valeurs par défaut.

```powershell
$dsc = New-DatastoreCluster -Name "DatastoreCluster-CLI" -Location $dc
```

Explication : crée le datastore cluster, géré comme une ressource unique.

```powershell
Get-Datastore -Name "ds-iscsi-cli" | Move-Datastore -Destination $dsc
```

Explication : ajoute un datastore iSCSI au cluster. Ajoutez un second datastore **du même type** (le manuel utilise deux datastores iSCSI), car on ne mélange pas NFS et VMFS dans un même cluster.

```powershell
Set-DatastoreCluster -DatastoreCluster $dsc -SdrsAutomationLevel FullyAutomated -IOLoadBalanceEnabled $true
```

Explication : active Storage DRS en mode entièrement automatisé avec équilibrage des I/O. En mode `Manual`, Storage DRS ne fait que des recommandations.

```powershell
Set-DatastoreCluster -DatastoreCluster $dsc -IOLatencyThresholdMillisecond 15 -SpaceUtilizationThresholdPercent 80
```

Explication : règle les paramètres d'exécution : seuil de latence (15 ms) et seuil d'espace utilisé (80 %) au-delà desquels Storage DRS recommande ou effectue des migrations.

```powershell
Get-DatastoreCluster | Select-Object Name, SdrsAutomationLevel, IOLoadBalanceEnabled, CapacityGB, FreeSpaceGB
Get-DatastoreCluster -Name "DatastoreCluster-CLI" | Get-Datastore | Select-Object Name, CapacityGB, FreeSpaceGB
```

Explication : vérifie la configuration du cluster et la liste des datastores membres avec leur espace libre.

---

## 3.9 Parcours de certification

Ce lab prépare notamment la certification **VCAP-DCV Deploy 2023**. Les vidéos complémentaires du manuel sur Storage DRS sont listées en Annexe D.

---

## 3.10 Questions de révision du Module 3

1. **Quelle est la différence entre stockage local et stockage réseau ?** Le local est lié à un hôte. Le réseau (FC, iSCSI, NFS, SAS partagé) est partagé et permet HA, DRS, vMotion et FT.
2. **VMFS ou NFS : lequel est un système de fichiers en bloc ?** VMFS (bloc). NFS est un partage fichier.
3. **Quels sont les trois formats de disque virtuel ?** Thin, Thick Lazy Zeroed, Thick Eager Zeroed.
4. **Quel format est requis pour la Fault Tolerance ?** Thick Eager Zeroed.
5. **Quelles étapes pour utiliser iSCSI sur un hôte ?** Activer l'initiateur logiciel, déclarer la cible (Dynamic Discovery), rescanner, puis créer le datastore VMFS.
6. **Que préserve un snapshot ?** Paramètres, état d'alimentation, état des disques et, en option, mémoire.
7. **Un snapshot est-il une sauvegarde ?** Non. Il dépend du disque d'origine et doit être de courte durée.
8. **Quelle est la différence entre vMotion et Storage vMotion ?** vMotion déplace le calcul (hôte), Storage vMotion déplace les fichiers (datastore), les deux sans interruption.
9. **À quoi sert un datastore cluster ?** À équilibrer capacité et I/O entre datastores grâce à Storage DRS.
10. **Pourquoi mettre un nouvel hôte en maintenance avant de l'ajouter au cluster ?** Pour vérifier sa configuration avant de l'activer et éviter des conflits avec les hôtes existants.

---

# ANNEXE A : Nettoyage du lab

**Objectif** : remettre l'environnement dans son état initial. Toutes ces commandes sont destructrices : relisez-les avant exécution. Le lab HOL étant jetable, le nettoyage est surtout utile pour rejouer une section.

```powershell
Get-VM -Name "VM-*" | Where-Object { $_.PowerState -eq "PoweredOn" } | Stop-VM -Confirm:$false
Get-VM -Name "VM-*" | Remove-VM -DeletePermanently -Confirm:$false
Get-Template -Name "TPL-Photon-CLI" | Remove-Template -DeletePermanently -Confirm:$false
```

Explication : éteint puis supprime les VM de test et le template, avec leurs fichiers.

```powershell
Get-ResourcePool -Name "RP-Production" | Remove-ResourcePool -Confirm:$false
Get-DrsRule -Cluster (Get-Cluster "Cluster-CLI") | Remove-DrsRule -Confirm:$false
Get-Tag -Name "Production" | Remove-Tag -Confirm:$false
Get-TagCategory -Name "Environnement" | Remove-TagCategory -Confirm:$false
```

Explication : supprime le resource pool, les règles DRS, le tag et sa catégorie.

```powershell
Get-VDSwitch -Name "VDS-CLI" | Get-VMHostNetworkAdapter -VMKernel | Where-Object { $_.PortGroupName -eq "PG-vMotion-CLI" } | Remove-VMHostNetworkAdapter -Confirm:$false
Get-VDSwitch -Name "VDS-CLI" | Remove-VDSwitch -Confirm:$false
```

Explication : retire les VMkernel vMotion puis le VDS (impossible tant que des interfaces ou des VM l'utilisent).

```powershell
Get-DatastoreCluster -Name "DatastoreCluster-CLI" | Remove-DatastoreCluster -Confirm:$false
```

Explication : supprime le datastore cluster (les datastores restent).

```powershell
Get-Cluster -Name "Cluster-CLI" | Get-VMHost | ForEach-Object {
  Set-VMHost -VMHost $_ -State Maintenance -Confirm:$false
  Move-VMHost -VMHost $_ -Destination (Get-Cluster -Name "RegionA01-COMP01")
  Set-VMHost -VMHost $_ -State Connected -Confirm:$false
}
Get-Cluster -Name "Cluster-CLI" | Remove-Cluster -Confirm:$false
```

Explication : remet les hôtes dans le cluster d'origine `RegionA01-COMP01`, puis supprime `Cluster-CLI`. Les VM doivent d'abord avoir été migrées ou éteintes, comme en 1.3.1 (sinon la mise en maintenance reste à 0 %). Le datacenter `RegionA01` est celui du lab : ne le supprimez pas.

```powershell
Get-VIPermission -Entity (Get-Datacenter "RegionA01") | Where-Object { $_.Principal -like "*prestataire*" } | Remove-VIPermission -Confirm:$false
Get-VIRole -Name "Role-Reseau-CLI" | Remove-VIRole -Confirm:$false
```

Explication : supprime la permission puis le rôle personnalisé.

```powershell
Disconnect-VIServer -Server * -Confirm:$false
```

Explication : ferme la session vCenter. Les hôtes déplacés vers `Cluster-CLI` peuvent être remis à leur emplacement d'origine avec `Move-VMHost`, après passage en maintenance.

---

# ANNEXE B : Journal des erreurs connues

| Erreur | Cause | Solution |
| --- | --- | --- |
| `No route to host` | Le DNS ne résout pas `vcsa-01a.vcf.sddc.lab` | Utiliser l'adresse IP `10.0.0.22` |
| `proxy tunnel request failed` | Un proxy bloque la connexion PowerCLI | `Set-PowerCLIConfiguration -ProxyPolicy NoProxy -Scope Session` |
| Erreur de certificat non approuvé | Certificat auto-signé du lab | `Set-PowerCLIConfiguration -InvalidCertificateAction Ignore -Scope Session` |
| `network VLAN10-Serveurs doesn't exist` | Sélection d'un Uplink (port group physique) au lieu d'un port group de VM | Utiliser `Get-VDPortgroup` et choisir un nom sans « Uplink » |
| `Move-VMHost` échoue | Hôte non placé en maintenance, ou VM encore en cours d'exécution | Migrer ou arrêter les VM, puis passer l'hôte en maintenance |
| `Move-VMHost` : The operation is not supported on the object | Déplacement d'un hôte vers un cluster d'un **autre datacenter** (ici `Datacenter-CLI` alors que les hôtes sont dans `RegionA01`) | Créer le cluster dans le datacenter existant (voir 1.3.1) |
| `Set-VMHost -State Maintenance` reste à `percent complete: 0` | L'hôte héberge encore des VM et DRS est désactivé : aucune évacuation automatique | Migrer les VM avec `Move-VM` (ou les éteindre), puis relancer. Les VM `vCLS` se gèrent seules (voir 1.3.1) |
| vMotion refusé | Pas de stockage partagé, VMkernel vMotion absent ou injoignable, ou VM sur un datastore local | Placer la VM sur un datastore partagé, vérifier les VMkernel (2.3.6) |
| La VM ne démarre pas, espace insuffisant | Disque Thick sur un datastore trop petit | Utiliser `-DiskStorageFormat Thin` (le manuel insiste sur Thin Provision) |
| Perte de connexion à l'hôte après ajout au VDS | Carte de gestion déplacée sur le VDS | Ne migrer que des cartes libres, jamais celle de la gestion |
| Impossible de se connecter à l'hôte en SSH | Lockdown Mode actif, ou service SSH arrêté | Désactiver le mode via vCenter, démarrer le service TSM-SSH |
| Le datastore n'apparaît pas sur le deuxième hôte | Rescan non effectué | `Get-VMHostStorage -VMHost $h2 -RescanAllHba -RescanVmfs` |
| Une VM s'éteint au lieu de redémarrer | VMware Tools absents (`Restart-VMGuest` indisponible) | Rallumer la VM manuellement, installer les Tools |
| `Operation "Restart VM guest" failed for VM "VM-Deploy-01" ... VMware Tools is not running in this virtual machine` avec `Restart-VMGuest` (1.11) | `Restart-VMGuest` exige VMware Tools actif dans la VM. Ici la VM Windows 10 n'a pas les Tools en cours d'exécution | Tester `$_.ExtensionData.Guest.ToolsRunningStatus`, utiliser `Restart-VMGuest` si `guestToolsRunning`, sinon `Restart-VM` (redémarrage forcé), ou installer les Tools |
| `Restart-VMGuest` ... `Permission to perform this operation was denied. Required privileges: 'VirtualMachine.Interact.Reset'` sur `vCLS-xxxxxxxx-...` (1.11) | Les VM `vCLS` sont des VM système gérées par vCenter : leur redémarrage est interdit à l'utilisateur, même administrateur | Les exclure du pipeline : `Where-Object { $_.Name -notlike "vCLS*" }` |
| `The node didn't join any Active Directory` | Aucune source AD dans le lab | Normal dans le lab, ignorer |
| Compte SSO bloqué | Trop d'échecs de connexion ou mot de passe expiré (90 jours) | Déverrouiller via un administrateur SSO, changer le mot de passe |
| « Unknown » sous Technology dans l'Host Client | Environnement imbriqué | Comportement attendu, ignorer |
| Bouton OK invisible dans un assistant | Fenêtre trop petite | Dézoomer le navigateur |
| Cmdlet SSO inconnue | Module `VMware.vSphere.SsoAdmin` absent ou autre version | `Get-Command *Sso*` et `Get-Help <cmdlet> -Full` |
| `Get-VIAccount : Parameter set cannot be resolved using the specified named parameters` (2.7, 2.8.2) | `-Name` et `-Domain` ne forment pas une combinaison valide pour cette cmdlet dans cette version de PowerCLI | Lister puis filtrer : `Get-VIAccount -User \| Where-Object { $_.Name -like "*prestataire*" }`. Voir aussi `Get-Help Get-VIAccount -Full` pour les jeux de paramètres de votre version |
| `New-SsoGroup`, `Add-UserToSsoGroup`, `Get-SsoGroup`, `Get-SsoPersonUser`, `Get-SsoLockoutPolicy`, `Get-SsoPasswordPolicy` : `The term '...' is not recognized as a name of a cmdlet` (2.8.2, 2.8.3, 2.8.4) | Même cause que `Connect-SsoAdminServer` : le module `VMware.vSphere.SsoAdmin` est absent du lab | Créer le groupe et ajouter les membres dans le vSphere Client (Administration, Single Sign On, Users and Groups, Groups, Add Group, puis Add Members), voir 2.8.3. Politiques de verrouillage et de mots de passe : Administration, Single Sign On, Configuration, Local Accounts, voir 2.8.4 |
| `Connect-SsoAdminServer: The term 'Connect-SsoAdminServer' is not recognized as a name of a cmdlet` (idem `New-SsoPersonUser`) (2.8) | Le module `VMware.vSphere.SsoAdmin` n'est pas installé ou pas importé dans la session. Il est distinct de VMware.PowerCLI | `Get-Module -ListAvailable VMware.vSphere.SsoAdmin` puis `Import-Module VMware.vSphere.SsoAdmin`. Dans le lab HOL, le module n'est pas disponible : **créer l'utilisateur dans le vSphere Client** (Administration, Single Sign On, Users and Groups, Users, Add), voir 2.8.2. Ailleurs, `Install-Module VMware.vSphere.SsoAdmin -Scope CurrentUser` (nécessite Internet) |
| `Insufficient resources to satisfy configured failover level for vSphere HA` au `Move-VM`, puis `Set-VMHost -State Maintenance` bloqué à 0 % (1.3.1) | Le cluster `Cluster-CLI` n'a qu'un hôte et le contrôle d'admission HA (niveau 1) refuse toute VM entrante. Les VM restées sur `esx-02a` ne peuvent pas partir, donc le mode maintenance ne se termine pas | `Ctrl+C`, puis `Get-Task -Status Running \| Stop-Task -Confirm:$false`. Désactiver l'admission control (`Set-Cluster -HAAdmissionControlEnabled:$false`), refaire les `Move-VM`, la maintenance et le `Move-VMHost`, puis le réactiver en 1.6.1 |
| `The specified VM ... is not under the cluster Cluster-CLI` et `You must specify at least 2 virtual machines` avec `New-DrsRule` (1.6.2) | Les VM sont sur un hôte resté hors de `Cluster-CLI` (cas constaté : `esx-02a` encore dans `RegionA01-COMP01`, 1.3.1 non terminée pour le second hôte), ou `$cluster` désigne un autre cluster | Vérifier avec `Get-Cluster -VM`, déplacer les hôtes (1.3.1) puis les VM dans `Cluster-CLI` avec `Move-VM`, relancer `New-DrsRule` |
| `Parameters for device 'ide0:0' may not be modified while the virtual machine is powered on` avec `Set-CDDrive` (1.3) | Le lecteur CD est sur un contrôleur IDE : modification impossible VM allumée | `Stop-VM`, puis `Set-CDDrive -NoMedia`, puis `Start-VM` |
| `Set-CDDrive : A parameter cannot be found that matches parameter name 'ContentLibraryIso'` (1.3) | Version de PowerCLI trop ancienne : le paramètre n'existe pas | Retrouver l'ISO sur le datastore (`New-PSDrive -PSProvider VimDatastore` puis `Get-ChildItem`) et monter avec `-IsoPath $isoFile.DatastoreFullPath`, ou passer par Edit Settings dans le vSphere Client |
| `VDPortgroup with name 'PG-Serveurs-CLI' was not found` à la création de la VM (1.3) | Le VDS et ses port groups ne sont créés qu'au module 2.3. Le module 1 est fait avant | Utiliser le port group standard `VM Network` : `$vmPortGroup = Get-VirtualPortGroup -VMHost $h1 -Name "VM Network"`. Le README le fait maintenant automatiquement si `PG-Serveurs-CLI` est absent |
| `Get-Datastore -Name "ds-nfs-cli"` ne retourne rien (ou erreur) à la création de la VM (1.3) | Les datastores `ds-nfs-cli` et `ds-iscsi-cli` ne sont créés qu'au module 3. Dans le lab, il n'existe que `ds-iscsi01` (VMFS, 55 Go libres) et `ds-nfs01` (NFS, 5,8 Go libres) | Utiliser `ds-iscsi01` : `$ds = Get-Datastore -Name "ds-iscsi01"`. Éviter `ds-nfs01`, trop petit |
| `Set-VDSwitch : Cannot validate argument on parameter 'VDSwitch'. The argument is null or empty` (2.2, propriétés avancées) | `Get-VDSwitch -Name "VDS-CLI" -ErrorAction SilentlyContinue` ne trouve rien : le VDS `VDS-CLI` n'est créé qu'en 2.3.1, et `SilentlyContinue` masque l'erreur, donc `$vds` reste vide | Créer le VDS en 2.3.1 avant ces commandes, ou utiliser le nom d'un VDS existant (`Get-VDSwitch`). Le README contient maintenant un test qui affiche un message explicite |
| `The VirtualNic capability 'vmotion' is not supported on VMHost 'esx-01a...'` avec `New-VMHostNetworkAdapter ... -VMotionEnabled $true`, puis `Failed to add VMotion tag when dedicated stack is instantiated` avec `esxcli ... tag add` (2.3.6) | L'hôte a une pile TCP/IP vMotion dédiée (netstack `vmotion`). Dans ce cas, le service vMotion ne peut pas être activé sur un VMkernel de la pile par défaut, ni par `-VMotionEnabled` ni par tag. L'interface (`vmk3`) est tout de même créée avec son IP | Supprimer `vmk3` (`Remove-VMHostNetworkAdapter`), puis la recréer sur la pile dédiée : `New-VMHostNetworkAdapter ... -NetworkStack (Get-VMHostNetworkStack -VMHost $h -Name "vmotion")`, ou via le vSphere Client (TCP/IP stack : vMotion) |
| `UpdateLockdownExceptions` : `The user or group named 'svc-sauvegarde' does not exist` (2.5) | Le compte donné en exception n'existe pas sur l'hôte : `svc-sauvegarde` est un exemple, pas un compte du lab | Utiliser un compte existant (`root` dans le lab), ou créer le compte local avant d'activer le Lockdown. Le Lockdown reste actif malgré l'erreur : le désactiver à la fin (`ChangeLockdownMode("lockdownDisabled")`) |
| `Set-VMHostFirewallException : Cannot change the host configuration. Required ruleset cannot be disabled` sur `"SSH Server"` (2.6) | La règle de pare-feu SSH est requise tant que le service `TSM-SSH` tourne : on a tenté de la désactiver avant d'arrêter le service | Arrêter d'abord le service (`Stop-VMHostService`, politique `Off`), vérifier `Running = False`, puis relancer la désactivation de la règle. Si cela échoue encore, laisser la règle : le service arrêté suffit |
| `$am.LockdownMode` affiche encore `lockdownDisabled` juste après `ChangeLockdownMode("lockdownNormal")` (2.5) | `$am` est une copie des données prise avant le changement | `$am.UpdateViewData()` puis relire `$am.LockdownMode` |
| `New-VIPermission : Could not find VIAccount with name 'VSPHERE.LOCAL\prestataire-reseau'`, suivi de `Value cannot be found for the mandatory parameter Principal` (2.7) | L'utilisateur SSO `prestataire-reseau` n'existe pas encore : il est créé en 2.8.2, après la section 2.7. Le message sur `Principal` est la conséquence : la commande n'a pas pu résoudre le compte. Une faute de frappe produit le même effet (`Entity` sans tiret, espace dans `VSPHERE. LOCAL`) | Créer l'utilisateur (`Connect-SsoAdminServer`, puis `New-SsoPersonUser`, voir 2.8.2), vérifier avec `Get-VIAccount -User | Where-Object { $_.Name -like "*prestataire*" }`, puis relancer `New-VIPermission -Entity $dc -Principal "VSPHERE.LOCAL\prestataire-reseau" -Role $role -Propagate:$true`. Le rôle `Role-Reseau-CLI` déjà créé est conservé |

---

# ANNEXE C : Checklist finale de validation

| Élément | Commande de contrôle | Résultat attendu |
| --- | --- | --- |
| Connexion vCenter | `$global:DefaultVIServers` | `10.0.0.22` connecté |
| Datacenter et cluster | `Get-Datacenter; Get-Cluster` | `RegionA01` (existant), `Cluster-CLI` |
| Hôtes | `Get-Cluster Cluster-CLI \| Get-VMHost` | 2 hôtes `Connected` |
| HA et DRS | `Get-Cluster Cluster-CLI` | HA et DRS activés |
| Réseau distribué | `Get-VDPortgroup -VDSwitch $vds` | `PG-Serveurs-CLI`, `PG-vMotion-CLI` |
| vMotion VMkernel | `Get-VMHostNetworkAdapter -VMKernel` | `VMotionEnabled = True` sur les 2 hôtes |
| Stockage | `Get-Datastore` | NFS et iSCSI visibles par les 2 hôtes |
| Datastore cluster | `Get-DatastoreCluster` | SDRS actif |
| VM et template | `Get-VM; Get-Template` | VM créées, template présent |
| Tags | `Get-VM -Tag $tag` | VM étiquetée retrouvée |
| vMotion | `Get-VM \| Select Name, VMHost` | L'hôte change sans arrêt de la VM |
| Storage vMotion | `Get-VM \| Get-Datastore` | Datastore changé sans arrêt |
| Snapshot | `Get-Snapshot -VM <vm>` | Création, restauration, suppression réussies |
| Ressources | `Get-VMResourceConfiguration -VM <vm>` | Shares High appliqués |
| Sécurité | `Get-VIPermission -Entity $dc` | Rôle réseau attribué |
| Lockdown | `$am.LockdownMode` | `lockdownDisabled` après les tests |

---

# ANNEXE D : Ressources complémentaires du manuel

Vidéos et références proposées dans la section Additional Resources du manuel HOL-2535.

| Ressource | Sujet | Module concerné |
| --- | --- | --- |
| How to Configure a vSphere Standard Switch (VSS), 4 min 22 | Configurer le réseau de base avec le VSS | Module 2, section 2.1 |
| vSphere 8, Overview, 8 min 18 | Vue d'ensemble de vSphere 8 et haute disponibilité de vCenter | Module 1 |
| Configure Alarms and Notification for VMware vSphere, 5 min 20 | Alarmes, alertes et notification par e-mail | Module 1, section 1.8 |
| Enable vSphere Host Lockdown Mode, 4 min 48 | Sécuriser les hôtes par le Lockdown | Module 2, section 2.5 |
| (Facultatif) Add VMware vSphere Hosts to Active Directory, 3 min 40 | Joindre un hôte à AD | Module 2, section 2.6 |
| Configure vSphere Host Firewall, 4 min 34 | Pare-feu ESXi et services | Module 2, section 2.6 |
| What is vSphere Storage DRS, 5 min 08 | Principe de Storage DRS et datastore cluster | Module 3, section 3.8 |
| Creating a Datastore Cluster with Storage DRS, 3 min 23 | Création et gestion d'un datastore cluster | Module 3, section 3.8 |
| Installing and Configuring vSphere, 4 min 36 | Installation d'ESXi | Module 1, section 1.1 |
| Overview of the DCUI, 4 min 58 | Console locale de l'hôte | Module 1, section 1.1 |
| What is New with DRS in vSphere 7, 5 min 47 | Nouveautés de DRS | Module 1, section 1.6 |
| Getting Started with vSphere Distributed Services Engine, 9 min 55 | DPU et vSphere 8 | Module 1 |

Notes de référence du manuel : déverrouiller un utilisateur SSO, changer son mot de passe et utiliser le Snapshot Manager (voir 2.8 et 3.7).

---

# ANNEXE E : Interface Hands-on Labs

Notions utiles pour travailler dans la plateforme HOL :

* La **Main Console** occupe la zone principale. Le **manuel du lab** est dans l'onglet à droite.
* Un **minuteur** décompte la durée. Le lab **ne peut pas être sauvegardé** et s'arrête à l'expiration. Le bouton **EXTEND** prolonge la durée selon ce que le lab autorise.
* Saisie de données : taper directement dans la console, **glisser-déposer** du texte ou des commandes du manuel vers la fenêtre active de la console, ou activer le **clavier à l'écran** (icône Universal Access, puis Enable Screen Keyboard), utile pour les claviers non américains.
* Les modules du lab sont **indépendants** : on peut commencer par n'importe lequel via la table des matières. Prévoir 60 à 90 minutes par module, et plus de 90 minutes pour tout faire.
* Casque recommandé pour les vidéos. Certaines vidéos montrent des éditions antérieures de vSphere, mais les concepts restent valables.
* Pour reprendre à la page d'accueil du lab ou changer de module, utiliser la table des matières du manuel.
* Avant de commencer, vérifier que le lab affiche **Ready**. Si ce n'est pas le cas après cinq minutes, demander de l'aide.

---

# ANNEXE F : Tableau de couverture du manuel

Chaque sujet de la table des matières du manuel HOL-2535 et sa section dans ce README.

| Sujet du manuel | Section |
| --- | --- |
| Lab Overview, Lab Description, Lab Guidance | 0.3, Annexe E |
| Virtualization 101 | 0.1 |
| M1 : Introduction, ESXi Host Client | 1.0.2, 1.1 |
| M1 : vCenter 8 Overview | 1.0.1 à 1.0.4, 1.2 |
| M1 : vCenter Server and Creating a Virtual Machine | 1.3 |
| M1 : Cloning Virtual Machines and Using Templates | 1.4 |
| M1 : Using Tagging and Search to Find Objects Quickly | 1.5 |
| M1 : vSphere High Availability and DRS | 1.6 |
| M1 : vSphere 8 Fault Tolerance | 1.7 |
| M1 : Monitoring Events and Creating Alarms | 1.8 |
| M1 : Configure Shares and Resources | 1.9 |
| M1 : Migrating Virtual Machines with vMotion | 1.10 |
| M1 : vSphere Monitoring and Performance | 1.11 |
| M1 : Introduction to vSphere with Tanzu | 1.12 |
| M1 : Certification Path | 1.13 |
| M1 : ESXi Installation and Configuration (simulation) | 1.1.1 |
| M2 : Introduction, types de switchs | 2.0 |
| M2 : Adding and Configuring vSphere Standard Switch | 2.1 |
| M2 : Working with the vSphere Distributed Switch | 2.2 |
| M2 : Adding and Configuring a vSphere Distributed Switch | 2.3, 2.4 |
| M2 : Using Host Lockdown Mode | 2.5 |
| M2 : Configuring the Host Services and Firewall | 2.6 |
| M2 : User Access and Authentication Roles | 2.7 |
| M2 : Understanding Single Sign On | 2.8 |
| M2 : Certification Path | 2.9 |
| M3 : vSphere Storage Overview | 3.0, 3.1 |
| M3 : Creating and Configuring vSphere Datastores (NFS, iSCSI, ajout d'hôte) | 3.2, 3.3, 3.4 |
| M3 : Storage vMotion | 3.5 |
| M3 : Managing Virtual Machine Disks | 3.6 |
| M3 : Working with Virtual Machine Snapshots | 3.7 |
| M3 : vSphere Datastore Cluster | 3.8 |
| M3 : Certification Path | 3.9 |
| Additional Resources | Annexe D |
| Appendix : Hands-on Labs Interface | Annexe E |

---

# ANNEXE G : Sources

* VMware Hands-on Labs, manuel HOL-2535-01-VCF-L : Virtualization 101 (édition 2024), Modules 1 à 3, informations additionnelles et annexe.
* README initial de configuration vCenter et PowerCLI fourni pour le lab.
* Documentation VMware PowerCLI et vSphere, pour les cmdlets de référence et les passages marqués Complément.

> Avertissement : les cmdlets de ce document ont été rédigées d'après le manuel et la documentation PowerCLI, mais n'ont pas été exécutées dans votre lab. Testez chaque section dans l'environnement HOL, et adaptez noms, adresses IP, versions de PowerCLI et syntaxe si une commande échoue. Les cmdlets SSO, les appels d'API (health check, gestion d'énergie, Lockdown) et les paramètres de Storage DRS sont les plus sensibles à la version du module.
