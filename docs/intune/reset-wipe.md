---
title: Intune — Reset et Wipe
description: Modes de réinitialisation Intune, différences et cas d'usage MSP
---

# Intune — Reset et Wipe

Intune propose plusieurs actions pour retirer ou réinitialiser un appareil. Choisir la mauvaise action peut laisser des données accessibles ou bloquer l'accès au poste. Ce guide couvre les 6 actions disponibles, leurs différences, et les cas d'usage typiques en MSP PME.

## Les 6 actions disponibles

| Action | Ce qui disparaît | Ce qui reste | Entra ID |
|---|---|---|---|
| Mettre hors service | Gestion MDM, profils et politiques Intune | Windows intact, données utilisateur | Retiré |
| Wipe + conserver inscription | Windows réinstallé, apps et paramètres | Inscription Intune, compte utilisateur associé | Maintenu |
| Wipe complet (options décochées) | Windows réinstallé, apps, paramètres, données utilisateur | Hash Autopilot dans le tenant | Retiré |
| Réinitialisation Autopilot (depuis Intune) | Profil utilisateur, apps et paramètres | Hash Autopilot, profil de déploiement, configs Intune | Maintenu |
| Autopilot Reset local (Ctrl+Win+R) | Profil utilisateur, apps et paramètres | Hash Autopilot, profil de déploiement, configs Intune | Maintenu |
| Fresh Start | Apps installées y compris bloatware OEM | Entra ID join (toujours), MDM selon option cochée | Toujours maintenu |

### Wipe complet — les deux options décochées

Quand tu lances un Wipe depuis Intune, deux cases sont proposées :

- "Conserver l'état d'inscription" — garde l'enrôlement Intune actif après le wipe
- "Conserver le compte utilisateur" — garde les fichiers du bureau et le profil

Wipe complet = les deux cases décochées = vraie remise à zéro. Seul le hash Autopilot enregistré dans le tenant survit.

!!! warning "Windows.old — bug corrigé"
    Il existait un bug sur Windows 11 où un Wipe complet avec BitLocker déplaçait les données utilisateur dans `Windows.old` au lieu de les effacer. Microsoft a corrigé ce problème sur les versions récentes.

### Autopilot Reset — profil uniquement, pas un vrai wipe

L'Autopilot Reset n'efface que le profil utilisateur. Ce n'est pas un wipe complet du disque.

!!! warning "Autopilot Reset = même utilisateur uniquement"
    Réserver l'Autopilot Reset aux cas où le poste est réattribué au même utilisateur. Pour un changement de personne, utiliser le Wipe complet — sinon des données de l'ancien utilisateur peuvent rester accessibles.

### Fresh Start — cas particulier

Fresh Start réinstalle Windows avec uniquement les apps built-in (Signature Edition). Contrairement au Wipe, le PC reste toujours Entra joined, même si tu ne coches pas "conserver les données utilisateur".

Cas d'usage typique : supprimer le bloatware OEM sur un nouveau PC avant de le remettre à un utilisateur, sans casser la jointure Entra.

!!! tip "Fresh Start et compte admin local"
    Ne pas bloquer la création du compte administrateur local dans les politiques Intune avant de lancer un Fresh Start — sinon erreur "There was a problem resetting your PC. No changes were made."

## Cas d'usage MSP

| Situation | Action recommandée | Raison |
|---|---|---|
| Départ employé, PC non récupéré | Mettre hors service | Coupe la gestion MDM sans besoin du matériel |
| Départ employé, PC récupéré pour réattribution | Wipe complet | Efface tout, OOBE Autopilot repart proprement |
| Changement d'utilisateur, même poste | Wipe complet | Supprime le profil de l'ancien utilisateur |
| Réattribution rapide, même utilisateur | Wipe + conserver inscription | Réinstalle Windows, garde l'enrôlement |
| Problème logiciel grave, on garde l'utilisateur | Réinit. Autopilot depuis Intune | Wipe profil + profil Autopilot réappliqué à distance |
| Technicien sur site, sans accès console Intune | Autopilot Reset local (Ctrl+Win+R) | Ne nécessite pas d'accès Intune, requiert un compte admin local |
| PC bloqué ou malware, action urgente à distance | Wipe complet depuis Intune | Le plus radical, efface tout |
| Fin de contrat prestataire, BYOD | Mettre hors service | Ne touche pas aux données perso, retire le MDM uniquement |
| Nouveau PC avec bloatware OEM à supprimer | Fresh Start | Réinstalle Windows proprement, garde la jointure Entra |

## Pièges MSP

### BitLocker et changement d'utilisateur

!!! danger "Piège BitLocker post-wipe"
    Si le primary user n'est pas retiré avant un Wipe, la clé BitLocker peut être cassée pour le prochain utilisateur — il ne pourra pas la récupérer depuis Entra. Toujours retirer le primary user dans Intune avant de lancer un Wipe pour une réattribution.

### LAPS et wipe

LAPS crée un compte admin local géré par Intune. Ce compte disparaît lors d'un wipe complet.

- LAPS est utile avant un "Mettre hors service" : il permet de conserver un accès local après que la gestion MDM est coupée.
- Après un wipe, Autopilot reprend la main. Le poste se réenrôle et Intune recrée automatiquement le compte LAPS via la politique.

!!! warning "Réflexe avant un Mettre hors service"
    Si aucun compte local n'existe et que LAPS n'est pas configuré, créer un compte local avant de lancer l'action — sinon perte de tout accès au poste.

## OneDrive et données hors profil

Un wipe (toutes variantes) efface `C:\scan`, `C:\clients` et tout dossier hors profil utilisateur standard. Intune ne fait aucune sauvegarde.

Checklist avant tout wipe ou réattribution :

- [ ] Primary user retiré dans Intune (évite le bug BitLocker)
- [ ] OneDrive configuré avec la sauvegarde des dossiers connus (Bureau, Documents, Images)
- [ ] Dernière sync OneDrive récente vérifiée dans la console de l'utilisateur
- [ ] Dossiers hors profil identifiés et sauvegardés manuellement (`C:\scan`, `C:\clients`, etc.)
- [ ] Compte local ou LAPS disponible si "Mettre hors service" prévu

!!! danger "Données hors OneDrive"
    OneDrive ne sauvegarde que Bureau, Documents et Images si la sauvegarde des dossiers connus est activée. Tout le reste est perdu définitivement lors d'un wipe.

## À lire ensuite

- [Autopilot](autopilot)
- [LAPS — gestion des comptes admin locaux](laps)
- [Conformité et Accès Conditionnel](intune-compliance.md) *(à venir)*
