# Aufgabe 1 – Ein Feuerwerk an UDP-Nachrichten (Pseudo-Verteilt)

## Struktur

| Datei | Zweck |
|---|---|
| `firework.py` | Ein Prozess des logischen Rings (Protokoll + Messung) |
| `messwerte.py` | Automatisierung: Ringe mit n = 2, 4, 8, … starten, Messwerte aggregieren |

## Nutzung

```bash
python messwerte.py                 # volle Messreihe bis zum Fehlschlag
python messwerte.py --max-n 64 --repeats 1    # schneller Testlauf
```

Ergebnis: `results/results.csv` (eine Zeile pro Lauf) sowie pro Lauf die
Roh-Statistiken jedes Prozesses (`node_*.json`, P0 enthält alle Rundenzeiten).

## Protokoll

Token-Ring nach **Le Lann (1977)** (Vorlesung Kap. 3.1): Prozess i wartet auf
das Token von P(i−1), trifft im "kritischen Abschnitt" seine Zündentscheidung
(Rakete = UDP-Multicast an `239.10.10.10`, danach `p = p/2`) und reicht das
Token per UDP-Unicast an P(i+1). Da nur der Token-Halter zündet, ist der
wechselseitige Ausschluss der Zündaktion per Konstruktion garantiert; der
Ring ist fair (jeder Prozess erhält das Token genau einmal pro Runde).
Die Ringtopologie steckt vollständig in der Port-Arithmetik
(`50000 + id`, Nachfolger `50000 + (id+1) mod n`).

Terminierung: Das Token trägt ein `fired`-Flag. P0 wertet es bei jeder
Rückkehr aus und zählt stille Runden; nach k davon sendet er STOP per
Multicast und absorbiert das Token.

## Klassische Token-Probleme (Vorlesung: "Fehlertoleranz?")

| Problem | Maßnahme in `firework.py` |
|---|---|
| Token-**Verlust** (UDP ist unzuverlässig) | Hop-zu-Hop-Zuverlässigkeit: Jede Übergabe wird nach 200 ms wiederholt, bis der Nachfolger per ACK bestätigt. |
| Token-**Duplikat** | Das Token wird genau einmal erzeugt, nie regeneriert (Timeout-Regeneration könnte ein nur verzögertes Token verdoppeln). Retransmit-Duplikate filtert die pro Hop wachsende Transfer-ID; auch Duplikate werden erneut bestätigt, da das vorige ACK verloren sein kann. |
| **Prozessausfall** | Nach 50 vergeblichen Wiederholungen gibt der Sender auf; Idle-Timeout (30 s) beendet alle übrigen Prozesse. Der Lauf gilt als gescheitert: Im Fehlerfall wird Liveness geopfert, nie Safety. |
| **STOP-Verlust** | STOP wird 3× gesendet (idempotent); Idle-Timeout als letztes Netz. |

## Messgrößen (pro Lauf, in Abhängigkeit von n)

Gefordert: `rounds` (Gesamtanzahl Token-Runden), `multicasts` (gesendete
Multicasts = Raketen + STOPs), `t_min/mean/max_ms` (Rundenzeit, gemessen
von P0 als Abstand zweier Token-Ankünfte mit `time.monotonic()`).

Zusätzlich sinnvoll: `retransmits` (wiederholte Token-Übergaben — misst
UDP-Paketverlust unter Last und erklärt, *woran* das maximale n scheitert),
`delivery_pct` (empfangene / erwartete Raketen, erwartet = Raketen × n —
misst die Zuverlässigkeit des ungesicherten Multicasts; die Raketen sind
bewusst ungesichert, sie sind die verbleibende Inkonsistenzquelle und damit
der Anknüpfungspunkt für Aufgabe 4), `wall_s` (Gesamtdauer).

**Maximales n** := größtes n, bei dem alle Wiederholungen regulär
terminieren (alle Prozesse Exit-Code 0).

## Hinweise für Windows

Der Code ist plattformabhängig: Windows beherrscht kein Multicast über das
Loopback-Interface, daher läuft die Multicast-Gruppe dort über das
Standard-Netzwerkinterface (`0.0.0.0`); `IP_MULTICAST_LOOP` hält die Pakete
auf dem Rechner. Das Windows-spezifische UDP-Verhalten (ICMP "Port
unreachable" → `ConnectionResetError`) wird an allen Sende- und
Empfangsstellen defensiv abgefangen.

Vor der ersten Messreihe: `python selftest.py` prüft in 2 Sekunden, ob
Unicast und Multicast auf dem Rechner funktionieren (Firewall-Diagnose).

Praktisch zu beachten:
- Beim ersten Start fragt die **Windows-Firewall**, ob Python kommunizieren
  darf → für private Netzwerke zulassen, sonst wird Multicast blockiert.
- Ein **aktives Netzwerkinterface** (WLAN/LAN verbunden) muss vorhanden
  sein, da die Multicast-Gruppe darüber läuft.
- Start in PowerShell: `python messwerte.py` (bzw. `py experiment.py`).
- Limitierende Faktoren für das maximale n sind hier v. a. die langsame
  Prozess-Erzeugung und Speicher; unter Linux/macOS zusätzlich
  `ulimit -n` (File Descriptors) und UDP-Puffer (`net.core.rmem_max`).