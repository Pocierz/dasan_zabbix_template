# DASAN V8102 (SLE / NOS 2.13) — mapa OID-ów z `snmpwalk_dasan`

Zdekodowane na podstawie `SLE-GPON-MIB` i `DASAN-SWITCH-MIB`
(`~/LMSK-lms-krzysztof/plugins/LMSGponDasanPlugin/mibs/`).

Urządzenie: 32 porty GPON (`GPON1/1..16`, `GPON2/1..16`), 4× 10G uplink
(`XE0/1..4`), 1767 zaprovisionowanych ONT (1582 aktywne w momencie walka).

## Korzenie

| OID | Nazwa |
|---|---|
| `.1.3.6.1.4.1.6296.101` | `sleMgmt` |
| `.1.3.6.1.4.1.6296.101.23` | `sleGpon` |
| `.1.3.6.1.4.1.6296.9.1.1.1` | `dsSwitchSystem` (DASAN-SWITCH-MIB) |

## Ruch na porcie — IF-MIB (standard)

Indeks = `ifIndex`: `900001..900004` = XE0/1..4, `1000001..1000016` = GPON1/x,
`2000001..2000016` = GPON2/x. `ifHighSpeed` = 10000 (XE) / 2500 (GPON).

| OID | Obiekt |
|---|---|
| `.1.3.6.1.2.1.31.1.1.1.6` / `.10` | `ifHCInOctets` / `ifHCOutOctets` |
| `.1.3.6.1.2.1.31.1.1.1.7` / `.11` | `ifHCInUcastPkts` / `ifHCOutUcastPkts` |
| `.1.3.6.1.2.1.31.1.1.1.8` / `.12` | `ifHCInMulticastPkts` / `ifHCOutMulticastPkts` |
| `.1.3.6.1.2.1.31.1.1.1.9` / `.13` | `ifHCInBroadcastPkts` / `ifHCOutBroadcastPkts` |
| `.1.3.6.1.2.1.2.2.1.14` / `.20` | `ifInErrors` / `ifOutErrors` |
| `.1.3.6.1.2.1.2.2.1.13` / `.19` | `ifInDiscards` / `ifOutDiscards` |

Interfejsy `br*`, `lo`, `default`, `mgmt` mają liczniki wyzerowane — LLD je odfiltrowuje.

## ONT — `sleGponOnuTable` (`.1.3.6.1.4.1.6296.101.23.3.1.1.<kol>`)

Indeks = `<ifIndex portu PON>.<ontId>`, np. `1000001.1`.

| Kol. | Obiekt | Uwagi |
|---|---|---|
| 2 | `sleGponOnuStatus` | `1` = inactive, `2` = active |
| 4 | `sleGponOnuSerial` | np. `HALN0842d6f0` |
| 8 | `sleGponOnuProfile` | np. `NET_3163_U` |
| 10 | `sleGponOnuDistance` | metry |
| 11 | `sleGponOnuHwAddress` | MAC |
| 12 | `sleGponOnuNosActiveVersion` | firmware ONT |
| 16 | `sleGponOnuRxPower` | 0.1 dBm; offline → `-400` |
| 17 | `sleGponOnuModelName` | np. `HL-1GE` |
| 18 | `sleGponOnuDescription` | opis abonenta |
| 21 | `sleGponOnuOltRxPower` | 0.1 dBm, moc odbierana przez OLT |
| 22 | `sleGponOnuRTD` | w bitach |
| 23 | `sleGponOnuLinkUpTime` | sekundy |
| 45 | `sleGponOnuDeactiveReason` | `2`=dying gasp, `3`/`100`=LOS, `4`=LOF |
| 51 / 52 | `sleGponOnuUpCount` / `DownCount` | detekcja flapowania |
| 58 | `sleGponOnuEqD` | equalization delay |

## ONT — diagnostyka optyczna `sleGponOnuAniTable`

