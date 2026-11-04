# pfSense — installation, accès à la GUI et bascule sur l'IP failover

**En cours.** pfSense est installé, accessible, et son WAN est sur l'IP failover. Le VPN WireGuard fonctionne et le sas est retiré. Il reste les VLAN.

Cette page résume ce que j'ai fait, pourquoi je l'ai fait comme ça, et ce qui s'est mal passé. Toutes les commandes sont dans le [pas-à-pas complet](03-pfsense-pas-a-pas.md). La sécurité de l'accès (tunnel SSH, clés, pare-feu OVH) est dans [04-hardening.md](04-hardening.md).

## Statut

| Étape | Statut |
|---|---|
| Bridges `vmbr100` (sas temporaire) et `vmbr101` (LAN) | Fait |
| NAT sur l'hôte pour le sas | Fait |
| Installation de pfSense CE 2.9.0 (VM 100) | Fait |
| Interfaces : WAN `10.0.0.2/24`, LAN `192.168.100.1/24` | Fait |
| Accès à la GUI par tunnel SSH, en HTTPS | Fait |
| Mot de passe admin changé | Fait |
| Instantané `apres-install` de la VM | Fait |
| Bascule du WAN sur l'IP failover (MAC virtuelle, `/32`, passerelle non locale) | Fait |
| Pare-feu OVH de l'IP failover | Fait |
| Retrait du sas `vmbr100` et de son NAT | Fait |
| VPN WireGuard (détails dans [04-hardening.md](04-hardening.md)) | Fait |
| VLAN (DMZ, APP, DATA, MGMT) | À faire |

## Le problème : installer pfSense avec une IP failover

pfSense a besoin d'Internet pour terminer son installation. Mon IP failover OVH est en `/32` avec une MAC virtuelle, et la passerelle d'OVH n'est pas dans son sous-réseau. Sur ce montage, pfSense n'a donc pas d'accès Internet en direct pendant l'installation.

## Mon choix : un sas vNAT temporaire

Je donne à pfSense un WAN privé derrière l'hôte, qui fait le NAT. Une fois pfSense installé, je supprime ce sas.

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

 PC d'administration --(tunnel SSH)--> hôte Proxmox --> GUI de pfSense
```

| Élément | Bridge | Adresse | Rôle |
|---|---|---|---|
| Hôte Proxmox | `vmbr100` | `10.0.0.1/24` | passerelle et NAT du sas |
| pfSense WAN | `vmbr100` | `10.0.0.2/24` | sortie Internet par l'hôte |
| pfSense LAN | `vmbr101` | `192.168.100.1/24` | réseau interne, sans DHCP |
| Hôte Proxmox | `vmbr101` | `192.168.100.2/24` | accès à la GUI |

`vmbr0` n'est pas touché : c'est lui qui me garde l'accès à l'hôte.

## Pourquoi ces choix

| Choix | Pourquoi |
|---|---|
| Sas vNAT sur `vmbr100` | donne à pfSense un accès Internet pour l'installation, sans toucher à `vmbr0` |
| LAN sur un bridge séparé (`vmbr101`) | un LAN doit être un segment isolé derrière le firewall ; WAN et LAN ne peuvent pas partager le même bridge ni le même réseau |
| Bridges sans port physique (`bridge-ports none`) | ce sont des switches virtuels privés : rien n'y entre depuis Internet |
| Accès à la GUI par tunnel SSH | la GUI n'écoute que sur le LAN privé ; je ne l'expose jamais sur Internet |
| NAT du sas : une règle de masquerade sur l'hôte | pfSense sort par l'hôte pendant l'installation. Je l'ai posée à la main avec `iptables` (la couche de compatibilité de nftables sous Debian). Une table nftables dédiée serait la méthode propre et persistante : elle est décrite dans le pas-à-pas |
| Bascule sur l'IP failover ensuite | le sas n'est qu'un échafaudage pour l'installation |

## Comment ça circule

Deux trajets, deux rôles pour l'hôte :

- **GUI** : mon navigateur parle à un port local de mon PC. Le client SSH envoie les données à l'hôte, qui ouvre lui-même une connexion vers `192.168.100.1:443` à travers `vmbr101`. L'hôte est un simple client, aucun routage ni NAT n'intervient.
- **Sortie Internet de pfSense** : pfSense envoie ses paquets à l'hôte par `vmbr100`. L'hôte les route, remplace l'adresse source `10.0.0.2` par son IP publique (masquerade), et les envoie par `vmbr0`. Sans ce NAT, OVH bloquerait une adresse source privée. Les schémas détaillés sont dans le [pas-à-pas](03-pfsense-pas-a-pas.md#13-comprendre-les-flux-réseau).

## Ce qui s'est passé

**WAN et LAN dans le même réseau.** Mon premier schéma mettait les deux interfaces de pfSense sur `vmbr100`, dans le même `10.0.0.0/24`. Ce n'est pas un vrai LAN séparé : le routage devient ambigu et le pare-feu bloque ce qui arrive par le mauvais côté. J'ai créé `vmbr101` pour le LAN.

**`vmbr101` sans adresse sur l'hôte.** Le bridge existait, mais l'hôte n'avait aucune IP dessus : il ne pouvait pas joindre pfSense. J'ai ajouté `192.168.100.2/24` dans `/etc/network/interfaces` ; le ping vers `192.168.100.1` est alors passé.

**HTTPS muet, HTTP fonctionnel.** Au premier accès à la GUI, le `curl` en HTTPS ne répondait rien, alors que le HTTP sur le port 80 fonctionnait. J'ai ouvert un tunnel vers le port 80, puis, depuis la GUI, j'ai activé le HTTPS sur le port 443 (**System → Advanced → Admin Access**). Le tunnel sert ensuite uniquement le port 443.

**Message à la prise de l'instantané.** `skipping guest filesystem freeze - disabled in VM options` : l'agent invité QEMU n'est pas activé, donc le système de fichiers n'est pas gelé avant l'instantané. Sans conséquence sur une installation neuve ; l'instantané reste utilisable pour un retour en arrière.

**Passerelle `Offline (packet loss)` après la bascule.** Juste après avoir configuré le WAN en `/32`, **Status → Gateways** affichait `OVH_GW` en `Offline (packet loss)`, alors que la passerelle répondait au ping. La cause : des changements n'avaient pas été appliqués (**Apply Changes**), donc le statut n'était pas à jour. Je ne me suis pas fié à ce seul statut : j'ai vérifié la sortie par un ping direct vers `8.8.8.8` depuis le WAN, puis le DNS, l'heure et les mises à jour de pfSense. Je clique désormais sur **Apply Changes** après chaque modification de passerelle ou d'interface.

**NAT du sas posé à la main.** Au retrait du sas, le fichier de règles prévu (`/etc/nftables-vnat.conf`) n'existait pas et le bloc `vmbr100` n'avait aucune ligne `post-up` : le masquerade avait été ajouté à la main avec `iptables` (`-A POSTROUTING -s 10.0.0.0/24 -o vmbr0 -j MASQUERADE`), et le service `nftables` est désactivé. Il n'était donc pas persistant. Je l'ai retiré avec `iptables -t nat -D …`, puis j'ai vérifié qu'aucune règle ne restait (`iptables -t nat -S`, `nft list ruleset`). Pour un NAT durable, j'utiliserais une table nftables dédiée, comme décrit dans le pas-à-pas.

## Accéder à la GUI

```bash
ssh -N -i ~/.ssh/id_ed25519_tunnel -L 8443:192.168.100.1:443 adminsys@IP-DU-SERVEUR
```

Puis `https://localhost:8443`. Le premier nombre est le port ouvert sur mon PC ; l'adresse qui suit est vue depuis Proxmox. J'utilise une clé dédiée au tunnel, sans shell, limitée aux deux interfaces web : elle est décrite dans [04-hardening.md](04-hardening.md).

