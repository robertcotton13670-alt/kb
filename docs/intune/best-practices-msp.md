---
title: Intune — Bonnes pratiques MSP
description: Best practices issues des sessions BCloud 1 et 2, orientées déploiement PME/MSP.
---

# Intune — Bonnes pratiques MSP

Synthèse des bonnes pratiques issues des webinaires BCloud sessions 1 et 2, complétées par le lab `crecas84.onmicrosoft.com`.

---

## Tenant / Entra

- Configurer le domaine personnalisé avant de créer les utilisateurs — les UPN créés avec le domaine `onmicrosoft.com` ne peuvent pas être renommés proprement ensuite. Si le domaine client est ajouté après, les utilisateurs gardent une adresse en `@client.onmicrosoft.com` sauf recréation manuelle.

- Scope MDM sur groupe ciblé en production — mettre "Tous" signifie que n'importe quel utilisateur qui joint un appareil à Entra déclenche automatiquement l'enrôlement Intune, y compris sur un PC personnel. Un groupe ciblé permet de contrôler qui est géré.

- Bloquer les appareils personnels dans les restrictions de plateforme — un appareil est considéré "professionnel" uniquement s'il est dans la liste Autopilot. Sans cette restriction, un utilisateur peut enrôler son PC personnel et recevoir toutes les politiques de l'entreprise.

- Activer LAPS sur tous les tenants clients — crée un compte admin local avec un mot de passe unique par machine, tournant automatiquement. Permet l'intervention technique sans partager un mot de passe global. Récupérable depuis Intune ou Entra sur la fiche appareil.

- Désactiver "l'utilisateur qui inscrit est admin local" — sans cette option désactivée, le premier utilisateur à enrôler la machine devient admin local. En Autopilot user-driven, c'est l'utilisateur final qui enrôle, pas le technicien.

---

## Autopilot

- Toujours définir un Group Tag à l'import du hash — sans Group Tag, l'appareil n'intègre aucun groupe dynamique basé sur `OrderID`. Il ne reçoit donc ni profil de déploiement ni ESP, et passe en OOBE standard sans configuration MSP.

- Affectation des profils sur groupes d'appareils uniquement — au moment où le profil Autopilot est appliqué, l'utilisateur n'est pas encore authentifié. Un groupe utilisateur ne peut pas être évalué à ce stade.

- Un seul profil par appareil — si un appareil est membre de deux groupes ayant chacun un profil Autopilot, c'est le profil le plus ancien (par date de création) qui s'applique, pas le plus récent. Difficile à déboguer si non anticipé.

- Supprimer l'appareil de l'ancien tenant Autopilot avant réimport — un hash ne peut appartenir qu'à un seul tenant à la fois. Supprimer l'appareil dans Intune ne suffit pas, il faut aussi le supprimer de la liste Autopilot séparément.

- Convention de nommage : préfixe client + aléatoire (ex : `PCP-%RAND:4%`) — facilite l'identification dans la console Intune et dans Datto RMM. Le suffixe `%RAND:4%` génère 4 caractères alphanumériques aléatoires au moment de l'enrôlement.

!!! warning "Suppression Autopilot"
    Supprimer un appareil dans Intune > Appareils ne le retire pas de la liste Autopilot. Les deux suppressions sont indépendantes.

---

## ESP (Enrollment Status Page)

- Apps bloquantes limitées au strict minimum — l'ESP attend que toutes les apps bloquantes soient installées avant de libérer le poste. Chaque app ajoutée est un risque de timeout ou d'échec. Seules les apps critiques pour la sécurité ou la gestion méritent d'être bloquantes.

- Exclure Office 365 des apps bloquantes — l'installation pèse entre 2 et 4 Go. Avec un timeout de 15 à 60 minutes, c'est souvent insuffisant selon la connexion client. L'utilisateur peut recevoir son poste sans Office, l'installation se termine en arrière-plan.

- Ne pas inclure les EXE, MSI user-scope, ou apps Store en bloquant — les EXE ne sont pas supportés pendant l'ESP. Les MSI qui s'installent dans le profil utilisateur échouent car le profil n'est pas encore créé. Les apps Store nécessitent également un profil utilisateur existant.

!!! tip "Apps bloquantes recommandées"
    Portail d'entreprise, Datto RMM, ThreatDown — uniquement. Office 365 à exclure systématiquement.

---

## Profils de configuration

- Catalogue de paramètres plutôt que Modèles pour tous les nouveaux profils — les Modèles n'ont que deux états : Activé ou Désactivé. Si un paramètre est mal configuré et poussé, il reste tatoué sur le poste même si la stratégie est supprimée. Le Catalogue offre un troisième état "Non configuré" qui efface le tatouage et permet de corriger sans réinstaller.

