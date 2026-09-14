# Sécurité — règles choisies et mise en place

Liste des mesures de sécurité du lab, une par une. Le statut est mis à jour au fur et à mesure.

## Le principe : plusieurs barrières, pas une seule

Chaque mesure bloque un type de problème différent. Aucune ne remplace les autres.

| Barrière | Elle sert à... |
|---|---|
| Firewall OVH | Décider qui a le droit d'atteindre le serveur, avant même d'y arriver |
| pfSense / VLAN | Décider quelles parties du réseau interne peuvent se parler entre elles |
| nftables (sur chaque machine) | Décider qui peut joindre CETTE machine précise, sur quel port |
| SSH sécurisé | Décider comment on a le droit de se connecter à une machine |
| Fail2ban | Bloquer automatiquement une IP qui essaie de se connecter trop de fois |
| WAF (reverse proxy) | Vérifier qu'une requête web n'est pas suspecte |

## Firewall OVH — sur l'IP principale

**Fait.**

S'applique à l'IP principale du serveur (`IP-DU-SERVEUR`), celle qui donne accès à Proxmox (`:8006`) et au SSH (`:22`).

| Priorité | Action | Protocole | IP autorisée | Port | État |
|---|---|---|---|---|---|
| 0 | Autoriser | TCP | mon IP perso | 22 | - |
| 1 | Autoriser | TCP | mon IP perso | 8006 | - |
| 2 | Autoriser | TCP | tous | - | established |
| 3 | Refuser | TCP | (vide) | (vide) | - |
| 4 | Refuser | UDP | (vide) | (vide) | - |

**Problème connu** : mon IP à la maison change de temps en temps (elle n'est pas fixe). Quand ça arrive, la règle devient fausse et je perds l'accès au serveur. Solution prévue plus tard : un VPN sur pfSense, qui vérifie une clé au lieu de vérifier une IP — plus besoin de mettre à jour cette règle à chaque changement.

**Ce qui s'est passé** : mon IP a changé, ça m'a coupé l'accès SSH et web. J'ai réglé le problème en récupérant ma nouvelle IP (`curl -4 ifconfig.me`) et en corrigeant les règles à la main. J'ai pensé à automatiser ça avec un script qui met à jour la règle tout seul, mais j'ai préféré ne pas m'éparpiller — le VPN va rendre ce script inutile de toute façon. Peut-être à faire plus tard, mais pas maintenant.

## Ce qu'il reste à faire, dans l'ordre utile

Le firewall OVH bloque déjà tout sauf mon IP. Donc les mesures suivantes ne sont pas classées dans l'ordre où j'y ai pensé, mais dans l'ordre de ce qu'elles apportent vraiment en plus :

1. **Couper la connexion SSH par mot de passe** — fait — utile si jamais quelqu'un réussit à se faire passer pour mon IP, ou si j'élargis la règle un jour
2. **Double authentification sur l'interface web Proxmox** — fait — la seule mesure ici qui protège contre un problème qui n'a rien à voir avec le réseau (mot de passe volé par phishing par exemple)
3. **Fail2ban** — à faire — pas très utile tant que seule mon IP peut atteindre le serveur ; deviendra utile une fois le VPN en place
4. **nftables sur le serveur lui-même** — à faire — pour l'instant, ferait juste la même chose que le firewall OVH (bloquer tout sauf mon IP), donc pas urgent

## Couper la connexion SSH par mot de passe

**Fait.**

```bash
ssh root@IP-DU-SERVEUR
sed -i 's/^#\?PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin prohibit-password/' /etc/ssh/sshd_config
systemctl restart sshd
```

- `PasswordAuthentication no` — coupe la connexion par mot de passe, seule la clé fonctionne
- `PermitRootLogin prohibit-password` — la clé passe pour root, le mot de passe est refusé même pour root

**Avant de fermer la session en cours** : ouvrir un second terminal et tester une nouvelle connexion SSH pour confirmer que la clé fonctionne toujours, avant de couper la session existante.

## Double authentification — interface web Proxmox

**Fait.**

1. Interface web `:8006` → **Datacenter** (menu de gauche) → **Permissions** → **Two Factor**
2. Ajouter (bouton "Add") → choisir **TOTP**
3. Scanner le QR code avec une app d'authentification
4. Confirmer avec le code à 6 chiffres généré par l'app

**App utilisée** : open-source, gratuite — **Aegis Authenticator** (Android) ou **2FAS Auth** (iOS et Android). Les deux permettent une sauvegarde chiffrée exportable soi-même, sans dépendre d'un compte cloud tiers.

Protège même si le mot de passe web fuite un jour.

## Fail2ban

**À faire. Priorité 3 — vraiment utile une fois le VPN en place.**

```bash
apt update
apt install fail2ban -y
systemctl enable --now fail2ban
fail2ban-client status sshd
```

Surveille les tentatives de connexion SSH ratées, bannit l'IP après trop d'essais. La jail `sshd` est activée par défaut sur les configurations Debian récentes.

## nftables sur le serveur Proxmox

**À faire. Priorité 4 — la moins urgente pour l'instant.**

```bash
apt install nftables -y
nft add table inet filter
nft add chain inet filter input { type filter hook input priority 0 \; policy drop \; }
nft add rule inet filter input ip saddr TON-IP/32 tcp dport { 22, 8006 } accept
nft add rule inet filter input ct state established,related accept
nft add rule inet filter input iif lo accept
```

Règle locale qui limite `22`/`8006` à mon IP, en plus du firewall OVH — redondant tant qu'il n'y a rien d'autre à filtrer sur ce serveur.

**Rendre permanent** (sinon perdu au reboot) :
```bash
nft list ruleset > /etc/nftables.conf
systemctl enable nftables
```

## pfSense / VLAN

**En cours.** La VM pfSense est installée, le réseau WAN est configuré à la main (temporaire). Les VLAN (DMZ/APP/DATA/MGMT) ne sont pas encore créés.

## nftables sur chaque VM (Reverse Proxy / Web / BDD)

**Pas encore utile** — ces VM n'existent pas encore.

## Fail2ban sur chaque VM

**Pas encore utile** — ces VM n'existent pas encore.

## WAF (reverse proxy)

**Pas encore utile** — le reverse proxy n'est pas encore installé.

## Leçon apprise : anonymiser les IP dans la doc publique

Erreur de débutant faite sur ce repo : l'IP principale du serveur (`5.135.138.58`) a été laissée en clair dans deux docs déjà poussées sur GitHub (`00-installation-proxmox.md`, `03-pfsense.md`), avant de réaliser que ça reste visible dans l'historique même après correction. Réécrire l'historique Git pour l'effacer a été écarté — le coût (casser l'historique construit comme preuve de travail) dépasse le bénéfice (une IP de serveur dédié n'est de toute façon jamais vraiment secrète, repérable par un scan automatique en quelques heures).

**Consigne pour la suite** : dans toute nouvelle doc destinée à GitHub, remplacer l'IP réelle par un placeholder (`IP-DU-SERVEUR`, `TON-IP`, etc.) avant le premier commit — plus simple de le faire dès le départ que de corriger après coup une fois poussé.
