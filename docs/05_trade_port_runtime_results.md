# Trade Port Runtime Results

## Purpose

This document records the in-game runtime results of the temporary Trade GUI debug view that exposes:

```text
Trade.GetFromPort
Trade.GetToPort
Trade.GetQuantityOfGoodsActuallyMoved
Trade.GetDesiredGoodsShipped
Location.IsCoastal
Location.HasPort
Location.GetPortSeaZone
```

The test goal was initially to determine whether a valid `GetFromPort` / `GetToPort` pair can be used as an engine-side proxy for `uses_sea`. That hypothesis was rejected. A second runtime pass then tested whether the endpoint locations' `GetPortSeaZone` validity is a better proxy.

## Runtime observations

### Brugge Market -> Paris Market

Observed debug values:

```text
FromPort: Tournai
FromPort IsCoastal: NO
FromPort HasPort: NO
FromPort SeaZone: NONE / INVALID

ToPort: Valenciennes
ToPort IsCoastal: NO
ToPort HasPort: NO
ToPort SeaZone: NONE / INVALID
```

The highlighted route is an overland route. Both endpoint sea zones are invalid.

### Paris Market -> London Market

Observed debug values:

```text
FromPort: Arques
FromPort IsCoastal: YES
FromPort HasPort: YES
FromPort SeaZone: Alabaster Coast

ToPort: London
ToPort IsCoastal: YES
ToPort HasPort: YES
ToPort SeaZone: Thames
```

The route visibly contains a maritime crossing between the continent and England. Both endpoint sea zones are valid.

### London Market -> Köln Market

Observed debug values:

```text
Shipping Volume: 1.24
Desired Volume: 1.24

FromPort: London
FromPort IsCoastal: YES
FromPort HasPort: YES
FromPort SeaZone: Thames

ToPort: Cuijk
ToPort IsCoastal: NO
ToPort HasPort: NO
ToPort SeaZone: NONE / INVALID
```

The route visibly uses a maritime leg from England toward the Low Countries before continuing inland. Exactly one endpoint sea zone is valid.

### London Market -> Sevilla Market

Observed debug values:

```text
Shipping Volume: 1.72
Desired Volume: 1.72

FromPort: London
FromPort IsCoastal: YES
FromPort HasPort: YES
FromPort SeaZone: Thames

ToPort: Sevilla
ToPort IsCoastal: YES
ToPort HasPort: YES
ToPort SeaZone: Bay of Cadiz
```

The route is maritime. Both endpoint sea zones are valid.

### Lübeck Market -> Brugge Market

Observed debug values:

```text
FromPort: Lübeck
FromPort IsCoastal: YES
FromPort HasPort: YES
FromPort SeaZone: Bay of Lübeck

ToPort: Brugge
ToPort IsCoastal: YES
ToPort HasPort: YES
ToPort SeaZone: Scheldt Estuary
```

This case is especially important because Lübeck and Brugge are also connected through continental land geography. The highlighted Trade route nevertheless runs over sea between the two markets. Both endpoint sea zones are valid.

This supports interpreting the classifier as a proxy for the **engine-selected Trade route**, not for whether sea transport is geographically unavoidable.

For the mod design, this is desirable: if the game's route calculation chooses the maritime path because it is cheaper/faster/better, the Trade should consume shipping capacity even when a land alternative exists.

## Rejected hypothesis: valid endpoint locations alone imply sea transport

The original hypothesis remains rejected:

```text
valid GetFromPort/GetToPort
    != proof that the Trade uses sea transport
```

A clearly overland Brugge -> Paris route still returns valid endpoint locations (`Tournai` and `Valenciennes`). Therefore the engine's `FromPort` / `ToPort` terminology must not be interpreted as "seaport used by this Trade" without further evidence.

The values may represent route transfer/entry/exit nodes or a broader engine routing concept. Their exact semantics remain open.

## Current candidate maritime classifier

The runtime pass has produced the following pattern:

| Trade | Observed selected route | From endpoint SeaZone | To endpoint SeaZone | Candidate result |
|---|---|---:|---:|---:|
| Brugge -> Paris | overland | invalid | invalid | NO |
| Paris -> London | maritime | valid | valid | YES |
| London -> Köln | mixed land/sea | valid | invalid | YES |
| London -> Sevilla | maritime | valid | valid | YES |
| Lübeck -> Brugge | maritime despite land alternative | valid | valid | YES |

All tested cases are consistent with the candidate:

```text
uses_shipping =
    Trade.GetFromPort.GetPortSeaZone.IsValid
    OR
    Trade.GetToPort.GetPortSeaZone.IsValid
```

More defensively, accounting for invalid endpoint objects:

```text
uses_shipping =
    (Trade.GetFromPort.IsValid AND Trade.GetFromPort.GetPortSeaZone.IsValid)
    OR
    (Trade.GetToPort.IsValid AND Trade.GetToPort.GetPortSeaZone.IsValid)
```

### Semantics

For the mod, the intended meaning should now be:

```text
uses_shipping =
    the engine-selected Trade relation uses a maritime leg
```

not:

```text
requires_shipping =
    no land route exists
```

This distinction matters for relations such as Lübeck -> Brugge, where a land alternative exists but the actual Trade route selected by the engine runs over sea.

### Status

**Strong runtime hypothesis; still not yet a verified gameplay-script classifier.**

The current evidence includes:

- pure overland -> both endpoint SeaZones invalid,
- pure maritime -> endpoint SeaZones valid,
- mixed maritime + inland -> one endpoint SeaZone valid,
- maritime route chosen despite a land alternative -> endpoint SeaZones valid.

The highest-value remaining false-positive test is a relation where one or both markets are coastal, but the highlighted selected Trade route is unambiguously overland. If such a route still produces invalid endpoint sea zones, confidence rises further.

## GUI side effect observed

With the first minimalist debug override of `trade_details_lateral_view.gui`, the selected Trade route line was drawn when the Trade was opened but disappeared after the monthly Trade refresh. The Trade itself continued to function and its values updated.

The official GUI datatype dump contains:

```text
PdxGuiWidget.SetHighlightTrade( Arg0 )
```

This supports treating the route line as UI highlight state rather than persistent Trade state.

The debug view had replaced the Vanilla panel content and omitted the Vanilla `ExistingTradeCard` structure. The root debug GUI was therefore updated to restore that Vanilla card while retaining the debug readout.

## Performance architecture if the classifier is validated

The classifier should not be recomputed for every Trade every month.

A suitable V1 architecture is a route-classification cache keyed by a market relation, initially treated as directed until route symmetry is proven:

```text
RouteKey = FromMarket -> ToMarket
RouteValue = uses_shipping yes/no
```

If route ownership or country-specific access proves relevant, the owner/country must also be part of the key.

For initialization, prefer iterating existing Trades and classifying each unique unknown key once rather than evaluating every theoretical Market x Market pair. New unknown market relations can be classified lazily when first encountered. This avoids an unnecessary O(M^2) full-market cross product.

Cache invalidation rules remain open until runtime tests establish which game-state changes can alter a route.

## Remaining API limitation

Even if the GUI/data-model classifier proves reliable, normal gameplay-script access is still unresolved. `Trade.GetFromPort`, `Location.GetPortSeaZone`, and related methods are currently verified in the GUI/data-model layer, not in the normal Jomini gameplay-script surface.

The internal Trade path remains a parallel high-value target:

```text
TradePathItem.GetLocation
TradePathItem.GetAdjacencyCost
```

If a mod-visible iterator/accessor for the Trade path can be found, sea-zone traversal could be classified directly.
