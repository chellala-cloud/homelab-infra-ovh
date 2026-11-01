# Sécurité — règles choisies et mise en place

Liste des mesures de sécurité du lab, une par une. Le statut est mis à jour au fur et à mesure.

## Le principe : plusieurs barrières, pas une seule

Chaque mesure bloque un type de problème différent. Aucune ne remplace les autres.

| Barrière | Elle sert à... |
|---|---|
| Firewall OVH | Décide qui a le droit d'atteindre le serveur, avant même d'y arriver |
| pfSense / VLAN | Décide quelles parties du réseau interne peuvent se parler entre elles |
| nftables (sur chaque machine) | Décide qui peut joindre quelle machine précise, sur quel port |
| SSH sécurisé | Décide comment on a le droit de se connecter à une machine |
| Clé de tunnel restreinte | Limite ce que mon accès de tous les jours peut atteindre : deux interfaces web, sans shell |
| VPN WireGuard (pfSense) | Donne accès au réseau interne avec une clé, sans dépendre de mon IP |
| Fail2ban | Bloquer automatiquement une IP qui essaie de se connecter trop de fois |
| WAF (reverse proxy) | Vérifie qu'une requête web n'est pas suspecte |

## Les accès au lab, une fois tout en place

| Accès | Chemin | Dépend de | Usage |
|---|---|---|---|
| SSH avec l'utilisateur d'administration (clé avec shell) | PC → port 22 | règle OVH sur mon IP | administration en ligne de commande |
| Tunnel SSH (clé dédiée) | PC → port 22 → GUI pfSense et GUI Proxmox | règle OVH sur mon IP | accès aux interfaces web |
| GUI Proxmox directe | `IP-DU-SERVEUR:8006` | règle OVH sur mon IP + TOTP | dernier recours |
| VPN WireGuard (à faire) | PC → IP failover (UDP) → LAN | VM pfSense en marche | accès aux interfaces web, sans dépendre de mon IP |
| KVM/IPMI et rescue OVH | manager OVH | compte OVH uniquement | secours quand le SSH ne passe plus |

Le KVM ne dépend ni du réseau du serveur, ni de pfSense, ni du SSH : c'est la dernière porte.

## Firewall OVH — sur l'IP principale

**Fait.**

S'applique à l'IP principale du serveur (`IP-DU-SERVEUR`), celle qui donne accès à Proxmox (`:8006`) et au SSH (`:22`).

| Ordre | Action | Protocole | IP source | Port source | Port destination | État TCP |
|---|---|---|---|---|---|---|
| 1 | Autoriser | TCP | tous | - | - | established |
| 2 | Autoriser | TCP | mon IP perso | - | 22 | - |
| 3 | Autoriser | TCP | mon IP perso | - | 8006 | - |
| 4 | Autoriser | ICMP | tous | - | - | - |
| 5 | Autoriser | TCP | tous | 53 | - | - |
| 6 | Autoriser | UDP | tous | 53 | - | - |
| 7 | Refuser | IPv4 | tous | - | - | - |

Ce pare-feu ne suit pas les connexions : la règle `established` (la première) est ce qui laisse revenir les réponses TCP aux connexions lancées depuis le serveur. Pour le DNS, les deux règles avec le port source 53 (TCP et UDP) laissent revenir les réponses des résolveurs. L'ICMP autorisé permet de tester avec `ping`. La dernière règle refuse tout le reste.

**Vérifié** : `apt update`, la résolution DNS et la synchronisation de l'heure fonctionnent. Il n'y a pas de règle pour les réponses NTP (UDP, port source 123) : si l'heure cesse de se synchroniser, en ajouter une avant la règle de refus.

