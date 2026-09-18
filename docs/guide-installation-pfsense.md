# Installer pfSense sur Proxmox (serveur dédié OVH) — guide sans blocage

Basé sur les blocages réels rencontrés et leurs solutions.

## Prérequis réseau (avant de toucher à la VM)

1. Commander une Additional IP chez OVH (Bare Metal Cloud → IP → Commander → IPv4, ~1,99 €HT/mois, sans engagement)
2. Assigner une MAC virtuelle à cette IP
3. Noter la passerelle : c'est l'IP principale du serveur avec le dernier octet en `.254` (ex: serveur en `5.135.138.58` → passerelle `5.135.138.254`) — pas dans le bloc de l'Additional IP
4. Créer `vmbr1` sur l'hôte Proxmox : Système → Réseau → Créer → Linux Bridge → nom `vmbr1`, pas d'IP, pas de port physique, VLAN aware coché → cliquer "Apply Configuration" (étape souvent oubliée, vérifier avec `ip a | grep vmbr1`)
5. Créer un bridge NAT temporaire pour donner un accès Internet à pfSense **pendant** l'installation uniquement (voir "Bridge NAT temporaire" ci-dessous)

## Créer la VM pfSense

| Paramètre | Valeur |
|---|---|
| ISO | Netgate Installer, type "AMD64 ISO IPMI/Virtual Machines" |
| OS Type | Other |
| SCSI Controller | VirtIO SCSI single |
| Disque | 8 Go, qcow2, IO thread coché |
| CPU | 1 core |
| RAM | 2048 Mo |
| net0 | bridge `vmbr0`, MAC = MAC virtuelle OVH (pas "auto"), firewall décoché |
| net1 | bridge `vmbr1`, MAC = vide (laisser Proxmox en générer une, jamais dupliquer celle de net0), no VLAN, firewall décoché |

Avant de démarrer : Options → Boot Order → mettre `ide2` (CD) avant `scsi0`, le remettre après installation réussie.

## Bridge NAT temporaire (pour l'installation uniquement)

**Pourquoi** : configurer manuellement l'IP additionnelle + passerelle hors sous-réseau en Rescue Shell pendant l'installation ne fonctionne pas de façon fiable (méthode abandonnée, testée sans succès). La méthode qui fonctionne : donner à pfSense un accès Internet classique et temporaire via NAT, le temps de télécharger les composants et terminer l'installation — la vraie config WAN (Additional IP) sera faite après, une fois dans l'interface web.

Créer un bridge NAT sur l'hôte Proxmox (Système → Réseau → Créer → Linux Bridge) :

| Champ | Valeur |
|---|---|
| Name | `vnetNAT` (ou autre nom explicite) |
| IPv4/CIDR | `192.168.100.254/24` |
| Bridge ports | vide |

Puis activer le forwarding + NAT sur l'hôte (remplacer `vmbr0` par l'interface qui sort vers Internet si différente) :
```bash
echo 1 > /proc/sys/net/ipv4/ip_forward
iptables -t nat -A POSTROUTING -s 192.168.100.0/24 -o vmbr0 -j MASQUERADE
```

Brancher temporairement `net0` de la VM pfSense sur ce bridge `vnetNAT` (au lieu de `vmbr0`) pour la durée de l'installation. pfSense obtient une IP dans ce réseau (ex: `192.168.100.1`, passerelle `192.168.100.254`).

