# Trade Shipping Capacity – Konzept

## 1. Ziel

Diese Mod soll Seeschifffahrt zu einem eigenständigen wirtschaftlichen Faktor im Handelssystem von Europa Universalis V machen.

Der Kern des Systems ist die Annahme, dass maritime Handelsströme nicht allein durch Merchant Capacity und Merchant Power funktionieren, sondern zusätzlich eine ausreichend große Handels- und Transportflotte benötigen.

Schiffe erhalten dadurch einen wirtschaftlichen Nutzen unabhängig davon, ob sie gerade militärisch eingesetzt werden. Insbesondere Transportschiffe sollen außerhalb von Kriegen dauerhaft wertvoll sein.

Die erste Version des Systems soll bewusst abstrakt und global funktionieren. Es wird **nicht** versucht, einzelne Flotten konkreten Handelsrouten oder Märkten physisch zuzuordnen.

---

## 2. Grundmodell

Das System besteht aus drei Größen:

1. **Sea Trade Demand** eines Marktes
2. **Country Transport Capacity** eines Landes
3. **Market Transport Capacity** eines Marktes

Aus Nachfrage und verfügbarer Kapazität wird anschließend eine **Shipping Coverage** berechnet.

---

## 3. Sea Trade Demand

### 3.1 Definition

`SeaTradeDemand` beschreibt, wie viel ausgehendes Handelsvolumen eines Marktes über See transportiert werden muss.

Die Grundlage ist der Vanilla-Wert:

```text
trade_volume
```

Nicht Merchant Capacity, sondern das tatsächlich gehandelte Volumen ist für dieses System relevant.

### 3.2 Zuordnung zum Ursprungsmarkt

Ein Trade wird ausschließlich dem Markt zugerechnet, aus dem er stammt.

Für einen Trade:

```text
London -> Lisboa
trade_volume = 10
```

wird die Nachfrage nur London zugerechnet:

```text
SeaTradeDemand_London += 10
```

Lisboa erhält durch diesen Trade keine zusätzliche Sea Trade Demand.

Damit wird jeder Trade genau einmal gezählt.

### 3.3 Maritime Klassifikation

Ein Trade soll nur dann zur Sea Trade Demand beitragen, wenn zwischen seinem Ursprungsmarkt und seinem Zielmarkt ein Seetransport erforderlich ist.

Konzeptionell:

```text
Trade
  -> from_market
  -> to_market
  -> prüfen, ob eine reine Land-/Strait-Verbindung existiert
      -> ja: kein Sea Trade Demand
      -> nein: trade_volume zu SeaTradeDemand des from_market addieren
```

Die exakte technische Methode zur sicheren Unterscheidung zwischen Land- und Seetrade ist noch zu verifizieren.

### 3.4 Formel

Für Markt `M`:

```text
SeaTradeDemand(M)
    = Summe aller trade_volume
      für ausgehende Trades aus M,
      die Seetransport benötigen
```

Formal:

```text
SeaTradeDemand_M = Σ trade_volume_t
```

für alle Trades `t` mit:

```text
t.from_market = M
requires_sea(t) = true
```

---

## 4. Country Transport Capacity

### 4.1 Definition

Jedes Land besitzt eine globale maritime Transportkapazität.

Diese wird aus der Vanilla-Statistik der vorhandenen Schiffe abgeleitet:

```text
transport_capacity
```

Es soll **keine separate erfundene Trade-Capacity pro Schiffsklasse** geben. Stattdessen wird die bereits vorhandene Transportfähigkeit der Schiffe als Grundlage verwendet.

### 4.2 Alle Schiffe zählen

Für die erste Version zählt jedes existierende Schiff eines Landes zu dessen globaler Transportkapazität.

Nicht relevant sind:

- aktuelle Position
- aktuelle Flotte
- Mission
- Hafenstatus
- Krieg oder Frieden
- Truppentransport
- aktuelle Nutzung der militärischen Transportkapazität

Die Existenz des Schiffes genügt.

### 4.3 Formel

Für Land `C`:

```text
CountryTransportCapacity_C
    = Σ TransportCapacity_ship
```

über alle Schiffe des Landes.