**Problème connu** : mon IP à la maison change de temps en temps (elle n'est pas fixe). Quand ça arrive, la règle devient fausse et je perds l'accès au serveur. Solution prévue : un VPN sur pfSense, qui vérifie une clé au lieu de vérifier une IP — plus besoin de mettre à jour cette règle à chaque changement.

**Ce qui s'est passé** : mon IP a changé, ça m'a coupé l'accès SSH et web. J'ai réglé le problème en récupérant ma nouvelle IP (`curl -4 ifconfig.me`) et en corrigeant les règles à la main. J'ai pensé à automatiser ça avec un script qui met à jour la règle tout seul, mais j'ai préféré ne pas m'éparpiller — le VPN va rendre ce script inutile de toute façon. Peut-être à faire plus tard, mais pas maintenant.

### Firewall OVH — sur l'IP failover de pfSense

**Fait.**

Le pare-feu d'OVH est propre à chaque IP : les règles de l'IP principale ne protègent pas l'IP failover. Il faut lui créer les siennes.

| Ordre | Action | Protocole | IP source | Port source | Port destination | État TCP |
|---|---|---|---|---|---|---|
| 1 | Autoriser | TCP | tous | - | - | established |
| 2 | Autoriser | TCP | tous | 53 | - | - |
| 3 | Autoriser | UDP | tous | 53 | - | - |
| 4 | Autoriser | UDP | tous | 123 | - | - |
| 5 | Refuser | IPv4 | tous | - | - | - |

pfSense sort par cette IP : les mêmes règles DNS que sur l'IP principale s'appliquent, plus une règle pour les réponses NTP (UDP, port source 123), car pfSense synchronise son heure auprès de serveurs extérieurs.

**Vérifié après l'activation** : résolution DNS, synchronisation de l'heure, mises à jour de pfSense et passerelle en ligne.

**À ajouter avec le VPN** : une règle qui autorise le port UDP de WireGuard (`PORT-WG`, en destination), placée avant le refus de l'IPv4. Ce sera le seul port ouvert à tout Internet.

## SSH — couper la connexion par mot de passe

**Fait.**

```bash
sed -i 's/^#\?PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
systemctl restart sshd
```

- `PasswordAuthentication no` — coupe la connexion par mot de passe, seule la clé fonctionne

**Avant de fermer la session en cours** : ouvrir un second terminal et tester une nouvelle connexion SSH pour confirmer que la clé fonctionne toujours, avant de couper la session existante.

## SSH — utilisateur dédié, connexion root interdite

**Fait.**

Je ne me connecte plus en `root` : j'ai créé un utilisateur d'administration, avec son propre dossier `.ssh` et sa propre clé. `adminsys` est un nom d'exemple, à remplacer par le vrai.

**Pourquoi** : `root` existe sur toutes les machines Linux, c'est donc le nom que les scans automatiques essaient en premier. Avec un utilisateur dédié, une clé volée ne donne pas directement `root` : il faut encore le mot de passe de `sudo`. Et les logs montrent qui a fait quoi.

**1. Créer l'utilisateur et lui donner `sudo`** (Proxmox n'installe pas `sudo` par défaut)

```bash
apt install sudo
adduser adminsys
usermod -aG sudo adminsys
```

Le mot de passe choisi sert à `sudo`, pas à la connexion SSH.

**2. Lui donner son propre dossier `.ssh` avec ma clé publique**

```bash
install -d -m 700 -o adminsys -g adminsys /home/adminsys/.ssh
nano /home/adminsys/.ssh/authorized_keys
chown adminsys:adminsys /home/adminsys/.ssh/authorized_keys
chmod 600 /home/adminsys/.ssh/authorized_keys
```

Si le dossier appartenait à `root`, ou s'il était accessible à d'autres, `sshd` ignorerait `authorized_keys` (option `StrictModes`), souvent sans message clair côté client.

**3. Tester avant de couper `root`**, depuis un second terminal :

```bash
ssh -i ~/.ssh/id_ed25519 adminsys@IP-DU-SERVEUR
sudo qm list
```

Les commandes `qm`, `pvesm`, `ifreload` et `nft` demandent root : je les lance avec `sudo`, ou dans un shell root ouvert par `sudo -i`.

**4. Interdire `root` en SSH**

```bash
sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
sshd -t
systemctl reload ssh
```

