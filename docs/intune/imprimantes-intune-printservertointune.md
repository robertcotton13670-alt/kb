---
title: Imprimantes — Migration vers Intune (PrintServerToIntune)
description: Migrer les imprimantes d'un serveur d'impression vers des Win32 apps Intune disponibles dans le Portail d'entreprise.
---

# Imprimantes — Migration vers Intune avec PrintServerToIntune

Sans Universal Print, la méthode la plus efficace pour exposer des imprimantes TCP/IP dans le Portail d'entreprise est de les packager en Win32 apps Intune. PrintServerToIntune automatise entièrement cette opération depuis le serveur d'impression existant.

!!! tip "Pertinence MSP Poweriti"
    Idéal pour les clients PME qui veulent supprimer leur serveur d'impression AD tout en offrant du self-service aux utilisateurs via le Portail d'entreprise. Testé jusqu'à ~12 imprimantes — OK pour la majorité des clients.

---

## Prérequis

- Serveur d'impression Windows avec les imprimantes connectées **en port TCP/IP** (pas via nom de partage UNC)
- Si pas de serveur d'impression : connecter toutes les imprimantes en IP port sur un poste relais temporaire
- PowerShell 5.1+ et accès Internet sur le serveur (pour télécharger IntuneWinAppUtil.exe)
- Compte avec droits **DeviceManagementApps.ReadWrite.All** sur le tenant Intune cible
- Module PowerShell Graph (installé automatiquement par le script si absent)

!!! warning "Imprimantes non exportées automatiquement"
    Les imprimantes Microsoft intégrées (Print to PDF, OneNote, Fax) sont ignorées. Seules les imprimantes avec un driver ayant un `infpath` 64 bits valide sont exportées. Les imprimantes non TCP/IP (USB direct, partage UNC) ne fonctionnent pas avec cet outil.

---

## Mise en place

### Ressources

