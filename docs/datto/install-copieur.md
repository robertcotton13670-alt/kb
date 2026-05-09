---
title: Composant - Install Copieur Réseau
description: Déployer un copieur réseau via Datto RMM — un composant par marque avec driver pré-embarqué, profil N/B et recto-verso, imprimante par défaut.
---

# Composant - Install Copieur Réseau

Six composants Datto RMM de type **Application**, un par marque de copieur. Chaque composant embarque le driver de sa marque directement dans le `.cpt` — le technicien n'a qu'à renseigner l'IP et le nom, aucun fichier à attacher.

Tous partagent le même script source : `myrepo/Datto RMM/install printer/Install-Copieur.ps1`

---

## Composants disponibles

| Composant Datto | Marques couvertes |
|---|---|
| `Install - Copieur Kyocera-TA [SYSTEM][WIN]` | Kyocera, Triumph-Adler |
| `Install - Copieur Konica-Develop [SYSTEM][WIN]` | Konica Minolta, Develop |
| `Install - Copieur Canon [SYSTEM][WIN]` | Canon |
| `Install - Copieur Ricoh [SYSTEM][WIN]` | Ricoh |
| `Install - Copieur Epson [SYSTEM][WIN]` | Epson |
| `Install - Copieur Sharp [SYSTEM][WIN]` | Sharp |

---

## Variables du composant

| Variable | Type | Défaut | Obligatoire | Description |
|---|---|---|---|---|
| `usrPrinterIP` | Chaîne | — | ✅ | Adresse IP du copieur (ex. `192.168.1.50`) |
| `usrPrinterName` | Chaîne | — | ✅ | Nom affiché dans Windows (ex. `MARSEILLE - RDC`) |
| `usrDriverName` | Chaîne | *(1er modèle de `modeles.txt`)* | ❌ | Nom exact du driver — pré-rempli depuis `modeles.txt`, détecté auto depuis le `.inf` si vide |
| `usrReplace` | Booléen | `Faux` | ❌ | Supprime et réinstalle si l'imprimante existe déjà |
| `usrSetDefault` | Booléen | `Faux` | ❌ | Définit comme imprimante par défaut pour l'utilisateur connecté |
| `usrMonochrome` | Booléen | `Faux` | ❌ | Coché = Noir et Blanc — décoché = couleur automatique |
| `usrRectoverso` | Booléen | `Faux` | ❌ | Coché = recto-verso bord long — décoché = recto uniquement |

!!! warning "Casse des variables dans l'interface Datto"
    Entrer les noms **sans le préfixe `env:`**.  
    ✅ Correct : `usrMonochrome`  
    ❌ Incorrect : `env:usrMonochrome`

---

## Procédure admin — Construire ou mettre à jour un composant

À effectuer lors de l'ajout d'un nouveau driver ou d'une mise à jour. Le `.cpt` généré est ensuite importé dans Datto RMM.

### Étape 1 — Préparer `drivers.zip`

Récupérer le driver PCL6 ou PS du fabricant et le préparer selon la structure attendue.

