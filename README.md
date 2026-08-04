# Szablon Zabbix 7.4 dla OLT DASAN/DZS GPON (platforma SLE)

Monitorowanie po SNMP dla OLT-ow DASAN/DZS na platformie SLE — powstalo i zostalo
zweryfikowane na **V8102** z NOS 2.13: 32 porty GPON, 4 uplinki 10G,
1767 zaprovisionowanych ONT-ow.

OID-y sa zdekodowane na podstawie oryginalnych MIB-ow producenta
(`SLE-GPON-MIB`, `DASAN-SWITCH-MIB`), a nie zgadniete. Pelna mapa:
[README_OID_MAP.md](README_OID_MAP.md).

## Co zbiera

**Ruch na porcie** — dla uplinkow XE i portow PON: bity in/out oraz osobno
**unicast, multicast i broadcast** w obu kierunkach (64-bitowe liczniki IF-MIB),
bledy, discardy, stan lacza i predkosc.

**Dane ONT** — status i powod deaktywacji (dying gasp kontra LOS), moc optyczna
Rx/Tx ONT-a, moc odbierana przez OLT, temperatura/napiecie/prad modulu, dystans
swiatlowodu, uptime, liczniki up/down oraz bledy warstwy PON (BIP8, FEC).

**Kondycja OLT-a** — CPU, pamiec, uptime, model, numer seryjny, firmware.

**Liczniki ONT** — lacznie/aktywne/nieaktywne oraz **aktywne ONT-y na kazdym
porcie PON osobno**.

## Architektura

Wszystkie dane masowe pobieraja **master itemy `walk[]`**, a kazdy item
per-interfejs i per-ONT jest itemem zaleznym. Na referencyjnym OLT daje to
~27 000 metryk z 10 zapytan SNMP na cykl. Master itemy nie trzymaja historii.

Nic nie jest zaszyte na sztywno — porty i ONT-y pochodza z trzech regul LLD,
wiec dowolna liczba portow, slotow i ONT-ow obsluzy sie sama.

## Instalacja

1. Zaimportuj `template_dasan_gpon_olt_snmp.yaml` (Data collection → Templates → Import).
2. Podepnij szablon pod hosta z interfejsem SNMP.
3. **Ustaw `max_repetitions` na interfejsie SNMP hosta na 50** (domyslne 10 daje
   5x wiecej zapytan GETBULK i walki ONT-ow sie nie miesza w timeoucie).

Punkt 3 jest obowiazkowy przy wiekszej liczbie ONT-ow. Same walki maja juz
ustawiony `timeout: 30s` w szablonie.

## Progi alarmowe

Wartosci domyslne sa **wyliczone z realnego rozkladu** na 1582 aktywnych ONT-ach,
a nie wziete z katalogu. Najwazniejsze:

* Upstream i downstream **nie sa porownywalne** — mediana ONT Rx to −21.1 dBm,
  a mediana OLT Rx −26.5 dBm. Te 5.4 dB roznicy to norma, nie usterka.
* Alarm o niskiej mocy ONT Rx jest **bramkowany asymetria** `(ONT Rx − OLT Rx)`:
  lacze slabe rowno w obie strony to po prostu dluga albo mocno podzielona trasa.
* Triggery na moc odbierana przez OLT sa domyslnie **wylaczone** — generowaly
  zgloszenia na dzialajacych laczach. Itemy zbieraja dane dalej.

Szczegolowa kalibracja z liczbami: [README_OID_MAP.md](README_OID_MAP.md).

## Znane ograniczenie

**Brak licznikow ruchu per ONT.** Tabele PM (`sleGponOnuPMStatistics`, `.7.2`
i `.7.3.x`) sa obecne, ale zwracaja zera dla kazdego ONT-a, bo performance
monitoring jest wylaczony na OLT. Zamiast tego szablon zbiera liczniki warstwy
PON (BIP8, FEC), ktore dzialaja.

## Zgodnosc

Pasuje do kazdego OLT-a DASAN/DZS na platformie SLE implementujacego
`SLE-GPON-MIB` (`.1.3.6.1.4.1.6296.101.23`). Nie pasuje do galezi EPON
(`sleEpon`) ani do starszej platformy DSSHE.

## Uwaga

W repozytorium **nie ma** surowego pliku `snmpwalk_dasan`, na ktorym powstal
szablon — zawiera opisy abonentow, adresy MAC i wewnetrzna adresacje IP.
