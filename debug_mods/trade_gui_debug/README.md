# Trade GUI Debug Mod

Purpose: inspect the engine-side `Trade.GetFromPort` and `Trade.GetToPort` values on real trades.

## Install

Copy `debug_mods/trade_gui_debug` as its own mod folder under:

```text
Documents/Paradox Interactive/Europa Universalis V/mod/
```

Enable **EU5 Trade Ships - Trade GUI Debug** in the launcher.

## Use

1. Load a save.
2. Open the Trade interface.
3. Open a concrete existing trade so the Trade Details lateral view appears.
4. Read the debug card.
5. Compare at least one obvious overland trade and one obvious maritime trade.

The view reports:

```text
From Market
To Market
Shipping Volume
Desired Volume
FromPort
ToPort
```

If a port endpoint is invalid, the view explicitly shows:

```text
NONE / INVALID
```

## Interpretation target

If clear overland trades have `FromPort = NONE / INVALID` and `ToPort = NONE / INVALID`, while clear maritime trades have named ports, then the port endpoints are an excellent engine-side indicator for maritime routing.

This test does not yet make the values available to normal gameplay script; it only verifies their semantics in the GUI/data-model layer.
