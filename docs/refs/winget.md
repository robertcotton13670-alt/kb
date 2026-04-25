---
title: winget
description: Guide winget pour MSP — déploiement, scripts PowerShell, contexte Intune et Datto RMM
---

# winget

`winget` est le gestionnaire de paquets natif Windows. En MSP, il est utilisé pour installer, mettre à jour et inventorier les applications, aussi bien manuellement sur un poste qu'automatiquement via Intune ou Datto RMM.

## Commandes essentielles

```powershell linenums="1"
# Recherche
winget search <nom>                     # Chercher une app
winget show <id>                        # Détails d'un paquet (version, source, etc.)

# Installation
winget install <id>                     # Installer une app
winget install <id> --silent --accept-package-agreements --accept-source-agreements

# Mises à jour
winget upgrade                          # Lister les apps avec mise à jour disponible
winget upgrade --all --silent --accept-package-agreements --accept-source-agreements

# Inventaire
winget list                             # Apps installées (reconnues par winget)

# Export / Import
winget export -o apps.json              # Exporter la liste des apps installées
winget import -i apps.json --accept-package-agreements  # Réimporter et installer
```

## Options utiles

| Option | Description |
|---|---|
| `--silent` / `-h` | Installation silencieuse, sans interface |
| `--accept-package-agreements` | Accepter les licences automatiquement |
| `--accept-source-agreements` | Accepter les conditions de la source |
| `--id <exact.id>` | Forcer l'ID exact pour éviter les ambiguïtés |
| `--source winget` | Forcer la source winget (évite le Microsoft Store) |
| `--scope machine` | Installer pour tous les utilisateurs |
| `--scope user` | Installer pour l'utilisateur courant uniquement |
| `--version <x.x.x>` | Installer une version spécifique |

!!! tip "Trouver l'ID exact d'une app"
    Utiliser [winget.run](https://winget.run) pour trouver l'ID exact avant de scripter. Exemple : `Mozilla.Firefox` et non `Firefox`.

## Exemples d'installation multi-apps

```powershell linenums="1"
# Installer Git, VS Code et Python en une commande
winget install Git.Git Microsoft.VisualStudioCode Python.Python.3.12 --silent --accept-package-agreements

# Installer une version spécifique
winget install Mozilla.Firefox --version 120.0

# Forcer scope machine (pour tous les utilisateurs)
winget install VideoLAN.VLC --scope machine --silent
```

## Contexte MSP — Droits admin

winget fonctionne partiellement sans droits admin :

| Action | Sans admin | Avec admin |
|---|---|---|
| `search`, `list`, `show` | ✅ | ✅ |
| Install scope `user` | ✅ | ✅ |
| Install scope `machine` | ❌ | ✅ |
| `upgrade --all` (apps machine) | ❌ | ✅ |

!!! warning "Sans droits admin"
    Sur un poste client standard, `winget upgrade --all` échouera partiellement pour les apps installées en scope machine. En contexte Intune ou Datto RMM, les scripts tournent sous SYSTEM — droits suffisants.

## Déploiement via Datto RMM

Sous SYSTEM (contexte Datto), `winget` n'est pas dans le PATH par défaut. Il faut résoudre le chemin dynamiquement :

```powershell linenums="1"
# Résoudre le chemin winget sous SYSTEM
$wingetPath = (Get-ChildItem -Path "C:\Program Files\WindowsApps\Microsoft.DesktopAppInstaller_*" -Recurse -Filter "winget.exe" -ErrorAction SilentlyContinue | Select-Object -Last 1).FullName

if (-not $wingetPath) {
    Write-Error "winget introuvable"
    exit 1
}

# Installer une app
& $wingetPath install Mozilla.Firefox --silent --accept-package-agreements --accept-source-agreements --scope machine
```

!!! warning "UAC sous Datto RMM"
    Si UAC se déclenche lors de l'installation, vérifier que le composant Datto est configuré en "Run As : System" et non "Logged-on User". Si winget n'est pas résolu correctement, Windows tente de le relancer en contexte utilisateur, ce qui déclenche UAC.

## Déploiement via Intune (Win32 App)

Structure d'un package Intune Win32 avec winget :

```
MonPackage/
├── install.ps1
└── detect.ps1
```

```powershell linenums="1"
# install.ps1
$wingetPath = (Get-ChildItem -Path "C:\Program Files\WindowsApps\Microsoft.DesktopAppInstaller_*" -Recurse -Filter "winget.exe" -ErrorAction SilentlyContinue | Select-Object -Last 1).FullName
& $wingetPath install Mozilla.Firefox --silent --accept-package-agreements --scope machine
```

```powershell linenums="1"
# detect.ps1 — détection basée sur la présence du dossier d'installation
if (Test-Path "C:\Program Files\Mozilla Firefox\firefox.exe") {
    Write-Output "Detected"
    exit 0
}
exit 1
```

Commande d'installation dans Intune :
```
powershell.exe -ExecutionPolicy Bypass -File install.ps1
```

!!! tip "Règle de détection"
    Ne jamais baser la détection sur winget lui-même — utiliser la présence du dossier d'installation ou d'une clé de registre. Winget peut ne pas être disponible dans le contexte de détection Intune.

## Générer un rapport des apps à mettre à jour

```powershell linenums="1"
# Lister les apps avec mise à jour disponible et exporter en CSV
winget upgrade | Out-File -FilePath "C:\tmp\winget-upgrades.txt" -Encoding UTF8
```

## À lire ensuite

- [Commandes & références MSP](index.md)
- [Get-WindowsAutopilotInfo — Hardware hash](autopilot-hash.md)