### 4.4 Designabsicht

Dadurch entsteht ein dauerhafter wirtschaftlicher Nutzen für Schiffe.

Insbesondere Transporttypen erhalten eine neue strategische Rolle:

- im Krieg: militärischer Truppentransport
- im Frieden: Teil der nationalen Handelsschifffahrt

Andere Schiffstypen können ebenfalls beitragen, sofern sie Vanilla-`transport_capacity` besitzen.

---

## 5. Verteilung auf Märkte

### 5.1 Grundidee

Die nationale Transportkapazität wird nicht auf einzelne Märkte aufgeteilt oder verbraucht.

Stattdessen steht sie grundsätzlich in allen Märkten zur Verfügung, in denen das Land Merchant Power besitzt.

Der effektive Beitrag wird durch den Merchant-Power-Anteil des Landes im jeweiligen Markt gewichtet.

### 5.2 Formel pro Land und Markt

Für Land `C` und Markt `M`:

```text
CountryContribution(C, M)
    = CountryTransportCapacity_C
      * MerchantPowerShare(C, M)
```

Beispiel:

```text
England CountryTransportCapacity = 20
```

Merchant-Power-Anteile:

```text
London      = 70 %
Antwerpen   = 25 %
Lisboa      = 10 %
```

Daraus entstehen:

```text
London      = 14
Antwerpen   = 5
Lisboa      = 2
```

Die gleiche nationale Flotte kann somit gleichzeitig zu mehreren Märkten beitragen.

Das ist in Version 1 ausdrücklich beabsichtigt.

### 5.3 Bedeutung der Abstraktion

Das System modelliert nicht die physische Allokation einzelner Schiffe.

Es modelliert stattdessen die allgemeine Fähigkeit einer Handelsnation, maritime Warenströme in Märkten zu bedienen, in denen sie kommerziell präsent ist.

Merchant Power dient dabei als Gewicht für den Zugriff auf die nationale Handelsflotte.

---

## 6. Market Transport Capacity

Die gesamte für einen Markt verfügbare Transportkapazität ergibt sich aus den Beiträgen aller dort kommerziell aktiven Länder.

```text
MarketTransportCapacity_M
    = Σ CountryContribution(C, M)
```

Formal:

```text
MarketTransportCapacity_M
    = Σ (
        CountryTransportCapacity_C
        * MerchantPowerShare(C, M)
      )
```

Dadurch können mehrere Länder gemeinsam die maritime Transportversorgung eines Marktes sicherstellen.

---

## 7. Shipping Coverage

### 7.1 Verhältnis von Angebot und Nachfrage

Die zentrale Kennzahl des Systems ist die maritime Transportabdeckung eines Marktes.

```text
ShippingCoverage_M
    = min(
        1,
        EffectiveMarketTransportCapacity_M
        / SeaTradeDemand_M
      )
```

### 7.2 Scaling Factor

Vanilla-`transport_capacity` und `trade_volume` verwenden voraussichtlich unterschiedliche Größenordnungen.

Daher wird wahrscheinlich ein globaler Skalierungsfaktor benötigt:

```text
EffectiveMarketTransportCapacity_M
    = MarketTransportCapacity_M
      * ShippingCapacityScale
```

Damit ergibt sich:

```text
ShippingCoverage_M
    = min(
        1,
        (MarketTransportCapacity_M * ShippingCapacityScale)
        / SeaTradeDemand_M
      )
```

Der konkrete Wert von `ShippingCapacityScale` ist eine Balancefrage und wird erst nach einem funktionierenden Prototypen festgelegt.

---

## 8. Beispiel

Ein Markt besitzt folgende maritime ausgehende Trades:

```text
Trade A: volume 8
Trade B: volume 12
Trade C: volume 6
```

Alle drei benötigen Seetransport.

```text
SeaTradeDemand = 26
```

Drei Länder besitzen Merchant Power im Markt:

```text
England:
CountryTransportCapacity = 20
MerchantPowerShare = 60 %
Contribution = 12

Netherlands:
CountryTransportCapacity = 16
MerchantPowerShare = 25 %
Contribution = 4

France:
CountryTransportCapacity = 10
MerchantPowerShare = 15 %
Contribution = 1.5
```

