# Technical Feasibility

## Purpose

This document tracks the technical feasibility of the first implementation of the trade-shipping concept described in `01_trade_shipping_capacity_concept.md`.

The goal of the first version is deliberately simple:

1. Sum the total naval `transport_capacity` owned by each country.
2. Determine the country's merchant-power share in every market it influences.
3. Make a proportional share of the country's total transport capacity available to each such market.
4. Calculate the maritime trade demand of each market from outgoing trades that require sea transport.
5. Compare available shipping capacity against maritime trade demand.

This first implementation intentionally ignores fleet position, current missions, embarked armies, war status, route assignment, and competition between markets for the same physical ships.

---

## Feasibility status legend

| Status | Meaning |
|---|---|
| **Confirmed** | Verified in Vanilla game files or official engine/script documentation. |
| **Partially confirmed** | The underlying engine value exists, but normal gameplay-script access is not yet confirmed. |
| **Workaround available** | No direct getter is known, but the value can plausibly be reconstructed from script-visible data. |
| **Open** | A required script interface has not yet been verified. |

---

# 1. Country-wide naval transport capacity

## Requirement

For every country `C`, calculate:

```text
CountryTransportCapacity(C)
    = sum of transport_capacity of all ships owned by C
```

The result is global for the country. The current position and mission of each fleet are irrelevant in V1.

## Status

**Workaround available**

## Verified engine/game behavior

`transport_capacity` is a real naval unit statistic.

It is defined on naval unit categories and/or unit-type templates. Examples found in Vanilla include:

- Galley category: `transport_capacity = 0.05`
- Light Ship category: `transport_capacity = 0.05`
- Heavy Ship category: `transport_capacity = 0.10`
- Transport age templates: increasing transport-capacity values across the ages

The engine also calculates the maximum transport capacity of an existing fleet internally. The GUI exposes values including:

```text
Unit.GetMaxTransportCapacity
Unit.GetAvailableTransportCapacity
Unit.GetUsedTransportCapacity
Unit.GetUsedTransportCapacityValue
```

However, no equivalent normal gameplay-script value has yet been confirmed.

## Script-visible reconstruction

Country scope can iterate its units with:

```txt
every_unit = { ... }
```

Unit scope can iterate individual subunits with:

```txt
every_sub_unit = { ... }
```

Subunit scope can identify ship types with:

```txt
sub_unit_type = unit_type:...
```

and ship categories with:

```txt
sub_unit_category = sub_unit_category:...
```

Therefore the country total can be reconstructed by iterating all naval subunits and adding their effective transport-capacity contribution to a country variable.

### Prototype structure

```txt
set_variable = {
    name = trade_shipping_total_transport_capacity
    value = 0
}

every_unit = {
    limit = {
        is_army = no
    }

    every_sub_unit = {
        # Determine the effective transport capacity
        # of this ship and add it to the owner total.
    }
}
```

The unresolved part is not iteration itself, but how to avoid maintaining a duplicated lookup table for every ship type if the engine does not expose the effective `transport_capacity` statistic to gameplay script.

## Preferred implementation order

1. Search for a hidden/scriptable getter for effective unit-type or subunit `transport_capacity`.
2. If unavailable, maintain a scripted lookup table matching Vanilla ship types and transport-capacity values.
3. Keep the lookup isolated in one scripted value/effect so it is easy to update after game patches.

---

# 2. Fleet position and fleet activity

## Requirement for V1

None.

## Status

**Confirmed as intentionally ignored**

The first implementation does not care whether a ship is:

- in port,
- at sea,
- transporting troops,
- in combat,
- on a naval mission,
- close to the market receiving its economic contribution.

Its existence is sufficient for it to contribute to the country's global shipping pool.

This removes the need to solve Unit -> Location -> Market assignment for V1.

---

# 3. Market merchant power per country

## Requirement

For every market `M` and country `C` with merchant power in that market, calculate:

```text
MarketPowerShare(C, M)
    = MerchantPower(C, M) / TotalMerchantPower(M)
```

Then:

```text
CountryMarketShippingContribution(C, M)
    = CountryTransportCapacity(C)
      * MarketPowerShare(C, M)
```

## Status