`.1.3.6.1.4.1.6296.101.23.6.6.1.1.<kol>`, indeks = `<ifIndex>.<ontId>.1`
(**uwaga na końcowe `.1`**). Wartości jako łańcuchy dziesiętne, np. `"-21.0200"`.

| Kol. | Obiekt | Jednostka |
|---|---|---|
| 16 | `...OpticModuleTemper` | °C |
| 17 | `...OpticModuleVoltage` | V |
| 18 | `...OpticModuleTxbias` | mA |
| 19 | `...OpticModuleTxPower` | dBm |
| 20 | `...OpticModuleRxPower` | dBm |

Kolumna 20 zgadza się z `sleGponOnuRxPower` (kol. 16 tabeli ONU) — np. `-21.02`
vs `-209`.

## ONT — warstwa PON `sleGponOltStatsOnuTable`

`.1.3.6.1.4.1.6296.101.23.7.1.3.1.<kol>`, indeks = `<ifIndex>.<ontId>`.

| Kol. | Obiekt |
|---|---|
| 4 | `...PonBip8Errors` |
| 6 | `...PonFecUncorrectedCodewords` |
| 7 | `...PonFecCorrectedCodewords` |
| 8 | `...PonFecReceivedCodewords` |

To **nie są** liczniki bajtów — mimo dużych wartości (kol. 8 ≈ 2·10¹¹) są to
słowa kodowe FEC.

## System OLT — `dsSwitchSystem` (`.1.3.6.1.4.1.6296.9.1.1.1.<kol>.0`)

| Kol. | Obiekt | Wartość w walku |
|---|---|---|
| 3 | `dsOsVersion` | `2.13 #0099` |
| 8 / 9 / 10 | `dsCpuLoad5s` / `1m` / `10m` | 67 / 17 / 13 |
| 14 / 15 / 16 | `dsTotalMem` / `UsedMem` / `FreeMem` | 2 GB / 492 MB / 1.5 GB |
| 17 | `dsHardwareVersion` | `DS-F6-63G-A1(O)` |
| 19 | `dsSerialNumber` | `MP710UG278A0071` |
| 20 | `dsFirmwareVersion` | `01.48.0003` |

## Uniwersalność szablonu

W szablonie **nie ma zaszytych ifIndex-ów ani ID ONT-ów** — wszystko idzie przez
dwie reguły LLD, więc dowolna liczba portów, slotów i ONT-ów obsłuży się sama.

* **Discovery interfejsów** filtruje po `ifHighSpeed <> 0`, a nie po nazwie portu.
  Na tym urządzeniu to dokładnie 36 portów fizycznych + `mgmt`; odpadają
  `br*`, `lo`, `default`, `CG1`. Porty, które są tylko *down* (XE0/3, GPON2/13…),
  nadal raportują prędkość nominalną, więc nie giną. Działa tak samo dla portów
  nazwanych `GE1`, `TE1/1`, `PON0/1` itd.
* **Discovery ONT-ów** bierze nazwę portu PON z `ifDescr`, a nie z arytmetyki na
  ifIndex — chassis z innym schematem numeracji dostanie poprawne etykiety.
  Zweryfikowane na tym pliku: 1813 wierszy walka → 1767 ONT, 28 portów PON,
  0 nierozwiązanych nazw.

Szablon pasuje do **każdego OLT-a DASAN/DZS na platformie SLE** implementującego
`SLE-GPON-MIB`. Nie pasuje do gałęzi EPON (`sleEpon` = `sleMgmt 20`) ani do
starszej platformy DSSHE. Itemy CPU/RAM pochodzą z `DASAN-SWITCH-MIB` i są
wspólne dla urządzeń DASAN — jeśli konkretny model ich nie ma, te kilka itemów
przejdzie w stan unsupported i nic poza tym się nie zmieni.

Do dostrojenia pod inne urządzenie służą makra `{$IF.DESCR.MATCHES}` (domyślnie
`.*`), `{$IF.HIGHSPEED.NOT_MATCHES}` (domyślnie `^0$`) i
`{$GPON.ONT.DESC.MATCHES}`.