**Important** : désactiver le firewall OVH (Edge Network Firewall sur l'IP principale) pendant cette étape — le trafic NAT sortant/retour a été bloqué avec le firewall actif, même avec la règle "established" en place. Réactiver le firewall une fois l'installation terminée.

Une fois pfSense installé, rebrancher `net0` sur `vmbr0` avec la vraie MAC virtuelle OVH, et configurer l'Additional IP définitive via l'interface web (pas besoin de repasser par cette étape manuelle en shell).

## Installation

1. Boot → Accept

2. Écran **Welcome to pfSense** → **Install** directement (le vNAT donne déjà accès à Internet, plus besoin du Rescue Shell manuel)
3. Si un écran réseau apparaît (Connectivity Check) : laisser en DHCP — `net0` étant branché sur `vnetNAT`, il obtient automatiquement une IP (`192.168.100.x`) et un accès Internet fonctionnel
4. **Software Version to Install** (peut apparaître selon la version de l'installeur) : choisir la version en haut de liste marquée **"Current Stable Version"** (déjà présélectionnée) → OK
5. **Interface Assignment** : WAN = `vtnet0`, LAN = `vtnet1` → Continue
6. **ZFS Configuration** → Stripe (un seul disque, pas besoin de redondance ici — déjà assurée au niveau Proxmox)
7. Écran suivant : cocher le disque avec **Espace** (pas juste valider), sinon erreur "Not enough disks selected"
8. **Select System Components** : garder seulement `base` coché, décocher `kernel-dbg` et `lib32`
9. **Time Zone** : `8` Europe → France/Paris
10. **System Configuration** (services au boot) : cocher `sshd`, `ntpd`, `local_unbound` — décocher `powerd`, `moused`
11. **System Hardening** : cocher 0,1,2,3,4,5,6,8 (hide_uids/gids/jail, read_msgbuf, proc_debug, random_pid, clear_tmp, secure_console) — laisser 7 et 9 décochés
12. **Final Configuration** → aller dans **Root Password** en premier, le définir, puis **Finish**
13. "Open a shell for final modifications ?" → **No**
14. Avant tout reboot : dans Proxmox, Hardware → `ide2` → **Remove**, puis Boot Order → `scsi0` en premier
15. Reboot → menu console principal (pas iPXE, pas boucle installateur)

## Après le premier boot réussi

**Clavier AZERTY** (si besoin), en shell (option 8 du menu) :
```
sysrc keymap="fr.iso.kbd"
kbdcontrol -l fr.iso.kbd < /dev/console
```

**Réseau WAN — passage du vNAT à l'IP définitive** :
1. Dans Proxmox, Hardware de la VM pfSense → `net0` → rebrancher sur `vmbr0`, remettre la MAC virtuelle OVH
2. Supprimer/désactiver le bridge NAT temporaire (`vnetNAT`) une fois l'installation terminée, plus nécessaire
3. Créer la VM Xubuntu sur `vmbr1` pour accéder à l'interface web pfSense (`https://192.168.1.1`)
4. Configurer le WAN définitif (Additional IP, passerelle hors sous-réseau) via **Interfaces → WAN** dans l'interface web — plus fiable et plus tolérant que la config en shell

## Sécuriser l'IP principale (Edge Network Firewall OVH)

À faire sur l'IP principale du serveur (celle qui porte Proxmox `:8006` et SSH `:22`), pas sur l'Additional IP.

1. Récupérer son IP publique : `curl -4 ifconfig.me`
2. Manager OVH → Bare Metal Cloud → serveur → IP → cliquer sur l'IP principale → Firewall
3. Ajouter les règles, dans l'ordre (priorité basse = évaluée en premier) :

| Priorité | Mode | Protocole | IP source | Port dest | État TCP |
|---|---|---|---|---|---|
| 0 | Autoriser | TCP | TON-IP/32 | 22 | - |
| 1 | Autoriser | TCP | TON-IP/32 | 8006 | - |
| 2 | Autoriser | TCP | tous | - | established |
| 3 | Refuser | TCP | (vide) | (vide) | - |
| 4 | Refuser | UDP | (vide) | (vide) | - |

4. Vérifier que l'IP dans les règles 0 et 1 est bien l'IP publique actuelle (dynamique, peut avoir changé)
5. Activer le toggle firewall (en haut à droite, "Désactivé" → actif)

Note : ce firewall ne concerne que l'IP principale. L'Additional IP (pfSense) aura son propre firewall séparé si besoin, notamment pour le futur VPN en UDP — aucun conflit entre les deux.

## Pièges à ne jamais refaire

- Ne pas laisser `/24` sur une IP OVH additionnelle → toujours `/32`
- Ne jamais supposer que la passerelle est dans le même bloc que l'IP OVH → toujours `.254` du bloc de l'IP principale
- Toujours vérifier "Apply Configuration" après une modif réseau Proxmox, pas juste la création visible dans l'UI
- Toujours éjecter/retirer l'ISO avant reboot post-installation
- Ne pas essayer de configurer l'Additional IP + passerelle hors sous-réseau en Rescue Shell pendant l'installation → ne fonctionne pas de façon fiable, utiliser le vNAT temporaire à la place
- Si le NAT temporaire ne fonctionne pas (pas d'accès Internet malgré la config), vérifier que le firewall OVH (Edge Network Firewall, IP principale) n'est pas en train de bloquer le trafic — le désactiver temporairement pendant l'installation, le réactiver après
