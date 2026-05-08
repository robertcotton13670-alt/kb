---
title: Composant - Report Printers
description: Script Datto RMM qui liste les imprimantes installees (locales et reseau) avec driver, version, IP et statut, avec ecriture optionnelle dans un UDF.
---

# Composant - Report Printers

Script Datto RMM de type **Script** qui inventorie toutes les imprimantes installees sur un poste : nom, type (local/reseau), driver, version du driver, fabricant, IP du port, statut de partage et imprimante par defaut. Les imprimantes virtuelles (PDF, XPS, OneNote, Fax) sont exclues par defaut. Resultat complet dans le stdout Datto, resume optionnel dans un UDF.

Script source : `datto-rmm/scripts/report-printers.ps1`

---

## Variables du composant

| Variable | Type | Defaut | Description |
|---|---|---|---|
| `UDFNumber` | Integer | vide | Numero du UDF Datto (1-30) ou ecrire le resume. Laisser vide pour ne pas ecrire. |
| `ShowVirtual` | Boolean | `false` | Inclure les imprimantes virtuelles (PDF, XPS, OneNote, Fax) |

---

## Creer le composant dans Datto RMM

1. **Components > New Component**
2. Remplir :
   - **Name** : `Script - Report Printers`
   - **Category** : `Scripts`
   - **OS** : `Windows`
3. Coller le contenu de `report-printers.ps1` dans l'onglet **Script**
4. Onglet **Variables** — ajouter :

| Name | Type | Default Value | Description |
|---|---|---|---|
| `UDFNumber` | Integer | *(vide)* | UDF cible pour le resume |
| `ShowVirtual` | Boolean | `false` | Inclure imprimantes virtuelles |

5. **Save**

!!! tip "UDF recommande"
    Configurer `UDFNumber` au niveau du site plutot qu'en variable de composant pour homogeneiser l'emplacement du resume sur tous les postes du client.

---

## Exemple de sortie stdout

```
============================================================
  PRINTER REPORT -- 2026-05-08 10:32
  Host: PC-CLIENT01
  Total printers found: 2
============================================================

  Name    : HP LaserJet Pro M404n [DEFAULT]
  Type    : Network
  Status  : Normal
  Driver  : HP LaserJet Pro M404
  Version : 61.273.1.21716
  Provider: HP
  Port    : IP_192.168.1.50 [192.168.1.50]
  Shared  : False
  ShareName: -
------------------------------------------------------------

  Name    : RICOH MP C307 PCL 6
  Type    : Local
  Status  : Normal
  Driver  : RICOH PCL6 UniversalDriver V4.22
  Version : 4.22.0.0
  Provider: Ricoh
  Port    : USB001
  Shared  : False
  ShareName: -
------------------------------------------------------------
```

---

## Contenu du UDF (si configure)

Le UDF recoit un resume sur une ligne, tronque a 5 imprimantes max :

```
HP LaserJet Pro M404n [Network] DEFAULT | RICOH MP C307 PCL 6 [Local]
```

!!! info "Limite UDF Datto"
    Les UDFs Datto acceptent jusqu'a 255 caracteres. Si plus de 5 imprimantes, le script ajoute `+N more` pour rester dans la limite.

---

## Imprimantes exclues par defaut

Les patterns suivants sont filtres automatiquement quand `ShowVirtual = false` :

| Pattern | Exemples |
|---|---|
| `*OneNote*` | Microsoft OneNote |
| `*Microsoft*` | Microsoft Print to PDF, Microsoft XPS |
| `*Fax*` | Windows Fax |
| `*XPS*` | XPS Document Writer |
| `*PDF*` | Adobe PDF, Foxit PDF |
| `*Virtual*` | Imprimantes virtuelles tierces |
| `*Generic*` | Generic / Text Only |

Passer `ShowVirtual = true` pour tout inclure (utile pour auditer les drivers virtuels en place).

---

## Cas d'usage typiques

| Situation | Action |
|---|---|
| Migration de serveur d'impression | Lancer sur tous les postes pour identifier les queues a recreer |
| Ticket "imprimante disparue" | Verifier si le driver est encore present avant de reinstaller |
| Audit parc avant deploiement Universal Print | Inventorier les modeles et versions de drivers |
| Poste lent au demarrage | Detecter les imprimantes reseau avec IP injoignable |

!!! warning "Execution en contexte SYSTEM"
    Le script tourne sous `NT AUTHORITY\SYSTEM`. Les imprimantes deployees uniquement dans le profil utilisateur (HKCU) peuvent ne pas apparaitre. Pour un inventaire complet incluant les imprimantes utilisateur, lancer egalement en contexte utilisateur via une tache planifiee.

---

## Codes de sortie

| Code | Signification |
|---|---|
| `0` | Succes -- rapport genere |
| `1` | Erreur -- `Get-Printer` a echoue ou exception non geree |

---

## A lire ensuite

- [Moniteur Defender - Age des signatures](defender-signature-age.md)
- [Toast Notifications Datto RMM](Toast notif depuis datto.md)
- [Composants Datto RMM -- bonnes pratiques](composants-datto.md) *(a venir)*
