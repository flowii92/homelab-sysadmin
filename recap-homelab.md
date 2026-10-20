# Récapitulatif du homelab

Laboratoire VMware reconstruit de zéro pour s'entraîner à l'administration systèmes et réseaux (orientation future : DevOps ou cybersécurité). Chaque phase est documentée dans son propre dossier.

## Environnement

| Élément | Valeur |
|---|---|
| Hôte | Windows, Intel i7-10700KF (8 cœurs / 16 threads), 32 Go de RAM |
| Hyperviseur | VMware Workstation Pro 25H2 |
| Pare-feu / routeur | pfSense CE 2.9.0 |
| Serveur Linux | Ubuntu Server 24.04 LTS |

## Architecture actuelle

```
Internet
   |
VMnet8 (NAT VMware, DHCP)
   |
[pfSense-FW01]  WAN em0 : DHCP (192.168.216.x)
                LAN em1 : 192.168.10.254/24
   |
VMnet2 (hôte uniquement, 192.168.10.0/24)
   |
[Ubuntu Server]  192.168.10.10/24 (statique)
```

### Réseaux virtuels

| Réseau | Type | Subnet | Usage | État |
|---|---|---|---|---|
| VMnet8 | NAT | géré par VMware (DHCP activé) | WAN de pfSense | utilisé |
| VMnet2 | Hôte uniquement, sans DHCP VMware | 192.168.10.0/24 | LAN | utilisé |
| VMnet3 | Hôte uniquement, sans DHCP VMware | 192.168.20.0/24 | DMZ | créé, pas encore utilisé |
| VMnet4 | Hôte uniquement, sans DHCP VMware | 192.168.99.0/24 | Management | créé, pas encore utilisé |

### Plan d'adressage du LAN (192.168.10.0/24)

| Plage | Usage |
|---|---|
| 192.168.10.1 | Carte virtuelle VMnet2 de l'hôte (VMware) |
| 192.168.10.10 à .50 | Serveurs (IP fixes) |
| 192.168.10.100 à .200 | DHCP dynamique (clients) |
| 192.168.10.254 | pfSense (passerelle, DNS, DHCP) |

### Machines virtuelles

| VM | Ressources | Réseau | IP |
|---|---|---|---|
| pfSense-FW01 | 1 vCPU, 2 Go, 20 Go (ZFS) | NAT (WAN) + VMnet2 (LAN) | WAN en DHCP, LAN 192.168.10.254 |
| Ubuntu Server | 2 vCPU, 4 Go, 25 Go | VMnet2 | 192.168.10.10 (netplan) |

Ordre de démarrage : pfSense d'abord, puis les autres VM (pfSense fournit le DHCP et la passerelle).

## Phases réalisées

### Phase 0 : GitHub
- Dépôt public `homelab-sysadmin`, Git installé et configuré sur le poste.
- Cycle de travail : `git status`, `git add .`, `git commit -m "..."`, `git push`.
- Règle : jamais de mots de passe, de clés privées SSH ni de `config.xml` pfSense dans le dépôt.

### Phase 1 : fondations (`01-fondations`)
- Création des réseaux virtuels dans le Virtual Network Editor.
- VM Ubuntu Server avec OpenSSH, d'abord en NAT.
- Lecture de `ip a` : interfaces `lo` et `ens33`, adresse, masque `/24`, bail DHCP.
- Connexion SSH depuis PowerShell, après activation du service (`systemctl enable --now ssh`). Sur Ubuntu 24.04, c'est `ssh.socket` qui écoute sur le port 22.

### Phase 2 : pfSense (`02-pfsense`)
- Installation de pfSense CE 2.9.0 avec l'installateur Netgate.
- Ajout de la 2e carte réseau (VMnet2) après l'installation, puis assignation des interfaces : WAN = em0, LAN = em1.
- LAN configuré en 192.168.10.254/24 avec serveur DHCP de .100 à .200.
- Assistant web : nom d'hôte, domaine en `.lan` (jamais `.local`), DNS, fuseau Europe/Paris, mot de passe admin changé.
- « Block private networks » décoché sur le WAN (le réseau NAT est en 192.168.x.x).
- Ubuntu rebranchée sur VMnet2 et connectivité validée du proche au lointain.

