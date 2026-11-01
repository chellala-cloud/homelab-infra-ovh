# 🛡️ Infrastructure Réseau & Sécurité Multi-VLAN sur Proxmox VE

![Proxmox](https://img.shields.io/badge/Proxmox_VE-9.x-E57000?logo=proxmox&logoColor=white)
![pfSense](https://img.shields.io/badge/pfSense-2.9.x-212F3D?logo=pfsense&logoColor=white)
![Traefik](https://img.shields.io/badge/Traefik-v3-24A1DE?logo=traefik&logoColor=white)
![Licence](https://img.shields.io/badge/License-MIT-blue.svg)

> **Projet de déploiement d'une infrastructure d'entreprise sécurisée sur serveur dédié OVHcloud.**  
> Ce dépôt documente l'architecture réseau complète : segmentation par VLANs sous pfSense (Routed Bridge), filtrage applicatif (Traefik, WAF, CrowdSec), durcissement système et observabilité.

---

## Vue d'ensemble de l'architecture

Architecture cible :

```mermaid
graph TD
    Internet[Internet / IP Additionnelle OVH] -->|vMAC OVH| VMBR0[vmbr0 - Bridge WAN L2]

    subgraph Proxmox [Hôte Proxmox VE]
        VMBR0 -->|WAN| PFSENSE[VM pfSense]
        PFSENSE -->|LAN / 192.168.100.1| VMBR101[vmbr101 - Bridge LAN]

        subgraph Network_Segmentation [Réseau Interne & VLANs segmentés]
            VMBR101 --> VLAN10[VLAN 10 - DMZ]
            VMBR101 --> VLAN20[VLAN 20 - APP]
            VMBR101 --> VLAN30[VLAN 30 - DATA]
            VMBR101 --> VLAN40[VLAN 40 - MGMT]

            VLAN10 --> DMZ_Stack[Traefik / CrowdSec / WAF]
            VLAN20 --> APP_Stack[Serveur web / Apache / PHP]
            VLAN30 --> DATA_Stack[Base de données]
            VLAN40 --> MGMT_Stack[Monitoring / Backup / PBS]
        end
    end

    classDef host fill:#2b2b2b,stroke:#4caf50,stroke-width:2px,color:#fff;
    classDef router fill:#003366,stroke:#0066cc,stroke-width:2px,color:#fff;
    class Proxmox host;
    class PFSENSE router;
```

---

## Où j'en suis

| Étape | Statut |
| :--- | :--- |
| Serveur Proxmox installé et accès sécurisé (pare-feu OVH, SSH par clé, double authentification) | Fait |
| pfSense installé et accessible par tunnel SSH, sur un réseau d'installation temporaire | Fait |
| Bascule du WAN de pfSense sur l'IP Additional (vMAC OVH), avec son pare-feu OVH | Fait |
| VPN WireGuard | À faire |
| VLAN, reverse proxy, WAF, CrowdSec | À faire |
| Application hébergée | À choisir |
| Sauvegarde et supervision | À faire |

pfSense a besoin d'Internet pour s'installer, alors que l'IP Additional d'OVH n'a pas d'accès direct à la passerelle. Je l'ai donc installé d'abord derrière un **sas NAT temporaire** sur l'hôte, puis j'ai basculé son WAN sur l'IP Additional. Le sas sera retiré une fois le VPN en place. Le raisonnement et les incidents sont dans [docs/03-pfsense.md](docs/03-pfsense.md).

---

## Infrastructure physique

| Composant | Spécification |
| :--- | :--- |
| **Fournisseur** | OVHcloud, serveur dédié |
| **CPU** | Intel Xeon E3-1230v6 — 4c/8t — 3.5/3.9 GHz |
| **RAM** | 32 Go ECC 2133 MHz |
| **Stockage** | 2×2 To HDD SATA, RAID logiciel |
| **Hyperviseur** | Proxmox VE 9 (template officiel OVHcloud) |

---

## Segmentation réseau

pfSense est le point d'entrée unique du réseau, configuré en architecture **Routed Bridge** (spécificité OVHcloud : l'IP Additional est associée à une vMAC attribuée à la carte réseau virtuelle de pfSense, qui assure le routage et le filtrage interne vers chaque VM).

Quatre VLANs séparent les rôles par fonction :

| VLAN | Rôle | Accès entrant autorisé |
| :--- | :--- | :--- |
| **DMZ (10)** | Reverse proxy, WAF | Internet via pfSense (ports 80/443 uniquement) |
| **APP (20)** | Serveur web (Apache, PHP) | VLAN DMZ uniquement |
| **DATA (30)** | Base de données | VLAN APP uniquement (port de la base, 3306 pour MariaDB) |
| **MGMT (40)** | Sauvegarde, supervision, PVE | Accès distant via **VPN WireGuard (pfSense)** ou tunnel SSH |

Le détail des règles de filtrage inter-VLAN est documenté dans [docs/02-reseau-vlan.md](docs/02-reseau-vlan.md).

---

## Détail des composants

* **Firewall / Routeur / DNS — pfSense :** 
  * Filtrage réseau L3/L4, NAT sortant, routage inter-VLAN strict.
  * Serveur **VPN WireGuard** (à venir) pour l'accès d'administration au VLAN MGMT et à l'IHM Proxmox (`:8006`). En attendant, l'accès passe par un tunnel SSH.
  * Service **Unbound DNS (Split-Horizon)** pour la résolution directe des noms de domaine internes sans passer par le WAN (évite le Hairpin NAT). Configuration détaillée : [docs/03-pfsense.md](docs/03-pfsense.md).

* **Reverse Proxy & Edge Security (VLAN DMZ) :** 
  * **Traefik :** Gestion SSL/TLS automatique (Let's Encrypt), routage dynamique vers les conteneurs/VMs internes.
  * **WAF + CrowdSec :** Inspection applicative HTTP (L7) et blocage collaboratif des IP malveillantes.

* **Serveur Web (VLAN APP) :** 
  * Apache + PHP, VirtualHosts nommés. L'application hébergée reste à choisir.

* **Base de données (VLAN DATA) :** 
  * MariaDB à ce stade, accessible uniquement depuis le VLAN APP. Le moteur sera confirmé quand l'application sera choisie : certaines applications imposent PostgreSQL.

---

## Sécurité — défense en profondeur

Plusieurs couches de sécurité indépendantes, chacune répondant à un besoin spécifique :

* **Pare-feu OVH (Edge Network Firewall) :** N'autorise l'accès au SSH et à l'interface Proxmox que depuis mon IP, avant même que le trafic n'atteigne le serveur.
* **pfSense / VLANs :** Isolation réseau L2/L3 et filtrage inter-VLAN strict.
* **nftables (sur l'hôte et sur chaque VM) :** Pare-feu local filtrant strictement les flux entrants/sortants. Sur l'hôte, une règle par interface (LAN et VPN) est prévue après la mise en place du VPN.
* **CrowdSec :** Détection d'intrusions et bannissement automatique comportemental.
* **Fail2ban (sur l'hôte) :** Bannissement des IP qui échouent trop de fois à se connecter, une fois le VPN en place.
* **WAF (sur Traefik) :** Filtrage applicatif contre les injections SQL, XSS et failles OWASP Top 10.
* **SSH durci :** Authentification par clé uniquement, mot de passe désactivé, connexion `root` interdite (utilisateur d'administration dédié avec `sudo`), limites sur les tentatives et les redirections de ports, journalisation de la clé utilisée à chaque connexion.
* **Tunnel SSH par clé dédiée :** Clé sans shell, limitée aux interfaces web de pfSense et de Proxmox.
* **Double authentification (TOTP) :** Sur l'interface web de Proxmox.

Détail des règles de durcissement : [docs/04-hardening.md](docs/04-hardening.md).

---

## Sauvegarde et observabilité

* **Sauvegarde :** Règle 3-2-1-1-0 (3 copies, 2 supports, 1 hors site, 1 immuable, 0 erreur après test), via Proxmox Backup Server ou Borgbackup/Restic selon l'arbitrage documenté.
* **Supervision :** Prometheus + Grafana + Alertmanager pour les métriques et alertes, logs centralisés.

---

## Limites connues

Cette infrastructure applique les principes de production, mais reste un projet personnel — les limites suivantes sont assumées et documentées plutôt que masquées :

* **Single point of failure physique :** Un seul serveur, aucune haute disponibilité réelle (pas de second nœud Proxmox, pas de cluster, pas de stockage partagé type Ceph).
* **Pas de redondance géographique :** En dehors de la copie de sauvegarde hors site.
* **Accès d'administration lié à mon IP :** Mon IP à la maison n'est pas fixe. Tant que le VPN n'est pas en place, la règle du pare-feu OVH est mise à jour à la main quand elle change, et le KVM/IPMI d'OVH sert d'accès de secours.

Ces écarts avec une architecture pleinement HA sont volontairement documentés : ils indiquent précisément ce qu'il faudrait ajouter à l'échelle d'une vraie production (second nœud physique, cluster, stockage partagé).

---

## Documentation complète

* [docs/01-architecture.md](docs/01-architecture.md)
* [docs/02-reseau-vlan.md](docs/02-reseau-vlan.md)
* [docs/03-pfsense.md](docs/03-pfsense.md) — installation de pfSense, sas NAT temporaire, accès à la GUI, bascule sur l'IP Additional. Le pas-à-pas complet est dans [docs/03-pfsense-pas-a-pas.md](docs/03-pfsense-pas-a-pas.md).
* [docs/04-hardening.md](docs/04-hardening.md) — sécurité : pare-feu OVH, SSH, tunnel, double authentification, secours.
* [docs/06-adr-migration-ovh.md](docs/06-adr-migration-ovh.md) — pourquoi ce projet succède à la version lab (8 Go RAM) et ce qui a changé.

---

## Stack technique

* **Hyperviseur :** Proxmox VE 9
* **Réseau & Sécurité :** pfSense (Routed Bridge, Unbound DNS, WireGuard)
* **Reverse Proxy & WAF :** Traefik, Coraza WAF, CrowdSec
* **Web & Data :** Apache, PHP, MariaDB (application à choisir)
* **Système :** Debian, nftables, OpenSSH durci, Fail2ban
* **Supervision & Sauvegarde :** Prometheus, Grafana, Proxmox Backup Server / Borgbackup

---

## Roadmap

* [ ] VPN WireGuard sur pfSense, puis retrait du sas NAT temporaire
* [ ] Choix et déploiement de l'application hébergée (VLAN APP et DATA)
* [ ] VPN site-à-site vers une seconde machine (Windows Server / Active Directory)
* [ ] Automatisation de la configuration (Ansible)
* [ ] Provisioning des ressources OVH via Terraform
* [ ] Tests de restauration de sauvegarde documentés

---

## Auteur & Contact

* **GitHub :** (https://github.com/chellala-cloud)
* **LinkedIn :** (https://www.linkedin.com/in/amina-hamza33)
* **Projet :** Conçu et documenté dans le cadre de mon portfolio technique Systems & Networks.
