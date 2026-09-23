# Scripted GUI Bridge and Directed Route Cache

## Status

The GUI -> Scripted GUI -> normal Jomini bridge is runtime-confirmed on EU5 1.3.

The current debug Trade view passes the selected Trade plus its From/To Markets through `MakeScope` / `GuiScope.AddScope(...)` into a normal scripted GUI effect.

Observed in-game after pressing the debug bridge button:

```text
Bridge status: JOMINI EFFECT EXECUTED
scope:trade received by Jomini: YES
scope:from_market received: YES
scope:to_market received: YES
```

This was confirmed for both:

- a maritime relation (London Market -> Dublin Market), and
- overland relations (Nürnberg Market -> Köln Market, Paris Market -> Bordeaux Market).

Therefore the mod can use GUI/data-model-only route information to decide a classification, then transfer the selected Trade and Market scopes into normal gameplay script and persist the result there.

## Bridge architecture

The working flow is:

```text
GUI Trade object
    -> Trade.GetFromPort / Trade.GetToPort
    -> endpoint GetPortSeaZone validity
    -> GUI classifier: MARITIME / LAND
    -> Trade.MakeScope
    -> Trade.GetFromMarket.MakeScope
    -> Trade.GetToMarket.MakeScope
    -> GuiScope.AddScope(...)
    -> GetScriptedGui(...).Execute(...)
    -> normal Jomini effect
```

The important point is that `GetPortSeaZone` itself does not need to exist in gameplay script. Only the classifier result needs to cross the GUI/Jomini boundary.

## Current classifier

Current runtime candidate:

```text
uses_shipping =
    (Trade.GetFromPort.IsValid AND Trade.GetFromPort.GetPortSeaZone.IsValid)
    OR
    (Trade.GetToPort.IsValid AND Trade.GetToPort.GetPortSeaZone.IsValid)
```

This means the engine-selected Trade relation uses a maritime leg, not that sea travel is geographically unavoidable.

## Directed relation semantics

A later runtime screenshot showed Brugge Market -> Paris Market classified as maritime while an earlier Paris Market -> Brugge Market observation had been overland.

The exact cause is not yet proven. Possibilities include:

- actual directional route asymmetry,
- route changes between game states,
- or another dynamic routing input.

Until symmetry is explicitly proven, route caching must therefore be directed:

```text
A -> B != B -> A
```

## Directed Market-pair cache prototype

The next debug build writes the classification into a variable map on the `from_market`, keyed by `to_market`:

```text
Map owner: FromMarket
Map name: eu5ts_shipping_routes
Key: ToMarket
Value:
    1 = maritime / uses shipping
    0 = land / does not use shipping
```

Conceptually:

```text
London Market:
    Dublin Market -> 1
    Köln Market   -> 1

Paris Market:
    Bordeaux Market -> 0
```

The scripted GUI removes an existing entry before adding the new value, so rerunning the classifier overwrites the cached relation deterministically.

The Trade debug GUI then reads the map back through:

```text
FromMarket.MakeScope.GetVariableFromVariableMap(
    'eu5ts_shipping_routes',
    ToMarket.MakeScope
)
```

The runtime success criterion is that, after pressing the bridge button, the GUI reports:

```text
Jomini cache write attempted: YES
Cache readback for this From->To pair: MARITIME (1)
```

or:

```text
Jomini cache write attempted: YES
Cache readback for this From->To pair: LAND (0)
```

If this succeeds, we will have proven the complete path:

```text
GUI-only route data
    -> normal Jomini effect
    -> persistent directed Market-pair cache
    -> later lookup without repeating route classification
```

## Remaining problem after cache validation

The next major problem is automatic enumeration/classification of previously unseen market relations without requiring the player to open each Trade manually.

Potential sources already identified include GUI Trade collections such as `ImportExportLateralView.GetTradesInMarket`, but a global or automatically driven enumeration mechanism still needs to be proven.
