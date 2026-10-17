# Phase 2 : pfSense (routeur / firewall)

## Objectif
Mettre en place un routeur/firewall central pour le lab : toutes les VM passent par pfSense pour accéder à Internet, et il distribue les adresses IP du réseau interne.

## Architecture

```
Internet -> VMnet8 (NAT) -> [pfSense] -> VMnet2 (LAN) -> VM du lab
```

| Élément | Valeur |
|---|---|
| WAN (em0) | DHCP via VMnet8 (NAT VMware) |
| LAN (em1) | VMnet2 (Hôte uniquement), 192.168.10.0/24 |
| IP de pfSense sur le LAN | 192.168.10.254 |
| DHCP du LAN | 192.168.10.100 à 192.168.10.200 |
| Nom d'hôte / domaine | pfsense-fw01 / lab.lan |

Pourquoi `.254` et pas `.1` : VMware attribue `192.168.10.1` à la carte virtuelle de l'ordinateur hôte sur VMnet2, ce qui aurait provoqué un conflit d'adresse.

## Machine virtuelle
- pfSense CE 2.9.0-RELEASE (édition gratuite, version stable)
- 1 vCPU, 2 Go de RAM, disque de 20 Go (ZFS)
- 2 cartes réseau : NAT (WAN) et VMnet2 (LAN)

## Étapes réalisées
1. Création de la VM et installation avec l'installateur Netgate.
2. Ajout de la 2e carte réseau (VMnet2) après l'installation.
3. Assignation des interfaces depuis la console : WAN = em0, LAN = em1.
4. Configuration de l'IP du LAN (192.168.10.254/24) et activation du serveur DHCP.
5. Assistant de configuration web : nom d'hôte, domaine, DNS, fuseau horaire Europe/Paris, changement du mot de passe admin.
6. Décochage de « Block private networks » sur le WAN (le réseau NAT est en 192.168.x.x).
7. Branchement de la VM Ubuntu sur VMnet2 (derrière pfSense).

## Tests de validation
Depuis la VM Ubuntu, du plus proche au plus lointain :

```
ip a                      # adresse en 192.168.10.x reçue du DHCP de pfSense
ip route                  # passerelle par défaut : 192.168.10.254
ping -c 3 192.168.10.254  # pfSense joignable (LAN)
ping -c 3 8.8.8.8         # sortie Internet (routage + NAT)
ping -c 3 google.com      # résolution DNS
```

Tous les tests réussissent. Cet ordre indique où chercher en cas de panne : LAN, puis routage/NAT, puis DNS.

## Problèmes rencontrés
- **Pas d'IP sur VMnet8** : le service DHCP local de VMnet8 était décoché dans le Virtual Network Editor. Cause : la consigne de décocher le DHCP ne concerne que les réseaux gérés par pfSense (VMnet2, 3, 4), pas VMnet8. Diagnostic : `ip a`, puis vérification du réseau virtuel et du service DHCP.
- **Une seule carte réseau détectée à l'installation** : le LAN n'a pu être assigné qu'après l'ajout de la 2e carte réseau, depuis le menu console de pfSense.
- **Console en QWERTY** : la console VMware n'utilise pas le clavier AZERTY. Solution : passer par l'interface web et SSH.

## Ce que j'ai appris
- La différence entre WAN et LAN, et le rôle du NAT.
- Pourquoi on évite l'adresse `.1` quand l'hôte VMware l'utilise déjà.
- Une méthode de diagnostic réseau progressive (LAN, puis routage, puis DNS).
- Les snapshots VMware avant chaque modification importante.

## Prochaines étapes
- IP fixe sur Ubuntu et réservation DHCP côté pfSense.
- Comprendre et écrire des règles de firewall.
- Installer Windows Server et Active Directory sur le LAN.