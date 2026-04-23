---
title: Collecter le hardware hash Autopilot
description: Utiliser Get-WindowsAutopilotInfo pour exporter le hash matériel et l'enregistrer dans Intune — local, distant ou upload direct
---

# Collecter le hardware hash avec `Get-WindowsAutopilotInfo`

Le hardware hash est l'empreinte matérielle unique d'un device Windows. Pour Autopilot v1, il doit être importé dans Intune **avant** que le device passe par l'OOBE. `Get-WindowsAutopilotInfo` est le script PowerShell de référence pour cette collecte.

## Prérequis

### Modules et outils PowerShell

| Composant | Rôle | Obligatoire |
|---|---|---|
| `Get-WindowsAutopilotInfo` | Script de collecte du hash | Oui |
| `Microsoft.Graph.Authentication` | Authentification Graph (upload direct) | Upload direct uniquement |
| `WindowsAutopilotIntune` | Module complémentaire (méthodes v2 Graph) | Upload direct uniquement |
| PowerShell 5.1 ou 7+ | Environnement d'exécution | Oui |
| NuGet provider | Requis par `Install-Script` / `Install-Module` | Oui (auto-installé) |

### Droits requis

=== "Collecte locale (export CSV)"

    - **Administrateur local** sur le device cible (ou exécution depuis le compte `defaultuser0` pendant l'OOBE via Shift+F10)

=== "Collecte distante (WinRM)"

    - **Administrateur local** sur le device cible
    - WinRM activé sur le device cible (`Enable-PSRemoting`)
    - Pare-feu autorisant le port **5985** (HTTP) ou **5986** (HTTPS) vers la cible

=== "Upload direct vers Intune"

    - **Administrateur local** sur le device (ou droits WinRM pour la collecte distante)
    - Compte Entra avec le rôle **Intune Administrator** ou **Windows Autopilot deployment profile manager** (minimum)
    - Accès Internet vers `graph.microsoft.com`

!!! warning "Collecte depuis l'OOBE"
    Pendant l'OOBE, le compte `defaultuser0` dispose de droits limités. Ouvrir un terminal avec `Shift+F10` puis exécuter `PowerShell` en tant qu'administrateur. Aucun module ne pouvant être installé depuis le Store en OOBE, **préparer le script sur une clé USB ou le déposer en pré-boot via WinPE**.

---

## Installation du module

### PowerShell Gallery (méthode standard)

```powershell
# Installer le script depuis la PSGallery
Install-Script -Name Get-WindowsAutopilotInfo -Force -Scope AllUsers

# Vérifier la version installée
Get-InstalledScript -Name Get-WindowsAutopilotInfo | Select-Object Name, Version
```

### Avec les modules Graph (pour l'upload direct)

```powershell
# Installer les dépendances Graph
Install-Module -Name Microsoft.Graph.Authentication -Scope AllUsers -Force
Install-Module -Name WindowsAutopilotIntune -Scope AllUsers -Force

# Puis le script
Install-Script -Name Get-WindowsAutopilotInfo -Force -Scope AllUsers
```

!!! tip "Environnement sans accès Internet"
    Télécharger le script sur un poste connecté, puis le copier sur la cible :

    ```powershell
    Save-Script -Name Get-WindowsAutopilotInfo -Path C:\Temp\AutopilotScript
    ```

    Copier ensuite le dossier sur la machine isolée. Le script `.ps1` peut être exécuté directement sans installation dans la PSGallery.

---

## Commandes de collecte

### Collecte locale — export CSV

Cas d'usage : technicien sur place, device pas encore connecté au réseau d'entreprise.

```powershell
# Export dans un fichier CSV local
Get-WindowsAutopilotInfo -OutputFile C:\Temp\autopilot-hash.csv

# Export avec le numéro de série et le nom du device dans le nom de fichier
Get-WindowsAutopilotInfo -OutputFile "C:\Temp\$(hostname)-$(Get-Date -Format yyyyMMdd).csv"

# Ajouter un GroupTag (utile pour l'affectation de profil dynamique)
Get-WindowsAutopilotInfo -OutputFile C:\Temp\autopilot-hash.csv -GroupTag "SITE-PARIS"

# Ajouter un AssignedUser (pre-provisioning user-driven)
Get-WindowsAutopilotInfo -OutputFile C:\Temp\autopilot-hash.csv -AssignedUser "jean.dupont@contoso.com"
```

!!! info "Contenu du CSV"
    Le fichier généré contient quatre colonnes : `Device Serial Number`, `Windows Product ID`, `Hardware Hash`, `Group Tag` (vide si non renseigné). C'est le format accepté directement par le portail Intune pour l'import manuel.

### Collecte distante — WinRM

Cas d'usage : batch de devices accessibles sur le réseau, sans intervention physique.

```powershell
# Un seul device distant
Get-WindowsAutopilotInfo -ComputerName PC-FINANCE-01 -OutputFile C:\Temp\autopilot-hash.csv

# Plusieurs devices — liste depuis un fichier texte
$computers = Get-Content C:\Temp\liste-devices.txt
Get-WindowsAutopilotInfo -ComputerName $computers -OutputFile C:\Temp\autopilot-hash-batch.csv

# Avec des credentials explicites (compte admin local commun)
$cred = Get-Credential
Get-WindowsAutopilotInfo -ComputerName PC-FINANCE-01 -Credential $cred -OutputFile C:\Temp\autopilot-hash.csv
```

!!! warning "WinRM et pare-feu"
    Avant de lancer la collecte distante, s'assurer que WinRM est actif sur chaque cible. Sur un parc déjà géré par Intune ou GPO, cela peut être poussé centralement :

    ```powershell
    # Sur la machine cible (à exécuter en admin local)
    Enable-PSRemoting -Force
    Set-Item WSMan:\localhost\Client\TrustedHosts -Value "*" -Force
    ```

    Ne jamais laisser `TrustedHosts = *` en production. Restreindre à la plage IP de la console d'administration après la collecte.

### Upload direct vers Intune

Cas d'usage : atelier OEM ou MSP, device neuf, pas d'accès au portail Intune souhaité.

```powershell
# Connexion Graph + upload immédiat
Get-WindowsAutopilotInfo -Online

# Avec GroupTag
Get-WindowsAutopilotInfo -Online -GroupTag "SITE-LYON"

# Avec AssignedUser
Get-WindowsAutopilotInfo -Online -AssignedUser "marie.martin@contoso.com" -GroupTag "FINANCE"

# Depuis un device distant avec upload direct
Get-WindowsAutopilotInfo -ComputerName PC-NEUF-42 -Online -GroupTag "POSTE-STD"
```

!!! note "Flux d'authentification avec `-Online`"
    Le script ouvre une fenêtre de connexion Microsoft. Le compte doit avoir les droits Intune décrits dans les prérequis. L'upload passe par l'API Graph (`/deviceManagement/importedWindowsAutopilotDeviceIdentities`). Le device apparaît dans **Intune → Devices → Enrollment → Windows → Windows Autopilot devices** sous quelques minutes.

!!! danger "Délai de synchronisation"
    Après un upload via `-Online` ou via import CSV dans le portail, un **délai de 15 à 60 minutes** est normal avant que le profil Autopilot soit assigné au device. Ne pas lancer l'OOBE immédiatement après l'import. Attendre la confirmation dans le portail que le device affiche le bon **Deployment Profile Status = Assigned**.

---

## Import CSV manuel dans le portail Intune

Si l'upload direct n'est pas possible, importer le CSV via le portail :

1. **Intune admin center** → `Devices` → `Enrollment` → `Windows` → `Windows Autopilot devices`
2. `Import` → sélectionner le fichier CSV généré par `Get-WindowsAutopilotInfo`
3. Attendre que le statut passe à **Complete** (rafraîchir manuellement)
4. Assigner un profil Autopilot au device (directement ou via groupe dynamique sur le GroupTag)

!!! tip "Groupe dynamique sur GroupTag"
    Créer un groupe Entra dynamique avec la règle :

    ```
    (device.devicePhysicalIds -any (_ -eq "[OrderID]:SITE-PARIS"))
    ```

    Tout device importé avec `-GroupTag "SITE-PARIS"` intégrera automatiquement ce groupe, et le profil Autopilot assigné au groupe s'appliquera sans action manuelle.

---

## Tableau comparatif des méthodes

| Méthode | Commande clé | Résultat | Accès Internet | Droits requis | Idéal pour |
|---|---|---|---|---|---|
| **Local → CSV** | `-OutputFile` | Fichier `.csv` local | Non | Admin local | OOBE, device isolé, batch manuel |
| **Distant → CSV** | `-ComputerName` + `-OutputFile` | Fichier `.csv` centralisé | Non (réseau LAN) | Admin local + WinRM | Parc géré, collecte en masse avant expédition |
| **Local → Upload direct** | `-Online` | Import Graph immédiat | Oui | Admin local + rôle Intune | Atelier OEM, device neuf avec accès Internet |
| **Distant → Upload direct** | `-ComputerName` + `-Online` | Import Graph immédiat | Oui | Admin local + WinRM + rôle Intune | Console centrale MSP, batch avec upload auto |
| **Import CSV portail** | *(portail Intune)* | Import manuel | Oui (navigateur) | Rôle Intune | Consolidation de CSV issus de plusieurs sites |

---

## Vérification post-import

```powershell
# Via Graph (vérifier que le device est bien enregistré)
Connect-MgGraph -Scopes "DeviceManagementServiceConfig.Read.All"

Get-MgDeviceManagementImportedWindowsAutopilotDeviceIdentity | 
    Select-Object SerialNumber, State, AssignedUserPrincipalName |
    Sort-Object SerialNumber
```

Dans le portail, le device doit afficher :

- **Profile status** : `Assigned` (sinon vérifier l'assignation du profil au groupe)
- **Deployment profile** : le nom du profil attendu

!!! warning "Device déjà existant dans Intune"
    Si le device est déjà enrôlé dans Intune (via OOBE standard ou MDM enrollment), l'enregistrement Autopilot se crée mais le profil ne s'appliquera qu'au **prochain reset** (`Settings → Recovery → Reset this PC`). Un re-enrôlement sans reset ne prend pas en compte le profil Autopilot.

---

## Troubleshooting rapide

| Symptôme | Cause probable | Action |
|---|---|---|
| `Install-Script` échoue sans erreur claire | NuGet provider absent | `Install-PackageProvider -Name NuGet -Force` |
| Erreur `Access Denied` sur la cible distante | WinRM non activé ou droits insuffisants | `Enable-PSRemoting -Force` sur la cible |
| Upload `-Online` bloqué après auth | MFA ou Conditional Access bloquant | Utiliser un compte exempt de CA pour l'upload, ou exporter en CSV |
| Device importé mais profil non assigné | GroupTag ne correspond à aucun groupe dynamique | Vérifier la règle du groupe Entra et l'opérateur `[OrderID]:` |
| Délai > 1h après import | Sync Autopilot service lente | Forcer via **Intune → Devices → Windows Autopilot devices → Sync** |

---

## À lire ensuite

- [Windows Autopilot — vue d'ensemble](autopilot.md)
- [Configurer une politique Device Preparation pas à pas](autopilot-v2-setup.md) *(à venir)*
- [Troubleshooting ESP et codes d'erreur courants](autopilot-troubleshooting.md) *(à venir)*
