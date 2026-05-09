---
title: Composant - Install Copieur Réseau
description: Procédure complète pour déployer un copieur réseau via Datto RMM — driver, profil N/B et recto-verso, imprimante par défaut.
---

# Composant - Install Copieur Réseau

Composant Datto RMM de type **Application** qui installe un copieur réseau sur un poste Windows : staging du driver depuis un `drivers.zip`, création du port TCP/IP, ajout de l'imprimante et application d'un profil d'impression (couleur, recto-verso) sans fichier XML externe.

Script source : `myrepo/Datto RMM/install printer/Install-Copieur.ps1`

---

## Variables du composant

| Variable | Type | Défaut | Obligatoire | Description |
|---|---|---|---|---|
| `usrPrinterIP` | Chaîne | — | ✅ | Adresse IP du copieur (ex. `192.168.1.50`) |
| `usrPrinterName` | Chaîne | — | ✅ | Nom affiché dans Windows (ex. `MARSEILLE - RDC`) |
| `usrDriverName` | Chaîne | *(vide)* | ❌ | Nom exact du driver — détecté auto depuis le `.inf` si vide |
| `usrReplace` | Booléen | `Faux` | ❌ | Supprime et réinstalle si l'imprimante existe déjà |
| `usrSetDefault` | Booléen | `Faux` | ❌ | Définit comme imprimante par défaut pour l'utilisateur connecté |
| `usrColorMode` | Chaîne | `Auto` | ❌ | `Auto` (couleur) ou `Monochrome` (N/B) |
| `usrDuplexMode` | Chaîne | `OneSided` | ❌ | `OneSided`, `TwoSidedLongEdge`, `TwoSidedShortEdge` |

!!! warning "Casse des variables dans l'interface Datto"
    Entrer les noms **sans le préfixe `env:`**.  
    ✅ Correct : `usrDuplexMode`  
    ❌ Incorrect : `env:usrDuplexMode`

---

## Procédure de déploiement

### Étape 1 — Préparer `drivers.zip`

Le composant attend un fichier `drivers.zip` contenant le dossier driver du fabricant.

=== "Driver téléchargé depuis le site fabricant"

    1. Télécharger le package PCL6 ou PS depuis le site du fabricant
    2. Extraire l'archive et repérer le **dossier contenant le `.inf`**
    3. Zipper ce dossier uniquement
    4. Renommer le zip `drivers.zip`

=== "Driver déjà présent sur un poste"

    ```powershell
    # Trouver le chemin du dossier driver
    Get-PrinterDriver -Name "NOM DU DRIVER" | Select-Object InfPath
    ```

    Zipper le dossier indiqué dans `InfPath` et renommer en `drivers.zip`.

!!! tip "Structure attendue dans le zip"
    ```
    drivers.zip
    └── dossier_driver/
        ├── mondriver.inf    ← obligatoire
        ├── mondriver.cat    ← recommandé (approbation certificat auto)
        └── *.dll, *.gpd…
    ```

### Étape 2 — Trouver le nom exact du driver

Laisser `usrDriverName` **vide** en priorité — le script détecte automatiquement le nom depuis le `.inf`.

Si la détection échoue (log : `Cannot auto-detect driver name from .inf`), récupérer le nom manuellement :

```powershell
# Sur un poste où le driver est déjà installé
Get-PrinterDriver | Select-Object Name | Sort-Object Name
```

Copier-coller **exactement** le nom retourné dans `usrDriverName`.

??? note "Noms courants par fabricant"
    | Fabricant | Nom driver typique |
    |---|---|
    | Konica Minolta / Develop | `Konica Minolta Universal PCL` |
    | Kyocera | `Kyocera TASKalfa XXXXX KX` |
    | Canon | `Canon Generic Plus PCL6` |
    | Ricoh | `RICOH PCL6 UniversalDriver V4.X` |
    | Sharp | `SHARP MX-XXXX PCL6` |
    | Epson | `EPSON AL-CXXXX Advanced PCL6` |

    Ces noms varient selon la version du package driver — toujours vérifier avec `Get-PrinterDriver`.

### Étape 3 — Attacher le zip et renseigner les variables

1. Dans le composant Datto : **Fichiers > Ajouter** → sélectionner `drivers.zip`
2. Renseigner les variables selon le copieur à déployer

**Exemples :**

