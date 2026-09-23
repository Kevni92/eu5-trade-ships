# Trade Pathfinding and Maritime Route Classification Research

## Purpose

This document records the research into whether Europa Universalis V exposes enough of its Trade pathfinding system to determine, from normal gameplay script, whether an individual Trade uses maritime transport.

The target question is:

```text
Given Trade T:
    can script determine whether T's actual route traverses sea?
```

This is required for the V1 demand formula:

```text
SeaTradeDemand(M)
    = sum(trade_volume)
      for outgoing trades from M
      whose route requires maritime transport
```

---

# Executive conclusion

The engine clearly computes and stores a detailed Trade route internally.

Confirmed engine/data-model concepts include:

```text
Trade.GetFromPort
Trade.GetToPort
TradePathItem.GetLocation
TradePathItem.GetAdjacencyCost
TradePathItem.HasPreviousLocation
```

However, the currently verified normal gameplay-script interface does **not** expose:

```text
Trade path list
TradePathItem iterator
from_port / to_port Trade scope targets
uses_sea / is_sea_trade flag
route distance
route cost
sea-distance share
land-distance share
```

The official script datatype dump exposes `Trade.MakeScope`, but not the GUI/data-model functions above. `TradePathItem` is also absent from the script datatype dump.

Therefore:

> **The engine has the information required for an exact maritime classifier, but no verified normal gameplay-script bridge to that information has been found.**

This keeps exact maritime Trade classification blocked for a pure gameplay-script implementation.

---

# 1. `can_find_trade_route`

## Status

**Confirmed gameplay trigger; reachability only**

Vanilla uses:

```txt
can_find_trade_route = {
    from = <market>
    to = <market>
}
```

The `create_trade` generic action explicitly comments that this is an expensive pathfinding check for human market selection.

This proves that normal gameplay logic can ask the engine whether a valid Trade route exists.

What the trigger returns:

```text
boolean route reachability
```

What it does not expose:

```text
path locations
selected ports
route cost
route length
land/sea composition
```

No alternate arguments were found that constrain or report route medium.

Conclusion:

```text
can_find_trade_route != maritime classifier
```

---

# 2. Trade object: selected ports exist engine-side

## Status

**Confirmed engine/GUI data model; no gameplay-script accessor found**

The official data model exposes:

```text
Trade.GetFromPort -> Location
Trade.GetToPort   -> Location
```

Vanilla `trade_details.gui` directly sets its data context to these values for the rows labelled as exporting/importing through a port.

The Trade object therefore knows port endpoints associated with its route.

This is important because a direct gameplay equivalent would give us a strong maritime signal.

### Missing bridge

Targeted searches found no normal Trade-scope equivalent such as:

```txt
from_port
to_port
has_from_port
has_to_port
```

The official script datatype dump contains:

```text
Trade
Trade.MakeScope
```

but does not expose `Trade.GetFromPort` or `Trade.GetToPort` there.

Conclusion:

> Treat Trade port endpoints as GUI/data-model-only until a gameplay-script bridge is demonstrated.

---

# 3. Internal Trade path representation

## Status

**Confirmed engine/data-model type**

The official uncategorized datatype dump contains:

```text
TradePathItem
```

with:

```text
TradePathItem.GetLocation        -> Location
TradePathItem.GetAdjacencyCost   -> CFixedPoint
TradePathItem.HasPreviousLocation -> bool
```

This is the strongest evidence found so far that EU5 internally represents the Trade path as a sequence of locations/adjacencies with explicit per-edge cost.

Conceptually, an exposed path could support:

```text
for each TradePathItem P in TradePath(T):
    location = P.GetLocation
    edge_cost = P.GetAdjacencyCost
```

A maritime classifier could then inspect whether any path location is a sea zone or whether any adjacency is a sea transition.

A route-cost model could also sum:

```text
RouteCost(T)
    = sum(P.GetAdjacencyCost)
```

## Critical limitation

No accessor returning a `TradePathItem` or a collection of `TradePathItem` objects was found.

Targeted searches returned no:

```text
Trade.GetPath
Trade.GetTradePath
Trade.GetRoute
GetTradePath
AccessTradePath
Return type: TradePathItem
```

Repository searches also found no Vanilla GUI/script use of:

```text
TradePathItem
GetAdjacencyCost
```

outside the generated datatype documentation.

Therefore the type exists, but is currently **orphaned from the mod-visible API** as far as the checked corpus shows.

---

# 4. Normal Trade gameplay-script surface

## Status

**Confirmed limited surface**

Vanilla Trade trigger localization exposes:

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

Normal gameplay also supports Trade iteration and the `from_market` / `to_market` relationships used by Vanilla.

No normal Trade trigger/value was found for:

```text
route
path
port
sea
land
route_cost
route_distance
```

This means the values required by the shipping concept are split across two API layers:

