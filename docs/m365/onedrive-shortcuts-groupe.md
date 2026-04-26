---
title: Raccourcis OneDrive SharePoint par groupe
description: Créer automatiquement des raccourcis OneDrive vers une bibliothèque SharePoint pour tous les membres d'un groupe Azure AD.
---

# Raccourcis OneDrive SharePoint par groupe

Script PowerShell standalone qui crée un raccourci OneDrive vers une bibliothèque de documents SharePoint pour tous les membres d'un groupe Azure AD, via Microsoft Graph API.

Le raccourci apparaît à la racine du OneDrive de chaque utilisateur. OneDrive préfixe automatiquement le nom avec le nom du site (ex : `TEST - Documents`).

---

## Prérequis

### App Registration Azure AD

| Champ | Valeur |
|---|---|
| Nom | Datto-OneDrive-Shortcuts |
| TenantId | `417a0229-5d8c-4750-911c-a7551b91bc85` |
| ClientId | `27522e60-eae5-4884-8c98-946da6272dd0` |
| Permissions | `Files.ReadWrite.All`, `GroupMember.Read.All` (Application) |

Le ClientSecret est saisi interactivement à l'exécution, jamais stocké.

### Environnement

- Windows, PowerShell 5.1+
- Pas de module tiers requis (Invoke-RestMethod natif)
- Accès internet vers `login.microsoftonline.com` et `graph.microsoft.com`

---

## Informations à préparer avant l'exécution

### 1. Lien odopen de la bibliothèque SharePoint

C'est le lien généré par SharePoint quand on clique sur "Ajouter un raccourci à OneDrive".

Comment l'obtenir :

1. Ouvrir la bibliothèque de documents SharePoint cible dans un navigateur
2. Cliquer sur `Ajouter un raccourci à OneDrive` (barre de commandes en haut)
3. Ne pas valider la fenêtre qui s'ouvre
4. Copier l'URL complète depuis la barre d'adresse du navigateur — elle commence par `odopen://` ou contient les paramètres `siteId=`, `webId=`, `listId=`

!!! tip "Astuce"
    Sur certains navigateurs, le lien odopen ne s'affiche pas dans la barre d'adresse. Dans ce cas, faire un clic droit sur le bouton "Ajouter un raccourci à OneDrive" → Copier l'adresse du lien.

!!! warning "Attention"
    Le lien doit contenir les paramètres `siteId`, `webId`, `listId` et `webUrl`. Le script vérifie leur présence et affiche une erreur si l'un d'eux est manquant.

### 2. Object ID du groupe Azure AD

1. Ouvrir [Entra ID](https://entra.microsoft.com) → Groupes
2. Rechercher le groupe cible
3. Copier la valeur du champ `Object ID`

---

## Exécution

```powershell linenums="1"
.\Shortcut-SharePoint-Groupe-V4.ps1 `
  -TenantId  "417a0229-5d8c-4750-911c-a7551b91bc85" `
  -ClientId  "27522e60-eae5-4884-8c98-946da6272dd0" `
  -ClientSecret "TON_SECRET"
```

Le script demande ensuite deux saisies interactives :

```
Colle le lien SharePoint complet (depuis 'Ajouter un raccourci a OneDrive') :
> odopen://...

Colle l'Object ID du groupe Azure AD :
> xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

---

## Comportement attendu

### Logs console

```
============================================
  Date        : 26/04/2026 10:00
  Site SPO    : https://contoso.sharepoint.com/sites/monsite
  Raccourci   : Documents
  Groupe      : xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
============================================
  Token       : OK
  Membres     : 5 trouvés
--------------------------------------------
  [OK]     Jean Dupont (jdupont@contoso.fr) → TEST - Documents
  [EXISTE] Marie Martin (mmartin@contoso.fr)
  [ERREUR] Paul Durand (pdurand@contoso.fr) → ...message d'erreur...
--------------------------------------------
  Créés       : 1
  Existants   : 1
  Erreurs     : 1
  Total       : 3
============================================
```

### Statuts par utilisateur

| Statut | Signification |
|---|---|
| `[OK]` | Raccourci créé avec succès |
| `[EXISTE]` | Raccourci déjà présent à la racine du OneDrive — aucune action |
| `[ERREUR]` | Échec de création — message d'erreur Graph affiché |

!!! tip "Idempotent"
    Le script peut être relancé sans risque sur le même groupe. Les utilisateurs qui ont déjà le raccourci reçoivent simplement un `[EXISTE]`.

---

## Points d'attention

!!! warning "Nom du raccourci"
    OneDrive ajoute automatiquement le nom du site en préfixe. Le raccourci demandé s'appelle `Documents` mais s'affichera `NomDuSite - Documents` dans l'explorateur. Ce comportement est imposé par OneDrive et ne peut pas être supprimé via Graph API.

!!! warning "Raccourcis déplacés par l'utilisateur"
    La détection d'existence vérifie uniquement la racine du OneDrive. Si un utilisateur a déplacé le raccourci dans un sous-dossier, le script ne le détecte pas et en crée un nouveau.

!!! warning "Utilisateurs sans OneDrive provisionné"
    Si le OneDrive d'un utilisateur n'a jamais été ouvert, la création du raccourci peut échouer. Il faut que le OneDrive soit initialisé au préalable.

---

## À lire ensuite

- [Audit des raccourcis manquants](audit-shortcuts-manquants.md) *(à venir)*
- [App Registration Graph API](../azure/app-registration-graph.md) *(à venir)*