## Kalibracja progów optycznych (zmierzone na 1582 aktywnych ONT)

**Upstream i downstream nie są porównywalne.** ONT nadaje ok. +2.5 dBm, OLT nadaje
mocniej, więc odczyt upstream leży systemowo niżej:

| | mediana | 5 pct | 1 pct | najgorszy |
|---|---|---|---|---|
| ONT Rx (downstream, ANI 20) | −21.1 | −23.9 | −25.4 | −30.0 |
| OLT Rx (upstream, kol. 21) | −26.5 | −30.4 | −32.2 | −33.9 |
| ONT Tx (ANI 19) | +2.58 | +1.96 | +1.76 | +0.53 |

Różnica median wynosi **5.4 dB** i jest normalna — to nie jest usterka.
Tłumienie upstream (ONT Tx − OLT Rx): mediana 29.1 dB, 95 pct 32.9 dB.

Próg `{$GPON.OLT.RX.MIN}` = −28 dBm (pierwotny) leżał **w środku zdrowej
populacji** → 399 alarmów. Obecne wartości domyślne:

| Makro | Wartość | Ile alarmów |
|---|---|---|
| `{$GPON.OLT.RX.CRIT}` | −32 (czułość odbiornika C+) | wyłączony |
| `{$GPON.OLT.RX.MIN}` | −30.5 | wyłączony |
| `{$GPON.ONT.RX.CRIT}` | −27 | 5 |
| `{$GPON.ONT.RX.MIN}` | −25 | 18 |
| `{$GPON.ONT.RX.MAX}` | −8 | 2 |

Razem 92 zamiast 584. Jeśli OLT ma optykę **B+** (czułość −28 dBm), a nie C+,
ustaw `{$GPON.OLT.RX.CRIT}` na −28 — wtedy 399 alarmów byłoby uzasadnione i
oznaczałoby realny problem z budżetem optycznym całej sieci.

### Bramka asymetrii na alarmach ONT Rx

Sam próg bezwzględny na ONT Rx alarmował też na łączach, które są słabe
**równomiernie w obie strony** — czyli po prostu długich albo mocno
podzielonych, a nie uszkodzonych. Oba triggery ONT Rx mają więc warunek
dodatkowy:

```
ALARM = ONT Rx < próg  AND  (ONT Rx − OLT Rx) > {$GPON.RX.ASYM.MIN}
```

`{$GPON.RX.ASYM.MIN}` = 5 dB. Rozkład asymetrii na tym OLT: mediana 5.55 dB,
min 0.06, max 12.62. Efekt: 23 → 13 alarmów, a zostają wyłącznie łącza słabe
**tylko w dół** — czyli z realną usterką po stronie toru odbiorczego.

Bramkę ma **tylko trigger ostrzegawczy** (`{$GPON.ONT.RX.MIN}` = −25 dBm).
Trigger krytyczny (`{$GPON.ONT.RX.CRIT}` = −27 dBm) celowo jej **nie ma** — to
podłoga bezwzględna, żeby ONT o fatalnym poziomie nie zniknął z radaru tylko
dlatego, że tłumi symetrycznie. Przykład: `GPON1/1:27` ma Rx −30.00 dBm przy
asymetrii 2.20 dB — ostrzeżenie milczy, ale poziom krytyczny go złapie.

### Triggery OLT Rx są wyłączone

Wszystkie trzy triggery na moc odbieraną przez OLT (`is low`, `is critically
low`, `dropped against its baseline`) mają w szablonie **status `DISABLED`** —
generowały 71 zgłoszeń na łączach, które działają. Itemy `gpon.ont.oltrxpower`
zbierają dane dalej: są w wykresach i zasilają bramkę asymetrii przy ONT Rx.
Żeby wrócić do alarmowania, wystarczy włączyć prototyp i przepuścić regułę LLD.