| Ressource | Lien |
|---|---|
| Repo GitHub | [gnon17/PrintServerToIntune](https://github.com/gnon17/PrintServerToIntune) |
| Article de référence | [smbtothecloud.com — PrintServerToIntune](https://smbtothecloud.com/printservertointune-migrate-printers-to-intune-as-tcp-ip-connections/) |
| Méthode manuelle (référence) | [call4cloud.nl — Deploy Printer Drivers Win32](https://call4cloud.nl/deploy-printer-drivers-intune-win32app/) |
| Alternative (print server maintenu) | [Rock My Printers — Nicklas Ahlberg](https://www.rockenroll.tech/2023/03/14/rock-my-printers/) |

### Téléchargement

1. Aller sur le [repo GitHub](https://github.com/gnon17/PrintServerToIntune) > **Releases** > télécharger le zip de la dernière version
2. Extraire sur le serveur d'impression (ex: `C:\PrintServerToIntune\`)
3. Vérifier la présence des 3 fichiers :
    - `PackageMyPrinters.ps1`
    - `UploadIntuneWinPrinters.ps1`
    - `PrinterIcon.jpg` (remplaçable, garder le même nom)

!!! tip "Personnalisation publisher"
    Avant de lancer, éditer la ligne 169 de `UploadIntuneWinPrinters.ps1` pour changer le publisher affiché dans le Portail d'entreprise (par défaut : "SMBtotheCloud").

---

## Utilisation

### Étape 1 — Packager les imprimantes

Exécuter depuis le serveur d'impression en PowerShell administrateur :

```powershell linenums="1"
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
.\PackageMyPrinters.ps1
```

Une fenêtre de sélection s'ouvre — sélectionner les imprimantes à migrer (Ctrl+clic pour multi-sélection), puis OK.

Le script effectue automatiquement :

- Extraction des infos (IP, port, driver, INF)
- Génération des scripts `Install.ps1`, `Uninstall.ps1`, `Detect.ps1`
- Téléchargement de `IntuneWinAppUtil.exe` depuis GitHub
- Création d'un `.intunewin` par imprimante

Résultat dans le répertoire du script :

```
ExportedPrinters\
  └── NomImprimante\
        ├── Install.ps1
        ├── Uninstall.ps1
        ├── Detect.ps1
        ├── manifest.json
        ├── driver\
        └── install.intunewin
Logs\
IntuneWinAppUtil.exe
```

### Étape 2 — Uploader dans Intune

À la fin du packaging, le script propose de lancer l'upload :

```
Upload printers to Intune now? (Y/N)
```

Répondre `Y`. Le script demande alors :

1. **Authentification Graph** — fenêtre interactive Microsoft (utiliser un compte avec droits DeviceManagementApps.ReadWrite.All)
2. **Assigner en Available à All Users ?** — répondre `Y` pour que les imprimantes apparaissent directement dans le Portail d'entreprise

Le script crée une Win32 app par imprimante dans Intune et uploade les packages.

!!! tip "Upload séparé"
    Si tu as répondu N à l'upload ou si tu veux uploader plus tard, lancer `UploadIntuneWinPrinters.ps1` séparément — il doit être dans le dossier parent d'`ExportedPrinters`.

---

## Résultat dans Intune

Chaque imprimante apparaît dans **Intune > Applications > Windows** comme une Win32 app avec :

- Commande d'installation : `Install.ps1` (injection driver + création port + création imprimante)
- Commande de désinstallation : `Uninstall.ps1`
- Règle de détection : `Detect.ps1` (vérifie la présence de l'imprimante dans Windows)
- Icône générique imprimante (ou ta propre icône si remplacée)

Si assignée en **Available à All Users**, l'utilisateur ouvre le Portail d'entreprise et installe lui-même l'imprimante voulue — sans Universal Print, sans droits admin.

---

## Utilisation quotidienne MSP

### Ajouter une nouvelle imprimante chez un client

1. Connecter l'imprimante en port TCP/IP sur le serveur (ou poste relais)
2. Relancer `PackageMyPrinters.ps1` — sélectionner uniquement la nouvelle imprimante
3. Uploader via `UploadIntuneWinPrinters.ps1` ou répondre Y à la fin du packaging
4. Assigner la nouvelle app au groupe voulu dans Intune

### Supprimer une imprimante

Dans Intune > Applications > Windows > sélectionner l'app imprimante > Supprimer.

Si elle était assignée en Required, elle sera désinstallée automatiquement des postes. Si Available, elle disparaît du Portail d'entreprise.

### Changer l'IP d'une imprimante

Il n'y a pas de mise à jour en place — il faut recréer le package avec la nouvelle IP et supprimer l'ancienne app. Ou pousser un script de remédiation Intune qui met à jour le port TCP/IP.

---

## Pièges et bonnes pratiques

| Piège | Solution |
|---|---|
| Imprimante non exportée (pas de `infpath`) | Utiliser un driver 64 bits valide installé sur le serveur |
| Imprimante connectée en UNC (\\\\serveur\\imprimante) | Reconnecter en TCP/IP direct avant d'exporter |
| Upload échoue (auth) | Vérifier que le compte a `DeviceManagementApps.ReadWrite.All` |
| Publisher "SMBtotheCloud" dans le portail | Éditer ligne 169 de `UploadIntuneWinPrinters.ps1` avant le premier run |
| Imprimante réapparaît après désinstallation | Vérifier que l'app n'est pas aussi assignée en Required |
| Script bloqué par ExecutionPolicy | Lancer avec `-ExecutionPolicy Bypass` en scope Process uniquement |

!!! warning "Pas adapté aux grands parcs"
    Pour des flottes de 20+ imprimantes ou des besoins de gestion fine (quotas, couleur, recto-verso forcé), prévoir une solution cloud print dédiée (PaperCut, Printix, Universal Print).

---

## À lire ensuite

- [Déploiement d'applications Win32 dans Intune](applications-win32.md) *(à venir)*
- [Portail d'entreprise — configuration et déploiement](portail-entreprise.md) *(à venir)*
- [Scripts PowerShell via Intune](scripts-powershell-intune.md) *(à venir)*
