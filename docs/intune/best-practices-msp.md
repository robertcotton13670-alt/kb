---
title: Intune — Bonnes pratiques MSP
description: Best practices orientées déploiement PME/MSP.
---

# Intune — Bonnes pratiques MSP

Synthèse des bonnes pratiques issues des webinaires BCloud et du lab `crecas84.onmicrosoft.com`.

---

## Tenant / Entra

- Configurer le domaine personnalisé avant de créer les utilisateurs — les UPN créés avec le domaine `onmicrosoft.com` ne peuvent pas être renommés proprement ensuite. Si le domaine client est ajouté après, les utilisateurs gardent une adresse en `@client.onmicrosoft.com` sauf recréation manuelle.

- Vérifier que le tenant n'est pas en mode "Intune for Education" — ce mode revient régulièrement par défaut sur certains tenants et écrase silencieusement les configurations personnalisées. Les options modifiées ne sont pas appliquées ou sont réinitialisées sans avertissement. Si les politiques ne semblent pas s'appliquer correctement, vérifier le mode du tenant en premier.

!!! warning "Intune for Education"
    Si ce mode est actif sur un tenant PME standard, les personnalisations Intune peuvent ne pas être enregistrées ou appliquées. À vérifier systématiquement lors de la prise en charge d'un nouveau tenant.

- Scope MDM sur groupe ciblé en production — mettre "Tous" signifie que n'importe quel utilisateur qui joint un appareil à Entra déclenche automatiquement l'enrôlement Intune, y compris sur un PC personnel. Un groupe ciblé permet de contrôler qui est géré.

- Bloquer les appareils personnels dans les restrictions de plateforme — un appareil est considéré "professionnel" uniquement s'il est dans la liste Autopilot. Tout appareil hors Autopilot est classé "personnel" par défaut par Intune. Dans Intune > Appareils > Inscription > Restrictions de plateforme, la colonne "Propriété personnelle" permet de bloquer par type d'OS. Pour le BYOD Windows, mettre Windows MDM > Propriété personnelle sur Bloquer empêche tout PC non présent dans Autopilot de s'enrôler.

![Restrictions de plateforme Intune](../assets/images/restriction%20plateforme.png)

- Activer LAPS sur tous les tenants clients — crée un compte admin local avec un mot de passe unique par machine, tournant automatiquement. Permet l'intervention technique sans partager un mot de passe global. Récupérable depuis Intune ou Entra sur la fiche appareil.

- Désactiver "l'utilisateur qui inscrit est admin local" — par défaut, le premier utilisateur à enrôler la machine devient automatiquement admin local. En Autopilot user-driven, c'est l'utilisateur final qui enrôle, pas le technicien — il se retrouve donc admin de son propre poste sans que ce soit voulu. Depuis Entra ID > Appareils > Paramètres de l'appareil, le paramètre "L'utilisateur qui inscrit son appareil est ajouté en tant qu'administrateur local" propose trois options : Tout, Sélectionné, Aucun. Mettre sur Aucun empêche tout utilisateur enrôlant un appareil de devenir admin local.

![Paramètre admin local inscription Entra](../assets/images/inscription%20admin.png)

- Activer "Désactiver l'inscription MDM lors de l'ajout d'un compte professionnel" dans Entra > Mobilité > Microsoft Intune — sans cette option, quand un utilisateur ajoute son compte pro sur un PC personnel, Windows affiche un pop-up avec la case "Allow my organization to manage my device" cochée par défaut. Un simple clic sur OK enrôle le PC perso dans Intune et lui pousse toutes les politiques de l'entreprise. Ce toggle supprime ce pop-up et empêche l'enrôlement automatique des BYOD.

![Pop-up enrôlement automatique](../assets/images/popup%20inscription.png)
![Toggle désactiver inscription MDM](../assets/images/evider%20enrollement.png)

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

## Conformité / Sécurité

- MAM + Conditional Access app-based pour les BYOD non enrôlés — un appareil personnel non enrôlé dans Intune MDM ne peut pas reporter d'état de conformité à Entra ID. Une règle Conditional Access qui exige "appareil conforme" bloque donc systématiquement ces utilisateurs. La solution est de combiner deux mécanismes :

    - une App Protection Policy (MAM) appliquée sur Outlook, Teams, Edge — elle sécurise les données au niveau de l'app sans toucher au reste de l'appareil (pas de wipe total possible, pas de visibilité sur le perso)
    - une règle Conditional Access de type "app-based" qui exige que l'app utilisée soit protégée par une APP, plutôt qu'exiger un appareil conforme

    Résultat : l'utilisateur accède à sa messagerie et Teams depuis son téléphone perso, mais les données restent dans un conteneur sécurisé — copier-coller vers des apps personnelles bloqué, wipe sélectif possible en cas de départ.

- Tester les politiques Conditional Access en mode Report-Only avant activation — ce mode simule l'impact de la règle sans bloquer personne. Permet de détecter les utilisateurs ou appareils qui seraient bloqués avant le basculement en production.

- Conformité couplée à Conditional Access — une politique de conformité seule n'a aucun effet bloquant. Elle étiquette le poste "non conforme" dans la console, mais sans règle CA associée, l'utilisateur continue d'accéder à ses ressources normalement.

!!! tip "BYOD en PME"
    MAM-only (sans enrôlement MDM) est l'approche recommandée par défaut pour les appareils mobiles personnels en PME. Friction minimale pour l'utilisateur, protection maximale pour les données métier.

!!! warning "Conditional Access et comptes break-glass"
    Toujours exclure au moins un compte break-glass admin de toutes les politiques CA. Sans ça, une mauvaise configuration peut verrouiller l'accès total au tenant.

---

## Offboarding / cycle de vie

- Suppression Intune et Entra ID sont deux actions indépendantes — supprimer un appareil dans Intune ne le retire pas d'Entra ID, et inversement. À chaque offboarding, vérifier que les deux suppressions sont bien effectuées pour éviter des enregistrements fantômes avec des permissions résiduelles.

- Autopilot Reset pour la réaffectation interne — quand un appareil change d'utilisateur au sein du même tenant, l'Autopilot Reset est l'action adaptée : wipe du profil utilisateur, conservation de l'enrôlement Intune et du profil de déploiement. Le poste est prêt pour le nouvel utilisateur sans intervention physique.

- Wipe complet pour les départs définitifs ou changements de tenant — efface toutes les données, désactive l'enrôlement. Couplé à la suppression dans la liste Autopilot si le poste doit être transféré à un autre client.

- Audits trimestriels du parc — identifier les appareils inactifs depuis plus de 90 jours, les retirements non finalisés et les enregistrements obsolètes. Un appareil inactif qui reste enrôlé conserve un accès potentiel aux ressources du tenant.

!!! warning "Offboarding incomplet"
    Un appareil retiré d'Intune mais toujours présent dans Entra ID peut conserver des permissions d'accès résiduelles. Toujours supprimer les deux entrées.

---

## À lire ensuite

- [Autopilot — Lab session 1](intune_lab_session1.md) *(à venir)*
- [Wintuner — Packaging applications](wintuner_intune.md) *(à venir)*