Depuis un nouveau terminal, `ssh root@IP-DU-SERVEUR` doit être refusé, et `ssh adminsys@IP-DU-SERVEUR` doit fonctionner. Je garde la session ouverte jusqu'à ce test.

**À savoir** : le mot de passe de `sudo` est celui de l'utilisateur ; il va dans un gestionnaire de mots de passe. S'il est perdu, le KVM d'OVH permet de le réinitialiser (`passwd adminsys`). L'interface web de Proxmox continue d'utiliser `root@pam`, protégé par le TOTP.

## SSH — limites sur les tentatives, les redirections et les logs

**Fait.**

J'ajoute des limites dans un fichier dédié : sous Debian, les fichiers de `/etc/ssh/sshd_config.d/` sont lus avant `sshd_config` et priment sur lui, donc je n'ai pas à toucher au fichier principal.

```bash
nano /etc/ssh/sshd_config.d/99-hardening.conf
```

```
# Limiter les tentatives et les connexions
MaxAuthTries 3
LoginGraceTime 30
MaxStartups 10:30:60
ClientAliveInterval 300
ClientAliveCountMax 2

# Fonctionnalités inutiles désactivées
X11Forwarding no
AllowAgentForwarding no
PermitTunnel no
AllowTcpForwarding local

# Journalisation détaillée (empreinte de la clé utilisée à chaque connexion)
LogLevel VERBOSE
```

| Option | Effet |
|---|---|
| `MaxAuthTries 3`, `LoginGraceTime 30`, `MaxStartups 10:30:60` | limitent les essais par connexion, le temps pour s'authentifier et les connexions simultanées non authentifiées |
| `ClientAliveInterval`, `ClientAliveCountMax` | coupent une session inactive |
| `X11Forwarding no` | pas d'affichage graphique distant |
| `AllowAgentForwarding no` | le serveur ne peut pas utiliser mon `ssh-agent` (pour rebondir : `ssh -J`) |
| `PermitTunnel no` | interdit `ssh -w` (un VPN improvisé avec une interface `tun`) |
| `AllowTcpForwarding local` | autorise `ssh -L` (mes tunnels), interdit `ssh -R` |
| `LogLevel VERBOSE` | enregistre l'empreinte de la clé à chaque connexion réussie |

**Ne jamais mettre `AllowTcpForwarding no`** : ça casserait les tunnels vers les interfaces web.

```bash
sshd -t
systemctl reload ssh
sshd -T | grep -E 'passwordauthentication|permitrootlogin|allowtcpforwarding|maxauthtries'
```

`sshd -t` valide la syntaxe (ne pas recharger s'il affiche une erreur). `sshd -T` affiche les valeurs réellement appliquées : à vérifier avant de fermer la session ouverte, et tester une connexion depuis un second terminal.

Vérifier aussi le fuseau horaire (`timedatectl`). S'il est faux (`timedatectl set-timezone Europe/Paris`), les horodatages des logs sont décalés, ce qui gêne quand je cherche une connexion dans le journal.

## Tunnel SSH par clé dédiée

**Fait.**

J'ai deux clés SSH, avec deux rôles différents :

| Clé | Ce qu'elle permet | Si elle est volée |
|---|---|---|
| Clé d'administration (celle de l'utilisateur dédié) | ouvre un shell : je peux tout faire sur le serveur, et passer `root` avec `sudo` | le voleur a un shell sur le serveur ; il lui faut encore mon mot de passe `sudo` pour devenir `root` |
| Clé de tunnel (nouvelle) | une seule chose : ouvrir un tunnel vers la GUI de pfSense et la GUI de Proxmox, sans shell | le voleur n'atteint que ces deux pages de connexion, qu'il doit encore franchir (mot de passe, et TOTP pour Proxmox) |