=== "Nouveau copieur N/B recto-verso"

    | Variable | Valeur |
    |---|---|
    | `usrPrinterIP` | `192.168.1.50` |
    | `usrPrinterName` | `MARSEILLE - RDC` |
    | `usrColorMode` | `Monochrome` |
    | `usrDuplexMode` | `TwoSidedLongEdge` |
    | `usrReplace` | `Faux` |
    | `usrSetDefault` | `Faux` |

=== "Remplacement + imprimante par défaut"

    | Variable | Valeur |
    |---|---|
    | `usrPrinterIP` | `192.168.1.50` |
    | `usrPrinterName` | `MARSEILLE - RDC` |
    | `usrColorMode` | `Monochrome` |
    | `usrDuplexMode` | `TwoSidedLongEdge` |
    | `usrReplace` | `Vrai` |
    | `usrSetDefault` | `Vrai` |

=== "Copieur couleur recto"

    | Variable | Valeur |
    |---|---|
    | `usrColorMode` | `Auto` |
    | `usrDuplexMode` | `OneSided` |

### Étape 4 — Vérification après déploiement

Le log Datto doit se terminer par :

```
[SUCCESS] SUCCESS -- Printer 'NOM' installed in XXs
[INFO] Exit code: 0
```

Vérification sur le poste :

```powershell linenums="1"
# Imprimante présente
Get-Printer -Name "NOM DU COPIEUR"

# Profil appliqué — Color=False confirme N/B
Get-PrintConfiguration -PrinterName "NOM DU COPIEUR" |
    Select-Object Color, DuplexingMode

# Imprimante par défaut
(Get-WmiObject -Query "SELECT * FROM Win32_Printer WHERE Default=True").Name
```

---

## Compatibilité profil d'impression

Le script génère le `PrintTicketXml` dynamiquement — aucun fichier XML externe requis.
Il envoie les deux noms de feature duplex simultanément pour couvrir Canon et les autres.

| Fabricant | Feature duplex | Testé |
|---|---|---|
| Kyocera | `psk:JobDuplexAllDocumentsContiguously` | ✅ |
| Canon | `psk:DocumentDuplex` | ✅ |
| Epson | `psk:JobDuplexAllDocumentsContiguously` | ✅ |
| Ricoh | `psk:JobDuplexAllDocumentsContiguously` | ✅ |
| Develop / Konica | `psk:JobDuplexAllDocumentsContiguously` | ✅ |
| Sharp | Standard probable | ⏳ |

??? tip "Vérifier le profil d'un nouveau fabricant"
    Configurer manuellement l'imprimante sur un poste de référence, puis exporter :

    ```powershell
    (Get-PrintConfiguration -PrinterName "NOM EXACT").PrintTicketXml |
        Out-File "$env:USERPROFILE\Desktop\export.xml" -Encoding UTF8
    ```

    Vérifier dans le XML les features `psk:PageOutputColor` et duplex.

---

## Dépannage

### `Cannot auto-detect driver name from .inf`

La détection auto n'a pas trouvé le nom dans le `.inf`.  
→ Renseigner `usrDriverName` manuellement (voir Étape 2).

### `Attachment 'drivers.zip' not found`

→ Vérifier que le zip est bien attaché au composant et nommé exactement `drivers.zip`.

### `No ping response` — avertissement sur le ping

Normal si l'imprimante filtre l'ICMP. Le script continue automatiquement. Pas d'action requise.

### Imprimante non définie par défaut malgré `usrSetDefault=Vrai`

Windows 10/11 a l'option **"Laisser Windows gérer mon imprimante par défaut"** activée.  
Le script la désactive automatiquement via `LegacyDefaultPrinterMode=1` dans le registre utilisateur.

!!! warning
    Un utilisateur doit être connecté au moment du déploiement pour que l'imprimante par défaut soit appliquée. Si aucun utilisateur n'est connecté, relancer la tâche en session active.

### `pnputil exited X -- trying rundll32 fallback`

Avertissement normal — le fallback `rundll32` prend le relais. Pas d'action requise.

### Le profil N/B n'est pas appliqué après installation

1. Vérifier que `usrColorMode = Monochrome` est renseigné
2. Sur une imprimante déjà existante, passer `usrReplace = Vrai` pour forcer la réapplication du profil
3. Consulter le log pour la ligne `Config readback: Color=...` — `False` = N/B confirmé

---

## Logs

| Emplacement | Contenu |
|---|---|
| `C:\ProgramData\CentraStage\Logs\install-copieur-*.log` | Log complet horodaté sur le poste |
| Datto RMM > Tâche > StdOut | Sortie temps réel |

---

## À lire ensuite

- [Report Printers](report-printers.md) — inventaire des imprimantes installées sur un poste
