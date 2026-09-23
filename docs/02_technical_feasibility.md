# Technical Feasibility

## Purpose

This document tracks the technical feasibility of the first implementation of the trade-shipping concept described in `01_trade_shipping_capacity_concept.md`.

The V1 model is deliberately simple:

1. Sum the total naval `transport_capacity` owned by each country.
2. Determine the country's Merchant Power share in every market in which it participates.
3. Make a proportional share of the country's total transport capacity available to each such market.
4. Calculate maritime trade demand from outgoing trades that require sea transport.
5. Compare available shipping capacity against maritime trade demand.

V1 intentionally ignores fleet position, current missions, embarked armies, war status, route assignment, and competition between markets for the same physical ships.

---

## Feasibility status legend

| Status | Meaning |
|---|---|
| **Confirmed** | Verified in Vanilla game files or official engine/script documentation. |
| **Partially confirmed** | The underlying engine value exists, but normal gameplay-script access is not yet confirmed. |
| **Workaround available** | No direct getter is known, but the value can be reconstructed from script-visible data. |
| **Open** | A required script interface has not yet been verified. |

---

# 1. Country-wide naval transport capacity

## Requirement

For every country `C`:

```text
CountryTransportCapacity(C)
    = sum of transport_capacity of all ships owned by C
```

The result is global for the country. Ship position and current activity are irrelevant in V1.

## Status

**Workaround available**

## Confirmed engine/game behavior

`transport_capacity` is a real naval unit statistic.

Examples in Vanilla include:

- Galley category: `transport_capacity = 0.05`
- Light Ship category: `transport_capacity = 0.05`
- Heavy Ship category: `transport_capacity = 0.10`
- Transport age templates: increasing values across the ages

The engine also calculates fleet transport capacity internally. The GUI exposes values including:

```text
Unit.GetMaxTransportCapacity
Unit.GetAvailableTransportCapacity
Unit.GetUsedTransportCapacity
Unit.GetUsedTransportCapacityValue
```

No equivalent direct gameplay-script getter has yet been confirmed.

## Script-visible reconstruction

Country scope can iterate units:

```txt
every_unit = { ... }
```

Unit scope can iterate individual subunits:

```txt
every_sub_unit = { ... }
```

Subunit scope can identify ship types and categories:

```txt
sub_unit_type = unit_type:...
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
    limit = { is_army = no }

    every_sub_unit = {
        # Look up this ship's effective transport_capacity
        # and add it to the country total.
    }
}
```

## Remaining issue

The unresolved part is avoiding a duplicated lookup table for every ship type if the engine does not expose the effective `transport_capacity` statistic to gameplay script.

### Preferred implementation order

1. Search for a hidden gameplay-script getter for effective ship/subunit `transport_capacity`.
2. If unavailable, maintain a centralized scripted lookup matching Vanilla ship definitions.
3. Keep the lookup isolated so game-version changes are easy to audit.

---

# 2. Fleet position and fleet activity

## Requirement for V1

None.

## Status

**Confirmed as intentionally ignored**

A ship contributes to its country's global shipping pool regardless of whether it is:

- in port,
- at sea,
- transporting troops,
- in combat,
- on another naval mission,
- close to any particular market.

The existence of the ship is sufficient.

This means V1 does not require Unit -> Location -> Market assignment.

---

# 3. Market Merchant Power per country

## Requirement

For every market `M` and country `C` with Merchant Power in that market:

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

**Confirmed**

This is no longer a V1 blocker.

## Confirmed gameplay-script values

On Market scope, Vanilla exposes the country-specific Merchant Power as:

```txt
"merchant_power_in_market(country_scope)"
```

It also exposes total Merchant Power in the current market as:

```txt
total_merchant_power
```

Both values are available in normal gameplay script, not merely in GUI data.

Vanilla missions use expressions such as:

```txt
value = "scope:mission_target_market.merchant_power_in_market(root)"
```

and Market-scope triggers such as:

```txt
merchant_power_in_market = {
    country = root
    value >= some_value
}
```

## Confirmed Merchant Power share formula

Vanilla itself calculates a country's share of Merchant Power by dividing the country-specific value by total market Merchant Power.

The relevant pattern appears in:

```text
reference_game_files/game/in_game/common/scripted_relations/deny_market_access.txt
```

Conceptually, the Vanilla calculation is:

```txt
value = "merchant_power_in_market(country_scope)"

if = {
    limit = { total_merchant_power != 0 }
    divide = total_merchant_power
}
```

