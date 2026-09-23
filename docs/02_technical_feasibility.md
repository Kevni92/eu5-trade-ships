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
| **Partially confirmed** | The underlying engine value exists, but normal gameplay-script access is not confirmed. |
| **Workaround available** | No direct getter is known, but the value can be reconstructed from script-visible data. |
| **Open** | A required script interface or semantic detail has not yet been verified. |
| **Not found** | Targeted research found no corresponding gameplay-script interface in the checked Vanilla/reference corpus. This is not proof that no engine implementation exists. |

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

The pattern is:

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

## Zero-total handling

The implementation must protect against division by zero:

```text
if TotalMerchantPower(M) > 0:
    share = MerchantPower(C, M) / TotalMerchantPower(M)
else:
    share = 0
```

---

# 4. Enumerating relevant markets and merchants

## Requirement

For each country, find every market in which that country participates commercially and calculate its Merchant Power share.

## Status

**Confirmed**

## Preferred country-first iterator

Vanilla provides:

```txt
every_market_with_merchants = {
    ...
}
```

This makes the preferred V1 architecture viable:

```text
Country
  -> every_market_with_merchants
  -> merchant_power_in_market(country)
  -> total_merchant_power
  -> calculate share
  -> add weighted shipping contribution to market
```

A defensive filter can still require:

```txt
"merchant_power_in_market(scope:shipping_country)" > 0
```

## Alternative iterators

Vanilla also provides the world-market pattern:

```txt
every_market_in_world = {
    limit = {
        has_merchant = root
    }
}
```

and Market scope supports:

```txt
every_merchant_in_market = {
    ...
}
```

Both country-first and market-first architectures are therefore possible. Country-first is preferred for V1 because the country transport-capacity pool is already cached on Country scope.

---

# 5. Confirmed V1 market-allocation formula

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

## Confirmed Trade data

Trade scope exposes the values/triggers:

```text
goods
trade_volume
trade_buy
trade_sell
trade_profit
trade_capacity_usage_percent
is_export
is_locked
```

Vanilla also exposes `from_market` and `to_market` from Trade scope and uses them in normal gameplay script.

`trade_volume` is therefore the preferred demand basis for this mod. The custom system models cargo volume, not Vanilla Merchant Capacity.

## Open part

A reliable normal-gameplay-script classification of whether an individual Trade requires sea transport remains unresolved.

---

# 7. Classifying a Trade as maritime

## Requirement

Given:

```text
trade.from_market.location
trade.to_market.location
```

determine whether the actual Trade route requires maritime transport.

## Status

**Open / current primary V1 blocker**

This research pass resolves several sub-questions, but does not yet provide an exact classifier.

## 7.1 Engine-side port endpoints exist

The Vanilla Trade GUI directly uses:

```text
Trade.GetFromPort
Trade.GetToPort
```

The corresponding UI rows are labelled as exporting/importing through a port.

This is strong evidence that the engine's Trade object internally knows the port endpoints selected for a route.

### Important limitation

These functions are GUI/data-model functions. Their existence does **not** imply equivalent gameplay-script event targets.

The official script datatype dump confirms that `Trade` is a script scope (`Trade.MakeScope`), but the normal Trade trigger surface found in Vanilla does not expose these GUI methods.

## 7.2 Gameplay-script `from_port` / `to_port` targets were not found

Targeted repository searches found no verified Trade-scope expressions equivalent to:

```text
from_port
to_port
uses_sea
is_sea_trade
trade_distance
```

`from_port` string matches found in Vanilla were unrelated names such as `chinese_treasure_depart_from_port`, not Trade event targets.

The Vanilla Trade trigger-localization file contains only:

```text
goods
trade_volume
trade_buy
trade_sell
trade_profit
trade_capacity_usage_percent
is_export
is_locked
```

Therefore, based on the currently checked Vanilla/reference corpus:

> **The engine knows Trade port endpoints, but no normal gameplay-script access to those endpoints has been found.**

This should be treated as **GUI-only until proven otherwise**.

## 7.3 `can_find_trade_route` does not classify route medium

Vanilla exposes the Country trigger:

```txt
can_find_trade_route = {
    from = <market>
    to = <market>
}
```

Vanilla's `create_trade` action uses it as an expensive pathfinding/reachability check when selecting two markets.

What is verified:

```text
can_find_trade_route(from, to)
    -> whether the country can find a valid Trade route
```

What is **not** exposed by the trigger:

