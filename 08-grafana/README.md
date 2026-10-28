# Phase 8 – Visualisation avec Grafana

## Objectif

Ajouter une couche de visualisation au-dessus de Zabbix : un dashboard unique qui donne en un coup d'œil l'état du lab (alertes actives, charge des serveurs Windows, trafic et disponibilité du firewall).

Zabbix collecte et alerte, Grafana présente.

![Dashboard Homelab](dashboard-homelab.png)

## Architecture

| Élément | Valeur |
|---|---|
| VM | `claude-srv` (Ubuntu Server) – 192.168.10.10 |
| Grafana | Port **3000** (dépôt APT officiel Grafana) |
| Zabbix frontend | Port **8080** (même VM) |
| Plugin | `alexanderzobnin-zabbix-app` (Grafana Labs) |
| Connexion | API Zabbix en local (`http://localhost:8080/api_jsonrpc.php`) |
| Compte utilisé | `grafana`, utilisateur Zabbix **en lecture seule** |

```
[SRV-DC01] [CLI-W11-01] [pfSense-FW01]
     agent 2     agent 2      SNMP v2c / ICMP
        \           |           /
         └──── Zabbix server ───┘   (192.168.10.10:8080)
                     │ API JSON-RPC (lecture seule)
                  Grafana            (192.168.10.10:3000)
```

## 1. Installation de Grafana

```bash
sudo apt install -y apt-transport-https wget gnupg
sudo mkdir -p /etc/apt/keyrings
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee /etc/apt/sources.list.d/grafana.list
sudo apt update
sudo apt install grafana -y
sudo systemctl enable --now grafana-server
```

Accès : `http://192.168.10.10:3000`. Le mot de passe par défaut `admin/admin` est changé dès la première connexion.

## 2. Plugin Zabbix

```bash
sudo grafana cli --homepath /usr/share/grafana plugins install alexanderzobnin-zabbix-app
sudo systemctl restart grafana-server
```

Activation ensuite dans *Administration → Plugins and data → Plugins → Zabbix* : c'est l'**application** (icône grille) qu'il faut activer, elle embarque la source de données.

## 3. Compte Zabbix dédié en lecture seule

Plutôt que de connecter Grafana avec le compte Admin, j'ai créé un compte dédié (principe du **moindre privilège**) :

- Groupe d'utilisateurs `Grafana-RO` : permission **Lecture** sur les groupes d'hôtes (Firewalls, Windows servers, Linux servers, Zabbix servers)
- Utilisateur `grafana` : membre de `Grafana-RO`, rôle **User role**

Si Grafana était compromis, l'attaquant ne pourrait que lire les métriques, pas modifier la supervision.

## 4. Source de données

*Connections → Data sources → Add data source → Zabbix*

- URL : `http://localhost:8080/api_jsonrpc.php` (Grafana et Zabbix sont sur la même VM)
- Authentification : utilisateur `grafana`
- **Save & test** → « Zabbix API version: 7.0.x »

## 5. Dashboard « Homelab – Vue d'ensemble »

| Panneau | Visualisation | Requête |
|---|---|---|
| Alertes actives | Problems | Group `/.*/`, Host `/.*/` |
| CPU Windows | Time series | Host `/SRV-DC01\|CLI-W11-01/`, Item `CPU utilization` |
| Trafic pfSense WAN/LAN | Time series | Host `pfSense-FW01`, Item `/Bits (received\|sent)/` |
| pfSense en ligne | Stat | Item `/ICMP ping\|SNMP agent availability/`, mapping `1 → UP`, `0 → DOWN` |

Les `/…/` sont des **expressions régulières** : un seul panneau couvre plusieurs hôtes ou plusieurs interfaces.

Le JSON du dashboard est exporté dans [`dashboard-homelab.json`](dashboard-homelab.json) et peut être réimporté (*Dashboards → New → Import*).

## Problèmes rencontrés

| Problème | Cause | Solution |
|---|---|---|
| `grafana-cli` : *Could not find config defaults* | `grafana-cli` est déprécié et ne trouve plus son homepath | Utiliser `grafana cli --homepath /usr/share/grafana ...` |
| Deux plugins « Zabbix » affichés | Le plugin existe en tant qu'application et source de données | Activer l'application (icône grille) |
| Alertes Zabbix */etc/passwd has been changed* et *Number of installed packages has been changed* | L'installation de Grafana a créé l'utilisateur système `grafana` et ajouté des paquets | Changement attendu : alertes **acquittées** avec un commentaire dans Zabbix |

## Ce que j'ai appris

- La différence entre **collecte/alerting** (Zabbix) et **visualisation** (Grafana)
- Interroger une application via son **API** avec un compte de service dédié
- Utiliser des **regex** pour construire des panneaux génériques
- Une alerte provoquée par sa propre action n'est pas du bruit : on l'**acquitte et on la trace**

## Pistes d'amélioration

- Passer Grafana en **HTTPS** (reverse proxy Nginx)
- Utiliser un **token API** Zabbix plutôt qu'un mot de passe
- Ajouter un dashboard par hôte avec des **variables** Grafana (liste déroulante d'hôtes)