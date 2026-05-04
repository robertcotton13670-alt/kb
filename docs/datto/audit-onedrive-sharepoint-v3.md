---
title: Audit OneDrive / SharePoint V3
description: Audit de la configuration OneDrive Business et des bibliothèques SharePoint synchronisées via Datto RMM.
---

# Audit OneDrive / SharePoint V3

Script PowerShell exécuté en contexte **Utilisateur Connecté** via Datto RMM. Il audite le compte OneDrive Business, le statut de synchronisation, le Known Folder Move et les bibliothèques SharePoint synchronisées sur le poste.

---

## Informations générales

| Paramètre   | Valeur                                          |
|-------------|-------------------------------------------------|
| Version     | 3                                               |
| Date        | Mai 2026                                        |
| Langage     | PowerShell 5.1+                                 |
| Plateforme  | Datto RMM                                       |
| OS requis   | Windows 10 / Server 2016 minimum                |
| Contexte    | **Utilisateur Connecté** (Datto RMM)            |
| Dépendance  | Aucune (lecture registre + système de fichiers) |

---

## Données collectées

| Donnée            | Description                                                                    |
|-------------------|--------------------------------------------------------------------------------|
| Compte OneDrive   | Email du compte Business1 lu dans le registre `HKCU`                          |
| Version OneDrive  | Version de `OneDrive.exe` détectée sur le poste                                |
| Statut sync       | Un ou plusieurs états simultanés — voir tableau ci-dessous                     |
| Known Folder Move | Desktop, Documents, Images — redirigés ou non vers OneDrive                   |
| Bibliothèques SPO | Nom + URL des bibliothèques SharePoint synchronisées                           |

---

## Logique d'exécution

Le script suit une logique linéaire en 4 étapes :

1. **Vérification utilisateur** — Détecte si le script tourne en contexte SYSTEM ou compte machine (`USERNAME$`). Si oui : affiche un message d'avertissement explicite et `exit 3` propre. Sinon, continue normalement.
2. **Collecte OneDrive** — Lecture `HKCU:\Software\Microsoft\OneDrive\Accounts\Business1` pour l'email, le statut sync multi-état, et détection de la version via `OneDrive.exe`.
3. **KFM et SharePoint** — Lecture `User Shell Folders` pour KFM, scan `Tenants` + fallback récursif pour les bibliothèques synchronisées.
4. **Export et closeout** — Génération du fichier `.txt` dans `C:\tmp` avec URLs complètes, puis affichage du bloc closeout structuré.

Pas de tâche planifiée, pas de dépendance externe.

---

## Variables Datto RMM

Ce composant ne définit pas de variables d'entrée. Il utilise uniquement la variable système standard :

| Variable          | Type    | Description                                      |
|-------------------|---------|--------------------------------------------------|
| `CS_PROFILE_NAME` | Système | Nom du site Datto RMM — affiché dans le closeout |

---

## Sortie console

### Exécution normale

```
============================================
  AUDIT ONEDRIVE / SHAREPOINT V3
============================================
  Utilisateur : DOMAIN\bob
  Site Datto  : Nom du site
  Date        : 08/03/2026 14:30
  OneDrive    : bob@contoso.com
  Version OD  : 24.201.1003.0005
  Sync        : A jour
--------------------------------------------
  KFM         : Desktop, Documents
               (Non redirige : Images)
--------------------------------------------
  SharePoint  : Projet Alpha
               RH - Documents
               IT - Partages
--------------------------------------------
  Export      : C:\tmp\audit_OneDrive_SharePoint_20260308_1430.txt
============================================
  Statut      : SUCCES
============================================
```

### Plusieurs statuts sync simultanés

La ligne `Sync` peut afficher plusieurs états sur des lignes distinctes si plusieurs problèmes coexistent :

```
  Sync        : Erreur de synchronisation
               Conflit (2 fichier(s))
               Upload en cours (5 fichier(s) en attente)
```

Dans le fichier export, les états sont joints sur une seule ligne séparés par ` / `.

### Exécution hors contexte utilisateur

Si le composant est lancé en mode **System** au lieu de **Utilisateur Connecté** :

```
============================================
  AUDIT ONEDRIVE / SHAREPOINT V3
============================================
  Utilisateur : WORKGROUP\PCLAB01$
  Statut      : NON EXECUTE
  Raison      : Contexte SYSTEM detecte - relancer en mode
                [Utilisateur Connecte] dans Datto RMM
============================================
```

Le nom du compte machine est affiché pour identifier le poste concerné. Code de sortie : `3`.

---

## Valeurs possibles — Statut sync

Plusieurs valeurs peuvent être présentes simultanément :