**Open / highest-priority blocker**

## Known information

EU5 clearly tracks merchant power by country and market. Vanilla also provides effects for directly adding merchant power to a market, and several mechanics modify merchant power from maritime sources.

However, the exact normal Jomini-script numeric interface for either of the following has not yet been verified:

```text
MerchantPower(country, market)
TotalMerchantPower(market)
MerchantPowerShare(country, market)
```

GUI data may expose such information, but GUI data functions cannot be assumed to be usable from gameplay script.

## Required research

Search the following sources for a script-visible value:

- official `script_docs`
- official datatype dumps
- Vanilla `script_values`
- market trigger localization
- market effects and triggers
- Vanilla GUI only as a clue to engine-side terminology
- existing mods that calculate market-specific merchant power

Potential search terms:

```text
merchant_power
market_power
merchant_power_share
trade_power
market_share
merchant_share
power_in_market
```

## Acceptance criterion

V1 needs one of the following:

### Preferred

A direct value equivalent to:

```text
market.merchant_power(country)
```

plus total market merchant power.

### Acceptable

A direct merchant-power-share value.

### Fallback

A script-reconstructable country/market merchant-power contribution from lower-level values.

Without one of these, the proposed proportional market allocation cannot be implemented exactly as designed.

---

# 4. Enumerating relevant markets

## Requirement

For each country, find every market in which that country has non-zero merchant power, or alternatively iterate all markets and inspect participating countries.

## Status

**Open**

The exact most efficient iterator structure has not yet been selected.

Possible architectures:

### Country-first

```text
Country
  -> every market where country has merchant power
  -> calculate share
  -> contribute shipping capacity
```

This is conceptually ideal if an iterator for influenced markets exists.

### Market-first

```text
Every Market
  -> every country with merchant power
  -> read country cached transport capacity
  -> calculate weighted contribution
```

This may be easier if market participant iterators exist while country-to-market iterators do not.

### World-market sweep

As a last resort, periodically iterate every market and every relevant country. This is technically simple but potentially expensive and should not be chosen before profiling.

## Required research

Verify iterators for:

```text
every_market
every_country_in_market
every_merchant
every_market_with_merchant_power
```

Names above are search targets only, not assumed valid syntax.

---

# 5. Maritime trade demand

## Requirement

For each market `M`:

```text
SeaTradeDemand(M)
    = sum(trade_volume)
```

for every trade where:

```text
trade.from_market = M
```

and the trade is classified as requiring maritime transport.

Only the origin market receives the demand. The destination market does not receive the same demand again.

## Status

**Partially confirmed**

## Confirmed

Trade scope supports at least:

```text
trade_volume
from_market
to_market
goods
```

Vanilla uses `trade_volume` directly in trade conditions.

This makes `trade_volume` a much better basis for the shipping system than Vanilla Merchant Capacity, because the mod wants to represent the amount of cargo requiring transportation rather than the administrative/commercial cost of maintaining the trade.

## Open part

A reliable script-side classification of whether a trade requires sea transport is still unresolved.

---

# 6. Classifying a trade as maritime

## Requirement

Given:

```text
trade.from_market.location
trade.to_market.location
```

determine whether the trade requires maritime transport.

## Status

**Open**

## Confirmed related functionality

The engine/script layer contains several relevant connectivity concepts:

```text
is_connected_to
is_connected_to_through_realm
within_naval_range_of
is_coastal
is_land
can_find_trade_route
```

`is_connected_to_through_realm` is explicitly documented as checking whether two locations are connected by land/strait through the same realm.

That is not sufficient for arbitrary international market-to-market land connectivity.

## Important unresolved question

The exact semantics of the more general:

```text
is_connected_to
```

have not yet been verified strongly enough to use it as the foundation of the maritime classifier.

## Promising engine-side clue

The Trade GUI datatype exposes:

```text
Trade.GetFromPort
Trade.GetToPort
```

This suggests the engine internally knows port endpoints for trades that use maritime routing.

Research should determine whether Trade scope exposes equivalent gameplay-script targets such as:

```text
from_port
to_port
```

These names are hypotheses only and must not be used until verified.

## Preferred classification hierarchy

