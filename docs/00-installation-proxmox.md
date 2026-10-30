# Guide d'installation de Proxmox VE sur serveur dédié OVHcloud

Ce guide détaille l'installation initiale de l'hyperviseur Proxmox VE 9 via le template officiel OVHcloud, avec un accès d'administration sécurisé par tunnel SSH (bootstrap) pour garantir une IP principale 100 % étanche.

## 1. Contexte matériel & Spécifications

| Élément | Valeur |
|---|---|
| Fournisseur | OVHcloud, serveur dédié |
| CPU | Intel Xeon E3-1230v6 — 4c/8t — 3.5/3.9 GHz |
| RAM | 32 Go ECC 2133 MHz |
| Disques | 2×2 To HDD SATA |
| OS / Hyperviseur | Proxmox VE 9 (Template officiel OVHcloud) |

## 2. Lancement de l'installation (Manager OVHcloud)

**Chemin** : Bare Metal Cloud → Serveurs dédiés → [Ton Serveur] → Installer → Lancer une réinstallation à partir d'un template OVHcloud → Type d'OS : Virtualisation → Linux → Proxmox Virtual Environment 9.

### Étape 1/4 — OS et stockage

- **Grappe de disques** : Affichée comme `2 X Disk SATA 2000 GB, JBOD` dans le menu déroulant (libellé générique OVH).
- **Vérification du schéma de partitionnement** :

| Partition | Type / FS | Point de montage | RAID | Taille exploitable | Espace consommé |
|---|---|---|---|---|---|
| 1 | ext4 | /boot | 1 | 1,0 Gio | 2,0 Gio |
| 2 | ext4 | / | 1 | 20,0 Gio | 40,0 Gio |
| 3 | swap | swap | - | 2×1,0 Gio | 2,0 Gio |
| 4 | zfs | /var/lib/vz | 1 | 1,8 Tio | 3,6 Tio |

**Validation** : Le RAID 1 (miroir) est bien appliqué automatiquement par le template OVH (confirmé par la colonne `Raid=1` et le ratio 2× entre l'espace consommé et l'espace exploitable). La case "Hardware RAID" reste grisée, ce qui est normal (RAID logiciel ZFS/LVM). Aucune modification manuelle n'est nécessaire.

### Étape 2/4 — Réseau, hostname et accès

- **Hostname** : `proxmox-ovh` (éviter les noms génériques)
- **Authentification** : Sélectionner ta clé SSH publique (aucun mot de passe root au déploiement)
- **Cloud-init Config Drive Metadata** : Laisser vide (servira ultérieurement pour le provisioning des VMs via Ansible/Terraform)
- **LACP** : Désactivé (le serveur dispose d'une seule interface réseau active)

## 3. Sécurisation de l'IP principale & Bootstrap (Tunnel SSH)

L'interface de gestion de Proxmox (port 8006) ne doit jamais être exposée directement sur l'IP publique.

### A. Configuration du pare-feu OVH (Edge Network Firewall)

Sur l'IP principale de ton serveur dédié :

- **Règle 0 (Autoriser)** : Protocole TCP, IP Source `TON_IP_PUBLIQUE/32`, Port destination `22` (SSH)
- **Règle 1 (Refuser)** : Refuser tout le reste (DROP)
- Activer le pare-feu


### B. Établissement du tunnel SSH depuis ton poste local

Pour accéder à l'IHM web Proxmox de façon sécurisée, encapsule le trafic Web dans le tunnel SSH :

```bash
ssh -L 8006:127.0.0.1:8006 root@<IP_PRINCIPALE_OVH>
```

### C. Vérifications système initiales (en CLI)

Dans la session SSH ouverte, valide l'état du système :

```bash
pveversion          # Vérifier la version de Proxmox
free -h              # Valider la RAM disponible
nproc                # Valider le nombre de vCPUs
zpool status         # Confirmer l'état du ZFS
```

**Vérification RAID ZFS** : la commande `zpool status` doit afficher `state: ONLINE` avec un `mirror-0` regroupant les deux disques en état `ONLINE` sans erreur (`READ WRITE CKSUM` à `0 0 0`).

## 4. Définition du mot de passe root pour l'IHM Web

L'authentification par clé SSH ne permet pas de se connecter directement à l'IHM Web (`:8006`). Il faut définir un mot de passe pour l'utilisateur root :

```bash
passwd
```

## 5. Configuration de l'interface réseau interne (vmbr1)

L'IHM de Proxmox est désormais accessible de manière sécurisée sur ton navigateur à l'adresse `https://localhost:8006` (via le tunnel SSH).

### Création du bridge virtuel (vmbr1)

1. Dans l'IHM Proxmox, sélectionner le nœud `proxmox-ovh` dans le menu de gauche
2. Naviguer vers **Système → Réseau**
3. Cliquer sur **Créer → Linux Bridge** et renseigner les champs suivants :
   - **Name** : `vmbr1`
   - **IPv4/CIDR** : (laisser vide — Proxmox ne porte pas d'IP sur ce bridge)
   - **Bridge ports** : (laisser vide — bridge purement virtuel, non rattaché à une carte physique)
   - **VLAN aware** : coché (indispensable pour le trunk VLANs 10/20/30/40 de pfSense)
4. Cliquer sur **Créer**
5. Cliquer sur **Apply Configuration** en haut de la page pour appliquer la modification à chaud

### Validation en CLI

Vérifier que le bridge est bien actif en session SSH :

```bash
ip a | grep vmbr1
```

**Avertissement** : ne jamais modifier ou supprimer `vmbr0`. Il est rattaché à la carte réseau physique et porte l'IP principale du serveur dédié.

## Pièges évités & Rappels

- **Faux affichage JBOD** : ne pas se fier au libellé du menu déroulant OVH. Toujours valider la présence du RAID 1 via le tableau de partitionnement et la commande `zpool status`.
- **Étanchéité du port 8006** : l'IHM Web Proxmox ne doit être joignable qu'au travers de `https://localhost:8006` via le tunnel SSH en phase de bootstrap, puis via `https://10.10.40.254:8006` après la mise en place du VPN WireGuard sur le VLAN 40 (MGMT).
- **Application du réseau** : la création d'un bridge dans l'IHM Proxmox reste inactive tant que le bouton "Apply Configuration" n'a pas été cliqué.
