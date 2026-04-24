---
title: Winget — Déploiement d'applications vers Intune
description: Workflow MSP complet pour packager et publier des apps Win32 dans Intune via Winget et Wintuner
---

# Winget — Déploiement d'applications vers Intune

Wintuner (`svrooij/winget-intune-cli`) est un outil .NET CLI qui automatise deux étapes en une : packaging d'un installeur depuis Winget en `.intunewin`, puis publication directe dans Intune via Microsoft Graph.

En MSP, l'avantage clé est le workflow **package once, publish many** : un package généré une fois peut être publié sur N tenants clients sans repackager.

---

## Prérequis

| Prérequis | Valeur |
|---|---|
| OS | Windows 10/11 avec Winget disponible |
| .NET SDK | Version 8 minimum |
| Droits Intune | `DeviceManagementApps.ReadWrite.All` via Graph |
| Connexion internet | Requise (téléchargement installeurs + publication) |

---

## Installation

```powershell linenums="1"
# 1. Installer .NET 8 SDK
winget install microsoft.dotnet.sdk.8 --source winget

# 2. Ajouter la source NuGet
dotnet nuget add source https://api.nuget.org/v3/index.json --name nuget.org

# 3. Installer Wintuner
dotnet tool install --global svrooij.winget-intune.cli

# 4. Vérifier (fermer/rouvrir PowerShell avant)
wintuner --version
```

!!! warning "Nom de commande"
    Depuis la v1.1.0, la commande est `wintuner` (et non `winget-intune` comme dans les anciennes vidéos). L'exécutable est dans `%USERPROFILE%\.dotnet\tools\`.

---

## Workflow MSP

### 1. Trouver l'ID Winget

```powershell
winget search <nom_application>
# Exemple : VideoLAN.VLC, 7zip.7zip, Google.Chrome.EXE
```

### 2. Vérifier le scope d'installation

```powershell
winget show <ID>
# Chercher le champ "Scope"
```

| Type installeur | Scope typique | Compatible Intune SYSTEM ? |
|---|---|---|
| MSI | Machine | ✅ Oui |
| NSIS (.exe) | Machine | ✅ Oui |
| MSIX / AppX | User | ⚠️ Géré différemment |
| EXE user-scope | User | ⚠️ Contexte User requis |

!!! tip "MSP"
    Toujours viser `machine scope` pour les déploiements Intune. L'installation en contexte SYSTEM bypass UAC — aucune interaction utilisateur.

Si le champ `Scope` est absent dans `winget show`, consulter le manifeste sur GitHub (`microsoft/winget-pkgs`) dans le fichier `.installer.yaml`.

### 3. Packager

```powershell
mkdir C:\winget-packages
wintuner package <WingetID> --package-folder C:\winget-packages
```

Wintuner génère automatiquement dans le sous-dossier `<ID>\<version>` :

- `*.intunewin` — package Intune
- `app.json` — métadonnées complètes
- `win32LobApp.json` — payload Microsoft Graph
- `detection.txt` — règle de détection (MSI product code)
- `readme.txt` — commandes install/uninstall

!!! tip
    Wintuner sélectionne automatiquement le meilleur installeur disponible (préférence MSI > EXE). Vérifier le résultat dans le dossier généré.

### 4. Publier dans Intune

```powershell linenums="1"
# Tenant courant (authentification interactive)
wintuner publish <WingetID> --package-folder C:\winget-packages --username <UPN>

# Tenant spécifique (multi-client MSP)
wintuner publish <WingetID> --package-folder C:\winget-packages `
  --tenant <TenantID> --username <UPN>
```

!!! danger "Piège WAM/broker Windows"
    Toujours ajouter `--username <UPN>` lors du `publish`. Sans ce paramètre, l'authentification échoue avec `authentication_canceled` (problème de broker WAM Windows).

    Exemple : `--tenant crecas84.onmicrosoft.com --username admin@crecas84.onmicrosoft.com`

Le Tenant ID se trouve dans Entra ID > Vue d'ensemble.

---

## Exemple complet — VLC Media Player

```powershell linenums="1"
winget search VLC
winget show VideoLAN.VLC
wintuner package VideoLAN.VLC --package-folder C:\winget-packages
wintuner publish VideoLAN.VLC --package-folder C:\winget-packages `
  --tenant crecas84.onmicrosoft.com --username admin@crecas84.onmicrosoft.com
```

---

## Workflow multi-tenant MSP

```powershell linenums="1"
# Packager une seule fois
wintuner package VideoLAN.VLC --package-folder C:\winget-packages

# Publier sur chaque tenant client
wintuner publish VideoLAN.VLC --package-folder C:\winget-packages `
  --tenant <TenantID-Client1> --username admin@client1.onmicrosoft.com

wintuner publish VideoLAN.VLC --package-folder C:\winget-packages `
  --tenant <TenantID-Client2> --username admin@client2.onmicrosoft.com
```

!!! tip "MSP"
    Stocker les packages dans IT Glue ou un partage réseau MSP. Nommer les dossiers avec la version pour la traçabilité (ex : `C:\winget-packages\VideoLAN.VLC\3.0.23`).

---

## Configuration dans Intune après publication

Vérifier dans Intune > Applications > Windows :

| Paramètre | Valeur recommandée |
|---|---|
| Contexte d'installation | System (défaut Wintuner) |
| Comportement d'installation | Install silently |
| Redémarrage requis | Basé sur codes de retour |
| Attribution | Groupe d'appareils (Available ou Required) |

!!! warning "Règle de détection"
    La règle auto-générée par Wintuner utilise le MSI product code et la version. Peut poser problème si la version change sans mise à jour du package — vérifier avant assignation en production.

---

## Pièges courants

| Piège | Solution |
|---|---|
| `wintuner` introuvable après install | Fermer/rouvrir PowerShell — PATH non rechargé |
| Erreur `authentication_canceled` | Ajouter `--username <UPN>` à la commande `publish` |
| Erreur NU1202 | Installer .NET 8 SDK (pas .NET 7) |
| App user-scope publiée en System | Vérifier `winget show` avant de packager |
| Règle de détection bloquante sur MAJ | Désactiver la vérification de version dans la règle si nécessaire |
| Tenant ID incorrect | Le récupérer dans Entra ID > Vue d'ensemble, pas dans Intune |
| Nom de commande incorrect | `wintuner` (pas `winget-intune`) depuis v1.1.0 |

---

## Références

- GitHub : [svrooij/winget-intune-cli](https://github.com/svrooij/winget-intune-cli)
- NuGet : `svrooij.winget-intune.cli`
- Winget packages : [microsoft/winget-pkgs](https://github.com/microsoft/winget-pkgs)

## À lire ensuite

- [Déploiement d'applications Win32 dans Intune](win32-intune.md) *(à venir)*
- [Workflow Autopilot — Enrôlement Windows](autopilot-windows.md) *(à venir)*
- [Composant Datto RMM — Upload Hardware Hash](datto-autopilot-hash.md) *(à venir)*
