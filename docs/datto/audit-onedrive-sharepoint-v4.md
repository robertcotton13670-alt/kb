---
title: Audit OneDrive / SharePoint V4
description: Audit complet OneDrive Business et bibliothèques SharePoint synchronisées — multi-comptes, vérification processus, alerte version.
---

# Audit OneDrive / SharePoint V4

Script PowerShell exécuté en contexte **Utilisateur Connecté** via Datto RMM. Audite le ou les comptes OneDrive Business, le statut de synchronisation, le processus OneDrive, le Known Folder Move et les bibliothèques SharePoint synchronisées. Supporte deux comptes Business simultanés et alerte sur une version trop ancienne.

---

## Informations générales

| Paramètre   | Valeur                                          |
|-------------|-------------------------------------------------|
| Version     | 4                                               |
| Date        | Mai 2026                                        |
| Langage     | PowerShell 5.1+                                 |
| Plateforme  | Datto RMM                                       |
| OS requis   | Windows 10 / Server 2016 minimum                |
| Contexte    | **Utilisateur Connecté** (Datto RMM)            |
| Dépendance  | Aucune (lecture registre + système de fichiers) |

---

## Changements vs V3

| # | Quoi | Pourquoi |
|---|------|----------|
| 1 | **Pure ASCII** dans le `.ps1` | PS 5.1 peut mangler les caractères accentués selon la code page de l'agent |
| 2 | **Exit 2** pour contexte SYSTEM (au lieu de 3) | Code 2 = *Not Applicable* selon la convention Datto RMM ; code 3 = timeout |
| 3 | **Vérification du processus** `OneDrive.exe` | Détecte les cas où OneDrive est installé mais arrêté ou crashé |
| 4 | **Détection multi-comptes** (Business1 + Business2) | Sites avec deux comptes OneDrive for Business sur le même poste |
| 5 | **Variable `OD_MIN_VERSION`** | Alerte si la version installée est inférieure au seuil configuré |
| 6 | **Export dans `%TEMP%`** (au lieu de `C:\tmp`) | Chemin universel accessible par l'utilisateur connecté |
| 7 | **`Generic List`** pour les collections en pipeline | Évite le bug de scoping PS 5.1 avec `+=` dans `ForEach-Object` |

---

## Données collectées

| Donnée             | Description                                                                 |
|--------------------|-----------------------------------------------------------------------------|
| Version OneDrive   | Version de `OneDrive.exe` détectée (tous chemins per-user et per-machine)   |
| Processus          | `OneDrive.exe` en cours d'exécution ou non                                  |
| Compte(s) Business | Email + statut sync de Business1 et Business2 si présents                  |
| Bibliothèques SPO  | Nom + URL par compte (fichiers `ClientPolicy_*.ini`, fallback registre)     |
| Known Folder Move  | Desktop, Documents, Images — redirigés ou non vers OneDrive                 |

---

## Logique d'exécution

