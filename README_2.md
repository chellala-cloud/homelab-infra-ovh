# Homelab infra OVH — Architecture de production (pfSense / VLAN / Proxmox VE)

Infrastructure auto-hébergée sur un serveur dédié OVHCloud, conçue selon les principes d'une architecture de production réelle : segmentation réseau, cloisonnement des rôles, sécurité en profondeur, sauvegarde et supervision.

> **Deuxième itération du projet.** La première version, contrainte à 8 Go de RAM sur un poste personnel, est conservée telle quelle ici : [homelab-infra](#) — *lien à mettre à jour*. Ce nouveau projet reprend les mêmes principes d'architecture sur un vrai serveur dédié, avec les moyens de faire du réellement production-grade plutôt qu'un compromis matériel.

## Sommaire

- [Vue d'ensemble](#vue-densemble)
- [Infrastructure physique](#infrastructure-physique)
- [Segmentation réseau](#segmentation-réseau)
- [Détail des composants](#détail-des-composants)
- [Sécurité — défense en profondeur](#sécurité--défense-en-profondeur)
- [Sauvegarde et observabilité](#sauvegarde-et-observabilité)
- [Limites connues](#limites-connues)
- [Documentation complète](#documentation-complète)
- [Stack technique](#stack-technique)
- [Roadmap](#roadmap)

## Vue d'ensemble

```
Internet (IP failover OVH)
        │
        ▼
┌──────────────┐
│   pfSense    │  Routed Bridge — frontal réseau, seul point d'entrée
└──────┬───────┘
       │
┌──────▼──────────────────────────────────────────────────┐
│  Proxmox VE — serveur dédié OVH                          │
│                                                            │
│  ┌────────┐   ┌────────┐   ┌────────┐   ┌────────┐        │
│  │  DMZ   │   │  APP   │   │  DATA  │   │  MGMT  │        │
│  │ Reverse│   │ Web    │   │ MariaDB│   │ Backup │        │
│  │ proxy  │   │(Apache)│   │        │   │Monitor.│        │
│  │ + WAF  │   │        │   │        │   │        │        │
│  └────────┘   └────────┘   └────────┘   └────────┘        │
└────────────────────────────────────────────────────────────┘
```

## Infrastructure physique

| Composant | Spécification |
|---|---|
| Fournisseur | OVHCloud, serveur dédié |
| CPU | Intel Xeon E3-1230v6 — 4c/8t — 3.5/3.9 GHz |
| RAM | 32 Go ECC 2133 MHz |
| Stockage | 2×2 To HDD SATA, RAID logiciel |
| Hyperviseur | Proxmox VE 9 (template officiel OVHcloud) |

## Segmentation réseau

pfSense est le point d'entrée unique du réseau, configuré en architecture **Routed Bridge** (spécificité OVHCloud : les IP failover sont routées vers la MAC unique du serveur, pfSense assure ensuite le routage interne vers chaque VM).

Quatre VLAN séparent les rôles par fonction :

| VLAN | Rôle | Accès entrant autorisé |
|---|---|---|
| DMZ | Reverse proxy, WAF | pfSense uniquement (443/80) |
| APP | Serveur web (Apache, PHP) | VLAN DMZ uniquement |
| DATA | Base de données (MariaDB) | VLAN APP uniquement, port 3306 |
| MGMT | Sauvegarde, supervision | Accès restreint, jamais exposé publiquement |

Le détail des règles de filtrage inter-VLAN est documenté dans [`docs/02-reseau-vlan.md`](docs/02-reseau-vlan.md).

## Détail des composants

### Firewall — pfSense
Filtrage réseau (IP/port/protocole/état), NAT sortant pour les VLAN internes, routage inter-VLAN strict. Configuration détaillée : [`docs/03-pfsense.md`](docs/03-pfsense.md).

### Reverse Proxy (VLAN DMZ)
Termine le TLS, route selon le nom de domaine vers le serveur web interne, masque l'existence réelle de la VM web.

### Serveur Web (VLAN APP)
Apache + PHP, VirtualHosts nommés, hébergeant l'ERP OpenConcerto — [`docs/05-erp-openconcerto.md`](docs/05-erp-openconcerto.md).

### Base de données (VLAN DATA)
MariaDB, jamais accessible directement depuis l'extérieur ni depuis un autre VLAN que APP.

## Sécurité — défense en profondeur

Plusieurs couches de sécurité indépendantes, chacune répondant à une question différente :

- **pfSense / VLAN** — quels segments réseau ont le droit de se parler
- **nftables** (sur chaque VM) — quelle machine précise, sur quel port précis, a le droit de joindre celle-ci
- **SSH durci** — authentification par clé uniquement, connexion root désactivée
- **Fail2ban** — bannissement automatique après tentatives de connexion répétées
- **WAF** sur le reverse proxy — filtrage applicatif HTTP

Détail des règles : [`docs/04-hardening.md`](docs/04-hardening.md).

## Sauvegarde et observabilité

- **Sauvegarde** : règle 3-2-1-1-0 (3 copies, 2 supports, 1 hors site, 1 immuable, 0 erreur après test), via Proxmox Backup Server ou Borgbackup/Restic selon arbitrage documenté.
- **Supervision** : Prometheus + Grafana + Alertmanager pour les métriques et alertes, logs centralisés.

## Limites connues

Cette infrastructure applique les principes de production, mais reste un projet personnel — les limites suivantes sont assumées et documentées plutôt que masquées :

- **Single point of failure physique** : un seul serveur, aucune haute disponibilité réelle (pas de second nœud Proxmox, pas de cluster, pas de stockage partagé type Ceph).
- **Pas de vraie redondance géographique** au-delà de la copie de sauvegarde hors site.

Ces écarts avec une architecture pleinement HA sont volontairement documentés : ils indiquent précisément ce qu'il faudrait ajouter à l'échelle d'une vraie production (second nœud physique, cluster, stockage partagé).

## Documentation complète

- [`docs/01-architecture.md`](docs/01-architecture.md)
- [`docs/02-reseau-vlan.md`](docs/02-reseau-vlan.md)
- [`docs/03-pfsense.md`](docs/03-pfsense.md)
- [`docs/04-hardening.md`](docs/04-hardening.md)
- [`docs/05-erp-openconcerto.md`](docs/05-erp-openconcerto.md)
- [`docs/06-adr-migration-ovh.md`](docs/06-adr-migration-ovh.md) — pourquoi ce projet succède à la version lab (8 Go RAM) et ce qui a changé

## Stack technique

- Proxmox VE 9 (hyperviseur)
- pfSense (firewall, Routed Bridge)
- Nginx (reverse proxy, WAF)
- Apache + PHP (serveur web)
- MariaDB (base de données)
- OpenConcerto (ERP)
- nftables, Fail2ban, OpenSSH durci
- Prometheus, Grafana (supervision)
- Proxmox Backup Server / Borgbackup (sauvegarde)

## Roadmap

- [ ] VPN site-à-site vers une seconde machine (Windows Server / Active Directory)
- [ ] Automatisation de la configuration (Ansible)
- [ ] Provisioning des ressources OVH via Terraform
- [ ] Tests de restauration de sauvegarde documentés
