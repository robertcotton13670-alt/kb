---
title: RDS — Gestion des sessions
description: Commandes RDS pour gérer les sessions utilisateurs, le drain serveur et la communication avec les utilisateurs connectés
---

# RDS — Gestion des sessions

En environnement RDS (Remote Desktop Services), la gestion des sessions utilisateurs depuis la ligne de commande est essentielle pour les interventions de maintenance, les redémarrages planifiés et le support terrain.

## Voir les sessions actives

```powershell linenums="1"
# Lister toutes les sessions (ID, utilisateur, état)
query session

# Lister uniquement les utilisateurs connectés
query user
```

Exemple de résultat `query session` :

```
 SESSIONNAME       USERNAME         ID  STATE   TYPE        DEVICE
 services                            0  Disc
 console           Administrateur    1  Conn    wdtshare
 rdp-tcp#2         jdupont           2  Active  rdpwd
 rdp-tcp#3         mmartin           3  Active  rdpwd
```

| Colonne | Description |
|---|---|
| ID | Identifiant de session — utilisé pour les commandes ciblées |
| STATE | `Active` = connecté, `Disc` = déconnecté mais session active en mémoire |

## Gérer une session

```powershell linenums="1"
# Déconnecter une session (l'utilisateur est déconnecté, session conservée en mémoire)
logoff <ID>

# Réinitialiser une session bloquée (fermeture forcée)
reset session <ID>

# Prendre la main sur une session (shadow)
shadow <ID> /control        # Avec contrôle clavier/souris
shadow <ID>                 # Visualisation uniquement
```

!!! warning "logoff vs reset session"
    `logoff` déconnecte proprement l'utilisateur mais conserve la session en mémoire (données non sauvegardées perdues). `reset session` ferme la session de force — à utiliser uniquement si la session est bloquée.

!!! tip "Shadow — consentement utilisateur"
    Sur Windows Server 2012 R2 et ultérieur, `shadow` peut nécessiter le consentement de l'utilisateur selon la configuration GPO. Vérifier la politique "Set rules for remote control of Remote Desktop Services user sessions".

## Envoyer un message aux utilisateurs

```powershell linenums="1"
# Envoyer un message à une session spécifique
msg <ID> "Votre message ici"

# Envoyer un message à tous les utilisateurs connectés
msg * "Le serveur redémarre dans 15 minutes. Veuillez sauvegarder votre travail."

# Envoyer à un utilisateur par nom
msg jdupont "Votre session sera fermée dans 5 minutes."
```

!!! tip "Prévenir avant une maintenance"
    Toujours envoyer un message aux utilisateurs avant un redémarrage serveur. Laisser au moins 10 à 15 minutes pour qu'ils puissent sauvegarder.

## Drain — Bloquer les nouvelles connexions

Le mode drain empêche de nouveaux utilisateurs de se connecter sans déconnecter ceux déjà en session. Indispensable avant une maintenance planifiée.

```powershell linenums="1"
# Activer le drain (bloquer nouvelles connexions)
change logon /drain

# Drain jusqu'au prochain redémarrage uniquement
change logon /drainuntilrestart

# Vérifier l'état actuel
change logon /query

# Réactiver les connexions après maintenance
change logon /enable
```

!!! warning "change logon /drain"
    Le drain ne déconnecte pas les utilisateurs déjà connectés. Il faut attendre qu'ils terminent leur session ou les déconnecter manuellement avec `logoff`.

## Procédure de maintenance complète

Workflow recommandé avant un redémarrage serveur RDS :

```powershell linenums="1"
# 1. Activer le drain
change logon /drain

# 2. Prévenir les utilisateurs
msg * "Maintenance planifiée dans 15 minutes. Veuillez sauvegarder et vous déconnecter."

# 3. Vérifier les sessions restantes (répéter jusqu'à ce que tout soit vide)
query user

# 4. Si des sessions persistent après délai, déconnecter
logoff 2
logoff 3

# 5. Vérifier qu'il ne reste plus de sessions actives
query session

# 6. Redémarrer le serveur
shutdown /r /t 60 /c "Redémarrage maintenance - dans 60 secondes"
```

## Tableau récapitulatif des commandes

| Commande | Description |
|---|---|
| `query session` | Lister toutes les sessions avec leur état et ID |
| `query user` | Lister les utilisateurs connectés |
| `logoff <ID>` | Déconnecter une session |
| `reset session <ID>` | Réinitialiser une session bloquée |
| `shadow <ID> /control` | Prendre la main sur une session |
| `msg <ID> "texte"` | Envoyer un message à une session |
| `msg * "texte"` | Envoyer un message à tous |
| `change logon /drain` | Bloquer les nouvelles connexions |
| `change logon /drainuntilrestart` | Drain jusqu'au prochain redémarrage |
| `change logon /enable` | Réactiver les connexions |
| `change logon /query` | Vérifier l'état drain actuel |

## À lire ensuite

- [Commandes & références MSP](index.md)
- [dsregcmd — Diagnostic Entra / Intune](dsregcmd.md)