Pourquoi ne pas simplement limiter les tunnels de la clé d'administration ? Parce que ça ne protège de rien : quelqu'un qui vole cette clé obtient un shell, et depuis un shell il atteint tout, sans avoir besoin d'un tunnel. Une restriction n'a de sens que sur une clé qui n'a pas de shell. C'est aussi pour ça que je n'ai pas mis de `PermitOpen` global dans `sshd_config`.

Au quotidien, j'utilise donc la clé de tunnel pour les interfaces web, et je garde la clé d'administration pour la ligne de commande.

**1. Générer la clé sur le PC**

```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_tunnel -C "tunnel-lab"
```

Avec une passphrase : tout le risque d'une clé repose sur son vol.

**2. La déclarer sur le serveur**, dans le fichier `authorized_keys` de l'utilisateur dédié (`/home/adminsys/.ssh/authorized_keys`), en une seule ligne. La clé de tunnel se connecte avec ce compte, puisque `root` est interdit en SSH :

```
restrict,port-forwarding,permitopen="192.168.100.1:443",permitopen="127.0.0.1:8006" ssh-ed25519 AAAA... tunnel-lab
```

- `restrict` coupe tout par défaut (shell, agent, X11, etc.)
- `port-forwarding` ne rétablit que les redirections
- chaque `permitopen` fixe une destination autorisée

**3. Ouvrir le tunnel**

```bash
ssh -N -i ~/.ssh/id_ed25519_tunnel \
  -L 8443:192.168.100.1:443 \
  -L 8006:127.0.0.1:8006 \
  adminsys@IP-DU-SERVEUR
```

Puis `https://localhost:8443` pour pfSense et `https://localhost:8006` pour Proxmox. `-N` ouvre le tunnel sans shell. `Ctrl+C` le ferme.

Comment lire `-L 8443:192.168.100.1:443` : le premier nombre est le port ouvert **sur mon PC** ; ce qui suit est **vu depuis Proxmox**. `127.0.0.1` veut dire « Proxmox lui-même », `192.168.100.1` est pfSense.

**4. Vérifier que la restriction fonctionne**

```bash
ssh -N -i ~/.ssh/id_ed25519_tunnel -L 9999:127.0.0.1:25 adminsys@IP-DU-SERVEUR
```

En ouvrant `localhost:9999`, le message côté SSH doit être `administratively prohibited`. Cette clé ne doit pas non plus donner de shell.

## Double authentification — interface web Proxmox

**Fait.**

1. Interface web `:8006` → **Datacenter** (menu de gauche) → **Permissions** → **Two Factor**
2. Ajouter (bouton "Add") → choisir **TOTP**
3. Scanner le QR code avec une app d'authentification
4. Confirmer avec le code à 6 chiffres généré par l'app

**App utilisée** : open-source, gratuite — **Aegis Authenticator** (Android) ou **2FAS Auth** (iOS et Android). Les deux permettent une sauvegarde chiffrée exportable soi-même, sans dépendre d'un compte cloud tiers.

Protège même si le mot de passe web fuite un jour.

## pfSense / VLAN

**En cours.** La VM pfSense est installée, le réseau WAN est configuré à la main (temporaire). Les VLAN (DMZ/APP/DATA/MGMT) ne sont pas encore créés.

| Mesure | Statut |
|---|---|
| Mot de passe admin changé | Fait |
| GUI en HTTPS (port 443), accessible depuis le LAN uniquement, par le tunnel SSH | Fait |
| SSH de pfSense désactivé (ou par clé uniquement, jamais sur le WAN) | À vérifier : désactivé par défaut, à confirmer dans System → Advanced → Admin Access → Secure Shell |
| Règles du WAN : rien d'ouvert sauf le port UDP de WireGuard | Rien d'ouvert pour l'instant ; le port de WireGuard sera ajouté avec le VPN |

La GUI n'écoute que sur le LAN privé : aucune route d'Internet n'y mène. Je n'y accède que par le tunnel SSH.

Si pfSense affiche *Potential DNS Rebind attack detected* en passant par `localhost`, ajouter `localhost` dans **System → Advanced → Admin Access → Alternate Hostnames**.