### Phase 3 : adressage et IP statique (`03-adressage`)
- Plan d'adressage documenté.
- Différence entre réservation DHCP (côté serveur DHCP) et IP statique (sur la machine).
- Configuration netplan : désactivation de l'écrasement par cloud-init, fichier YAML en 192.168.10.10/24, passerelle et DNS en 192.168.10.254.
- Validation avec `netplan try`.

Configuration netplan utilisée :

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

## Méthode de diagnostic réseau

Du plus proche au plus lointain, pour savoir où ça casse :

```
ip a                        # l'interface a-t-elle une adresse ?
ip route                    # la passerelle par défaut est-elle présente ?
ping -c 3 192.168.10.254    # LAN : pfSense joignable ?
ping -c 3 8.8.8.8           # routage + NAT : sortie Internet ?
ping -c 3 google.com        # DNS : résolution de noms ?
```

- Échec au premier ping : problème de LAN (mauvais réseau VMware, pas d'IP).
- Échec au deuxième : routage, NAT ou WAN pfSense.
- Échec au troisième seulement : DNS.

## Problèmes rencontrés

| Problème | Cause | Résolution |
|---|---|---|
| Démarrage Ubuntu bloqué sur `systemd-networkd-wait-online`, puis « autoconfiguration failed » | Case « Use local DHCP service » décochée sur VMnet8 (la consigne ne concernait que VMnet2, 3 et 4) | Recocher la case dans le Virtual Network Editor |
| SSH `inactive (dead)` et `disabled` | Service non activé ; sur Ubuntu 24.04, `ssh.socket` écoute | `sudo systemctl enable --now ssh` |
| `systemctl enable` demandait un mot de passe | Commande lancée sans `sudo` | Préfixer par `sudo` (la saisie du mot de passe n'affiche rien) |
| Une seule carte réseau dans pfSense | 2e carte non ajoutée ; risque d'écraser la carte NAT en la modifiant au lieu d'en ajouter une | *Ajouter → Carte réseau*, puis assignation depuis la console |
| Console pfSense en QWERTY | Clavier de la console VMware | Passer par l'interface web et SSH |
| `netplan try` : « configuration could not be generated » | Cause exacte non élucidée ; fichier vérifié correct | `netplan generate`, `cat -A`, puis nouvel essai de `netplan try` |
| Notepad : « pas l'autorisation d'ouvrir ce fichier » | Problème de fichier ou de droits | Recréer le fichier avec `New-Item` |

## Compétences travaillées

- **Virtualisation** : réseaux virtuels (NAT, hôte uniquement), cartes multiples, snapshots, ordre de démarrage.
- **Réseau** : sous-réseaux et notation `/24` (254 hôtes utilisables), WAN/LAN, NAT, DHCP, passerelle, DNS, conflit d'adresse sur `.1`.
- **Linux** : `ip`, `systemctl`, `sudo`, netplan, YAML, cloud-init.
- **pfSense** : installation, assignation d'interfaces, console, interface web.
- **Diagnostic** : démarche progressive, lecture de messages d'erreur.
- **Git/GitHub** : cycle add/commit/push, documentation structurée, hygiène sur les secrets.

## Commandes clés

```
ip a
ip route
sudo netplan generate        # silence = syntaxe correcte
sudo netplan try             # retour arrière automatique après 120 s
sudo systemctl enable --now ssh
ssh utilisateur@192.168.10.10
git status
git add .
git commit -m "message"
git push
```

## Reste à faire

- [ ] Authentification SSH par clé, puis `PasswordAuthentication no` et `PermitRootLogin no`
- [ ] Règles de firewall pfSense (LAN vers WAN, isolation entre réseaux)
- [ ] Windows Server 2022 + Active Directory (AD DS, DNS, DHCP, GPO, jonction de machines)
- [ ] Supervision : Zabbix + Grafana
- [ ] Sauvegarde et restauration testée
- [ ] Logs centralisés
- [ ] Automatisation : scripts, Ansible, puis Docker et CI/CD
- [ ] Sécurité : VPN WireGuard, Wazuh, tests depuis une VM Kali