Therefore the V1 formula is directly scriptable:

```text
MarketPowerShare(C, M)
    = merchant_power_in_market(C)
      / total_merchant_power
```

No custom reconstruction of Merchant Power is required.

## Zero-power / zero-total handling

The implementation should explicitly protect against division by zero:

```text
if TotalMerchantPower(M) > 0:
    share = MerchantPower(C, M) / TotalMerchantPower(M)
else:
    share = 0
```

Markets in which the country has no Merchant Power should not receive a contribution from that country.

---

# 4. Enumerating relevant markets and merchants

## Requirement

For each country, find every market in which that country participates commercially and calculate its Merchant Power share.

## Status

**Confirmed**

This is also no longer a V1 blocker.

## Preferred country-first iterator

Vanilla provides:

```txt
every_market_with_merchants = {
    ...
}
```

This iterator is used from country context in Vanilla events and iterates markets in which the country has merchant participation.

For the trade-shipping system, the country-first architecture is therefore viable:

```text
Country
  -> every_market_with_merchants
  -> merchant_power_in_market(country)
  -> total_merchant_power
  -> calculate share
  -> add weighted shipping contribution to market
```

A defensive Merchant Power filter can still be applied inside the Market scope:

```txt
"merchant_power_in_market(scope:shipping_country)" > 0
```

## Alternative world-market sweep

Vanilla also provides:

```txt
every_market_in_world = {
    limit = {
        has_merchant = root
    }
}
```

This exact pattern is used in Vanilla Portugal content.

It provides a straightforward fallback if a world-market pass is ever preferable for initialization or debugging.

## Market-first iteration

Market scope also supports:

```txt
every_merchant_in_market = {
    ...
}
```

Inside this iterator the iterated scope is the merchant country.

Therefore both architectures are technically possible:

### Country-first

```text
Country
  -> every_market_with_merchants
```

### Market-first

```text
Market
  -> every_merchant_in_market
```

For V1, **country-first is preferred** because the country transport-capacity pool is already calculated and cached on Country scope.

---

# 5. Confirmed V1 market-allocation formula

With Sections 3 and 4 resolved, the market allocation can now be defined without speculative syntax or data reconstruction.

For country `C` and market `M`:

```text
PowerShare(C, M)
    = MerchantPower(C, M) / TotalMerchantPower(M)
```

```text
ShippingContribution(C, M)
    = CountryTransportCapacity(C)
      * PowerShare(C, M)
```

For the whole market:

```text
MarketTransportCapacity(M)
    = sum over all merchant countries C of
      ShippingContribution(C, M)
```

Example:

```text
England total transport capacity = 20

London Merchant Power share     = 70% -> contribution 14
Antwerpen Merchant Power share  = 25% -> contribution 5
Lisboa Merchant Power share     = 10% -> contribution 2
```

The same national shipping pool is intentionally reused across all influenced markets in V1. This is an abstraction, not physical ship allocation.

---

# 6. Maritime trade demand

## Requirement

For each market `M`:

```text
SeaTradeDemand(M)
    = sum(trade_volume)
```

for every Trade where:

```text
trade.from_market = M
```

and the Trade is classified as requiring maritime transport.

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

Vanilla uses `trade_volume` directly in Trade conditions.

`trade_volume` is therefore the preferred demand basis for this mod. The custom system is intended to model physical cargo volume, not Vanilla Merchant Capacity.

## Open part

A reliable script-side classification of whether a Trade requires sea transport remains unresolved.

---

# 7. Classifying a Trade as maritime

## Requirement

Given:

```text
trade.from_market.location
trade.to_market.location
```

determine whether the Trade requires maritime transport.

## Status

**Open / current primary V1 blocker**

## Confirmed related functionality

Relevant connectivity concepts include:

```text
is_connected_to
is_connected_to_through_realm
within_naval_range_of
is_coastal
is_land
can_find_trade_route
```

`is_connected_to_through_realm` is explicitly documented as checking land/strait connectivity inside the same realm. That is insufficient for arbitrary international market-to-market routing.

## Important unresolved question

The exact semantics of the more general:

```text
is_connected_to
```

still need to be verified before using it as the maritime classifier.

## Promising engine-side clue

The Trade GUI datatype exposes:

```text
Trade.GetFromPort
Trade.GetToPort
```

This proves that the engine internally knows port endpoints for Trade objects.