Si pfSense affiche *Potential DNS Rebind attack detected*, j'ajoute `localhost` dans **System → Advanced → Admin Access → Alternate Hostnames**.

## Bascule sur l'IP failover

**Fait**, sas compris. Une fois pfSense installé, il n'a plus besoin du sas pour fonctionner en `/32`. J'ai suivi cet ordre, pour ne jamais perdre l'accès à pfSense pendant la transition :

1. Instantané de la VM (`qm snapshot 100 avant-bascule`).
2. VM arrêtée, interface WAN passée sur `vmbr0` avec la MAC virtuelle d'OVH (`qm set 100 --net0 virtio=<MAC>,bridge=vmbr0`), puis redémarrée. Le LAN reste sur `vmbr101` : mon tunnel vers la GUI n'est pas coupé.
3. Dans la GUI : passerelle OVH créée avec **Use non-local gateway** (**System → Routing → Gateways**), WAN en `/32` sur l'IP failover, passerelle par défaut remplacée, ancienne passerelle `10.0.0.1` supprimée. La console ne permet pas cette configuration : elle refuse une passerelle hors du sous-réseau.
4. **Block private networks** et **Block bogon networks** remis sur le WAN.
5. Sortie Internet vérifiée : ping vers `8.8.8.8` depuis le WAN, DNS, heure (NTP) et mises à jour de pfSense.
6. Règles du pare-feu OVH de l'IP failover créées, puis activées (voir [04-hardening.md](04-hardening.md)), puis mêmes vérifications.
7. VPN WireGuard installé et testé (voir [04-hardening.md](04-hardening.md)), puis sas retiré : bloc `vmbr100` supprimé de `/etc/network/interfaces` et règle de masquerade retirée (`iptables -t nat -D POSTROUTING -s 10.0.0.0/24 -o vmbr0 -j MASQUERADE`). Je garde `vmbr101`, le LAN des futures VM. J'ai ensuite retesté tous les accès : VPN, tunnel SSH, SSH direct, GUI de Proxmox directe et sortie de pfSense.

Le double NAT a disparu avec la bascule : les VM sortiront par pfSense avec l'IP failover.

## Ce qu'il reste à faire

- Les VLAN : DMZ, APP, DATA, MGMT.
- Vérifier que le SSH de pfSense est désactivé, et qu'aucune règle du WAN n'ouvre autre chose que le port de WireGuard.