1. Direct script-exposed sea-route/port information on Trade.
2. Verified arbitrary land/strait connectivity test between market centers.
3. Approximation based on known geographic/connectivity triggers.

---

# 7. Market shipping capacity

## Requirement

Once country transport capacity and market-power shares are available:

```text
MarketTransportCapacity(M)
    = sum over countries C of
      CountryTransportCapacity(C)
      * MarketPowerShare(C, M)
```

## Status

**Conceptually complete; technically dependent on Section 3**

The calculation itself can be represented with normal script variables once the market-power share is accessible.

A market-level cached variable could conceptually store:

```text
trade_shipping_available_capacity
```

The exact variable location and update strategy remain to be decided after confirming market scope variable support and iterator availability.

---

# 8. Unit conversion / scaling factor

## Requirement

Vanilla `transport_capacity` and `trade_volume` are not naturally expressed in the same scale.

Therefore define:

```text
EffectiveShippingCapacity(M)
    = MarketTransportCapacity(M)
      * SHIPPING_CAPACITY_MULTIPLIER
```

## Status

**Confirmed design requirement; balance value open**

Example only:

```text
SHIPPING_CAPACITY_MULTIPLIER = 5
```

This would mean:

```text
1.0 transport_capacity
= 5.0 supported maritime trade_volume
```

The real value must be derived from representative save-game data rather than guessed during implementation.

## Balancing method

Collect samples for:

- small coastal countries,
- major maritime powers,
- land powers with limited fleets,
- early game,
- mid game,
- late game.

Compare:

```text
CountryTransportCapacity
SeaTradeDemand of major markets
MerchantPowerShare distribution
```

Then choose a multiplier that creates meaningful but not universally crippling shipping shortages.

---

# 9. Shipping coverage

## Requirement

For each market:

```text
ShippingCoverage(M)
    = min(
        1,
        EffectiveShippingCapacity(M) / SeaTradeDemand(M)
      )
```

Special case:

```text
if SeaTradeDemand(M) <= 0
    ShippingCoverage(M) = 1
```

## Status

**Technically straightforward once inputs exist**

Suggested cached market values:

```text
trade_shipping_sea_demand
trade_shipping_available_capacity
trade_shipping_coverage
```

The final names should follow the mod's namespace convention once one is selected.

---

# 10. Economic consequence of insufficient coverage

## Requirement

A market with insufficient shipping capacity should suffer an economic/trade penalty.

## Status

**Intentionally undecided**

Do not implement this before the upstream calculation is working and measurable.

Potential targets to research later include:

- trade efficiency,
- merchant capacity,
- merchant power,
- trade profitability,
- goods moved,
- market access,
- other market-level modifiers.

The first technical milestone should expose the calculated coverage value without yet applying a strong gameplay penalty.

---

# 11. Update frequency

## Requirement

Recalculate often enough to respond to:

- ships being built or destroyed,
- changing merchant power,
- new or removed trades,
- changes in trade volume,
- market creation/destruction or reassignment.

## Status

**Open design choice**

## Recommended V1

Use a monthly recalculation.

Reasons:

- transport-capacity ownership changes relatively infrequently,
- trade and merchant-power values do not require daily precision for this abstraction,
- monthly calculation limits script cost,
- cached market values are easier to inspect during debugging.

A future implementation may split recalculation into separate country and market caches if profiling shows a need.

---

# 12. Proposed V1 calculation pipeline

Conceptual monthly sequence:

```text
STEP 1 — COUNTRY SHIPPING POOL

for each country C:
    CountryTransportCapacity(C) = 0

    for each naval unit owned by C:
        for each ship/subunit:
            CountryTransportCapacity(C) += ship transport capacity


STEP 2 — RESET MARKET VALUES

for each market M:
    MarketTransportCapacity(M) = 0
    SeaTradeDemand(M) = 0


STEP 3 — DISTRIBUTE COUNTRY SHIPPING CAPACITY

for each market M:
    for each country C with merchant power in M:
        share = MerchantPowerShare(C, M)

        MarketTransportCapacity(M) +=
            CountryTransportCapacity(C) * share


STEP 4 — CALCULATE MARITIME TRADE DEMAND

for each trade T:
    if T requires maritime transport:
        T.from_market.SeaTradeDemand += T.trade_volume


STEP 5 — CALCULATE COVERAGE

for each market M:
    effective_capacity =
        MarketTransportCapacity(M)
        * SHIPPING_CAPACITY_MULTIPLIER

    if SeaTradeDemand(M) > 0:
        ShippingCoverage(M) =
            min(1, effective_capacity / SeaTradeDemand(M))
    else:
        ShippingCoverage(M) = 1
```

