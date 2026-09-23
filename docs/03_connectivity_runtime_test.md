# Connectivity Runtime Test

## Purpose

This runtime test exists to resolve whether Location connectivity triggers can be used as a V1 proxy for maritime trade classification.

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

The test therefore evaluates both triggers side by side on fixed geography.

## What this test does not test

The engine GUI exposes:

```text
Trade.GetFromPort
Trade.GetToPort
```

but no equivalent normal gameplay-script Trade target has been verified. This runtime mod does not invent `from_port`, `to_port`, `uses_sea`, or similar syntax.

Likewise, this test does not claim that connectivity is identical to the actual route chosen by the trade pathfinder. It only establishes what the two Location connectivity triggers return in practice for the tested pairs.

---

# Test matrix

| Test | Pair | Purpose |
|---|---|---|
| 1 | London -> Oxford | Same-realm land control |
| 2 | Kobenhavn -> Malmo | Strait-oriented control |
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

# Observed runtime results

The test was executed in-game as England. The observed values were:

| Test | Pair | GENERAL | REALM |
|---|---|---:|---:|
| 1 | London -> Oxford | YES | YES |
| 2 | Kobenhavn -> Malmo | NO | NO |
| 3 | Paris -> Madrid | NO | NO |
| 4 | Rome -> Naples | NO | NO |
| 5 | London -> Paris | NO | NO |

Raw report:

```text
YES/YES
NO/NO
NO/NO
NO/NO
NO/NO
```

---

# Interpretation

## `is_connected_to` is not arbitrary transitive land connectivity

The decisive controls are:

```text
Paris -> Madrid = NO
Rome  -> Naples = NO
```

Both are continental land pairs, yet `is_connected_to` returns `NO`.

Therefore the trigger cannot be interpreted as:

```text
"there exists some land/strait path between these two arbitrary locations"
```

This rules out the proposed V1 classifier:

```text
is_connected_to = yes -> land trade
is_connected_to = no  -> maritime trade
```

because it would incorrectly classify continental examples such as Paris -> Madrid and Rome -> Naples as maritime.

## `is_connected_to_through_realm` remains realm-specific

The official description of `is_connected_to_through_realm` remains useful for its intended purpose: checking land/strait connectivity inside a realm.

It is not a general international trade-route classifier.

## Kobenhavn -> Malmo

The Kobenhavn -> Malmo control returned `NO/NO`. This should not be generalized into a statement that straits are unsupported globally; it only proves that this exact pair did not return connected in the tested state/build.

## London -> Paris

London -> Paris also returned `NO/NO`, but this is not evidence that `NO` means "requires sea" because the same result occurs for the continental land controls.

---

# Conclusion

The connectivity fallback is rejected for V1.

Status after runtime verification:

```text
is_connected_to
    confirmed real trigger
    not suitable as arbitrary land-path classifier

is_connected_to_through_realm
    confirmed land/strait semantics
    explicitly realm-limited
    not suitable for arbitrary international trade routes
```

The next research target is the Trade/pathfinding data model itself: route path objects, selected ports, adjacency costs, and any gameplay-script bridge to that information.

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
│       ├── english/
│       └── german/
└── docs/
```

Manual console trigger:

```text
event eu5_trade_ships_connectivity_test.1
```
