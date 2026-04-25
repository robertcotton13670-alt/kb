---
title: Commandes & références MSP
description: Index des commandes, outils et liens utiles — références rapides MSP
---

# Commandes & références MSP

Référence rapide des commandes, outils en ligne et portails admin. Les sujets complexes ont leur propre page dédiée.

## Diagnostic Entra / Intune / AD

| Commande | Description |
|---|---|
| [`dsregcmd /status`](dsregcmd.md) | État Entra Join, Hybrid Join, PRT, MDM — voir page dédiée |
| `dsregcmd /leave` | Quitter le join (redémarrage requis) |
| `%windir%\system32\deviceenroller.exe /c /AutoEnrollMDM` | Forcer l'enrollment Intune |
| `gpupdate /force` | Forcer l'application des GPO |
| `gpresult /h rapport.html` | Rapport HTML des GPO appliquées sur un poste |
| `eventvwr.msc` | Observateur d'événements (logs DeviceManagement, EnterpriseMgmt) |

## Réseau

Voir la page dédiée [ipconfig](ipconfig.md) pour toutes les options.

| Commande | Description |
|---|---|
| `ipconfig /all` | Config réseau complète (IP, MAC, DNS, DHCP) |
| `ipconfig /flushdns` | Vider le cache DNS |
| `ipconfig /release` + `/renew` | Renouveler le bail DHCP |
| `tracert <host>` | Tracer le chemin réseau hop par hop |
| `pathping <host>` | Combine ping + tracert avec latence par saut |
| `nslookup <host>` | Résolution DNS |
| `netstat -ano` | Connexions actives + PID |
| `arp -a` | Table ARP (MAC / IP) |
| `netsh int ip reset` | Reset la pile TCP/IP |
| `netsh winsock reset` | Reset le winsock (redémarrage requis) |
| `netsh wlan show profile name="X" key=clear` | Afficher le mot de passe WiFi d'un profil |
| `Test-NetConnection <host> -Port <port>` | Test port TCP (PowerShell) |
| `ping -t <host>` | Ping continu |

## Maintenance Windows

| Commande | Description |
|---|---|
| `sfc /scannow` | Vérifie et répare les fichiers système |
| `DISM /Online /Cleanup-Image /RestoreHealth` | Répare l'image Windows via Windows Update |
| `chkdsk C: /f /r` | Vérification et réparation du disque (redémarrage requis) |

!!! warning "chkdsk"
    Sur le volume système, chkdsk s'exécute au prochain redémarrage. Ne pas interrompre.

## winget

Voir la [page dédiée winget](winget.md) pour les scripts et le contexte Intune/Datto.

| Commande | Description |
|---|---|
| `winget search <nom>` | Chercher une app dans le catalogue |
| `winget install <id> --silent --accept-package-agreements` | Installer une app |
| `winget upgrade --all --silent` | Mettre à jour toutes les apps |
| `winget list` | Lister les apps installées |
| `winget export -o apps.json` | Exporter la liste des apps installées |
| `winget import -i apps.json` | Réimporter et installer depuis un fichier |

## Autopilot

Voir la [page dédiée Get-WindowsAutopilotInfo](autopilot-hash.md).

| Commande | Description |
|---|---|
| `Install-Script -Name Get-WindowsAutopilotInfo` | Installer le script |
| `Get-WindowsAutopilotInfo -OutputFile hash.csv` | Exporter le hardware hash en CSV |
| `Get-WindowsAutopilotInfo -Online` | Upload direct vers Intune |

## RDS / Sessions

Voir la [page dédiée RDS](rds-sessions.md) pour les cas d'usage MSP complets.

| Commande | Description |
|---|---|
| `change logon /drain` | Bloquer les nouvelles connexions (maintenance) |
| `change logon /enable` | Réactiver les connexions |
| `query session` | Lister toutes les sessions |
| `query user` | Utilisateurs connectés avec ID session |
| `logoff <ID>` | Déconnecter une session |
| `msg * "message"` | Envoyer un message à tous les utilisateurs connectés |

## PowerShell / Environnement

| Commande | Description |
|---|---|
| `Set-ExecutionPolicy -ExecutionPolicy Bypass -Scope Process` | Bypass temporaire (session uniquement) |
| `Set-ExecutionPolicy RemoteSigned -Scope CurrentUser` | Pour scripts locaux |
| `powershell.exe -ExecutionPolicy Bypass -File script.ps1` | Bypass à l'appel du script |
| `pwsh.exe` | Lancer PowerShell 7 |

## Exchange Online / M365

| Commande | Description |
|---|---|
| `Connect-ExchangeOnline -UserPrincipalName admin@domain.com` | Connexion Exchange Online |
| `Connect-MgGraph -Scopes "..."` | Connexion Graph API |
| `Get-Mailbox -ResultSize Unlimited \| Get-MailboxStatistics \| Select-Object DisplayName, TotalItemSize, ItemCount \| Export-Csv -Path "C:\tmp\rapport.csv" -NoTypeInformation -Encoding UTF8` | Export tailles boîtes mail |

## Graph API — OneDrive Shortcuts

Voir la [page dédiée Graph API OneDrive Shortcuts](graph-onedrive-shortcuts.md) pour le script V3 complet.