Gesamt:

```text
MarketTransportCapacity = 17.5
```

Bei einem hypothetischen Skalierungsfaktor von `1.0`:

```text
ShippingCoverage = 17.5 / 26
                 = 0.673
                 = 67.3 %
```

Der Markt hätte damit ein maritimes Transportdefizit.

---

## 9. Auswirkungen eines Transportdefizits

Die konkrete wirtschaftliche Strafe bei unzureichender Shipping Coverage ist noch nicht festgelegt.

Mögliche Ansätze sind beispielsweise:

- geringere Trade Efficiency
- geringere Merchant Power
- geringere effektive Handelskapazität
- reduzierte Warenbewegung
- zusätzliche Handelskosten
- marktbezogene wirtschaftliche Modifier

Für Version 1 soll zunächst nur die Berechnung von Nachfrage, Kapazität und Coverage zuverlässig funktionieren.

Die ökonomische Wirkung wird anschließend separat entschieden und balanciert.

---

## 10. Version-1-Scope

Die erste Implementierung soll bewusst einfach bleiben.

### Enthalten

- `trade_volume` als maritime Nachfrage
- Sea Trade Demand pro Ursprungsmarkt
- globale Country Transport Capacity
- alle existierenden Schiffe eines Landes zählen
- Vanilla-`transport_capacity` als Grundlage
- Gewichtung über Merchant-Power-Anteil im Markt
- Summierung aller Länderbeiträge pro Markt
- Shipping Coverage pro Markt
- konfigurierbarer Skalierungsfaktor

### Nicht enthalten

- Zuordnung einzelner Flotten zu Märkten
- Flottenpositionen
- konkrete Seehandelsrouten
- Entfernung von Handelsrouten
- Schiffsmissionen
- Flotten- oder Hafenstatus
- Berücksichtigung aktuell transportierter Armeen
- Verbrauch von Transportkapazität durch einzelne Trades
- exklusive Allokation eines Schiffes auf nur einen Markt
- Piraterieinteraktion
- Blockadeinteraktion
- Kriegsmodifikatoren
- dynamische Routenreservierung

Diese Einschränkungen sind Designentscheidungen für den ersten Prototypen und keine Aussage darüber, was später technisch möglich sein könnte.

---

## 11. Technischer Berechnungsablauf

Konzeptionell kann ein Update-Zyklus in drei Phasen erfolgen.

### Phase A – Country Transport Capacity

Für jedes Land:

```text
CountryTransportCapacity = 0

für alle eigenen Schiffe:
    CountryTransportCapacity += effektive transport_capacity des Schiffes
```

### Phase B – Sea Trade Demand

Für jeden Markt:

```text
SeaTradeDemand = 0

für alle Trades mit from_market = dieser Markt:
    wenn Trade Seetransport benötigt:
        SeaTradeDemand += trade_volume
```

### Phase C – Market Transport Capacity

Für jeden Markt:

```text
MarketTransportCapacity = 0

für jedes Land mit Merchant Power im Markt:
    MarketTransportCapacity +=
        CountryTransportCapacity
        * MerchantPowerShare
```

Danach:

```text
ShippingCoverage =
    min(
        1,
        MarketTransportCapacity
        * ShippingCapacityScale
        / SeaTradeDemand
    )
```

Sonderfall:

```text
SeaTradeDemand = 0
```

soll als vollständig versorgt behandelt werden:

```text
ShippingCoverage = 1
```

---

## 12. Bereits bestätigte EU5-Bausteine

Die bisherige Recherche hat folgende relevante Engine-/Script-Bausteine bestätigt:

- Trades besitzen `trade_volume`.
- Trades besitzen `from_market` und `to_market`.
- Trades können über Trade-Iteratoren untersucht werden.
- Units können über Country-Iteratoren durchlaufen werden.
- Units können ihre Subunits iterieren.
- Subunits können nach `sub_unit_type` und `sub_unit_category` unterschieden werden.
- Schiffstypen und Schiffskategorien besitzen den Naval Modifier `transport_capacity`.
- Die GUI kennt für eine Navy eine aggregierte maximale, belegte und verfügbare Transportkapazität.
- Locations besitzen eine Zuordnung zu einem Market.

