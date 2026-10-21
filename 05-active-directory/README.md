# Phase 5 : Windows Server 2022 et Active Directory

## Objectif
Mettre en place un contrôleur de domaine Active Directory sur le LAN, derrière pfSense, avec le DNS intégré et une heure synchronisée.

## Architecture

```
Internet -> VMnet8 (NAT) -> [pfSense] -> VMnet2 (LAN 192.168.10.0/24) -> [SRV-DC01] 192.168.10.20
```

| Élément | Valeur |
|---|---|
| Machine | SRV-DC01 (Windows Server 2022 Standard Evaluation, expérience de bureau) |
| Ressources | 2 vCPU, 4 Go de RAM, 60 Go de disque, UEFI sans démarrage sécurisé |
| Réseau | VMnet2, IP fixe 192.168.10.20/24, passerelle 192.168.10.254 |
| Domaine | ad.lab.lan (NetBIOS : LAB) |
| DNS du DC | lui-même (127.0.0.1), avec pfSense (192.168.10.254) en redirecteur |
| Source de temps | pfSense (192.168.10.254) |

Le domaine est `ad.lab.lan` et non `lab.lan` (utilisé par pfSense) ou `.local` : le TLD `.local` entre en conflit avec mDNS.

## Étapes réalisées
1. Création de la VM sur VMnet2 et installation de Windows Server 2022 (installation manuelle).
2. Installation des VMware Tools.
3. IP fixe, DNS provisoire vers pfSense et renommage en SRV-DC01 :

```
New-NetIPAddress -InterfaceAlias "Ethernet0" -IPAddress 192.168.10.20 -PrefixLength 24 -DefaultGateway 192.168.10.254
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses 192.168.10.254
Rename-Computer -NewName "SRV-DC01" -Restart
```

4. Installation du rôle AD DS et création de la forêt :

```
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
Install-ADDSForest -DomainName "ad.lab.lan" -DomainNetbiosName "LAB" -InstallDns
```

5. Après le redémarrage, le DC utilise son propre DNS et transmet le reste à pfSense :

```
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses 127.0.0.1
Add-DnsServerForwarder -IPAddress 192.168.10.254
```

6. Synchronisation de l'heure avec pfSense (Kerberos tolère 5 minutes d'écart au maximum) :

```
w32tm /config /manualpeerlist:"192.168.10.254,0x8" /syncfromflags:manual /reliable:yes /update
Restart-Service w32time
w32tm /resync /rediscover
w32tm /query /source
```

## Validation
- `Get-ADDomain` : DNSRoot `ad.lab.lan`, NetBIOS `LAB`, les rôles PDC, RID et Infrastructure sur SRV-DC01.
- `nslookup ad.lab.lan` et `nslookup srv-dc01.ad.lab.lan` : réponse `192.168.10.20`.
- `nslookup google.com` : le redirecteur vers pfSense fonctionne.
- `w32tm /query /source` : `192.168.10.254,0x8`.
- `w32tm /stripchart /computer:192.168.10.254 /samples:3 /dataonly` : décalage inférieur à une seconde.
- `Get-NetIPAddress` : `192.168.10.20` en Manual.

## Problèmes rencontrés

| Problème | Cause | Résolution |
|---|---|---|
| « Windows ne trouve pas le Contrat de licence Microsoft » au démarrage de l'installation | Le fichier de réponses d'Easy Install de VMware était mal interprété par l'installateur | Installation manuelle, sans Easy Install |
| `ad.lab.lan` résolu en 192.168.10.101 au lieu de 192.168.10.20 | Le serveur avait gardé une adresse reçue en DHCP (plage .100 à .200) et l'avait enregistrée dans le DNS | DHCP désactivé sur la carte, IP fixe appliquée, `ipconfig /registerdns`, `nltest /dsregdns`, puis suppression des anciens enregistrements A |
| Source de temps affichée « Local CMOS Clock » | Le service n'avait pas encore pris en compte pfSense | `w32tm /resync /rediscover` après avoir vérifié avec `stripchart` que pfSense répondait |
| `Add-DnsServerForwarder` : « adresse déjà configurée » | La promotion avait déjà repris pfSense comme redirecteur | Aucune action, simple vérification avec `Get-DnsServerForwarder` |
| `dcdiag /q` signale `DFSREvent` et `SystemLog` en échec | Avertissements DFSR normaux juste après la création du domaine, et événements du premier démarrage de Windows (avant AD) | À surveiller : devrait disparaître sous 24 heures |

## Ce que j'ai appris
- Le rôle du contrôleur de domaine et du DNS intégré : AD repose sur le DNS (enregistrements SRV).
- Un DC doit avoir une IP fixe et utiliser son propre DNS.
- Pourquoi éviter `.local` et choisir un nom de domaine distinct de celui du routeur.
- L'importance de l'heure pour Kerberos, et comment la synchroniser avec `w32tm`.
- Diagnostiquer avec `nslookup`, `Get-NetIPAddress`, `dcdiag`, `w32tm`.

## Prochaines étapes
- Créer les unités d'organisation, les groupes de sécurité et les utilisateurs.
- Écrire une première GPO.
- Joindre une machine cliente au domaine.