| Endpoint | Description |
|---|---|
| `POST https://login.microsoftonline.com/{tenantId}/oauth2/v2.0/token` | Obtenir un token app (client_credentials) |
| `GET /users/{email}/drive/root:/{FolderName}` | Vérifier si un dossier existe |
| `POST /users/{email}/drive/root/children` | Créer un dossier ou raccourci à la racine |
| `POST /users/{email}/drive/root:/{FolderName}:/children` | Créer un raccourci dans un sous-dossier |
| `GET /users/{userId}/drive/root/children` | Lister les items racine OneDrive |
| `DELETE /users/{userId}/drive/items/{itemId}` | Supprimer un raccourci |
| `GET /groups/{groupId}/members/microsoft.graph.user` | Membres d'un groupe Azure AD |

## Applications client M365

### Outlook (nouveau client)

| Élément | Description |
|---|---|
| `%localappdata%\Packages\MSTeams_8wekyb3d8bbwe\LocalCache\Microsoft\MSTeams` | Cache Teams nouveau client à vider |
| `%localappdata%\Packages\Microsoft.OutlookForWindows_8wekyb3d8bbwe\LocalCache\Local\Microsoft\Outlook\EBWebView` | Cache nouveau Outlook à vider pour reset profil |

!!! tip "Reset nouveau Outlook"
    Fermer Outlook, tuer `NewOutlook.exe` et `msedgewebview2.exe` dans le gestionnaire des tâches, puis vider le dossier EBWebView.

### Office / Activation

| Commande | Description |
|---|---|
| `cscript "C:\Program Files\Microsoft Office\Office16\OSPP.VBS" /dstatus` | État activation Office |
| `cscript OSPP.VBS /unpkey:<key>` | Supprimer une clé Office |
| `cscript OSPP.VBS /act` | Forcer l'activation Office |

## Raccourcis clavier

| Raccourci | Description |
|---|---|
| `Ctrl + Win + R` | Autopilot Reset local (nécessite droits admin) |
| `Win + V` | Historique presse-papiers |
| `Win + Shift + S` | Capture d'écran partielle (Snipping Tool) |
| Android : 5 appuis sur "Bienvenue" | Active le mode enrôlement Android Enterprise |

## Portails admin Microsoft

| Portail | URL |
|---|---|
| Admin M365 | `https://admin.cloud.microsoft` |
| Entra ID | `https://entra.microsoft.com` |
| Intune | `https://intune.microsoft.com` |
| Exchange admin | `https://admin.exchange.microsoft.com` |
| Suivi des messages | `https://admin.exchange.microsoft.com/#/messagetrace` |
| Quarantaine mails | `https://security.microsoft.com/quarantine` |
| Règles de flux (transport rules) | `https://admin.exchange.microsoft.com/#/transportrules` |
| Defender / Sécurité | `https://security.microsoft.com` |
| Teams admin | `https://admin.teams.microsoft.com` |
| SharePoint admin | `https://admin.microsoft.com/sharepoint` |
| État des services M365 | `https://status.cloud.microsoft` |
| Tous les portails Microsoft | `https://cmd.ms` |

## Sites et outils en ligne

### Mail / DNS / Délivrabilité

| Site | Usage |
|---|---|
| [mxtoolbox.com](https://mxtoolbox.com) | MX, SPF, DKIM, DMARC, blacklist, headers mail |
| [mail-tester.com](https://mail-tester.com) | Score de délivrabilité d'un mail |
| [dmarcian.com](https://dmarcian.com) | Analyse DMARC |
| [viewdns.info](https://viewdns.info) | Boîte à outils DNS (historique IP, whois, reverse DNS) |

### Diagnostic / Sécurité

| Site | Usage |
|---|---|
| [haveibeenpwned.com](https://haveibeenpwned.com) | Vérifier si un email est compromis |
| [whatismyip.com](https://whatismyip.com) | IP publique d'un site client |
| [ping.eu](https://ping.eu) | Ping / traceroute / port check en ligne |
| [ssllabs.com/ssltest](https://ssllabs.com/ssltest) | Audit certificat SSL d'un site |

### Références M365 / Licences / Outils

| Site | Usage |
|---|---|
| [m365maps.com](https://m365maps.com) | Cartes interactives des licences M365 (Business, E3, Intune, Entra...) |
| [intuneqlinks.net](https://intuneqlinks.net) | Liens directs vers les paramètres Intune |
| [cmd.ms](https://cmd.ms) | Annuaire de tous les portails Microsoft cloud |
| [winget.run](https://winget.run) | Moteur de recherche IDs paquets winget |
| [Fluent UI Theme Designer](https://fluentuipr.z22.web.core.windows.net/heads/master/theming-designer/index.html) | Créer un thème SharePoint / Teams aux couleurs d'un client |

## À lire ensuite

- [dsregcmd — Diagnostic Entra / Intune](dsregcmd.md)
- [ipconfig — Toutes les options réseau](ipconfig.md)
- [winget — Guide MSP](winget.md)
- [Get-WindowsAutopilotInfo — Hardware hash](autopilot-hash.md)
- [RDS — Gestion des sessions](rds-sessions.md)
- [Graph API — OneDrive Shortcuts](graph-onedrive-shortcuts.md)
