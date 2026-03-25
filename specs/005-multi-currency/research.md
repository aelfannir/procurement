# Research: Multi-Currency MVP

## R1 — Currency API Integration Pattern

**Decision**: Use Symfony HttpClient (already available) to call fawazahmed0 currency API.

**Rationale**: `symfony/http-client` is already a dependency (composer.json) and used in AccountingService for Sage API calls. Same pattern: inject `HttpClientInterface`, call GET, parse JSON.

**Alternatives considered**:
- Guzzle: Not in project. Symfony HttpClient already available.
- BAM API: Tested and works, but MAD-only. fawazahmed0 supports any currency pair.

## R2 — PurchaseOrder.getCurrencyCode() Migration

**Decision**: Add stored `currencyCode` field using `HasCurrencyCode` trait. Override `getCurrencyCode()` to read from stored field with clinic fallback.

**Rationale**: The `HasCurrencyCode` trait already exists (`src/Entity/Traits/HasCurrencyCode.php`) and is used by Country and ProductPricing. It provides a 3-char string column with Currency validator. The existing `getCurrencyCode()` method (line 623-626) is called by:
- AccountingService.php (lines 100, 222) — Sage payload
- PdfController.php (lines 96, 131, 176, 228, 274, 319) — PDF exports
- ReceiptProduct.php (line 324) — convenience method

After change: method reads from stored field. All callers get vendor currency automatically.

**Null safety**: Migration adds nullable → backfills clinic currency → sets NOT NULL. No window with null values if migration runs before deployment.

## R3 — ProductPricing Currency Filter

**Decision**: Override CurrencyExtension behavior when fetching pricing for a PO. Pass explicit `currencyCode` filter parameter instead of relying on locale.

**Rationale**: Currently `CurrencyExtension` (`src/Doctrine/CurrencyExtension.php` lines 13-23) auto-filters ProductPricing by the user's locale currency (from `Accept-Language` header via `LocaleService`). This won't work for multi-currency POs — a user in Morocco (MAD locale) needs to see EUR pricing when the PO currency is EUR.

**Implementation**: The frontend ProductField.tsx (line 109-129) builds a filter for ProductPricing. Add `currencyCode` to this filter. The backend CurrencyExtension can be bypassed when an explicit `currencyCode` filter parameter is provided.

**Alternatives considered**:
- Disable CurrencyExtension entirely: Too broad, would affect other screens.
- Add a new endpoint: Unnecessary, just add a filter parameter.

## R4 — Invoice Rate Fetching

**Decision**: Fetch exchange rate in a Doctrine event listener (prePersist) on Invoice entity, not in a controller.

**Rationale**: Invoice creation happens through API Platform processors. A listener ensures the rate is always fetched regardless of creation path. Pattern consistent with existing `ProductPricingEventListener` which auto-sets currencyCode on prePersist.

**Alternatives considered**:
- State processor: Would work but requires knowing all creation paths.
- Controller: Invoices aren't created via custom controllers.

## R5 — AccountingService Changes

**Decision**: Convert all amounts by `invoice.exchangeRate` and send `clinic.currencyCode` to Sage.

**Rationale**: AccountingService.php line 222 currently sends `invoice->getPurchaseOrder()?->getCurrencyCode() ?? 'MAD'`. After our change, this returns vendor currency (e.g., SAR). Sage must receive clinic currency per PO requirement. Change to `$invoice->getClinic()->getCity()->getCountry()->getCurrencyCode()` and multiply all amounts by `$invoice->getExchangeRate()`.

**Affected lines**: 100 (error message), 137 (invoice amount), 172-194 (entries), 222 (payload), 452-531 (payment terms).

## R6 — Frontend Currency Dropdown

**Decision**: Add currency select field to PO form. Source: distinct `currencyCode` values from Country API endpoint with `eq[currencyCode]` filter support.

**Rationale**: Country entity already has API endpoint at `/countries` with `eq[:property]` filter (Country.php lines 49-59). Frontend can fetch distinct currencies. The PO Mapping.tsx already has a `currencyCode` field definition (line 264-268, currently read-only). Change to editable in create/update views.

**Alert dialog**: Reuse existing `ConfirmationModal` pattern from VendorField.tsx (lines 39-99) for currency change warning.

## R7 — Frontend Dual Columns

**Decision**: Add computed columns to PO product grid. Clinic amounts = `priceExclTax × exchangeRate`, calculated in frontend.

**Rationale**: The frontend already has:
- `CurrencyFormat` component that accepts explicit `currency` option
- `AmountUnit` component that reads `currencyCode` from form context
- Column definition pattern supports custom render functions

No backend changes for display — amounts are computed client-side from stored vendor amounts × stored rate.