| Valeur                                      | Source                           | Signification                                           |
|---------------------------------------------|----------------------------------|---------------------------------------------------------|
| `A jour`                                    | *(aucun problème détecté)*       | Synchronisation normale, aucun problème                 |
| `Erreur de synchronisation`                 | `SyncError` dans le registre     | Erreur signalée par le moteur de sync OneDrive          |
| `Synchronisation en cours`                  | ODSyncUtil — `CurrentState = 1`  | Sync active (ODSyncUtil requis)                         |
| `En pause`                                  | ODSyncUtil — `CurrentState = 2`  | Pause manuelle (ODSyncUtil requis — voir note)          |
| `Hors ligne`                                | ODSyncUtil — `CurrentState = 4`  | Poste hors ligne (ODSyncUtil requis)                    |
| `Upload en cours (N fichier(s) en attente)` | `NumberOfFilesToUpload > 0`      | Fichiers en attente d'envoi vers OneDrive               |
| `Conflit (N fichier(s))`                    | `NumberOfFilesWithConflict > 0`  | Fichiers en conflit de version                          |
| `Inconnu`                                   | Clé Business1 absente ou erreur  | OneDrive non configuré ou lecture registre impossible   |

!!! warning "Pause manuelle — détection partielle"
    La pause n'est pas exposée dans le registre (`Paused` n'existe pas, `global.ini` non fiable). Le script utilise **ODSyncUtil** en best-effort pour la détecter. Cependant, ODSyncUtil requiert `OneDriveFlyoutPS.dll` — ce fichier est **absent sur les installations OneDrive per-machine** (Windows 11, `Program Files`). Sur ces postes, une pause manuelle s'affiche comme `A jour`.

!!! note "Popup « Remplacer le dossier par votre raccourci »"
    Ce type de notification OneDrive (apparaît lors de l'ajout d'un raccourci SharePoint via Graph API) n'est **pas stocké dans le registre** — OneDrive le gère dans sa base interne. Il n'est donc pas détectable par ce script.

---

## Détection contexte SYSTEM

OneDrive stocke sa configuration dans `HKCU`, qui pointe vers le profil **SYSTEM** (et non celui de l'utilisateur connecté) quand le composant est mal configuré dans Datto RMM.

Le script détecte trois cas d'exécution non-utilisateur :

| Cas                   | Détection                                                      |
|-----------------------|----------------------------------------------------------------|
| Processus SYSTEM      | `$env:USERNAME -eq "SYSTEM"`                                   |
| Aucun utilisateur     | `$env:USERNAME` vide                                           |
| Compte machine AD/AAD | `$env:USERNAME` se termine par `$` (ex. `PCLAB01$`)           |

Dans ces cas, le script affiche le nom du compte détecté et sort proprement avec `exit 3` sans tenter de lire OneDrive.

---

## Export fichier

Un fichier texte est généré à chaque exécution dans `C:\tmp` :

```
audit_OneDrive_SharePoint_AAAAMMJJ_HHMM.txt
```

Le fichier contient le détail complet incluant les URLs SharePoint (`WebUrl`) et les chemins `MountPoint`, non affichés en console pour des raisons de lisibilité.

!!! tip "Création automatique"
    Si `C:\tmp` n'existe pas, il est créé automatiquement.

!!! warning "Échec d'écriture"
    En cas d'échec, le statut passe à `AVERTISSEMENT` et le script continue (`exit 0`). L'audit console reste complet.

---

## Codes de sortie

| Code | Signification                                                        |
|------|----------------------------------------------------------------------|
| `0`  | Succès — données collectées et affichées                             |
| `1`  | Erreur générale inattendue (exception non gérée)                     |
| `3`  | Contexte SYSTEM ou compte machine détecté — script ignoré proprement |

---

## Fichiers requis dans le composant

| Fichier                            | Rôle                                                    |
|------------------------------------|---------------------------------------------------------|
| `audit_onedrive_sharepoint_v3.ps1` | Script principal                                        |
| `ODSyncUtil.exe`                   | Optionnel — détection pause/hors ligne via OneDrive COM |
| `ODSyncLib.dll`                    | Requis si ODSyncUtil.exe présent (dépendance directe)   |

!!! tip "ODSyncUtil"
    ODSyncUtil (MIT, Rodney Viana) est uploadé dans le composant Datto en best-effort. S'il ne peut pas lire le statut OneDrive (ex. `OneDriveFlyoutPS.dll` absent), le script bascule automatiquement sur la lecture registre.

---

## À lire ensuite

- [Raccourcis OneDrive SharePoint par groupe](../m365/onedrive-shortcuts-groupe.md)
- [App Registration Graph API](../azure/app-registration-graph.md) *(à venir)*