```text
Gameplay script:
    trade_volume
    from_market
    to_market
    owner / trade scope logic

Engine / GUI data model:
    GetFromPort
    GetToPort
    TradePathItem
    GetAdjacencyCost
```

The missing piece is a bridge between these layers.

---

# 5. Additional engine-side geographic helpers

The uncategorized data model also exposes GUI-facing methods such as:

```text
Location.GetClosestPort
Location.GetPortSeaZone
Location.HasPort
```

These confirm that the engine has explicit port and sea-zone relationships available to UI code.

They were not found in the normal script datatype dump and should not be assumed usable from gameplay effects/triggers.

---

# 6. `distance_to` and `distance_to_squared`

## Status

**Confirmed gameplay-script numeric values**

Normal gameplay script can evaluate arbitrary Location distance:

```txt
"location_a.distance_to(location_b)"
"location_a.distance_to_squared(location_b)"
```

Vanilla uses these values for geographic weighting and sorting.

This is useful for balancing and approximate shipping-distance scaling.

It is **not** evidence of the actual Trade path.

There is no verified indication that `distance_to` includes:

- roads,
- route pathfinding,
- selected ports,
- sea currents,
- Trade access constraints,
- Trade adjacency costs.

Therefore:

```text
distance_to != Trade route cost
```

---

# 7. Connectivity runtime test result

The runtime connectivity test produced:

```text
London -> Oxford:    YES / YES
Kobenhavn -> Malmo:  NO  / NO
Paris -> Madrid:     NO  / NO
Rome -> Naples:      NO  / NO
London -> Paris:     NO  / NO
```

where:

```text
GENERAL = is_connected_to
REALM   = is_connected_to_through_realm
```

This rejects `is_connected_to` as a transitive arbitrary land-path classifier.

In particular:

```text
Paris -> Madrid = NO
Rome -> Naples  = NO
```

show that `NO` cannot be interpreted as "sea transport required".

The connectivity fallback is therefore no longer considered viable for V1.

See `03_connectivity_runtime_test.md` for the full test record.

---

# 8. Feasibility matrix

| Capability | Engine exists | Gameplay script | GUI/data model | V1 usefulness |
|---|---:|---:|---:|---|
| Trade reachability | Yes | **Yes** (`can_find_trade_route`) | Yes | insufficient alone |
| Trade volume | Yes | **Yes** | Yes | required |
| From/to market | Yes | **Yes** | Yes | required |
| Arbitrary geometric Location distance | Yes | **Yes** | Yes | optional approximation |
| Selected Trade from-port | Yes | Not found | **Yes** | highly useful if bridged |
| Selected Trade to-port | Yes | Not found | **Yes** | highly useful if bridged |
| Trade path item | **Yes** | Not found | engine/data model | ideal if bridged |
| Per-adjacency Trade path cost | **Yes** | Not found | engine/data model | ideal if bridged |
| Direct `uses_sea` flag | Not found | Not found | Not found | ideal but absent |
| Direct route distance/cost getter | route cost exists internally | Not found | no direct Trade getter found | blocked |

---

# 9. Consequence for V1 architecture

The remaining V1 blocker is no longer uncertainty about whether EU5 calculates a route.

It clearly does.

The blocker is specifically:

```text
No verified gameplay-script access to the computed route or its selected ports.
```

Therefore a pure Jomini implementation currently has three realistic directions.

## Direction A — Find an undocumented bridge

Continue searching/testing for a way to move from a Trade script scope into its GUI/data-model route information.

Highest-value targets:

```text
Trade.GetFromPort
Trade.GetToPort
TradePathItem
```

This is the preferred outcome.

## Direction B — GUI-assisted research only

A temporary GUI/debug modification can expose port information for real Trades and help us understand Vanilla routing behavior.

This would be useful for reverse-engineering and balancing, but does not by itself make the values available to gameplay effects.

## Direction C — Approximate classifier

Use script-visible geography to create a custom approximation.

This would no longer be the actual Vanilla Trade route and must be documented as such.

The tested `is_connected_to` approach is rejected. Any replacement approximation would need a different algorithm, potentially using Location-neighbor traversal and explicit land/sea graph logic.

Such reconstruction is substantially more complex and may be expensive if evaluated for every Trade.

---

# 10. Current recommendation

Do **not** implement a guessed maritime classifier yet.

The next technical experiment should be a small Trade GUI/debug overlay that displays for selected real Trades:

```text
From Market
To Market
From Port
To Port
Trade Volume
Assigned Merchant Capacity
```

This would answer two important empirical questions:

1. What do `GetFromPort` / `GetToPort` return for purely overland Trades?
2. Do port values reliably distinguish Trades that use a maritime path?

If the answer is yes, the remaining problem becomes purely one of finding or creating a script bridge to an already reliable engine classification.
