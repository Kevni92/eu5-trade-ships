# Trade Port Runtime Results

## Purpose

This document records the in-game runtime results of the temporary Trade GUI debug view that exposes:

```text
Trade.GetFromPort
Trade.GetToPort
Trade.GetQuantityOfGoodsActuallyMoved
Trade.GetDesiredGoodsShipped
```

The test goal was to determine whether a valid `GetFromPort` / `GetToPort` pair can be used as an engine-side proxy for `uses_sea`.

## Runtime observations

### Paris Market -> Brugge Market

Observed debug values:

```text
FromPort: Valenciennes
ToPort: Tournai
```

The highlighted route shown on the map is an overland route from Paris toward Brugge. Despite that, both port endpoints are valid.

### Paris Market -> London Market

Observed debug values:

```text
FromPort: Arques
ToPort: London
```

This route visibly contains a maritime crossing between the continent and England.

### London Market -> Köln Market

Observed debug values after the trade became active:

```text
Shipping Volume: 1.24
Desired Volume: 1.24
FromPort: London
ToPort: Cuijk
```

The route visibly uses a maritime leg from England toward the Low Countries before continuing inland.

## Conclusion

The original hypothesis is rejected:

```text
valid GetFromPort/GetToPort
    != proof that the Trade uses sea transport
```

A clearly overland Paris -> Brugge route still returns valid endpoint locations (`Valenciennes` and `Tournai`). Therefore the engine's `FromPort` / `ToPort` terminology must not be interpreted as "seaport used by this Trade" without further evidence.

The values may instead represent route transfer/entry/exit nodes or a broader engine routing concept. Their exact semantics remain open.

Consequently, `Trade.GetFromPort.IsValid` and `Trade.GetToPort.IsValid` are **not suitable as the V1 maritime classifier**.

## GUI side effect observed

With the first minimalist debug override of `trade_details_lateral_view.gui`, the selected Trade route line was drawn when the Trade was opened but disappeared after the monthly Trade refresh. The Trade itself continued to function and its values updated.

The official GUI datatype dump contains:

```text
PdxGuiWidget.SetHighlightTrade( Arg0 )
```

This supports treating the route line as UI highlight state rather than persistent Trade state.

The debug view had replaced the Vanilla panel content and omitted the Vanilla `ExistingTradeCard` structure. The root debug GUI was therefore updated to restore that Vanilla card while retaining the debug readout, so the next runtime pass can check whether the highlight survives the monthly refresh.

## Implication for maritime classification research

After these tests, the remaining high-value target is still the internal Trade path representation:

```text
TradePathItem.GetLocation
TradePathItem.GetAdjacencyCost
```

If a mod-visible iterator/accessor for the Trade path can be found, sea-zone traversal can be classified directly. No such gameplay-script accessor has yet been verified.
