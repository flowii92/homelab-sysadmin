# Phase 7 : supervision avec Zabbix

## Objectif
Superviser les machines du lab (contrôleur de domaine, poste client) depuis un serveur central : disponibilité, CPU, mémoire, disques, services Windows. Être alerté quand un seuil est dépassé ou qu'un service s'arrête.

## Architecture

```
[SRV-DC01]  192.168.10.20  ──┐  agent Zabbix 2 (port 10050)
[CLI-W11-01] 192.168.10.60 ──┤
                             │
              [Ubuntu : serveur Zabbix 7.0 LTS]  192.168.10.10
              PostgreSQL + Nginx (interface web, port 8080) + serveur Zabbix (port 10051)
```

| Composant | Rôle |
|---|---|
| Serveur Zabbix | Collecte les données, évalue les seuils (triggers), génère les problèmes |
| Base PostgreSQL | Stocke la configuration et l'historique des mesures |
| Interface web (Nginx + PHP-FPM) | Administration et tableaux de bord |
| Agent Zabbix 2 | Installé sur chaque machine supervisée, remonte les métriques |

## Principes
- **Vérification passive** : le serveur interroge l'agent (port TCP 10050 sur la machine supervisée).
- **Vérification active** : l'agent envoie ses données au serveur (port TCP 10051 sur le serveur).
- **Hôte** : une machine supervisée. **Élément (item)** : une mesure. **Trigger** : un seuil sur une mesure. **Problème** : un trigger déclenché.
- **Modèle (template)** : un ensemble d'éléments et de triggers réutilisable. Les machines Windows utilisent « Windows by Zabbix agent ».
- Le nom d'hôte saisi dans Zabbix doit être **identique** au `HOSTNAME` configuré dans l'agent, sinon les vérifications actives échouent.

## Installation du serveur (Ubuntu 24.04)
1. Ajout du dépôt officiel Zabbix 7.0 LTS, puis installation de `zabbix-server-pgsql`, `zabbix-frontend-php`, `zabbix-nginx-conf`, `zabbix-sql-scripts` et `zabbix-agent2`.
2. Installation de PostgreSQL (`sudo apt install -y postgresql`), création de l'utilisateur et de la base `zabbix`, import du schéma initial.
3. Renseignement du mot de passe de la base dans `/etc/zabbix/zabbix_server.conf` (`DBPassword=`), fichier jamais publié.
4. Interface web Nginx déplacée sur le port **8080** (`listen 8080;`) pour garder le port 80 libre.
5. Démarrage et activation des services : `zabbix-server`, `zabbix-agent2`, `nginx`, `php8.3-fpm`.
6. Assistant web de configuration, puis **changement du mot de passe par défaut du compte Admin**.

## Installation de l'agent sur Windows
Installation silencieuse de `zabbix_agent2-7.0.32-windows-amd64-openssl.msi` (même version que le serveur) :

```
msiexec /i zabbix_agent2.msi /qn SERVER=192.168.10.10 SERVERACTIVE=192.168.10.10 HOSTNAME=<NOM-DU-POSTE>
```

| Paramètre | Rôle |
|---|---|
| `SERVER` | Serveur autorisé à interroger l'agent (vérifications passives) |
| `SERVERACTIVE` | Serveur auquel l'agent envoie ses données (vérifications actives) |
| `HOSTNAME` | Nom de l'hôte, identique à celui saisi dans Zabbix |

Ouverture du port dans le pare-feu Windows, **limitée au serveur Zabbix** (moindre privilège) :

```
New-NetFirewallRule -DisplayName "Zabbix Agent" -Direction Inbound -Protocol TCP -LocalPort 10050 -RemoteAddress 192.168.10.10 -Action Allow
```

Vérifications :

```
Get-Service "Zabbix Agent 2"                       # Running
Get-NetTCPConnection -LocalPort 10050 -State Listen
nc -vz <IP-du-poste> 10050                          # depuis Ubuntu : succeeded
```

## Ajout d'un hôte dans Zabbix
*Collecte de données → Hôtes → Créer un hôte* : nom identique au `HOSTNAME`, modèle `Windows by Zabbix agent`, groupe d'hôtes, interface Agent (IP, port 10050). Après 1 à 2 minutes, l'icône **ZBX** passe au vert.

Le poste client étant en DHCP, une **réservation DHCP** pfSense lui donne une adresse stable (192.168.10.60, hors de la plage dynamique), sans quoi Zabbix perdrait le poste à chaque changement d'adresse.

## Résultat
- SRV-DC01 : agent actif, services Active Directory découverts automatiquement.
- CLI-W11-01 : agent actif, métriques CPU, mémoire, disque visibles dans *Dernières données*.
- Test de charge CPU sur le client : pic visible sur le graphique.

## Problèmes rencontrés et corrections

| Problème | Explication | Correction |
|---|---|---|
| `sudo: unknown user postgres` | Le serveur PostgreSQL n'était pas installé : les paquets Zabbix n'installent que le client | `sudo apt install -y postgresql` |
| Confusion sur le rôle de l'agent | L'agent s'installe sur la machine **supervisée**, pas sur le serveur Zabbix | Un agent par machine à superviser |
| Nom DNS du client inutilisable côté Ubuntu | Ubuntu interroge pfSense, qui ne connaît pas les noms du domaine AD (ils sont sur le DC) | Réservation DHCP et ajout de l'hôte par adresse IP |
| Mauvaise machine pour les commandes | Commandes d'administration lancées sur une autre VM | Toujours lancer `hostname` avant de commencer |

## Ce que j'ai appris
- Architecture serveur / agent, vérifications passives et actives, ports 10050 et 10051.
- Le nom d'hôte doit correspondre des deux côtés.
- Ouvrir un port en le limitant à la source qui en a besoin.
- Un service d'infrastructure (DHCP, DNS) doit donner une adresse stable aux machines supervisées.
- Un message d'erreur sur un utilisateur ou un service inconnu signale souvent un paquet non installé.
- Les mots de passe (base de données, compte Admin) ne vont jamais dans le dépôt.

## Prochaines étapes
- Supervision de pfSense (ICMP, puis SNMP avec une règle UDP 161 limitée au serveur Zabbix).
- Grafana relié à Zabbix pour les tableaux de bord.
- Alerte sur l'arrêt d'un service (exercice avec le spooler d'impression).