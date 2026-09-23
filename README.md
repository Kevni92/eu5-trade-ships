# EU5 Trade Ships

Current repository state: **runtime connectivity test prototype** for the planned maritime shipping-capacity system.

## Install

Place or link this repository folder into:

```text
Documents/Paradox Interactive/Europa Universalis V/mod/eu5-trade-ships/
```

The folder itself must contain:

```text
.metadata/metadata.json
```

Then enable **EU5 Trade Ships - Connectivity Test** in the EU5 launcher playset.

The metadata currently targets EU5 `1.3.*`.

## Run the connectivity test

Start a **new game** with the mod enabled.

At `on_game_start`, a debug event chain opens automatically for the human player. It tests five location pairs and reports two values for each:

```text
GENERAL = is_connected_to
REALM   = is_connected_to_through_realm
```

The test sequence is:

1. London -> Oxford
2. Kobenhavn -> Malmo
3. Paris -> Madrid
4. Rome -> Naples
5. London -> Paris

The final event tells you how to report the result sequence.

No gameplay state is intentionally changed by the test events.

## If the event does not appear

Confirm that:

- the mod is enabled in the active playset,
- `.metadata/metadata.json` is directly below the mod root,
- you started a new game after enabling the mod,
- EU5 was fully restarted after changing the mod structure.

Then inspect:

```text
Documents/Paradox Interactive/Europa Universalis V/logs/error.log
```

for entries mentioning `eu5_trade_ships_connectivity_test`.

## Documentation

- `docs/01_trade_shipping_capacity_concept.md` — design model
- `docs/02_technical_feasibility.md` — verified script capabilities and blockers
- `docs/03_connectivity_runtime_test.md` — exact runtime test and interpretation
