# Phase 6 : poste client Windows 11, jonction au domaine et GPO

## Objectif
Joindre un poste Windows 11 au domaine `ad.lab.lan`, s'y connecter avec un compte Active Directory, puis appliquer une stratégie de groupe (GPO) à des utilisateurs.

## Architecture

```
Internet -> VMnet8 (NAT) -> [pfSense] -> VMnet2 (LAN 192.168.10.0/24) -> [SRV-DC01] 192.168.10.20
                                                                      -> [CLI-W11-01] DHCP (192.168.10.1xx)
```

| Élément | Valeur |
|---|---|
| Machine | CLI-W11-01 (Windows 11 Enterprise, évaluation de 90 jours) |
| Ressources | (2 vCPU, 4 Go de RAM, 60 Go de disque), UEFI, TPM virtuel |
| Réseau | VMnet2, adresse reçue en DHCP de pfSense |
| Compte de connexion | LAB\bdurand (compte du domaine, groupe GG_Compta) |
| Emplacement dans l'annuaire | OU=Postes,OU=Lab,DC=ad,DC=lab,DC=lan |

## Prérequis : le DNS des clients
pfSense distribuait sa propre adresse comme DNS. Un poste Windows ne pouvait donc pas retrouver le contrôleur de domaine : AD est localisé grâce à des enregistrements SRV du DNS que seul le DC connaît. Dans *Services → DHCP Server → LAN* de pfSense :
- DNS Servers : `192.168.10.20` (le DC), sans DNS externe en secours
- Domain name : `ad.lab.lan`

Les clients passent toujours par le DC pour Internet : il transmet ce qu'il ne connaît pas à pfSense (redirecteur DNS).

## Étapes réalisées
1. Création de la VM sur VMnet2, avec un TPM virtuel (VMware chiffre seulement les fichiers nécessaires au TPM) et installation manuelle de Windows 11 Enterprise.
2. Configuration initiale sans compte Microsoft : « Configurer pour une utilisation professionnelle ou scolaire », puis « Options de connexion », puis « Joindre un domaine à la place », avec un compte local temporaire.
3. Installation des VMware Tools.
4. Vérification du réseau avant la jonction :

```
ipconfig /all           # DNS 192.168.10.20, suffixe ad.lab.lan
nslookup ad.lab.lan     # 192.168.10.20
ping 192.168.10.20
```

5. Renommage, jonction au domaine et placement dans l'OU `Postes` en une commande :

```
Add-Computer -DomainName "ad.lab.lan" -OUPath "OU=Postes,OU=Lab,DC=ad,DC=lab,DC=lan" -NewName "CLI-W11-01" -Credential "LAB\Administrateur" -Restart
```

6. Première connexion avec `LAB\bdurand` (choix « Autre utilisateur ») : le mot de passe temporaire doit être changé (`-ChangePasswordAtLogon`).

Vérification côté DC :

```
Get-ADComputer -Filter * | Select-Object Name,DistinguishedName
```
`CLI-W11-01` apparaît dans `OU=Postes,OU=Lab`. Sans `-OUPath`, un poste atterrit dans le dossier par défaut `Computers`, où aucune GPO ne peut être liée.

## Première GPO
- Nom : `GPO_Utilisateurs_Restrictions`
- Liée à : `OU=Utilisateurs,OU=Lab,DC=ad,DC=lab,DC=lan`
- Paramètre : Configuration utilisateur, Modèles d'administration, Panneau de configuration, « Interdire l'accès au Panneau de configuration et à l'application Paramètres du PC » activé. Techniquement, la GPO écrit la valeur de registre `NoControlPanel = 1` sous `HKCU\Software\Microsoft\Windows\CurrentVersion\Policies\Explorer`.

Vérification sur le client, dans la session de `bdurand` :
- ouverture de Paramètres ou du Panneau de configuration : message « Cette opération a été annulée en raison de restrictions sur cet ordinateur »
- `gpresult /r /scope user` dans un terminal non administrateur : la GPO est listée dans « Objets Stratégie de groupe appliqués »
- `gpupdate /force` puis déconnexion et reconnexion pour forcer l'application

## Problèmes rencontrés

| Problème | Cause | Résolution |
|---|---|---|
| L'écran de configuration initiale demande un compte Microsoft ou Entra ID | Windows pousse vers l'annuaire cloud de Microsoft, pas vers un domaine local | « Options de connexion », puis « Joindre un domaine à la place » avec un compte local |
| `Add-Computer` : « Le nom d'utilisateur ou le mot de passe est incorrect » | Sur Windows Server en français, le compte administrateur intégré s'appelle `Administrateur` ; `LAB\Administrator` n'existe pas | `-Credential "LAB\Administrateur"` |
| `gpresult /r` affichait « N/A » | Commande lancée sur le DC, sous le compte Administrateur (dossier `Users`, sans GPO liée) | La lancer sur le client, dans la session de l'utilisateur, avec un terminal non administrateur |

## Ce que j'ai appris
- Un poste joint au domaine a un compte d'ordinateur dans l'annuaire et fait confiance au DC pour authentifier les utilisateurs (Kerberos, d'où l'importance de l'heure).
- Les clients doivent utiliser le DNS du DC, sinon ils ne trouvent pas le domaine.
- Une GPO est liée à une OU et ne s'applique qu'aux objets qui s'y trouvent : le compte Administrateur (dossier `Users`) n'est pas concerné.
- Les stratégies utilisateur se chargent à l'ouverture de session ; `gpupdate /force` demande un nouveau chargement.
- Ordre d'application : local, site, domaine, OU (`LSDOU`), la dernière appliquée l'emporte en cas de conflit.
- `gpresult` décrit le contexte dans lequel il est lancé (machine et utilisateur).

## Prochaines étapes
- Restreindre la GPO au seul groupe `GG_Compta` avec le filtrage de sécurité. Piège à connaître (MS16-072) : une fois « Utilisateurs authentifiés » retiré du filtrage, il faut lui redonner le droit de **lecture** dans l'onglet Délégation, sinon la GPO ne s'applique à personne.
- Tester avec `LAB\amartin` (qui ne doit plus être bloquée).
- Autres GPO : lecteur réseau, fond d'écran, règles de mots de passe.
- Joindre Ubuntu au domaine avec SSSD.