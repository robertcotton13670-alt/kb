# Page d'exemple — test des fonctionnalités

Cette page sert à vérifier que toutes les fonctionnalités de rendu Markdown fonctionnent correctement.

## Texte et mise en forme

Du texte avec *italique*, **gras**, et `code inline`. On peut aussi faire des [liens externes](https://learn.microsoft.com).

## Listes à cocher

- [x] Créer le repo GitHub
- [x] Configurer MkDocs
- [x] Déployer via GitHub Actions
- [ ] Ajouter du vrai contenu
- [ ] Connecter un domaine custom

## Bloc de code PowerShell

```powershell
# Exemple : connexion à Microsoft Graph
Connect-MgGraph -Scopes "User.Read.All", "Group.Read.All"

Get-MgUser -Filter "accountEnabled eq true" -Top 10 |
    Select-Object DisplayName, UserPrincipalName, Mail
```

## Bloc de code Bash

```bash
# Installer MkDocs Material en local
pip install mkdocs-material
mkdocs serve
```

## Tableau

| Outil | Usage MSP | Licence requise |
|-------|-----------|-----------------|
| Intune | Gestion de postes | Business Premium |
| Datto RMM | Monitoring, patch | Abonnement Datto |
| CIPP | Multi-tenant M365 | Open source |

## Encadrés (admonitions)

!!! tip "Astuce"
    Les admonitions sont parfaites pour mettre en valeur une information clé dans un runbook.

!!! warning "Attention"
    À utiliser avec précaution sur un tenant de production.

!!! danger "Irréversible"
    Cette commande supprime définitivement les données.

## Diagramme Mermaid

```mermaid
flowchart LR
    A[Nouveau poste] --> B{Autopilot ?}
    B -->|Oui| C[Enrollment Intune]
    B -->|Non| D[Enrollment manuel]
    C --> E[ESP]
    D --> E
    E --> F[Poste prêt]
```
