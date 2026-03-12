# US8 - Blocking on Missing/Incomplete Clinic Config (Priority: P2)

Parent: [spec.md](../spec.md) | Requirements: FR-008

As a user comptabilising an invoice with a discrepancy on a clinic that has
no or incomplete discrepancy configuration, the system blocks with a clear
message.

**Why this priority**: Safety net — ensures no discrepancy is processed
without proper configuration. Also covers legacy clinics created before
this feature.

**Independent Test**: Attempt to comptabilise a discrepancy invoice on an
unconfigured clinic — verify blocking message.

## Acceptance Scenarios

1. **Given** an invoice with a discrepancy on a clinic with no discrepancy
   config, **When** I click Comptabiliser, **Then** blocked with: "Aucun
   paramétrage d'écart n'est défini pour la clinique [Nom]. Veuillez
   renseigner le seuil autorisé ainsi que les comptes comptables d'écart
   avant de comptabiliser cette facture."
2. **Given** a clinic with threshold set but missing clinic-favorable
   account, **When** I try to comptabilise, **Then** blocked with:
   "Le paramétrage d'écart de la clinique [Nom] est incomplet : compte
   d'écart favorable à la clinique manquant."
3. **Given** a clinic with threshold set but missing vendor-favorable
   account, **Then** blocked with: "Le paramétrage d'écart de la clinique
   [Nom] est incomplet : compte d'écart favorable au fournisseur manquant."