- Une stratégie = un sujet — si deux stratégies contiennent le même paramètre, Intune détecte le conflit et bloque les deux. Avec une stratégie par sujet (ex : `Edge - moteur de recherche`, `Bluetooth`), l'origine d'un problème est immédiatement identifiable.

- Rechercher en anglais dans le Catalogue — l'interface Intune est traduite en français mais le moteur de recherche du Catalogue indexe les termes anglais. Chercher `control panel` et non `panneau de configuration`.

- Ne jamais bloquer tout le panneau de configuration — certaines sections comme les paramètres d'affichage ou de résolution d'écran doivent rester accessibles à l'utilisateur. Un blocage total génère des tickets inutiles.

!!! danger "Tatouage"
    Un paramètre appliqué via un Modèle reste actif sur le poste même après suppression de la stratégie. Seul le Catalogue de paramètres permet de l'effacer proprement via l'état "Non configuré".

---

## Windows Update for Business (WUfB)

- Deux anneaux minimum — l'anneau Test (0 jour) reçoit les mises à jour immédiatement, idéalement sur les postes IT. L'anneau Production attend 14 jours, ce qui laisse le temps de détecter les régressions avant diffusion générale.

- Toujours affecter à des groupes d'appareils — une stratégie WUfB liée à un utilisateur changerait le comportement de mise à jour quand un autre utilisateur se connecte sur le même poste.

- Feature Update configuré séparément — les mises à jour de fonctionnalités (ex : passage à Windows 11 24H2) se configurent dans "Stratégies de mise à jour de fonctionnalités", pas dans les anneaux. Cela permet de verrouiller une version Windows précise pour tout un parc.

- Approbation manuelle des pilotes pour les clients critiques — dans les environnements médicaux, industriels ou avec du matériel spécifique, une mise à jour de pilote automatique peut casser un équipement ou un logiciel métier.

---

## Applications

- Microsoft Store nouveau en priorité — basé sur Winget, la mise à jour est gérée automatiquement par le Store sans intervention MSP. Méthode la moins coûteuse en maintenance.

- Chrome Enterprise MSI obligatoire — le Chrome standard (`.exe`) s'installe dans le profil utilisateur (`AppData`). Sans droits admin, il ne se met pas à jour correctement et ne supporte pas les stratégies de groupe ni Intune. Chrome Enterprise s'installe dans `Programme Files` en contexte Système.

- Office 365 exclu de l'ESP — voir section ESP ci-dessus.

- Vérifier le scope Winget avant de packager — un installeur en user-scope s'installe dans le profil utilisateur. Déployé en contexte SYSTEM via Intune, l'installation échoue silencieusement car le profil cible n'existe pas au moment de l'exécution.

- Portail d'entreprise en obligatoire sur tous les postes — c'est l'interface utilisateur d'Intune. Sans lui, l'utilisateur ne peut pas installer les apps disponibles, voir son état de conformité ni synchroniser manuellement ses politiques.

- Ne jamais cumuler Win32 en Obligatoire + ESP bloquante — une app Win32 en mode Obligatoire est installée lors de l'OOBE. Si cette même app est aussi dans les bloquantes de l'ESP, le moindre délai ou échec bloque l'OOBE entière.

!!! tip "Chrome Enterprise"
    URL de téléchargement : [https://enterprise.google.com/chrome/chrome-browser/](https://enterprise.google.com/chrome/chrome-browser/) — choisir le MSI 64 bits.

---

## Scripts PowerShell

- Contexte SYSTEM par défaut — les scripts s'exécutent sans profil utilisateur. Adapter uniquement si le script doit modifier le profil utilisateur (`HKCU`, Documents, etc.), auquel cas cocher "Exécuter avec les droits de l'utilisateur connecté".

- Chocolatey poussé avant les applications — l'ordre d'exécution Intune lors de l'Autopilot est : certificats → profils de configuration → scripts → applications. Pousser Chocolatey en script garantit sa disponibilité avant les déploiements d'applications qui en dépendent.

- Scripts de remédiation pour les services critiques — la correction fonctionne en deux scripts : le premier détecte un état problématique (`exit code 1`), le second corrige uniquement si la détection a échoué. Utile pour s'assurer que l'agent Datto ou ThreatDown est toujours actif.

```powershell linenums="1"
# Exemple — Détection service Datto RMM
$service = Get-Service -Name "CagService" -ErrorAction SilentlyContinue
if ($service -and $service.Status -eq "Running") {
    exit 0  # OK
} else {
    exit 1  # Problème détecté → déclenche le script de remédiation
}
```

---

## À lire ensuite

- [Autopilot — Lab session 1](intune_lab_session1.md)
- [Profils de configuration](intune_bcloud_session2.md)
- [Wintuner — Packaging applications](wintuner_intune.md)
