Phase 4 : règles de firewall pfSense
Objectif

Passer d'un LAN totalement ouvert (« tout est autorisé ») à une politique deny by default : seul le trafic nécessaire est autorisé, tout le reste est bloqué et journalisé.

Principes de fonctionnement
Les règles s'appliquent sur l'interface où le trafic entre. Une règle de l'onglet LAN filtre ce qui arrive des machines du LAN.
Les règles sont lues de haut en bas, et la première qui correspond s'applique. L'ordre compte.
Tout ce qui n'est pas explicitement autorisé est bloqué (règle de refus implicite en fin de liste).
pfSense est stateful : une connexion sortante autorisée voit ses paquets de retour acceptés automatiquement.
Pass / Block / Reject : Pass autorise, Block jette le paquet en silence (le client attend un timeout), Reject renvoie un refus immédiat. Block côté Internet, Reject en interne pour faciliter le diagnostic.
Règles du LAN
#	Action	Protocole	Source	Destination	Port	Description
0	Pass	*	*	LAN Address	80, 443	Anti-Lockout Rule (règle par défaut, à ne jamais supprimer)
1	Pass	TCP/UDP	LAN subnets	This Firewall (self)	53	DNS vers pfSense
2	Pass	UDP	LAN subnets	This Firewall (self)	123	NTP vers pfSense
3	Pass	ICMP (any)	LAN subnets	any	-	Ping autorisé
4	Pass	TCP	LAN subnets	any	alias WEB_PORTS	Web et mises à jour

Les règles par défaut « Default allow LAN to any rule » (IPv4 et IPv6) sont désactivées (Toggle), pas supprimées : on peut les réactiver en cas de besoin. L'IPv6 n'est pas utilisé dans le lab.

Alias
WEB_PORTS (type Port(s)) : 80 et 443. Créé dans Firewall → Aliases → Ports, puis utilisé dans la règle 4. Un alias garde les règles lisibles et réutilisables.
Tests réalisés (depuis la VM Ubuntu)
Test	Résultat attendu	Résultat
sudo apt update	Réussit (ports 80/443)	OK
curl -I https://example.com	Réussit (443)	OK
ping -c 3 8.8.8.8	Réussit (ICMP)	OK
nc -vz -w 3 github.com 22	Timeout (port 22 non autorisé)	Timeout, bloqué par la règle de refus implicite
Lire les logs

Status → System Logs → Firewall affiche les paquets bloqués :

croix rouge : paquet bloqué
Default deny rule IPv4 : règle de refus implicite
192.168.10.10:port → IP-de-GitHub:22 : la source, la destination et le port visé
TCP:S : paquet SYN, première étape d'une connexion TCP
plusieurs lignes espacées d'une seconde : le client retransmet son SYN faute de réponse
Problèmes rencontrés et corrections
Problème	Explication	Correction
Règle NTP avec la source « LAN address »	« LAN address » est l'adresse de pfSense lui-même, pas celle des machines du LAN	Source remplacée par LAN subnets
Port « 80 - 443 » sur la règle web	Une plage autorise tous les ports de 80 à 443 (135, 139, 143, 389...)	Remplacé par l'alias WEB_PORTS (80 et 443 seulement)
Source « WAN subnets » sur la règle web	Sans effet sur le trafic du LAN	Source remplacée par LAN subnets
Nouvelles règles à 0/0 B	Les règles « Default allow » placées au-dessus interceptaient tout le trafic	Règles « Default allow » désactivées
Test du port 22 vers 8.8.8.8 non concluant	Le DNS de Google n'écoute pas sur le port 22 : un timeout serait apparu même sans firewall	Test refait vers un port réellement ouvert ailleurs (SSH de GitHub), confirmé dans les logs
Synchronisation NTP d'Ubuntu bloquée	Ubuntu interrogeait ses serveurs sur Internet (UDP 123), non autorisé	Ubuntu configurée pour utiliser pfSense : NTP=192.168.10.254 dans /etc/systemd/timesyncd.conf

Configuration NTP côté Ubuntu :

sudo nano /etc/systemd/timesyncd.conf     # section [Time] : NTP=192.168.10.254
sudo systemctl restart systemd-timesyncd
timedatectl timesync-status
Ce que j'ai appris
Le modèle de pare-feu stateful et l'importance de l'ordre des règles.
Le principe du moindre privilège appliqué au réseau (deny by default).
L'usage des alias pour garder des règles propres.
Lire un journal de firewall (source, destination, port, type de paquet).
Un timeout seul ne prouve pas un blocage : il faut tester un cas qui réussirait sans la règle, puis le confirmer dans les logs.
Une heure synchronisée est indispensable pour la suite (Kerberos et Active Directory tolèrent très peu de décalage).
Prochaines étapes
Windows Server 2022 et Active Directory sur le LAN.
Règles adaptées aux services AD (DNS, Kerberos, LDAP, SMB).
Utilisation des réseaux VMnet3 (DMZ) et VMnet4 (management).