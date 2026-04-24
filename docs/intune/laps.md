---
title: LAPS via Intune
description: Activer et configurer Windows LAPS pour les postes gérés par Intune.
---

# LAPS via Intune

Windows LAPS (Local Administrator Password Solution) permet de gérer automatiquement le mot de passe du compte administrateur local de chaque poste. Intune prend en charge le déploiement natif de LAPS via une stratégie de gestion des comptes, sans dépendance on-premises.

## Prérequis

| Condition | Détail |
|---|---|
| Licence | Intune P1 (inclus dans M365 Business Premium / EMS E3+) |
| OS | Windows 11 22H2+ ou Windows 10 22H2+ avec KB5025221 |
| Join type | Entra Joined ou Hybrid Entra Joined |
| Rôle admin | Intune Administrator ou Global Administrator |

!!! info
    LAPS est intégré à Windows depuis la build 25145. Aucun agent MSI à déployer pour les postes à jour.

## Activer LAPS dans Intune

### 1. Vérifier que LAPS est activé au niveau du tenant

1. Ouvrir [Intune admin center](https://intune.microsoft.com) → **Tenant administration** → **Windows LAPS**.
2. Vérifier que l'état affiche **Enabled**. Si non, activer.

### 2. Créer une stratégie LAPS

1. Aller dans **Endpoint security** → **Account protection** → **+ Create policy**.
2. Platform : **Windows 10 and later** — Profile : **Local admin password solution (Windows LAPS)**.
3. Configurer les paramètres clés :

| Paramètre | Valeur recommandée |
|---|---|
| Backup directory | Azure Active Directory |
| Password age (days) | 30 |
| Password length | 20 |
| Password complexity | Large letters + small letters + numbers + special characters |
| Post-authentication actions | Reset password + Log off |
| Post-authentication reset delay (hours) | 24 |

4. Assigner la stratégie au groupe de machines cible.

!!! warning
    Ne pas cibler **All Devices** sans test préalable. Un mauvais paramètre de compte cible peut verrouiller l'accès local.

### 3. Vérifier la configuration sur un poste

Une fois la stratégie appliquée (synchronisation Intune ~15 min) :

```powershell linenums="1"
# Vérifier l'état LAPS local
Get-LapsAADPassword -DeviceIds (Get-WmiObject Win32_ComputerSystem).Name
```

Ou depuis le portail Intune : **Devices** → sélectionner le poste → **Local admin password**.

## Récupérer un mot de passe LAPS

### Via Intune admin center

1. **Devices** → rechercher le poste → onglet **Local admin password**.
2. Cliquer **Show local administrator password** (action journalisée dans les logs d'audit).

### Via PowerShell (Microsoft Graph)

```powershell linenums="1"
# Nécessite le module Microsoft.Graph
Connect-MgGraph -Scopes "DeviceLocalCredential.Read.All"

$deviceId = "<Azure AD Device ID>"
Get-MgDeviceLocalCredential -DeviceId $deviceId | Select-Object -ExpandProperty Credentials
```

!!! danger
    L'accès au mot de passe LAPS est journalisé. Toute consultation est traçable dans les logs d'audit Entra ID. Ne pas partager les mots de passe hors canal sécurisé.

## Rotation manuelle du mot de passe

Depuis le portail Intune → poste → **Local admin password** → **Rotate**.

Ou via PowerShell :

```powershell linenums="1"
# Forcer la rotation immédiate
Invoke-LapsRotation
```

!!! note
    La rotation peut prendre jusqu'à 15 minutes pour se refléter dans le portail.

## Dépannage

| Symptôme | Cause probable | Action |
|---|---|---|
| Onglet "Local admin password" absent | OS non supporté ou LAPS non activé | Vérifier build Windows + activation tenant |
| Mot de passe non disponible | Stratégie non appliquée | Vérifier sync Intune, logs `%SystemRoot%\debug\netlogon.log` |
| Erreur `BackupDirectory not configured` | Stratégie sans assignation | Contrôler l'assignation de groupe |

## À lire ensuite

- [Autopilot](autopilot)
- [Conditional Access](../entra/conditional-access) *(à venir)*
