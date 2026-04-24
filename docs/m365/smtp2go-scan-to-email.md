---
title: Scan-to-email via SMTP2GO
description: Configurer un copieur/scanner pour envoyer des mails via SMTP2GO en remplacement d'Exchange Online Basic Auth
---

# Scan-to-email via SMTP2GO

Microsoft a supprimé l'authentification basique (Basic Auth) sur Exchange Online. Les copieurs qui envoyaient directement via `smtp.office365.com` avec un identifiant/mot de passe ne fonctionnent plus. La solution recommandée est d'utiliser un relay SMTP tiers comme SMTP2GO, indépendant de Microsoft.

## Prérequis

- Accès au DNS du domaine client (Cloudflare, OVH, etc.)
- Un compte SMTP2GO (offre gratuite : 1 000 mails/mois, suffisant pour un copieur de bureau)
- Accès à l'interface web du copieur (via son IP sur le réseau local)

## 1. Vérifier le domaine dans SMTP2GO

Dans l'interface SMTP2GO : Sending > Verified Senders > Sender Domains > Add Sender Domain.

Entrer le domaine client (ex. `clientdomaine.fr`). SMTP2GO génère 3 enregistrements CNAME à ajouter au DNS :

| Type  | Hostname (exemple)              | Valeur                  |
|-------|---------------------------------|-------------------------|
| CNAME | `em895751.clientdomaine.fr`     | `return.smtp2go.net`    |
| CNAME | `s895751._domainkey.clientdomaine.fr` | `dkim.smtp2go.net` |
| CNAME | `link.clientdomaine.fr`         | `track.smtp2go.net`     |

!!! warning "Cloudflare"
    Les CNAME doivent être en mode "DNS only" (nuage gris). Le mode proxy (nuage orange) empêche la vérification.

!!! tip "Propagation DNS"
    Vérifier la propagation sur [dnschecker.org](https://dnschecker.org/) (type CNAME) avant de cliquer Verify. Délai possible jusqu'à 72h selon le bureau d'enregistrement.

Une fois les 3 CNAME en place, cliquer Verify dans SMTP2GO. Les 3 lignes passent au vert (✓).

!!! info "SPF et DKIM automatiques"
    Pas besoin de modifier l'enregistrement SPF existant. Le premier CNAME (return-path) gère le SPF automatiquement via VERP. DKIM est couvert par le second CNAME.

## 2. Créer un utilisateur SMTP

Sending > SMTP Users > Add SMTP User.

- Username : ex. `copieur@clientdomaine.fr` (format libre, pas besoin que la boîte existe)
- Password : utiliser le générateur SMTP2GO ou définir un mot de passe fort

!!! warning
    Ces identifiants sont propres à SMTP2GO. Ne pas utiliser un compte M365 ou Google existant.

Noter le username et le mot de passe — ils seront saisis dans le copieur.

## 3. Configurer le copieur

Accéder à l'interface web du copieur en saisissant son IP dans un navigateur (même réseau). L'IP se trouve en imprimant une page de configuration réseau depuis le menu du copieur, ou dans la liste des appareils connectés du routeur.

Chercher la section Email Settings ou SMTP Settings (généralement sous Network ou Scan) :

```
Serveur SMTP  : mail.smtp2go.com
Port          : 2525  (ou 587 si 2525 bloqué)
Authentification : SMTP Auth
Username      : (username créé à l'étape 2)
Password      : (mot de passe créé à l'étape 2)
```

!!! warning "Port 25"
    Ne pas utiliser le port 25 — il est bloqué en sortie par la quasi-totalité des FAI pour lutter contre le spam.

Lancer le test d'envoi depuis l'interface du copieur pour valider.

## Dépannage

| Symptôme | Cause probable | Action |
|---|---|---|
| Vérification domaine KO | DNS pas encore propagé | Attendre + vérifier sur dnschecker.org |
| Test d'envoi échoue | Mauvais identifiants | Vérifier username/password sans espace |
| Erreur connexion port 2525 | Port bloqué par le FAI | Essayer le port 587 |
| Mails en spam | DKIM pas encore propagé ou DMARC absent | Attendre la propagation, ajouter un enregistrement DMARC |

## DMARC (optionnel mais recommandé)

SMTP2GO ne gère pas le DMARC. Si le domaine n'en a pas, ajouter un enregistrement TXT minimal :

```
Nom  : _dmarc.clientdomaine.fr
Type : TXT
Valeur : v=DMARC1; p=none; rua=mailto:dmarc@clientdomaine.fr
```

Commencer avec `p=none` (monitoring uniquement) avant de passer à `p=quarantine` ou `p=reject`.

## À lire ensuite

- [Gestion multi-tenant M365](multi-tenant.md) *(à venir)*
- [DNS et délivrabilité email](dns-deliverabilite.md) *(à venir)*