- the path itself,
- locations traversed,
- whether sea zones are used,
- selected ports,
- land distance vs sea distance.

Therefore `can_find_trade_route` cannot by itself answer the mod's maritime-classification question.

## 7.4 `is_connected_to_through_realm` is precisely documented, but too narrow

Official script documentation defines:

```txt
is_connected_to_through_realm
```

as checking whether a Location is connected by **land/strait** to another Location inside the same realm, including the top overlord and subjects.

This makes its semantics useful and reliable, but it cannot classify arbitrary international Market pairs.

For example, an overland Trade between market centers belonging to unrelated countries is outside the guarantee given by this trigger.

Conclusion:

> `is_connected_to_through_realm` is not a general-purpose world Trade route classifier.

## 7.5 `is_connected_to` exists, but its exact semantics remain insufficiently documented

The normal Location trigger:

```txt
is_connected_to = <location>
```

is real and widely used by Vanilla.

Examples use it to check connectivity to a capital or between named Locations. However, in the available exact/reference documentation researched so far, no authoritative description was found that establishes all of the following properties needed by this mod:

1. that it searches arbitrary international geography,
2. that it is strictly land/strait connectivity,
3. that it ignores ownership/realm boundaries,
4. that its result corresponds to whether the Trade routing system can avoid sea transport.

Because the newer `is_connected_to_through_realm` trigger is explicitly documented while the older `is_connected_to` semantics are not, V1 must not silently assume these properties.

### Consequence

The tempting classifier:

```text
if from_market.location is_connected_to to_market.location:
    land trade
else:
    maritime trade
```

is **not yet accepted as verified implementation logic**.

It may become a viable approximation after controlled in-game tests, but it is not currently source-proven.

## 7.6 Current classification options

### Option A — Direct Trade port/path data

**Preferred, but currently unavailable in gameplay script.**

Ideal logic would be based on `Trade.GetFromPort` / `Trade.GetToPort` or a route/path object. No gameplay-script equivalent has been found.

### Option B — `is_connected_to` fallback

**Potentially viable, but requires runtime verification.**

If controlled tests demonstrate that `is_connected_to` means arbitrary land/strait connectivity across realms, V1 can classify:

```text
land-required = from_location is_connected_to to_location
sea-required  = NOT land-required
```

This would classify based on whether a land/strait connection exists between market centers, not necessarily the exact route chosen internally by the Trade engine.

### Option C — Geographic approximation

**Last resort only.**

Examples could combine coastal status and a verified connectivity test. Pure tests such as "both markets are coastal" are insufficient because two coastal markets can still be connected overland.

No geographic approximation should be implemented before Option B is tested.

## 7.7 Required runtime verification matrix

The next prototype should test `is_connected_to` against known Location pairs representing distinct cases:

| Case | Expected property to test |
|---|---|
| Two Locations on the same continuous landmass | should be connected if trigger is geographic land connectivity |
| Two Locations separated by ordinary sea | should not be connected |
| Island to mainland | should not be connected unless a strait counts |
| Locations connected by a defined strait | determines whether straits count |
| Locations in different independent countries but same landmass | tests whether realm/ownership matters |
| Two coastal Locations with an obvious overland path | verifies that coastal status does not force maritime classification |

The test must compare `is_connected_to` with `is_connected_to_through_realm` where applicable, and should be run in the exact target game version.

## 7.8 Current conclusion

The maritime-classifier problem is narrower than before:

- Trade port endpoints: **confirmed engine-side, GUI access only so far**.
- Direct gameplay Trade sea/path flag: **not found**.
- `can_find_trade_route`: **confirmed reachability only**.
- `is_connected_to_through_realm`: **confirmed land/strait semantics, realm-limited**.
- `is_connected_to`: **confirmed trigger, exact semantics not sufficiently documented for V1 yet**.

The highest-value next action is therefore a small runtime/debug test of `is_connected_to`, rather than more speculative script syntax.

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

The first milestone should expose calculated values for debugging before applying consequences.

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

```text
for each country C:
    for each M in C.every_market_with_merchants:
        if TotalMerchantPower(M) > 0:
            share = MerchantPower(C, M) / TotalMerchantPower(M)

            MarketTransportCapacity(M) +=
                CountryTransportCapacity(C) * share
```

Confirmed values:

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

This remains the unresolved core step.

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

The engine knows the selected Trade ports, but no gameplay-script accessor for those endpoints has been found. The best remaining candidate is a validated `is_connected_to`-based classifier.