Progi bezwzględne to lista obserwacyjna. Nowe usterki wykrywają triggery
**„dropped against its own baseline"** — porównują ostatnią godzinę z własną
średnią 7-dniową danego ONT (`{$GPON.RX.DROP}` = 4 dB), więc działają tak samo
dobrze przy −18 jak przy −29 dBm. Wymagają ok. tygodnia historii.

## Aktywne ONT-y na port PON

Agregat `GPON: ONTs active` nie pokazuje, **który** port traci ONT-y. Dlatego
osobna reguła LLD `gpon.pon.discovery` (filtr `{$GPON.PON.DESCR.MATCHES}` = `GPON`)
wykrywa same porty PON i tworzy po jednym itemie na port:

* `gpon.pon.ont.active[{#SNMPINDEX}]`, nazwa `PON {#IFDESCR}: Active ONTs`
* zależny od tego samego mastera `gpon.ont.walk.status` — **zero dodatkowego SNMP**
* JavaScript liczy linie `...3.1.1.2.<ifIndex>.<ontId> = INTEGER: 2`

Zweryfikowane na dwóch różnych OLT-ach: 32 porty / suma 1579 oraz 16 portów /
suma 386 — w obu przypadkach co do jednego tyle, ile agregat.

**Filtr jest celowo bez kotwicy `^`.** Nazewnictwo portów różni się między
modelami: `GPON2/3` na V8102 (Gdańska), ale `Port10-GPON` na innym chassis
(Kosynierów). Wzorzec `^GPON` na tym drugim **nie zgłasza błędu — po prostu nie
wykrywa niczego**, więc wykres per port zostaje pusty przy zdrowych regułach LLD.
Różni się też numeracja ifIndex: 1000001–2000016 kontra 1–16.

Wykres to widget **`svggraph` ze wzorcem nazwy** `PON *: Active ONTs`, a nie
zwykły wykres. Ma to znaczenie: prototyp wykresu dałby jeden wykres *na port*,
a wzorzec daje jeden wykres z jedną **linią** na port — o to chodziło.

### Alarmy o masowym wypadnięciu ONT-ów

| Trigger | Poziom | Warunek |
|---|---|---|
| `PON {#IFDESCR}: {$GPON.PON.DROP} or more ONTs went inactive at once` | **HIGH** | na jednym porcie PON ubyło ≥ `{$GPON.PON.DROP}` (10) aktywnych ONT-ów między dwoma odczytami |
| `GPON: Mass ONT outage` | **DISASTER** | w całym OLT ubyło ≥ `{$GPON.ONT.MASS.DROP}` (100) aktywnych ONT-ów między dwoma odczytami |

Oba mają **strażnika przed niepełnym walkiem**:

```
... and last(gpon.ont.count.total) = last(gpon.ont.count.total,#2)
```

Uciety walk SNMP obniża liczbę aktywnych tak samo jak realna awaria. Ale przy
awarii ONT-y pozostają zaprovisionowane, więc `total` się nie zmienia — a przy
uciętym walku spada razem z `active`. Bez tego warunku każdy timeout SNMP
zgłaszałby katastrofę.

Oba odpalają na *przejściu* i mają „Allow manual close" — zamyka się je po
obsłużeniu awarii, zamiast czekać, aż same wygasną.

`GPON: Share of active ONTs dropped` (HIGH, próg `{$GPON.ONT.ACTIVE.MIN}` = 80%)
**zależy** od triggera DISASTER, więc przy masowej awarii widać jedno
zgłoszenie, a nie dwa.

## Wykresy — dlaczego są w szablonie

Wykresy tworzone ręcznie w GUI na szablonie **znikają przy każdym imporcie**
z zaznaczonym „Delete missing". Dlatego są teraz w YAML-u:

