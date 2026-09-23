# EU5 Trade Ships

Current repository state: **Trade GUI debug prototype** for inspecting whether real EU5 trades expose maritime port endpoints through `Trade.GetFromPort` and `Trade.GetToPort`.

The repository root is the active mod root.

## Install

Place or link this repository folder into:

```text
Documents/Paradox Interactive/Europa Universalis V/mod/eu5-trade-ships/
```

The folder itself must contain:

```text
.metadata/metadata.json
in_game/gui/trade_details_lateral_view.gui
```

Then enable **EU5 Trade Ships - Trade GUI Debug** in the EU5 launcher playset.

The metadata currently targets EU5 `1.3.*`.

## Run the current test

1. Load a save with the mod enabled.
2. Open the Trade interface.
3. Open a concrete existing trade so the Trade Details lateral view appears.
4. Read the debug card.
5. Compare at least one obvious overland trade and one obvious maritime trade.

The debug view reports:

```text
From Market
To Market
Shipping Volume
Desired Volume
FromPort
ToPort
```

If a port endpoint is invalid, it explicitly shows:

```text
NONE / INVALID
```

## Interpretation target

If clear overland trades have:

```text
FromPort = NONE / INVALID
ToPort   = NONE / INVALID
```

while clear maritime trades have named port locations, then the engine-side port endpoints are a strong maritime-route indicator.

This test only verifies the semantics of the GUI/data-model values. It does not yet make those values available to normal gameplay script.

## Documentation

- `docs/01_trade_shipping_capacity_concept.md` — design model
- `docs/02_technical_feasibility.md` — verified script capabilities and blockers
- `docs/03_connectivity_runtime_test.md` — completed connectivity test and results
- `docs/04_trade_pathfinding_research.md` — Trade path/port research

The older connectivity event test is retained in documentation only and is no longer part of the active root mod.