## Blocker 2 — Effective ship transport-capacity getter

A fleet-level getter exists in the GUI, but normal gameplay-script access has not been found.

This already has a practical fallback: reconstruct capacity from ship type/category definitions.

## Resolved — Merchant Power share

Confirmed gameplay-script values:

```text
merchant_power_in_market(country)
total_merchant_power
```

## Resolved — Relevant market iteration

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
| Identify subunit type/category | **Confirmed** | No | — |
| Vanilla `transport_capacity` exists | **Confirmed** | No | — |
| Direct gameplay getter for effective fleet transport capacity | **Partially confirmed** | No | Reconstruct from ship definitions |
| Country-wide transport-capacity cache | **Feasible** | No | Script variable |
| Merchant Power in market, numeric | **Confirmed** | No | — |
| Total Merchant Power | **Confirmed** | No | — |
| Merchant Power share | **Confirmed** | No | Divide power by total |
| Relevant market iteration | **Confirmed** | No | Multiple iterator architectures available |
| Trade `trade_volume` | **Confirmed** | No | — |
| Trade `from_market` / `to_market` | **Confirmed** | No | — |
| Engine Trade port endpoints | **Confirmed GUI-side** | No by itself | `Trade.GetFromPort`, `Trade.GetToPort` |
| Gameplay Trade port endpoints | **Not found** | **Yes** for exact port classifier | Test connectivity fallback |
| Direct Trade `uses_sea` / sea-route flag | **Not found** | **Yes** for exact classifier | Test connectivity fallback |
| `can_find_trade_route` | **Confirmed** | No | Reachability only; no route-medium output |
| `is_connected_to_through_realm` | **Confirmed** | No | Land/strait, but realm-limited |
| `is_connected_to` | **Confirmed trigger; semantics open** | **Yes** | Runtime verification required |
| Market shipping-capacity sum | **Feasible** | No | Script variable |
| Shipping coverage calculation | **Feasible** | No | Script values/variables |
| Final economic penalty | **Open by design** | No | Delay until calculation works |

---

# 16. V1 non-goals

The first implementation explicitly does **not** model:

- fleet position,
- port assignment of player fleets,
- naval missions,
- physical allocation of ships to individual Trade routes,
- distance-weighted shipping cost,
- simultaneous overcommitment of the same national fleet across several markets,
- military use of transport capacity reducing commercial capacity,
- blockade effects,
- piracy effects,
- wartime disruption,
- convoy escorts,
- per-country shortages inside the same market.

These can be reconsidered only after the global-pool model is functional and produces useful gameplay.

---

# 17. Next research and prototype tasks

Proceed in this order:

1. **Build a minimal exact-version runtime/debug test for `is_connected_to`.**
   - same landmass,
   - cross-border land connection,
   - ordinary sea separation,
   - island/mainland,
   - strait connection,
   - coastal locations with an overland connection.
2. Compare results with `is_connected_to_through_realm` where applicable.
3. If `is_connected_to` proves suitable, define and document the V1 maritime classifier explicitly.
4. If it does not, inspect exact-version generated `script_docs` for any Trade route/path targets not present in the current reference corpus.
5. Continue searching for a direct gameplay getter for effective ship/fleet `transport_capacity`; otherwise implement the centralized lookup fallback.
6. Prototype cached country and market variables.
7. Add debug output/UI exposing:
   - country transport capacity,
   - country Merchant Power share in a selected market,
   - market sea-trade demand,
   - market available shipping capacity,
   - shipping coverage.
8. Only after validation, design the gameplay penalty for insufficient shipping coverage.

---

# 18. Current feasibility conclusion

The global shipping-pool and Merchant-Power allocation parts of V1 are technically well supported by verified gameplay-script primitives.

The remaining fundamental uncertainty is narrowly concentrated in **classifying individual Trade objects as requiring sea transport**.

The engine demonstrably knows the route's port endpoints through `Trade.GetFromPort` and `Trade.GetToPort`, but that information has not been found on the normal gameplay-script surface. `can_find_trade_route` proves only route reachability. `is_connected_to_through_realm` has clear land/strait semantics but is realm-limited. The ordinary `is_connected_to` trigger exists and is the most promising fallback, but its exact semantics must be verified in the target game version before it becomes production logic.

Therefore implementation can proceed on all other V1 components, while the maritime classifier should remain behind a small dedicated research/debug prototype rather than being based on guessed syntax or undocumented assumptions.
