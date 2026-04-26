---
title: Audit OneDrive / SharePoint V3
description: Audit de la configuration OneDrive Business et des bibliothèques SharePoint synchronisées via Datto RMM.
---

# Audit OneDrive / SharePoint V3

Script PowerShell exécuté en contexte Utilisateur Connecté via Datto RMM. Il audite le compte OneDrive Business, le statut de synchronisation, le Known Folder Move et les bibliothèques SharePoint synchronisées sur le poste.

---

## Informations générales

| Paramètre   | Valeur                                          |
|-------------|-------------------------------------------------|
| Version     | 3                                               |
| Date        | Mars 2026                                       |
| Langage     | PowerShell 5.1+                                 |
| Plateforme  | Datto RMM                                       |
| OS requis   | Windows 10 / Server 2016 minimum                |
| Contexte    | Utilisateur Connecté (Datto RMM)                |
| Dépendance  | Aucune (lecture registre + système de fichiers) |

---

## Données collectées

| Donnée            | Description                                                     |
|-------------------|-----------------------------------------------------------------|
| Compte OneDrive   | Email du compte Business1 lu dans le registre HKCU             |
| Version OneDrive  | Version de `OneDrive.exe` détectée sur le poste                |
| Statut sync       | A jour / En cours / En pause / Erreur / Inconnu                |
| Known Folder Move | Desktop, Documents, Images — redirigés ou non vers OneDrive    |
| Bibliothèques SPO | Nom + URL des bibliothèques SharePoint synchronisées           |

---

## Logique d'exécution

Le script suit une logique linéaire en 4 étapes :

1. Vérification utilisateur — `exit 3` propre si aucun utilisateur connecté ou contexte SYSTEM.
2. Collecte OneDrive — lecture `HKCU:\Software\Microsoft\OneDrive\Accounts\Business1` pour l'email, le statut sync, et détection de la version via `OneDrive.exe`.
3. KFM et SharePoint — lecture `User Shell Folders` pour KFM, scan `Tenants` + fallback récursif pour les bibliothèques synchronisées.
4. Export et closeout — génération du fichier `.txt` dans `C:\tmp` avec URLs complètes, puis affichage du bloc closeout structuré.

Pas de tâche planifiée, pas de dépendance externe.

---

## Variables Datto RMM

Ce composant ne définit pas de variables d'entrée. Il utilise uniquement la variable système standard :

| Variable          | Type    | Description                                      |
|-------------------|---------|--------------------------------------------------|
| `CS_PROFILE_NAME` | Système | Nom du site Datto RMM — affiché dans le closeout |

---

## Sortie console

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

### Valeurs possibles — Sync

| Valeur     | Signification                                          |
|------------|--------------------------------------------------------|
| `A jour`   | Aucun fichier en attente, pas d'erreur ni de pause     |
| `En cours` | Fichiers en attente d'upload (nombre affiché)          |
| `En pause` | Synchronisation mise en pause par l'utilisateur        |
| `Erreur`   | Clé `SyncError` présente dans le registre              |
| `Inconnu`  | Clé Business1 introuvable ou lecture impossible        |

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

| Code | Signification                                         |
|------|-------------------------------------------------------|
| `0`  | Succès — données collectées et affichées              |
| `1`  | Erreur générale inattendue (exception non gérée)      |
| `3`  | Aucun utilisateur connecté — script ignoré proprement |

---

## Fichiers requis dans le composant

| Fichier                            | Rôle             |
|------------------------------------|------------------|
| `audit_onedrive_sharepoint_v3.ps1` | Script principal |

---

## À lire ensuite

- [Raccourcis OneDrive SharePoint par groupe](../m365/onedrive-shortcuts-groupe.md)
- [App Registration Graph API](../azure/app-registration-graph.md) *(à venir)*