1. **Vérification utilisateur** — Si contexte SYSTEM ou compte machine : message + `exit 2`.
2. **Version et processus** — Détection `OneDrive.exe`, comparaison au seuil `OD_MIN_VERSION`.
3. **Comptes Business** — Boucle sur Business1 et Business2 (s'ils existent). Pour chacun : email, statut sync, bibliothèques SharePoint.
4. **KFM** — Lecture `User Shell Folders` pour les trois dossiers.
5. **Calcul du statut global** — `AVERTISSEMENT` si : processus arrêté, version sous seuil, ou erreur/conflit sync.
6. **Export** — Fichier `.txt` dans `%TEMP%` avec URLs complètes.

---

## Variables Datto RMM

| Variable          | Type    | Requis | Description                                              |
|-------------------|---------|--------|----------------------------------------------------------|
| `OD_MIN_VERSION`  | String  | Non    | Version minimale OneDrive (ex. `24.221.1110.0`). Si la version installée est inférieure, statut `AVERTISSEMENT`. |
| `CS_PROFILE_NAME` | Système | —      | Nom du site Datto RMM — affiché dans le closeout         |

---

## Sortie console

### Exécution normale — un compte, tout OK

```
============================================
  AUDIT ONEDRIVE / SHAREPOINT V4
============================================
  Utilisateur : DOMAIN\bob
  Site Datto  : Contoso - Montreal
  Date        : 10/05/2026 09:15
  Version OD  : 24.221.1110.0
  Processus   : En cours
--------------------------------------------
  [Business1] bob@contoso.com
  Sync        : A jour
  SharePoint  : RH - Documents
               IT - Partages
               Projet Alpha
--------------------------------------------
  KFM         : Desktop, Documents
               (Non redirige : Images)
--------------------------------------------
  Export      : C:\Users\bob\AppData\Local\Temp\audit_OneDrive_SharePoint_20260510_0915.txt
============================================
  Statut      : SUCCES
============================================
```

### Deux comptes Business détectés

```
  [Business1] bob@contoso.com
  Sync        : A jour
  SharePoint  : Documents partages
--------------------------------------------
  [Business2] bob@fabrikam.com
  Sync        : Conflit (1 fichier(s))
  SharePoint  : Aucune
--------------------------------------------
  Statut      : AVERTISSEMENT
  Detail      : [Business2] Conflit (1 fichier(s))
```

### Processus arrêté + version sous seuil

```
  Version OD  : 23.100.0508.0001 [ALERTE]
  Min Version : 24.221.1110.0
  Processus   : [ALERTE] Arrete
  ...
  Statut      : AVERTISSEMENT
  Detail      : Version OneDrive 23.100.0508.0001 inferieure au seuil 24.221.1110.0 | OneDrive.exe non detecte (arrete ou crash)
```

### Exécution hors contexte utilisateur

```
============================================
  AUDIT ONEDRIVE / SHAREPOINT V4
============================================
  Utilisateur : WORKGROUP\PCLAB01$
  Statut      : NON EXECUTE
  Raison      : Contexte SYSTEM detecte
                Relancer en mode [Utilisateur Connecte]
============================================
```

Exit code : `2` (Not Applicable).

---

## Statuts de synchronisation

| Valeur                                      | Source                           | Signification                                         |
|---------------------------------------------|----------------------------------|-------------------------------------------------------|
| `A jour`                                    | *(aucun problème détecté)*       | Synchronisation normale                               |
| `Erreur sync`                               | `SyncError` dans le registre     | Erreur signalée par le moteur OneDrive                |
| `Upload en attente (N fichier(s))`          | `NumberOfFilesToUpload > 0`      | Fichiers en attente d'envoi                           |
| `Conflit (N fichier(s))`                    | `NumberOfFilesWithConflict > 0`  | Fichiers en conflit de version                        |
| `Inconnu`                                   | Clé Business absente ou erreur   | OneDrive non configuré ou lecture registre impossible |

!!! warning "Pause manuelle non détectable"
    La pause OneDrive n'est pas exposée de façon fiable dans le registre. Une pause manuelle apparaît comme `A jour` dans le rapport.

---

## Détection contexte SYSTEM

| Cas                    | Condition détectée                              |
|------------------------|-------------------------------------------------|
| Processus SYSTEM       | `$env:USERNAME -eq "SYSTEM"`                    |
| Aucun utilisateur      | `$env:USERNAME` vide                            |
| Compte machine AD/AAD  | `$env:USERNAME` se termine par `$` (ex. `PC01$`) |

---

## Codes de sortie

| Code | Signification                                                        |
|------|----------------------------------------------------------------------|
| `0`  | Succès — données collectées et affichées                             |
| `1`  | Erreur générale inattendue (exception non gérée)                     |
| `2`  | Contexte SYSTEM ou compte machine — script ignoré proprement         |

---

## Export fichier

Fichier généré dans `%TEMP%` de l'utilisateur connecté :

```
audit_OneDrive_SharePoint_AAAAMMJJ_HHMM.txt
```

Contenu étendu : URLs SharePoint complètes, chemins KFM, tous les comptes détectés.

!!! warning "Échec d'écriture"
    En cas d'échec de l'export, le statut passe à `AVERTISSEMENT` et le détail précise l'erreur. L'audit console reste complet et le script continue (`exit 0`).

---

## Fichier requis dans le composant

| Fichier                            | Rôle            |
|------------------------------------|-----------------|
| `audit_onedrive_sharepoint_v4.ps1` | Script principal |

Aucune dépendance externe. Pas de téléchargement, pas de module PowerShell requis.

---

## À lire ensuite

- [Audit OneDrive / SharePoint V3](audit-onedrive-sharepoint-v3.md) *(version précédente)*
- [Raccourcis OneDrive SharePoint par groupe](../m365/onedrive-shortcuts-groupe.md)