**Ce qui s'est passé** : au premier accès, le HTTPS de la GUI ne répondait pas (le `curl` restait muet), alors que le HTTP sur le port 80 fonctionnait. J'ai ouvert un tunnel vers le port 80, puis, depuis la GUI, j'ai activé le HTTPS sur le port 443 (**System → Advanced → Admin Access**). Le tunnel sert ensuite uniquement le port 443.

## Secours — KVM/IPMI et mode rescue d'OVH

Quand le SSH ne passe plus, deux accès ne dépendent pas de lui.

**Si mon IP a changé** (avant le VPN) : relever la nouvelle IP depuis le PC (`curl -4 ifconfig.me`), corriger dans le manager OVH les deux règles qui autorisent mon IP (ports 22 et 8006). Le KVM n'est pas nécessaire.

**KVM/IPMI** : console du serveur ouverte depuis le manager OVH. Elle fonctionne même si le réseau, pfSense ou le SSH sont en panne. Connexion avec le mot de passe root local (il passe par le clavier virtuel, pas par le SSH). Utile après une erreur dans la configuration `sshd` : corriger ou supprimer le fichier en cause, puis `sshd -t` et `systemctl reload ssh`. Le mot de passe root local doit rester dans un gestionnaire de mots de passe.

**Mode rescue** : quand le système ne démarre plus.

1. Dans le manager OVH : **Netboot** → **Rescue**, puis redémarrer le serveur. Les identifiants arrivent par e-mail.
2. Se connecter en SSH au système de secours et repérer la partition racine (`lsblk`). Selon la disposition (RAID, LVM, ZFS), il faut d'abord l'assembler.
3. Monter le système et y entrer :

```bash
mount /dev/<racine> /mnt
mount --bind /dev /mnt/dev
mount --bind /proc /mnt/proc
mount --bind /sys /mnt/sys
chroot /mnt
```

4. Corriger, puis `exit` et `umount -R /mnt`.
5. **Remettre le netboot sur le disque dur** dans le manager, puis redémarrer. Sans cette étape, le serveur retombe en rescue.

## Ce qu'il reste à faire, dans l'ordre utile

Le firewall OVH bloque déjà tout sauf mon IP. Donc les mesures suivantes ne sont pas classées dans l'ordre où j'y ai pensé, mais dans l'ordre de ce qu'elles apportent vraiment en plus :

1. **Couper la connexion SSH par mot de passe et interdire `root`** — fait — utile si jamais quelqu'un réussit à se faire passer pour mon IP, ou si j'élargis la règle un jour
2. **Double authentification sur l'interface web Proxmox** — fait — la seule mesure ici qui protège contre un problème qui n'a rien à voir avec le réseau (mot de passe volé par phishing par exemple)
3. **Limites SSH et clé de tunnel restreinte** — fait — limitent ce qu'une clé volée ou un accès de tous les jours peut atteindre
4. **Basculer le WAN de pfSense sur l'IP failover** — fait — prérequis du VPN ; le sas temporaire est retiré après le VPN
5. **VPN WireGuard sur pfSense** — à faire — accès par clé, indépendant de mon IP
6. **Fail2ban** — à faire après le VPN — pas très utile tant que seule mon IP peut atteindre le serveur
7. **nftables sur le serveur lui-même** — à faire après le VPN — règle par interface, pas par IP
8. **Vérifier que le SSH de pfSense est désactivé** — à faire

## VPN WireGuard sur pfSense

**À faire.** La bascule sur l'IP failover est faite.

Pour que le VPN soit joignable depuis Internet, pfSense doit avoir une adresse publique : son WAN est donc passé sur l'IP failover. Pas de redirection de port (DNAT) sur l'hôte : elle obligerait à ouvrir un port UDP à tout Internet sur l'IP principale, celle que je protège pour ne laisser passer que mon IP, et elle serait jetée à la bascule.

