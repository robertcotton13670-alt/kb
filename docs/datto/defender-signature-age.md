---
title: Moniteur Defender - Age des signatures
description: Composant Datto RMM qui alerte si les signatures Windows Defender ont plus de 24h (ou seuil configurable).
---

# Moniteur Defender - Age des signatures

Moniteur Datto RMM qui interroge `Get-MpComputerStatus` et remonte une alerte si les signatures antivirus Windows Defender n'ont pas ete mises a jour depuis plus de `MaxAgeHours` heures (defaut : 24h). Couvre aussi la detection du service desactive et de l'antivirus desactive par un tiers.

Script source : `datto-rmm/monitors/defender-signature-age.ps1`

---

## Variables du composant

| Variable | Type | Defaut | Description |
|---|---|---|---|
| `MaxAgeHours` | Integer | `24` | Age maximum accepte des signatures en heures |

---

## Creer le composant dans Datto RMM

1. Ouvrir **ComStore** ou aller dans **Components > New Component**
2. Remplir les champs :
   - **Name** : `Monitor - Defender Signature Age`
   - **Category** : `Monitors`
   - **OS** : `Windows`
3. Dans l'onglet **Script**, coller le contenu de `defender-signature-age.ps1`
4. Dans l'onglet **Variables**, ajouter :

| Name | Type | Default Value | Description |
|---|---|---|---|
| `MaxAgeHours` | Integer | `24` | Max signature age in hours |

5. Dans l'onglet **Output Variables**, configurer :

| Output Variable Name | Comparison | Alert Threshold |
|---|---|---|
| `Status` | Contains | `CRITICAL` |

!!! warning "Nom de la variable de sortie"
    Le script ecrit `Status=...` dans le bloc resultat. Le champ **Output Variable Name** dans Datto doit etre exactement `Status` (sensible a la casse).

6. **Save** le composant

---

## Creer le moniteur (Monitor) dans Datto RMM

1. Aller dans **Sites > [Site cible] > Monitors > New Monitor**
2. Choisir **Component Monitor**
3. Selectionner le composant `Monitor - Defender Signature Age`
4. Configurer :
   - **Name** : `Defender - Signatures outdated`
   - **Interval** : `Every 4 hours` (recommande)
   - **Alert Priority** : `High`
   - **Auto-resolve** : activer (le moniteur repasse OK quand les signatures sont a jour)
5. Dans la section **Variables**, laisser `MaxAgeHours` vide pour utiliser 24h, ou saisir une valeur differente par client
6. **Assign** aux appareils ou groupe cible

!!! tip "Intervalle de verification"
    Avec un intervalle de 4h et un seuil de 24h, la fenetre d'alerte effective est 24h a 28h. Passer a 1h si tu veux une detection plus rapide.

---

## Interpreter les resultats

### Statuts possibles

| Status retourne | Signification | Action |
|---|---|---|
| `OK: Signatures 3.5 hours old (v1.419.xxx)` | Signatures a jour | Aucune |
| `CRITICAL: Signatures are 36.2 hours old (threshold: 24 h)` | Signatures perimees | Voir section troubleshooting |
| `CRITICAL: Windows Defender service is disabled` | Service MpsSvc arrete | Verifier si desactive intentionnellement ou par GPO |
| `CRITICAL: Antivirus is disabled -- third-party AV may have taken over` | Defender desactive par un AV tiers | Normal si Bitdefender/SentinelOne/etc. est en place |
| `CRITICAL: Cannot query Defender status -- cmdlet unavailable` | `Get-MpComputerStatus` absent | Defender completement desactive ou supprime |

!!! info "AV tiers installe"
    Si un antivirus tiers est present (ex. SentinelOne, Bitdefender), Defender se desactive automatiquement. L'alerte `Antivirus is disabled` est alors un faux positif -- exclure ces machines du moniteur ou adapter le seuil.

---

## Troubleshooting signatures perimees

1. **Verifier la connectivite Windows Update** -- les signatures se telechargent via WU ou WSUS
2. **Forcer la mise a jour manuellement** depuis le poste :
   ```powershell
   Update-MpSignature -UpdateSource MicrosoftUpdateServer
   ```
3. **Verifier l'etat du service** :
   ```powershell
   Get-Service -Name WinDefend, WdNisSvc | Select-Object Name, Status, StartType
   ```
4. **Verifier si WSUS bloque** les mises a jour de definitions (les signatures Defender utilisent le canal `Microsoft Malware Protection` dans WSUS)
5. **Verifier les evenements** :
   ```powershell
   Get-WinEvent -LogName "Microsoft-Windows-Windows Defender/Operational" -MaxEvents 20 |
       Where-Object { $_.Id -in @(1000, 1001, 2001, 2003, 2004) } |
       Select-Object TimeCreated, Id, Message | Format-List
   ```

!!! warning "WSUS et definitions Defender"
    Si le tenant utilise WSUS, verifier que les **Definition Updates** sont approuvees dans la console WSUS. Une definition bloquee `Not approved` maintient les signatures a la version du dernier push.

---

## Codes de sortie

| Code | Signification |
|---|---|
| `0` | Signatures a jour, service actif |
| `1` | Alerte : signatures perimees, service desactive, ou erreur d'interrogation |

---

## A lire ensuite

- [Toast Notifications Datto RMM](Toast notif depuis datto.md)
- [Audit OneDrive SharePoint](audit-onedrive-sharepoint-v3.md)
- [Composants Datto RMM -- bonnes pratiques](composants-datto.md) *(a venir)*