Research should determine whether equivalent Trade-scope information is exposed to gameplay script.

Possible names such as `from_port` or `to_port` are hypotheses only until verified.

## Preferred classification hierarchy

1. Direct gameplay-script sea-route / port information on Trade.
2. Verified arbitrary land/strait connectivity test between market centers.
3. Geographic approximation only if the first two are impossible.

---

# 8. Market shipping capacity cache

## Requirement

For each market:

```text
MarketTransportCapacity(M)
    = sum(
        CountryTransportCapacity(C)
        * MerchantPowerShare(C, M)
      )
```

## Status

**Feasible**

The previously missing Merchant Power inputs are now confirmed.

A cached value should conceptually store:

```text
trade_shipping_available_capacity
```

The exact storage target and variable naming convention should be finalized during the prototype.

---

# 9. Unit conversion / scaling factor

Vanilla `transport_capacity` and `trade_volume` are not naturally expressed in the same scale.

Define:

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

would mean:

```text
1.0 transport_capacity
= 5.0 supported maritime trade_volume
```

The real multiplier should be derived from representative save-game data rather than guessed.

---

# 10. Shipping coverage

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
if SeaTradeDemand(M) <= 0:
    ShippingCoverage(M) = 1
```

## Status

**Feasible once maritime demand classification is solved**

Suggested cached/debug values:

```text
trade_shipping_sea_demand
trade_shipping_available_capacity
trade_shipping_coverage
```

---

# 11. Economic consequence of insufficient coverage

## Status

**Intentionally undecided**

Do not implement a strong gameplay penalty before the upstream calculation works and can be inspected in-game.

Potential later targets include:

- trade efficiency,
- merchant capacity,
- merchant power,
- trade profitability,
- quantity of goods moved,
- market access,
- other market-level economic modifiers.

The first milestone should expose the calculated values for debugging before applying consequences.

---

# 12. Update frequency

## Recommended V1

Use a monthly recalculation.

Reasons:

- ship ownership changes relatively infrequently,
- Merchant Power does not require daily precision for this abstraction,
- Trade volumes can be sampled monthly,
- monthly calculation limits script cost,
- cached values are easier to inspect and audit.

---

# 13. Revised V1 calculation pipeline

## Step 1 — Calculate country shipping pools

```text
for each country C:
    CountryTransportCapacity(C) = 0

    for each naval unit owned by C:
        for each ship/subunit:
            CountryTransportCapacity(C) += ship transport_capacity
```

## Step 2 — Reset market caches

```text
for each market M:
    MarketTransportCapacity(M) = 0
    SeaTradeDemand(M) = 0
```

## Step 3 — Distribute each country's pool by Merchant Power share

Preferred architecture:

```text
for each country C:
    for each M in C.every_market_with_merchants:
        if TotalMerchantPower(M) > 0:
            share = MerchantPower(C, M) / TotalMerchantPower(M)

            MarketTransportCapacity(M) +=
                CountryTransportCapacity(C) * share
```

The key values are confirmed as:

```txt
"merchant_power_in_market(country_scope)"
total_merchant_power
```

## Step 4 — Calculate maritime Trade demand

```text
for each Trade T:
    if T requires maritime transport:
        T.from_market.SeaTradeDemand += T.trade_volume
```

This is the remaining unresolved core step because the maritime classifier is not yet confirmed.

## Step 5 — Calculate coverage

```text
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

# 14. Main technical blockers

## Blocker 1 — Maritime-route classification

**Current primary blocker.**

Need a reliable way to determine whether an outgoing Trade requires maritime transport.

## Blocker 2 — Effective ship transport-capacity getter

A fleet-level getter exists in the GUI, but normal gameplay-script access has not been found.

This already has a practical fallback: reconstruct capacity from ship type/category definitions.

## Resolved — Merchant Power share

No longer a blocker.

Confirmed gameplay-script values:

```text
merchant_power_in_market(country)
total_merchant_power
```

Confirmed Vanilla share formula:

```text
merchant_power_in_market(country) / total_merchant_power
```

## Resolved — Relevant market iteration

No longer a blocker.

Confirmed iterators/predicates include:

```text
every_market_with_merchants
every_market_in_world
has_merchant
every_merchant_in_market
```

---

# 15. Feasibility matrix