Nicht alle dieser GUI-Werte sind automatisch auch als normale Gameplay-Script-Values verfügbar. Die Implementierung muss daher für jede benötigte Größe separat verifizieren, welcher Zugriff im Jomini-Script tatsächlich möglich ist.

---

## 13. Offene technische Fragen

Vor der eigentlichen Implementierung müssen insbesondere folgende Punkte verifiziert werden:

### 13.1 Transport Capacity eines Landes

Zu klären:

- Gibt es einen direkt auslesbaren Gameplay-Script-Wert für die aggregierte Transportkapazität einer Unit oder Navy?
- Falls nicht: Wie wird die effektive `transport_capacity` eines einzelnen Subunits zuverlässig rekonstruiert?
- Müssen Unit-Type-, Template- und Category-Werte separat berücksichtigt werden?
- Wie wirkt beschädigte `subunit_strength` auf die effektive Transportkapazität?

### 13.2 Merchant Power Share

Benötigt wird für jedes Paar aus Country und Market:

```text
MerchantPowerShare(C, M)
```

Zu verifizieren:

- direkter Script-Getter für Merchant Power eines Landes in einem Markt
- direkter Getter für gesamten Merchant Power des Marktes
- alternativ direkter Getter für den prozentualen Anteil

### 13.3 Maritime Trade Classification

Benötigt wird:

```text
requires_sea(trade)
```

Zu verifizieren:

- ob ein Trade direkt From/To Port exponiert
- ob daraus zuverlässig Seetransport abgeleitet werden kann
- alternativ, ob eine allgemeine Land-/Strait-Konnektivität zwischen zwei Marktzentren geprüft werden kann

### 13.4 Speicherung auf Market Scope

Zu prüfen:

- ob Market Scope Variablen für die benötigten Cache-Werte geeignet sind
- wie Werte effizient aktualisiert und zurückgesetzt werden
- wie häufig der gesamte Rechenzyklus ausgeführt werden sollte

---

## 14. Balancefragen für später

Erst nach einem technisch funktionierenden Prototypen sollen folgende Punkte festgelegt werden:

- `ShippingCapacityScale`
- gewünschte durchschnittliche Shipping Coverage
- Stärke der Defizitstrafen
- Progression der Transportflotten über die Ages
- Verhältnis zwischen Kriegsschiffen und dedizierten Transporten
- möglicher wirtschaftlicher Wert beschädigter Schiffe
- mögliche Boni für maritime Handelsnationen
- mögliche Wechselwirkungen mit Piraterie, Blockaden und Kriegen

---

## 15. Leitprinzipien

Für die erste Version gelten folgende Designprinzipien:

1. **Einfachheit vor Simulationstiefe.**
2. **Bestehende Vanilla-Werte bevorzugen.**
3. **`trade_volume` misst maritime Nachfrage.**
4. **`transport_capacity` misst verfügbares Schifffahrtspotential.**
5. **Transportflotten sollen auch im Frieden wertvoll sein.**
6. **Merchant Power bestimmt, wie stark ein Land seine Handelsflotte in einem Markt wirtschaftlich nutzbar machen kann.**
7. **Keine physische Flottenallokation in Version 1.**
8. **Jeder ausgehende maritime Trade wird nur seinem `from_market` zugerechnet.**
9. **Unverifizierte Engine-Funktionen werden nicht vorausgesetzt.**
10. **Erst Berechnung funktionsfähig machen, danach wirtschaftliche Auswirkungen und Balance hinzufügen.**

---

## 16. Nächster Entwicklungsschritt

Der nächste technische Schritt ist die Verifikation der beiden zentralen numerischen Eingaben:

1. **Country Transport Capacity** zuverlässig aus den vorhandenen Schiffen berechnen.
2. **Merchant Power Share** eines Landes in einem Markt scriptseitig auslesen oder berechnen.

Erst wenn beide Werte belastbar verfügbar sind, sollte die erste vollständige `MarketTransportCapacity`-Berechnung implementiert werden.
