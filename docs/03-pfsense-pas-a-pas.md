# pfSense sur Proxmox (OVHCloud) — pas-à-pas

*Annexe de [03-pfsense.md](03-pfsense.md) : la procédure complète, avec toutes les commandes, pour l'architecture réseau temporaire.*

## 1. Objectif et principe

pfSense a besoin d'Internet pour terminer son installation, alors que l'IP failover OVH (en /32, avec vMAC) n'est pas dans le sous-réseau de la passerelle OVH. On contourne ce point avec un **sas vNAT temporaire** :

- `vmbr100` : bridge virtuel sans port physique. L'hôte Proxmox y est la passerelle (`10.0.0.1`) et fait du NAT vers Internet. C'est le **WAN** de pfSense pendant l'installation.
- `vmbr101` : bridge virtuel privé, c'est le **LAN** de pfSense. L'hôte y a une IP pour atteindre la GUI via un tunnel SSH.
- `vmbr0` : bridge relié à la carte physique. Il n'est **pas modifié** et conserve l'accès à l'hôte.

```
Internet
   |
 vmbr0 (inchangé, IP publique de l'hôte)
   |
 [Hôte Proxmox : routage + NAT]
   |
   +-- vmbr100  10.0.0.1/24 ---- WAN pfSense  10.0.0.2/24       (net0 / vtnet0)
   |
   +-- vmbr101  192.168.100.2/24 -- LAN pfSense  192.168.100.1/24 (net1 / vtnet1)

 PC d'administration --(tunnel SSH 8443 -> 192.168.100.1:443)--> hôte Proxmox
```

## 2. Plan d'adressage

| Élément | Bridge | Adresse | Remarque |
|---|---|---|---|
| Hôte Proxmox | vmbr100 | 10.0.0.1/24 | passerelle et NAT du sas |
| pfSense WAN (net0 / vtnet0) | vmbr100 | 10.0.0.2/24 | passerelle 10.0.0.1 |
| pfSense LAN (net1 / vtnet1) | vmbr101 | 192.168.100.1/24 | sans passerelle, sans DHCP |
| Hôte Proxmox | vmbr101 | 192.168.100.2/24 | accès à la GUI |

Dans ce document, la VM pfSense a l'identifiant `100`. Adaptez-le si besoin.

## 3. Prérequis

- Accès SSH à l'hôte Proxmox VE avec l'utilisateur d'administration (`root` est interdit en SSH). Les commandes de ce document s'exécutent en root : `sudo -i`, ou `sudo` devant chaque commande.
- L'ISO de pfSense CE, téléchargée depuis le site officiel, et son empreinte SHA256 pour la vérifier.
- Le nom du stockage disponible : `pvesm status`.
- L'IP failover n'est **pas** nécessaire pour cette phase.

## 4. Création des bridges

### 4.1 Sauvegarder la configuration réseau

```
cp /etc/network/interfaces /etc/network/interfaces.bak-$(date +%F)
```

Ne modifiez pas le bloc `vmbr0` : c'est lui qui vous donne l'accès à l'hôte.

### 4.2 Ajouter les deux bridges

```
nano /etc/network/interfaces
```

Ajoutez à la fin du fichier :

```
auto vmbr100
iface vmbr100 inet static
    address 10.0.0.1/24
    bridge-ports none
    bridge-stp off
    bridge-fd 0
    post-up   nft -f /etc/nftables-vnat.conf
    post-down nft delete table ip nat_pfsense

auto vmbr101
iface vmbr101 inet static
    address 192.168.100.2/24
    bridge-ports none
    bridge-stp off
    bridge-fd 0
```

`bridge-ports none` crée un switch virtuel privé, sans lien avec la carte physique. Si un bloc `vmbr100` ou `vmbr101` existe déjà (par exemple en `inet manual`), remplacez-le au lieu d'en ajouter un second.

Les lignes `post-up` et `post-down` appliquent et retirent la règle de NAT créée à l'étape 5.

### 4.3 Alternative par l'interface web

**Datacenter → nœud → System → Network → Create → Linux Bridge**, puis :

- Name : `vmbr100`, IPv4/CIDR : `10.0.0.1/24`, Bridge ports : laisser vide.
- Name : `vmbr101`, IPv4/CIDR : `192.168.100.2/24`, Bridge ports : laisser vide.
- Cliquer sur **Apply Configuration**.

Les lignes `post-up` et `post-down` du NAT doivent de toute façon être ajoutées dans le fichier.

### 4.4 Appliquer et vérifier

Créez d'abord le fichier de règles NAT (étape 5), sinon le `post-up` échouera. Puis :

```
ifreload -a
ip -br a show vmbr100
ip -br a show vmbr101
```

