# Connectivity Runtime Test

## Purpose

This runtime test exists to resolve the remaining uncertainty around using location connectivity as a V1 proxy for maritime trade classification.

The engine documentation explicitly describes:

```text
is_connected_to_through_realm
```

as land/strait connectivity inside the same realm.

The broader:

```text
is_connected_to
```

exists and is used by Vanilla, but its exact routing/ownership semantics are not sufficiently documented for the Trade Ships design.

The mod therefore tests both triggers side by side on fixed geography.

## What this test does not test

The engine GUI exposes:

```text
Trade.GetFromPort
Trade.GetToPort
```

but no equivalent normal gameplay-script Trade target has been verified. This runtime mod does not invent `from_port`, `to_port`, `uses_sea`, or similar syntax. Doing so would turn the test itself into invalid script rather than test runtime behavior.

Likewise, this test does not claim that connectivity is identical to the actual route chosen by the trade pathfinder. It only establishes what the two Location connectivity triggers mean in practice.

---

# Test matrix

The event chain runs these cases in order.

| Test | Pair | Purpose |
|---|---|---|
| 1 | London -> Oxford | Same-realm land control |
| 2 | Kobenhavn -> Malmo | Strait-oriented same-realm control |
| 3 | Paris -> Madrid | International continental land |
| 4 | Rome -> Naples | Second international continental land control |
| 5 | London -> Paris | Clear sea separation across the English Channel |

For every pair the game evaluates:

```txt
is_connected_to = <target>
is_connected_to_through_realm = <target>
```

The result screen reports them as:

```text
GENERAL = is_connected_to
REALM   = is_connected_to_through_realm
```

---

# Working hypothesis

The current working hypothesis is:

```text
London -> Oxford     YES / YES
Kobenhavn -> Malmo   YES / YES
Paris -> Madrid      YES / NO
Rome -> Naples       YES / NO
London -> Paris      NO  / NO
```

This is deliberately a hypothesis, not a scripted expected answer.

The actual runtime results are authoritative for the installed game build.

---

# Interpretation

## Pattern A

If the international land pairs return:

```text
GENERAL = YES
REALM   = NO
```

while London -> Paris returns:

```text
GENERAL = NO
REALM   = NO
```

then `is_connected_to` is a viable indicator for broad land/strait connectivity independent of realm ownership.

That would make it useful as a **geographic fallback** for V1.

It still would not prove that a real Trade object chose a land route rather than an available sea route.

## Pattern B

If London -> Paris returns:

```text
GENERAL = YES
```

then `is_connected_to` clearly crosses ordinary sea separation and cannot be used as a no-sea classifier.

## Pattern C

If Paris -> Madrid and Rome -> Naples return:

```text
GENERAL = NO
```

then `is_connected_to` is probably ownership/realm constrained or otherwise unsuitable for arbitrary international route classification.

## Strait case

Kobenhavn -> Malmo is included to check whether the installed build treats this Oresund case as connected under the two triggers.

---

# Running the test

The repository root is itself the mod root.

Expected structure:

```text
eu5-trade-ships/
├── .metadata/
│   └── metadata.json
├── in_game/
│   ├── common/
│   │   └── on_action/
│   │       └── eu5_trade_ships_connectivity_test_on_actions.txt
│   └── events/
│       └── eu5_trade_ships_connectivity_test_events.txt
├── main_menu/
│   └── localization/
│       └── english/
│           └── eu5_trade_ships_connectivity_test_l_english.yml
└── docs/
```

Place or link the repository folder under:

```text
Documents/Paradox Interactive/Europa Universalis V/mod/
```

Enable:

```text
EU5 Trade Ships - Connectivity Test
```

in the launcher playset and start a **new game**.

`on_game_start` triggers the test chain for every human country. The test does not modify game state.

---

# Reporting results

After the fifth test, report the five pairs in this order:

```text
London-Oxford:     GENERAL/REALM
Kobenhavn-Malmo:   GENERAL/REALM
Paris-Madrid:      GENERAL/REALM
Rome-Naples:       GENERAL/REALM
London-Paris:      GENERAL/REALM
```

Example format only:

```text
YES/YES
YES/YES
YES/NO
YES/NO
NO/NO
```

Once those runtime values are known, `02_technical_feasibility.md` can be updated with an evidence-based conclusion about the connectivity fallback.