- Le port UDP de WireGuard est ouvert dans le pare-feu OVH de l'IP failover (voir plus haut) et dans les règles du WAN de pfSense. Rien d'autre n'est ouvert sur le WAN.
- Une fois connecté au VPN, j'atteins la GUI de pfSense (`192.168.100.1`) et celle de Proxmox (`192.168.100.2:8006`) directement sur le LAN.
- Le tunnel SSH reste en secours : il ne dépend pas de pfSense.

## Fail2ban

**À faire. Après le VPN — vraiment utile une fois le VPN en place.**

```bash
apt update
apt install fail2ban -y
systemctl enable --now fail2ban
fail2ban-client status sshd
```

Surveille les tentatives de connexion SSH ratées, bannit l'IP après trop d'essais. La jail `sshd` est activée par défaut sur les configurations Debian récentes.

**Pourquoi pas avant** : seule mon IP peut atteindre le port 22, donc il n'y a presque rien à bannir. Et comme mon IP change, je ne peux pas la mettre dans `ignoreip` : quelques erreurs de connexion de ma part suffiraient à me bannir moi-même. Pour débannir : `fail2ban-client set sshd unbanip <IP>`, depuis le KVM si je suis bloqué.

## nftables sur le serveur Proxmox

**À faire. Après le VPN.**

Une règle locale par interface : le port 22 et le port 8006 ne sont joignables que depuis le LAN (`vmbr101`), le VPN, et mon IP. Elle ne dépend donc plus d'une adresse qui change, sauf pour la ligne sur mon IP.

```bash
nano /etc/nftables-hote.conf
```

```
add table inet hote
flush table inet hote
table inet hote {
    chain input {
        type filter hook input priority 0; policy drop;
        ct state established,related accept
        iif lo accept
        meta l4proto { icmp, ipv6-icmp } accept
        iifname "vmbr101" tcp dport { 22, 8006 } accept
        ip saddr TON-IP/32 tcp dport { 22, 8006 } accept
    }
}
```

Ajouter la ligne de l'interface du VPN (`iifname "wg0"`) quand il existe.

- **L'ICMP reste autorisé** : pfSense envoie un ping à sa passerelle pour surveiller son WAN. Sans lui, la passerelle apparaît hors ligne dans pfSense.
- **Une table dédiée**, et non `nft list ruleset > /etc/nftables.conf` : cette commande copierait aussi la table de NAS du sas et les tables du pare-feu Proxmox.
- **Charger avec** `nft -f /etc/nftables-hote.conf`, puis tester depuis un second terminal avant de fermer la session. Une erreur dans une chaîne `policy drop` coupe l'accès : le KVM est le secours.
- **Rendre permanent** : ajouter `post-up nft -f /etc/nftables-hote.conf` au bloc `vmbr101` dans `/etc/network/interfaces`.
- Si le pare-feu Proxmox est activé sur l'hôte, vérifier que les deux ne se contredisent pas.

## nftables sur chaque VM (Reverse Proxy / Web / BDD)

**Pas encore utile** — ces VM n'existent pas encore.

## Fail2ban sur chaque VM

**Pas encore utile** — ces VM n'existent pas encore.

## WAF (reverse proxy)

**Pas encore utile** — le reverse proxy n'est pas encore installé.

## Leçon apprise : anonymiser les IP dans la doc publique

Erreur de débutant faite sur ce repo : l'IP principale du serveur a été laissée en clair dans deux docs déjà poussées sur GitHub (`00-installation-proxmox.md`, `03-pfsense.md`), avant de réaliser que ça reste visible dans l'historique même après correction. Réécrire l'historique Git pour l'effacer a été écarté — le coût (casser l'historique construit comme preuve de travail) dépasse le bénéfice (une IP de serveur dédié n'est de toute façon jamais vraiment secrète, repérable par un scan automatique en quelques heures).

**Consigne pour la suite** : dans toute nouvelle doc destinée à GitHub, remplacer l'IP réelle par un placeholder (`IP-DU-SERVEUR`, `TON-IP`, etc.) avant le premier commit — plus simple de le faire dès le départ que de corriger après coup une fois poussé.