Sortie attendue : `10.0.0.1/24` sur `vmbr100` et `192.168.100.2/24` sur `vmbr101`.

## 5. Activer le routage et le NAT sur l'hôte

### 5.1 Activer le forwarding IPv4 (persistant)

```
echo 'net.ipv4.ip_forward=1' > /etc/sysctl.d/99-ip-forward.conf
sysctl --system
sysctl net.ipv4.ip_forward
```

La dernière commande doit afficher `net.ipv4.ip_forward = 1`.

### 5.2 Créer la règle de masquerade (nftables)

```
nano /etc/nftables-vnat.conf
```

Contenu :

```
add table ip nat_pfsense
flush table ip nat_pfsense
table ip nat_pfsense {
    chain postrouting {
        type nat hook postrouting priority srcnat; policy accept;
        ip saddr 10.0.0.0/24 oifname "vmbr0" masquerade
    }
}
```

Le NAT vit dans sa propre table : il ne touche pas aux règles du pare-feu Proxmox ni à celles de Debian.

### 5.3 Charger et vérifier

```
nft -f /etc/nftables-vnat.conf
nft list table ip nat_pfsense
```

Si le pare-feu Proxmox est activé sur l'hôte, vérifiez qu'il ne bloque pas le trafic transféré depuis `vmbr100` vers `vmbr0`.

## 6. Création de la VM pfSense

### 6.1 Récupérer l'ISO

Téléchargez l'ISO depuis le site officiel de pfSense CE (Netgate). Elle est souvent fournie en `.iso.gz`.

```
cd /var/lib/vz/template/iso
gunzip pfSense-CE-<version>-RELEASE-amd64.iso.gz
sha256sum pfSense-CE-<version>-RELEASE-amd64.iso
```

Comparez l'empreinte avec celle publiée. Vous pouvez aussi envoyer l'ISO par l'interface web : **local → ISO Images → Upload**.

### 6.2 Créer la VM en ligne de commande

Remplacez `<stockage>` par le nom relevé avec `pvesm status` (par exemple `local` ou `local-lvm`).

```
qm create 100 \
  --name pfsense \
  --ostype other \
  --cores 2 \
  --memory 2048 \
  --cpu host \
  --scsihw virtio-scsi-single \
  --scsi0 <stockage>:16 \
  --net0 virtio,bridge=vmbr100 \
  --net1 virtio,bridge=vmbr101 \
  --ide2 local:iso/pfSense-CE-<version>-RELEASE-amd64.iso,media=cdrom \
  --boot order='ide2;scsi0' \
  --onboot 1
```

- `net0` (WAN) est sur `vmbr100` et `net1` (LAN) sur `vmbr101`. Dans pfSense, ils apparaîtront comme `vtnet0` et `vtnet1`.
- Vérification : `qm config 100 | grep -E 'net|boot|ide2'`.

### 6.3 Démarrer

```
qm start 100
```

Ouvrez ensuite la console : **VM 100 → Console** (noVNC).

## 7. Installation de pfSense

1. Acceptez la licence, puis choisissez **Install pfSense**.
2. Choisissez la disposition du clavier (par exemple French).
3. Partitionnement : **Auto (UFS)** est le plus simple, **Auto (ZFS)** est possible. Sélectionnez le disque virtuel de 16 Go.
4. Si l'installateur propose de configurer le réseau, saisissez à la main : IP `10.0.0.2/24`, passerelle `10.0.0.1`, DNS `1.1.1.1`. Le sas vNAT n'a pas de serveur DHCP.
5. Laissez l'installation se terminer. À la fin, refusez la configuration manuelle (**No**) et choisissez **Reboot**.
6. Retirez l'ISO pendant le redémarrage, depuis l'hôte :

```
qm set 100 --ide2 none,media=cdrom
qm set 100 --boot order=scsi0
```

## 8. Configuration des interfaces dans la console pfSense

### 8.1 Assigner les interfaces (option 1)

- Should VLANs be set up now : `n`
- WAN interface : `vtnet0`
- LAN interface : `vtnet1`
- Optional interface : appuyer sur Entrée
- Do you want to proceed : `y`

### 8.2 Régler les adresses IP (option 2)

**WAN (choix 1)**

- Configure IPv4 address via DHCP : `n`
- Adresse IPv4 : `10.0.0.2`, masque : `24`
- Passerelle amont : `10.0.0.1`
- Configure IPv6 via DHCP6 : `n`, adresse IPv6 : Entrée (aucune)
- Revert to HTTP as the webConfigurator protocol : `n` (garder HTTPS)

**LAN (choix 2)**