=== "Driver téléchargé depuis le site fabricant"

    1. Télécharger le package driver PCL6 (ou PS) depuis le site du fabricant
    2. Extraire l'archive et repérer le **dossier contenant le fichier `.inf`**
    3. Zipper ce dossier uniquement (pas l'intégralité du package)
    4. Renommer le zip `drivers.zip`

=== "Driver déjà déployé sur un poste"

    ```powershell
    # Localiser le dossier du driver dans le store Windows
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

Placer ensuite le zip dans le dossier de la marque concernée :

```
myrepo/Datto RMM/install printer/brands/
├── kyocera-ta/
│   └── drivers.zip    ← placer ici
├── konica-develop/
│   └── drivers.zip
├── canon/
│   └── drivers.zip
...
```

!!! info
    Les fichiers `drivers.zip` ne sont pas versionnés (`.gitignore`). À conserver localement ou sur un partage réseau.

### Étape 2 — Générer `modeles.txt`

Avant de construire le `.cpt`, générer la liste des modèles détectés dans chaque `drivers.zip` :

```powershell linenums="1"
cd "myrepo\Datto RMM\install printer"

# Toutes les marques
.\Generate-ModelesTxt.ps1

# Une seule marque
.\Generate-ModelesTxt.ps1 -Brand kyocera-ta
```

Le script crée `brands/<marque>/modeles.txt` avec la liste complète des modèles trouvés dans le `.inf`.

**Éditer ensuite chaque fichier** pour supprimer les modèles obsolètes ou non utilisés — seuls les modèles conservés apparaîtront dans la variable `usrDriverName` du composant.

!!! tip "Format du fichier"
    Un modèle par ligne. Les lignes vides et les lignes commençant par `#` sont ignorées.

    ```
    # Actifs chez nos clients
    Kyocera TASKalfa 2554ci KX
    Kyocera TASKalfa 3554ci KX
    Kyocera TASKalfa 4054ci KX
    # Kyocera TASKalfa 181 KX   <-- ancien modèle, commenté
    ```

### Étape 3 — Générer les `.cpt`

Depuis le dossier du composant :

```powershell linenums="1"
cd "myrepo\Datto RMM\install printer"

# Toutes les marques (ignore les marques sans drivers.zip)
.\Build-Composants.ps1

# Une seule marque
.\Build-Composants.ps1 -Brand kyocera-ta
```

Les `.cpt` sont générés dans `dist/` :

```
dist/
├── Install-Copieur-Kyocera-Ta.cpt
├── Install-Copieur-Konica-Develop.cpt
├── Install-Copieur-Canon.cpt
├── Install-Copieur-Ricoh.cpt
├── Install-Copieur-Epson.cpt
└── Install-Copieur-Sharp.cpt
```

??? note "Exemple de sortie du script"
    ```
    [kyocera-ta]
      [INFO] 12 models injected from modeles.txt
      [OK]   Install-Copieur-Kyocera-Ta.cpt  (driver: 45.2 MB  total: 45.3 MB)

    [konica-develop]
      [INFO] 8 models injected from modeles.txt
      [OK]   Install-Copieur-Konica-Develop.cpt  (driver: 38.1 MB  total: 38.2 MB)

    [canon]
      [SKIP] drivers.zip missing -- copy the canon driver zip here as drivers.zip

    Done -- Built: 2  Skipped: 1
    ```

### Étape 4 — Importer dans Datto RMM

Dans Datto RMM : **Composants > Importer** → sélectionner le `.cpt`.

Les GUIDs sont fixes par marque — un re-import met à jour le composant existant sans en créer un nouveau.

---

## Procédure déploiement — Installer un copieur

### Étape 1 — Sélectionner le composant et renseigner les variables

Dans Datto RMM, choisir le composant correspondant à la **marque du copieur** (ex. `Install - Copieur Kyocera-TA`) puis renseigner les variables.

=== "Nouveau copieur N/B recto-verso"

    | Variable | Valeur |
    |---|---|
    | `usrPrinterIP` | `192.168.1.50` |
    | `usrPrinterName` | `MARSEILLE - RDC` |
    | `usrMonochrome` | ✅ Coché |
    | `usrRectoverso` | ✅ Coché |
    | `usrReplace` | Décoché |
    | `usrSetDefault` | Décoché |

=== "Remplacement + imprimante par défaut"

    | Variable | Valeur |
    |---|---|
    | `usrPrinterIP` | `192.168.1.50` |
    | `usrPrinterName` | `MARSEILLE - RDC` |
    | `usrMonochrome` | ✅ Coché |
    | `usrRectoverso` | ✅ Coché |
    | `usrReplace` | ✅ Coché |
    | `usrSetDefault` | ✅ Coché |

=== "Copieur couleur recto"

    | Variable | Valeur |
    |---|---|
    | `usrMonochrome` | Décoché |
    | `usrRectoverso` | Décoché |

Laisser `usrDriverName` **vide** — le script détecte automatiquement le nom depuis le `.inf`.  
Si la détection échoue (log : `Cannot auto-detect driver name from .inf`), voir la section [Dépannage](#cannot-auto-detect-driver-name-from-inf).

### Étape 2 — Vérification après déploiement

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
| Kyocera / TA | `psk:JobDuplexAllDocumentsContiguously` | ✅ |
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

La détection auto n'a pas trouvé le nom dans le `.inf`. Récupérer le nom manuellement sur un poste où le driver est installé :

```powershell
Get-PrinterDriver | Select-Object Name | Sort-Object Name
```

Copier-coller **exactement** le nom retourné dans `usrDriverName` (la casse compte).

??? note "Noms courants par fabricant"
    | Fabricant | Nom driver typique |
    |---|---|
    | Konica Minolta / Develop | `KONICA MINOLTA Universal PCL` |
    | Kyocera / TA | `Kyocera TASKalfa XXXX ci KX` |
    | Canon | `Canon Generic Plus PCL6` |
    | Ricoh | `RICOH MP CXXXX PCL 6` |
    | Sharp | `SHARP MX-XXXX PCL6` |
    | Epson | `EPSON AM-C4000 Series` |

    Ces noms varient selon la version du package driver — toujours vérifier avec `Get-PrinterDriver`.

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

1. Vérifier que `usrMonochrome` est coché
2. Sur une imprimante déjà existante, passer `usrReplace` à coché pour forcer la réapplication du profil
3. Consulter le log pour la ligne `Config readback: Color=...` — `False` = N/B confirmé

### Build : `[SKIP] drivers.zip missing`

Le script de build ignore les marques sans `drivers.zip`. Placer le zip dans `brands/<marque>/drivers.zip` et relancer `Build-Composants.ps1`.

---

## Logs

| Emplacement | Contenu |
|---|---|
| `C:\ProgramData\CentraStage\Logs\install-copieur-*.log` | Log complet horodaté sur le poste |
| Datto RMM > Tâche > StdOut | Sortie temps réel |

---

## À lire ensuite

- [Report Printers](report-printers.md) — inventaire des imprimantes installées sur un poste
