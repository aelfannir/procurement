# Multi-Currency MVP — Résumé

## Problème

L'équipe est **bloquée** — impossible de créer des factures pour un fournisseur saoudien car le système ne supporte qu'une seule devise (devise clinique).

## Solution

**5 nouveaux champs en base. Aucun nouvel écran. Aucune nouvelle entité.**

Le fournisseur a un pays. À la création d'un BC, le système lit la devise du pays du fournisseur, récupère le taux de change via une API gratuite, et sauvegarde les deux sur le BC. La facture hérite automatiquement du BC.

## Ce qui change pour les utilisateurs

- **Fournisseurs même devise (99% des cas)** : absolument rien ne change
- **Fournisseurs devise étrangère** : le BC affiche un badge "SAR → MAD (2.50)" et deux colonnes supplémentaires montrant les montants en devise clinique. C'est tout.

## Ce qu'on a éliminé du spec original

Le spec original contenait **11 stories et 3 nouvelles entités** (Currency, ExchangeRate, AnnualAverageRate) plus 3 écrans de paramétrage. Après analyse :

- **Entité Currency** → inutile. L'entité Country a déjà les codes devise (ISO 4217).
- **CRUD taux de change** → inutile. Une API gratuite et testée (Bank Al-Maghrib + fallback global) fournit les taux officiels à la demande.
- **Taux moyen annuel** → inutile. Le taux journalier à la création du BC est plus précis.
- **Saisie manuelle du taux sur la facture** → inutile. La facture hérite du BC.
- **Validation ±10%** → inutile. Les taux viennent d'une API officielle, pas de saisie manuelle.

**11 stories → 4. 3 nouvelles entités → 0. 3 écrans admin → 0.**

## Les 4 stories

| # | Story | Description |
|---|-------|-------------|
| 1 | ACHAT-2100 | Ajouter un champ "Pays" au formulaire fournisseur |
| 2 | ACHAT-2103 | Le BC stocke devise + taux du fournisseur, affiche les montants en double devise |
| 3 | ACHAT-2104 | La facture hérite automatiquement devise + taux du BC |
| 4 | ACHAT-2106 | Vérifier que la comptabilisation Sage fonctionne avec la devise fournisseur |

## Pourquoi un MVP

- **Débloque le fournisseur saoudien immédiatement**
- **Zéro risque** — les BC en même devise ne sont pas impactés
- **Évolutif** — le même pattern (currencyCode + exchangeRate) peut être ajouté aux demandes d'achat, avoirs, prix produits dans les itérations futures
- **Aucune charge admin** — pas d'écrans de paramétrage à configurer, pas de taux à saisir manuellement