- Adresse IPv4 : `192.168.100.1`, masque : `24`
- Passerelle amont : Entrée (aucune)
- Configure IPv6 via DHCP6 : `n`, adresse IPv6 : Entrée (aucune)
- Enable the DHCP server on LAN : `n`
- Revert to HTTP : `n`

Le menu doit alors afficher :

```
WAN (wan) -> vtnet0 -> v4: 10.0.0.2/24
LAN (lan) -> vtnet1 -> v4: 192.168.100.1/24
```

### 8.3 Tester la sortie Internet (option 7, Ping host)

- `10.0.0.1` : valide le sas.
- `8.8.8.8` : valide le NAT de l'hôte.

## 9. Tests depuis l'hôte Proxmox

```
ping -c2 192.168.100.1
curl -vk --max-time 5 https://192.168.100.1
```

Si vous obtenez du HTML ou une redirection, la GUI répond.

## 10. Accéder à la GUI par tunnel SSH

Depuis le PC d'administration :

```
ssh -N -L 8443:192.168.100.1:443 adminsys@IP-DU-SERVEUR
```

Ouvrez `https://localhost:8443` (le certificat est auto-signé, acceptez l'avertissement). Identifiants par défaut : `admin` / `pfsense`.

Au quotidien, j'utilise une clé dédiée au tunnel, sans shell, plutôt que la clé d'administration : voir [04-hardening.md](04-hardening.md).

Si pfSense affiche *Potential DNS Rebind attack detected*, ouvrez le shell (option 8 de la console) :

```
pfSsh.php playback disablewebguirdrcheck
```

Ou bien ajoutez `localhost` dans **System → Advanced → Admin Access → Alternate Hostnames** une fois dans la GUI.

### Si le HTTPS ne répond pas : accès initial en HTTP

Lors de cette installation, le HTTPS n'a pas répondu au premier accès alors que le port 80 fonctionnait. Passez alors par le HTTP : le tunnel SSH chiffre déjà le trajet entre le PC et l'hôte.

```
ssh -N -L 8080:192.168.100.1:80 adminsys@IP-DU-SERVEUR
```

Ouvrez `http://localhost:8080`. Une fois dans la GUI, activez le HTTPS : **System → Advanced → Admin Access**, protocole **HTTPS (SSL/TLS)**, port `443`, puis **Save**. Reconnectez-vous ensuite avec le tunnel `8443` décrit plus haut.

### Assistant de configuration initiale

1. Changez immédiatement le mot de passe `admin`.
2. Hostname, domaine et DNS (par exemple `1.1.1.1`).
3. Fuseau horaire.
4. Page WAN : **décochez** *Block RFC1918 Private Networks* et *Block bogon networks* pendant cette phase, car le WAN est en adresse privée (`10.0.0.2`).

## 11. Dépannage

| Symptôme | Cause probable | Action |
|---|---|---|
| `ping 192.168.100.1` échoue depuis l'hôte | `net1` pas sur `vmbr101`, ou pas d'IP sur `vmbr101` | `qm config 100 \| grep net`, `ip -br a show vmbr101`, `bridge link show` |
| Le ping passe mais pas le `curl` | Offload virtio, ou GUI non démarrée | Désactiver l'offload (ci-dessous), option 11 de la console |
| HTTP fonctionne, HTTPS non | HTTPS désactivé, ou réponse `y` à *Revert to HTTP* | `sockstat -4 -l \| grep -E ':(443\|80)'`, puis **System → Advanced → Admin Access** : protocole HTTPS |
| pfSense ne joint pas `8.8.8.8` | forwarding ou NAT absent sur l'hôte | `sysctl net.ipv4.ip_forward`, `nft list table ip nat_pfsense`, pare-feu Proxmox |
| Alerte DNS Rebind | accès via `localhost` | voir la section 10 |

Désactiver l'offload dans le shell pfSense (test rapide) :

```
ifconfig vtnet1 -rxcsum -txcsum -tso -lro
```

Pour le rendre permanent : **System → Advanced → Networking**, cochez *Disable hardware checksum offload*, *Disable hardware TCP segmentation offload* et *Disable hardware large receive offload*, puis redémarrez pfSense.

## 12. Étape suivante : basculer sur l'IP failover

Une fois pfSense installé, il n'a plus besoin du sas pour fonctionner en `/32`. Faites d'abord un instantané :

```
qm snapshot 100 avant-bascule
```

Puis :

1. Créer la MAC virtuelle pour l'IP failover dans le manager OVH.
2. Passer l'interface WAN sur `vmbr0` avec cette MAC : `qm set 100 --net0 virtio=<MAC_VIRTUELLE>,bridge=vmbr0`.
3. Dans pfSense, mettre le WAN en IP failover avec le masque `/32` et créer la passerelle OVH. Dans **System → Routing → Gateways**, cocher *Use non-local gateway through interface specific route*.
4. Remettre *Block private networks* et *Block bogon networks* sur le WAN.
5. Supprimer le sas : retirer les lignes `post-up` / `post-down` et le bloc `vmbr100` de `/etc/network/interfaces`, puis `nft delete table ip nat_pfsense` et `ifreload -a`.
6. Conserver `vmbr101` : c'est le LAN derrière lequel seront branchées les futures VM.

## 13. Comprendre les flux réseau

L'hôte Proxmox joue deux rôles différents selon le trajet. Pour la GUI, il est un simple **client** qui ouvre une connexion vers pfSense. Pour la sortie Internet de pfSense, il est un **routeur avec NAT**.

### 13.1 Accès à la GUI (vmbr101, l'hôte est client)

```
PC                                  Hôte Proxmox                         pfSense
 |                                       |                                  |
 | navigateur -> localhost:8443          |                                  |
 |        |                              |                                  |
 |   client ssh (écoute sur 8443)        |                                  |
 |        |                              |                                  |
 |========= tunnel SSH chiffré =========>| sshd (port 22, via vmbr0)        |
 |        (Internet, carte physique)     |        |                         |
 |                                       |   ouvre une connexion TCP        |
 |                                       |   depuis 192.168.100.2           |
 |                                       |        |-- vmbr101 (switch) ---->| 192.168.100.1:443
 |                                       |                                  |
```

1. Le navigateur vise `localhost:8443`, un port ouvert sur le PC par le client `ssh`.
2. Le client SSH envoie les données, chiffrées, vers l'IP publique de l'hôte sur le port 22. Ce tronçon passe par `vmbr0`.
3. `sshd` lit l'instruction du tunnel (`-L 8443:192.168.100.1:443`) et ouvre **sa propre** connexion TCP vers `192.168.100.1:443`.
4. Cette connexion part de `192.168.100.2`. `192.168.100.1` est dans le même réseau, donc l'hôte n'a besoin ni de passerelle ni de route : il demande l'adresse MAC de pfSense (ARP) et lui envoie les trames à travers le switch virtuel `vmbr101`.
5. pfSense répond, et la réponse remonte le même chemin.

Le trafic entre l'hôte et pfSense reste dans `vmbr101`, un switch purement virtuel : il ne touche ni la carte physique ni Internet. `ip_forward` et le NAT ne servent pas ici. La GUI n'est jamais exposée sur Internet.

### 13.2 Sortie Internet de pfSense (vmbr100 puis vmbr0, l'hôte est routeur)

Exemple : un ping vers `8.8.8.8` lancé depuis pfSense.

```
pfSense (WAN 10.0.0.2)        Hôte Proxmox                              Internet
        |                           |                                      |
        | src 10.0.0.2              |                                      |
        | dst 8.8.8.8               |                                      |
        |--- vmbr100 (switch) ----->| 10.0.0.1                             |
        |                           |  1. ip_forward : il route            |
        |                           |  2. route par défaut -> vmbr0        |
        |                           |  3. masquerade : src devient         |
        |                           |     l'IP publique de l'hôte          |
        |                           |--- vmbr0 -> carte physique --------->| 8.8.8.8
        |                           |                                      |
        |                           |<-- réponse vers l'IP publique -------|
        |                           |  4. conntrack : dst redevient        |
        |<-- vmbr100 ---------------| 10.0.0.2                             |
```

1. `8.8.8.8` n'est pas dans `10.0.0.0/24` : pfSense applique sa route par défaut et envoie le paquet à `10.0.0.1` (l'hôte) à travers `vmbr100`.
2. Le paquet n'est pas destiné à l'hôte. Avec `net.ipv4.ip_forward=1`, il le fait suivre, et sa route par défaut l'envoie par `vmbr0`.
3. La règle de masquerade remplace la source `10.0.0.2` par l'IP publique de l'hôte et mémorise la correspondance (conntrack).
4. Le paquet quitte le serveur par `vmbr0`. Vu de l'extérieur, il vient de l'hôte.
5. La réponse revient vers l'IP publique de l'hôte. Conntrack remet `10.0.0.2` comme destination et la renvoie par `vmbr100`.

Sans NAT, le paquet sortirait avec une source privée (`10.0.0.2`) qu'OVH bloquerait, et Internet ne saurait pas renvoyer la réponse.

Les futures VM branchées sur `vmbr101` subiront un double NAT (pfSense, puis l'hôte) tant que le sas est en place. Il disparaît avec la bascule sur l'IP failover (section 12).