| Obiekt | Gdzie w YAML | Pozycje |
|---|---|---|
| `Memory` | `zabbix_export.graphs` | total, used (filled), utilization |
| `ONTs` | `zabbix_export.graphs` | total (dashed), active, inactive |
| `CPU` | `zabbix_export.graphs` | Load 5s |
| `Interface {#IFDESCR}({#IFALIAS}): Network traffic` | `graph_prototypes` w `net.if.discovery` | bits in (filled), bits out, in errors, out errors |

UUID-y to te, które nadała już działająca instancja — import **aktualizuje**
istniejące wykresy zamiast zgłaszać kolizję nazw.

Dashboard szablonowy `Network traffic` też jest w pliku (`templates[].dashboards`)
— inaczej ginie tak samo jak wykresy.

**Plik jest teraz eksportem z żywej instancji** (`configuration.export`), a nie
tworem ręcznym. To jedyny pewny sposób na poprawny schemat widgetów: wartość
pola typu `GRAPH`/`GRAPH_PROTOTYPE` to obiekt `{host, name}`, a nie numeryczne
id. Po zmianach klikniętych w GUI wystarczy wyeksportować szablon ponownie.

Uwaga na jeszcze jedną niekonsekwencję: **eksport** zagnieżdża triggery
jednoitemowe pod itemem (`items[].triggers`), a tylko wieloitemowe zostawia w
korzeniu. **Import** akceptuje obie formy — ale nie akceptuje `triggers` w
środku elementu `template`.

Zasady, które kosztowały nas po jednym nieudanym imporcie:

1. Triggery i wykresy poziomu szablonu idą w **korzeniu** `zabbix_export`,
   obok `templates` — nie w środku elementu szablonu. W środku reguły LLD
   siedzą tylko *prototypy*.
2. Nazwa prototypu wykresu **musi zawierać makra LLD**. „Network traffic" bez
   makr dałoby 37 wykresów o identycznej nazwie i błąd discovery.

## Ustawienia wymagane po stronie Zabbixa

Bez tego duże walki ONT wpadają w NOT SUPPORTED i wszystkie itemy zależne
pokazują zera:

| Co | Gdzie | Wartość |
|---|---|---|
| `max_repetitions` | interfejs SNMP hosta | **50** (domyślne 10 = 5× więcej zapytań GETBULK) — ustawienie **hosta**, nie schodzi z szablonu, trzeba powtórzyć przy każdym nowym OLT |
| `timeout` | itemy `gpon.ont.walk.*` | **30 s** (globalne 3 s nie wystarcza) |

Zmierzone na tym OLT przy max-rep 50: jedna kolumna 1767 wierszy ~0.75 s,
walk FEC (3 kolumny) 2.7–9.2 s, walk optyczny (5 kolumn) ~3.8 s.

Format wyjścia `walk[]` to `OID = TYP: wartość`, np.
`.1.3.6.1.4.1.6296.101.23.3.1.1.2.1000001.1 = INTEGER: 2`. Preprocessing
`SNMP walk value` sam obcina typ, ale własny JavaScript musi go uwzględnić —
na tym poległy liczniki `GPON: ONTs *`.

Itemy optyczne nieaktywnych ONT-ów nie mają wiersza DDM — krok `SNMP walk value`
ma `Discard value`, więc nowo wykryte nie wpadają w NOT SUPPORTED. Uwaga: item,
który już jest niewspierany, odblokuje się dopiero przy realnej wartości —
disable/enable tego nie kasuje.

## Czego w tym walku NIE ma

* **Ruchu per ONT w bajtach/pakietach.** Tabele PM
  (`sleGponOnuPMStatistics` = `.7.2`, `sleGponOnuPMStatisticsDetail` = `.7.3.1..4`)
  są obecne, ale **wszystkie wartości = 0** — performance monitoring jest wyłączony
  na OLT. Po włączeniu PM te tabele dadzą ruch per ONT i per port UNI.
* Kolumn `sleGponOnuPortGemblockRx/Tx` (`.6.1.1.1.10/11`) — firmware ich nie zwraca.
* Diagnostyki optycznej SFP po stronie OLT (dla portów XE i PON).
