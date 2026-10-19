# Phase 3 : plan d'adressage et IP statique

## Objectif
Donner des adresses stables aux serveurs du lab. Un serveur dont l'adresse change avec le DHCP est ingérable : supervision, règles de firewall et DNS reposent sur des adresses fixes.

## Plan d'adressage du LAN (192.168.10.0/24)

| Plage | Usage |
|---|---|
| 192.168.10.1 | Carte virtuelle VMnet2 de l'ordinateur hôte (VMware) |
| 192.168.10.10 à .50 | Serveurs (IP fixes) |
| 192.168.10.100 à .200 | DHCP dynamique (clients) |
| 192.168.10.254 | pfSense (passerelle et DNS) |

| Machine | IP | Méthode |
|---|---|---|
| pfSense-FW01 | 192.168.10.254 | Statique |
| Ubuntu Server (srv-ubuntu01) | 192.168.10.10 | Statique (netplan) |

## Réservation DHCP ou IP statique
- **Réservation DHCP** : configurée sur le serveur DHCP (pfSense). La machine reste en DHCP et reçoit toujours la même adresse, associée à son adresse MAC. Avantage : tout est géré au même endroit.
- **IP statique** : configurée sur la machine elle-même. Elle ne dépend plus du DHCP, mais il faut penser à rester cohérent avec le plan d'adressage et éviter les doublons.
- Les adresses fixes sont placées **hors de la plage DHCP** pour qu'aucun client ne les reçoive par hasard.

## Configuration Ubuntu (netplan)

Fichier `/etc/netplan/50-cloud-init.yaml` :

```yaml
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: false
      addresses:
        - 192.168.10.10/24
      routes:
        - to: default
          via: 192.168.10.254
      nameservers:
        addresses:
          - 192.168.10.254
```

Désactivation de l'écrasement par cloud-init : `/etc/cloud/cloud.cfg.d/99-disable-network-config.cfg` contient `network: {config: disabled}`.

## Commandes utiles

```
ip a                 # adresses des interfaces
ip route             # table de routage, passerelle par défaut
sudo netplan generate   # vérifie la syntaxe du YAML (silence = OK)
sudo netplan try        # applique avec retour arrière automatique après 120 s
```

## Validation
- `ip a` : `192.168.10.10/24` sans la mention `dynamic`
- `ip route` : `default via 192.168.10.254 proto static`
- Ping vers pfSense, vers 8.8.8.8 et vers google.com : tous réussis

## Problème rencontré
`netplan try` annulait la configuration avec « the configuration could not be generated », sans détail. Diagnostic : `sudo netplan generate` pour voir l'erreur éventuelle, puis `sudo cat -A` sur le fichier pour détecter des tabulations ou caractères cachés. Le fichier était correct, et une nouvelle exécution de `netplan try` a fonctionné.

## Ce que j'ai appris
- Le rôle d'une IP statique et d'une réservation DHCP.
- La syntaxe YAML de netplan (indentation en espaces, jamais de tabulations).
- `netplan try` comme filet de sécurité pour ne pas se couper de sa propre machine.