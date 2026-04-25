---
title: Graph API — OneDrive Shortcuts
description: Créer, lister et supprimer des raccourcis SharePoint dans OneDrive via Microsoft Graph API
---

# Graph API — OneDrive Shortcuts

La création de raccourcis SharePoint dans OneDrive via Graph API permet de déployer automatiquement les raccourcis "Ajouter à OneDrive" pour tous les membres d'un groupe, sans intervention manuelle. C'est la seule méthode automatisable en masse — il n'existe pas de politique Intune native pour cette opération.

## Concept

Un raccourci OneDrive vers une bibliothèque SharePoint est un `driveItem` de type `remoteItem` stocké dans le OneDrive personnel de l'utilisateur. Contrairement à la sync classique (`odopen://`), le raccourci apparaît dans `OneDrive - NomOrganisation` et est disponible sur tous les appareils de l'utilisateur.

| Méthode | Chemin local | Multi-device | Admin déployable |
|---|---|---|---|
| Sync classique (`odopen://`) | `C:\Users\user\NomOrg\...` | ❌ | ✅ via Intune |
| Raccourci OneDrive | `C:\Users\user\OneDrive - NomOrg\...` | ✅ | ✅ via Graph API |

## Prérequis — App Registration

Créer une App Registration dans Entra ID avec ces permissions Application (pas déléguées) :

| Permission | Usage |
|---|---|
| `Files.ReadWrite.All` | Créer / lire / supprimer les raccourcis dans les OneDrive |
| `GroupMember.Read.All` | Lire les membres d'un groupe Azure AD |

!!! warning "Permissions Application vs Déléguées"
    Les permissions Application permettent d'agir sur le OneDrive de n'importe quel utilisateur sans être connecté en tant que lui. Les permissions Déléguées nécessitent un contexte utilisateur interactif (Graph Explorer uniquement).

## Obtenir un token

```powershell linenums="1"
$tenantId     = "votre-tenant-id"
$clientId     = "votre-client-id"
$clientSecret = "votre-client-secret"

$body = @{
    grant_type    = "client_credentials"
    client_id     = $clientId
    client_secret = $clientSecret
    scope         = "https://graph.microsoft.com/.default"
}

$response = Invoke-RestMethod `
    -Uri "https://login.microsoftonline.com/$tenantId/oauth2/v2.0/token" `
    -Method POST `
    -Body $body `
    -ContentType "application/x-www-form-urlencoded"

$token = $response.access_token
```

## Endpoints principaux

### Créer un raccourci à la racine du OneDrive

```powershell linenums="1"
$headers = @{
    Authorization  = "Bearer $token"
    "Content-Type" = "application/json"
}

$body = @{
    name       = "Nom du raccourci"
    remoteItem = @{
        sharepointIds = @{
            siteId   = "votre-siteId"
            webId    = "votre-webId"
            listId   = "votre-listId"
            siteUrl  = "https://tenant.sharepoint.com/sites/MonSite"
        }
    }
    "@microsoft.graph.conflictBehavior" = "rename"
} | ConvertTo-Json -Depth 5

Invoke-RestMethod `
    -Uri "https://graph.microsoft.com/v1.0/users/$email/drive/root/children" `
    -Method POST `
    -Headers $headers `
    -Body $body
```

### Créer un raccourci dans un sous-dossier

```powershell linenums="1"
# Créer d'abord le dossier parent si absent
$folderBody = @{
    name   = "_SharePoint"
    folder = @{}
    "@microsoft.graph.conflictBehavior" = "rename"
} | ConvertTo-Json -Depth 3

Invoke-RestMethod `
    -Uri "https://graph.microsoft.com/v1.0/users/$email/drive/root/children" `
    -Method POST -Headers $headers -Body $folderBody

# Créer le raccourci dans le dossier
Invoke-RestMethod `
    -Uri "https://graph.microsoft.com/v1.0/users/$email/drive/root:/_SharePoint:/children" `
    -Method POST -Headers $headers -Body $body
```

### Lister les raccourcis d'un utilisateur

```powershell linenums="1"
# Lister tous les items racine du OneDrive
$items = Invoke-RestMethod `
    -Uri "https://graph.microsoft.com/v1.0/users/$email/drive/root/children" `
    -Headers $headers

# Filtrer sur les raccourcis (remoteItem présent)
$shortcuts = $items.value | Where-Object { $_.remoteItem }
$shortcuts | Select-Object name, id
```

### Supprimer un raccourci

```powershell linenums="1"
Invoke-RestMethod `
    -Uri "https://graph.microsoft.com/v1.0/users/$userId/drive/items/$itemId" `
    -Method DELETE `
    -Headers $headers
```

### Récupérer les membres d'un groupe

```powershell linenums="1"
$members = @()
$url = "https://graph.microsoft.com/v1.0/groups/$groupId/members/microsoft.graph.user?`$select=displayName,mail&`$top=999"

do {
    $response = Invoke-RestMethod -Uri $url -Headers $headers
    $members += $response.value | Where-Object { $_.mail }
    $url = $response.'@odata.nextLink'
} while ($url)
```

## Trouver les IDs SharePoint (siteId, webId, listId)

Dans SharePoint, cliquer sur "Ajouter à OneDrive" et récupérer le lien `odopen://` — il contient tous les IDs encodés en URL.

```powershell linenums="1"
# Décoder un lien odopen
$link = "odopen://sync/?..." # coller le lien complet ici
$decoded = [System.Uri]::UnescapeDataString($link)

# Parser les paramètres
$params = @{}
foreach ($part in $decoded -split '&') {
    $kv = $part -split '=', 2
    if ($kv.Count -eq 2) { $params[$kv[0].Trim()] = $kv[1].Trim() }
}

$params['siteId']
$params['webId']
$params['listId']
$params['webUrl']
```

## Script V3 — Déploiement pour un groupe complet

Le script complet `Shortcut-SharePoint-Groupe-V3.ps1` gère :

- Parsing automatique du lien odopen
- Création du dossier `_SharePoint` si absent
- Création du raccourci pour tous les membres du groupe
- Rapport succès / erreurs

```powershell linenums="1"
# Appel du script
.\Shortcut-SharePoint-Groupe-V3.ps1 `
    -TenantId     "votre-tenant-id" `
    -ClientId     "votre-client-id" `
    -ClientSecret "votre-client-secret"

# Le script demande ensuite interactivement :
# 1. Le lien odopen complet (depuis SharePoint > Ajouter à OneDrive)
# 2. L'Object ID du groupe Azure AD
```

!!! tip "Stocker les secrets dans Datto RMM"
    En déploiement MSP, stocker `TenantId`, `ClientId` et `ClientSecret` dans les variables de site Datto RMM — une variable par client. Ne jamais mettre les secrets en dur dans le script.

## Supprimer la sync Intune si les droits sont retirés

Pour la sync classique (pas les raccourcis), une politique Intune permet de supprimer automatiquement le contenu local quand les droits SharePoint sont retirés :

Dans Intune > Configuration > Settings Catalog > OneDrive :
`Hard delete the contents of the OneDrive folder when user loses permissions to a shared folder`

!!! warning "Raccourcis vs sync"
    Cette politique ne s'applique qu'à la sync classique (`odopen://`). Pour supprimer un raccourci Graph API quand les droits sont retirés, il faut un script PowerShell qui appelle `DELETE /users/{userId}/drive/items/{itemId}`.

## À lire ensuite

- [Commandes & références MSP](index.md)
- [winget — Guide MSP](winget.md)
