# ADR 06 — Migration du lab 8 Go RAM vers un serveur dédié OVH

## Statut
Acceptée

## Contexte

Le projet a démarré sur un poste personnel Ubuntu disposant de 8 Go de RAM partagés avec un usage quotidien. Cette contrainte imposait des compromis techniques documentés dans le premier projet (`homelab-infra`) :

- Firewall pfSense en VM complète (obligatoire, FreeBSD)
- Reverse Proxy, Web et Base de données en conteneurs LXC/Incus plutôt qu'en VM complètes, pour limiter l'empreinte mémoire
- Absence de segmentation VLAN poussée, faute de marge pour la tester sereinement
- Lab démarré "à la demande" plutôt qu'en fonctionnement continu, pour ne pas monopoliser la RAM du poste de travail

L'accès à un serveur dédié OVHCloud (Xeon E3-1230v6, 32 Go ECC, 2×2 To RAID) a supprimé cette contrainte.

## Décision

Le projet est poursuivi sur un nouveau dépôt (`homelab-infra-ovh`), avec une architecture revue pour exploiter pleinement les ressources disponibles :

- Chaque rôle (Firewall, Reverse Proxy, Web, Base de données) redevient une VM complète sous Proxmox VE, plutôt qu'un mélange VM/conteneur
- Segmentation en quatre VLAN fonctionnels (DMZ, APP, DATA, MGMT) avec filtrage inter-VLAN strict sur pfSense
- Ajout d'une brique de supervision (Prometheus/Grafana) et d'une brique de sauvegarde dédiée, absentes de la version lab faute de ressources
- Le premier dépôt (`homelab-infra`) est conservé en l'état comme trace de cette première itération, plutôt que réécrit ou supprimé

## Conséquences

**Positives**
- Architecture plus proche d'un environnement de production réel, sans compromis matériel à justifier
- Possibilité de tester une vraie segmentation réseau (VLAN) et des couches de sécurité supplémentaires (WAF)
- Le serveur dédié étant physiquement séparé du poste de travail personnel, plus aucun impact sur l'usage quotidien

**Négatives / limites assumées**
- Le projet dépend désormais de la disponibilité du serveur dédié (coût récurrent, contrairement au lab local)
- Reste un seul serveur physique : aucune haute disponibilité réelle n'est possible sans un second nœud
