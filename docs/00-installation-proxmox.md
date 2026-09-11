# Installer Proxmox VE sur serveur dédié OVH — guide

## Contexte du serveur

| Élément | Valeur |
|---|---|
| Fournisseur | OVHCloud, serveur dédié |
| CPU | Intel Xeon E3-1230v6 — 4c/8t — 3.5/3.9 GHz |
| RAM | 32 Go ECC 2133 MHz |
| Disques | 2×2 To HDD SATA |

## Installation via template officiel OVHcloud

Plus simple et fiable qu'une installation manuelle par IPMI/ISO.

**Chemin dans le Manager** : Bare Metal Cloud → serveur → Installation à partir d'un template OVHcloud → Type de l'OS : **Virtualisation** → Linux → **Proxmox Virtual Environment 9**.

## Étape 1/4 — OS et stockage

**Grappe de disques cible** : affichée comme "2 X Disk SATA 2000 GB, JBOD" dans le menu déroulant, mais c'est trompeur — vérifier via le tableau de partitionnement détaillé :

| Partition | FS | Point de montage | RAID | Taille exploitable | Espace consommé |
|---|---|---|---|---|---|
| 1 | ext4 | /boot | 1 | 1.0 Gio | 2.0 Gio |
| 2 | ext4 | / | 1 | 20.0 Gio | 40.0 Gio |
| 3 | swap | swap | - | 2×1.0 Gio | 2.0 Gio |
| 4 | zfs | /var/lib/vz | 1 | 1.8 Tio | 3.6 Tio |

**Conclusion** : RAID 1 (miroir) bien appliqué automatiquement par le template OVH, malgré le libellé "JBOD" du menu. La colonne `Raid=1` et le ratio 2x entre espace consommé et exploitable le confirment. Case "hardware RAID" grisée car pas de carte RAID physique — RAID logiciel géré via ZFS/LVM au niveau des partitions. Aucune personnalisation nécessaire.

## Étape 2/4 — Réseau, hostname, accès

- **Hostname** : `proxmox-ovh` — éviter les noms génériques
- **Authentification** : clé SSH publique fournie (pas de mot de passe root) — cohérent avec le hardening SSH prévu (`PermitRootLogin no`, auth par clé uniquement)
- **Cloud-init Config Drive Metadata** : laissé vide — utile seulement pour du provisioning automatisé de VM plus tard (Ansible/Terraform, en roadmap), pas nécessaire pour l'hyperviseur lui-même
- **LACP** : désactivé — le serveur n'a qu'une interface réseau active, pas de bénéfice à l'agrégation ici

## Vérifications après installation

```bash
ssh root@5.135.138.58

pveversion          # version Proxmox installée
free -h              # RAM vue par le système
nproc                # CPU vus par le système
zpool status          # état du RAID/ZFS
```

`zpool status` doit afficher `state: ONLINE`, un `mirror-0` avec les deux disques `ONLINE`, `READ WRITE CKSUM` à `0 0 0` — confirmation finale du RAID 1 réel, au-delà du tableau affiché par l'installeur OVH.

## Mot de passe root pour l'interface graphique

L'auth par clé couvre SSH, pas l'interface web `:8006` — deux mécanismes séparés :
```bash
passwd
```

## Première étape de l'infra : créer vmbr1

Via l'interface web (plus sûr que l'édition manuelle du fichier réseau) :

1. Interface Proxmox `:8006` → clique sur ton nœud (le nom du serveur) dans le menu de gauche
2. **Système → Réseau**
3. Bouton **"Créer" → "Linux Bridge"**
4. Remplis :
   - **Name** : `vmbr1`
   - **IPv4/CIDR** : laisse vide — ce bridge n'a pas besoin d'IP côté Proxmox, c'est pfSense qui sera le seul à porter une IP dessus (`10.10.x.1`)
   - **Bridge ports** : laisse vide — c'est ce qui en fait un bridge purement interne, sans carte physique attachée
   - **VLAN aware** : coche cette case — indispensable pour que les VLAN (DMZ/APP/DATA/MGMT) puissent transiter dessus
5. Clique **"Créer"**, puis en haut de la page réseau, un bouton **"Apply Configuration"** apparaît — clique dessus pour appliquer sans reboot complet du serveur

Vérification que le bridge est bien actif (pas juste créé dans l'UI) :
```bash
ip a | grep vmbr1
```

Ne jamais toucher à `vmbr0` pendant cette manip — il porte l'IP principale, donc l'accès SSH/web actuel.

## Pièges à ne jamais refaire

- Ne pas se fier au libellé "JBOD" du menu déroulant → toujours vérifier le tableau de partitionnement détaillé et/ou `zpool status`
- Toujours cliquer "Apply Configuration" après une création/modif réseau, sinon la config reste inactive malgré son apparence dans l'UI
- Le mot de passe root (`passwd`) est nécessaire pour l'interface web même en étant déjà authentifié par clé SSH côté serveur