---

# 13. Main technical blockers

The current blockers are ordered by importance.

## Blocker 1 — Merchant-power share

Need a reliable numeric gameplay-script value for country merchant power in a market and either total merchant power or direct share.

**This is currently the most important blocker.**

## Blocker 2 — Maritime-route classification

Need a reliable way to determine whether an outgoing trade requires maritime transport.

## Blocker 3 — Effective ship transport-capacity getter

A fleet-level getter exists in the GUI, but normal gameplay-script access has not been found.

This blocker already has a practical fallback: reconstruct capacity from ship type/category definitions.

## Blocker 4 — Efficient market/country iteration

Need to identify the cleanest iterator structure for participating countries and markets.

This is mainly a performance/architecture question rather than a fundamental feasibility blocker.

---

# 14. Feasibility matrix

| Component | Status | V1 blocker? | Current fallback |
|---|---|---:|---|
| Iterate country units | **Confirmed** | No | — |
| Iterate ship subunits | **Confirmed** | No | — |
| Identify subunit type | **Confirmed** | No | — |
| Identify naval category | **Confirmed** | No | — |
| Vanilla `transport_capacity` exists | **Confirmed** | No | — |
| Direct gameplay getter for effective fleet transport capacity | **Partially confirmed** | No | Reconstruct from ship definitions |
| Country-wide transport-capacity cache | **Feasible** | No | Script variable |
| Trade `trade_volume` | **Confirmed** | No | — |
| Trade `from_market` / `to_market` | **Confirmed** | No | — |
| Determine sea-required trade | **Open** | **Yes** | Connectivity approximation if verified |
| Merchant power in market, numeric | **Open** | **Yes** | None confirmed yet |
| Merchant-power share | **Open** | **Yes** | Calculate from power if raw values are exposed |
| Market shipping-capacity sum | **Feasible** | No | Script variable |
| Shipping coverage calculation | **Feasible** | No | Script value/variables |
| Final economic penalty | **Open by design** | No | Delay until calculation works |

---

# 15. V1 non-goals

The first implementation explicitly does **not** model:

- fleet position,
- port assignment,
- naval missions,
- actual shipping lanes,
- individual route capacity allocation,
- distance-weighted shipping cost,
- simultaneous overcommitment of the same national fleet across several markets,
- military use of transport capacity reducing commercial capacity,
- blockade effects,
- piracy effects,
- wartime disruption,
- convoy escorts,
- per-country shipping shortages inside the same market.

These can be reconsidered only after the global-pool model is functional and produces useful gameplay.

---

# 16. Next research tasks

Research should proceed in this order:

1. **Find the script-visible merchant-power value/share for Country x Market.**
2. **Find the strongest reliable maritime-trade classifier.**
3. Search for a direct gameplay getter for effective ship/fleet `transport_capacity`.
4. Verify market and country iterators needed for an efficient monthly update.
5. Prototype cached country and market variables.
6. Add debug output/UI exposing:
   - country transport capacity,
   - market sea-trade demand,
   - market available shipping capacity,
   - shipping coverage.
7. Only after validation, design the gameplay penalty for insufficient shipping coverage.

---

# 17. Current feasibility conclusion

The core concept is technically plausible with the currently verified EU5 scripting primitives.

The country-side shipping pool is already implementable through unit/subunit iteration even if the engine's fleet-level transport-capacity getter remains GUI-only.

The two unresolved pieces that determine whether the intended V1 can be implemented exactly are:

1. **numeric Merchant Power / Merchant Power Share per country and market**, and
2. **a reliable sea-required classification for individual trades**.

Until those are verified, implementation should remain at the research/prototype stage rather than locking in guessed syntax or approximations.