| Component | Status | V1 blocker? | Current fallback |
|---|---|---:|---|
| Iterate country units | **Confirmed** | No | — |
| Iterate ship subunits | **Confirmed** | No | — |
| Identify subunit type | **Confirmed** | No | — |
| Identify naval category | **Confirmed** | No | — |
| Vanilla `transport_capacity` exists | **Confirmed** | No | — |
| Direct gameplay getter for effective fleet transport capacity | **Partially confirmed** | No | Reconstruct from ship definitions |
| Country-wide transport-capacity cache | **Feasible** | No | Script variable |
| `merchant_power_in_market(country)` | **Confirmed** | No | — |
| `total_merchant_power` | **Confirmed** | No | — |
| Merchant Power share | **Confirmed** | No | Divide country power by total power |
| `every_market_with_merchants` | **Confirmed** | No | `every_market_in_world` + `has_merchant` |
| `every_merchant_in_market` | **Confirmed** | No | — |
| Trade `trade_volume` | **Confirmed** | No | — |
| Trade `from_market` / `to_market` | **Confirmed** | No | — |
| Determine sea-required Trade | **Open** | **Yes** | Connectivity approximation only if verified |
| Market shipping-capacity sum | **Feasible** | No | Cached variable/value |
| Shipping coverage calculation | **Feasible** | No | Cached variable/value |
| Final economic penalty | **Open by design** | No | Delay until calculation works |

---

# 16. V1 non-goals

V1 explicitly does **not** model:

- fleet position,
- port assignment,
- naval missions,
- physical shipping lanes,
- individual route capacity allocation,
- distance-weighted shipping cost,
- simultaneous overcommitment of the same national fleet across several markets,
- military use reducing commercial capacity,
- blockade effects,
- piracy effects,
- wartime disruption,
- convoy escorts,
- per-country shortages inside the same market.

These can be reconsidered only after the global-pool model is functional and produces useful gameplay.

---

# 17. Source evidence for Merchant Power research

The Merchant Power result is based on Vanilla game files, not inferred from GUI behavior.

## Country-specific Merchant Power and total market Merchant Power

```text
reference_game_files/game/in_game/common/scripted_relations/deny_market_access.txt
```

Vanilla reads:

```text
merchant_power_in_market(country_scope)
total_merchant_power
```

and directly divides the former by the latter.

## Trigger registration

```text
reference_game_files/game/in_game/common/trigger_localization/market_triggers.txt
```

contains entries for:

```text
merchant_power_in_market
total_merchant_power
```

## Direct scoped numeric use

```text
reference_game_files/game/in_game/common/missions/generic_capital_economy_mission_pack.txt
```

uses:

```text
scope:mission_target_market.merchant_power_in_market(root)
```

as a numeric script value.

## Country -> relevant markets

Vanilla uses:

```text
every_market_with_merchants
```

in multiple events, including economy and flavor content.

## World-market alternative

```text
reference_game_files/game/in_game/events/DHE/flavor_por.txt
```

uses:

```txt
every_market_in_world = {
    limit = {
        has_merchant = root
    }
}
```

and also uses `any_market_with_merchants` / `random_market_with_merchants`.

## Market -> merchant countries

Vanilla uses:

```text
every_merchant_in_market
```

in multiple events and scripted relations.

---

# 18. Next research tasks

Research should now proceed in this order:

1. **Find the strongest reliable maritime-Trade classifier.**
2. Search for a direct gameplay getter for effective ship/subunit `transport_capacity`.
3. Prototype the country-first Merchant Power distribution using `every_market_with_merchants`.
4. Verify the preferred cache/storage scopes and monthly reset/update architecture.
5. Add debug output/UI exposing:
   - country transport capacity,
   - country Merchant Power share in selected markets,
   - market sea-trade demand,
   - market available shipping capacity,
   - shipping coverage.
6. Collect save-game samples to choose `SHIPPING_CAPACITY_MULTIPLIER`.
7. Only after validation, design the gameplay penalty for insufficient coverage.

---

# 19. Current feasibility conclusion

The Merchant Power side of the V1 concept is now technically confirmed.

EU5 gameplay script exposes all primitives required to distribute a country's global shipping pool across its commercially influenced markets:

```text
merchant_power_in_market(country)
total_merchant_power
every_market_with_merchants
```

Vanilla itself demonstrates the exact share calculation required by the design.

Therefore the main unresolved requirement for an exact V1 implementation is now **maritime classification of individual Trade objects**.

The absence of a direct gameplay getter for effective fleet `transport_capacity` is secondary because a ship-definition lookup provides a workable fallback.
