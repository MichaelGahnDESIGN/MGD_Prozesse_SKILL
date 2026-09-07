---
name: prozesse
description: Hintergrundaufgaben und Prozesse prüfen, korrigieren und beenden — was läuft wirklich, was ist eine Karteileiche, was ist ein Ausreißer. Enthält die Selbsttreffer-Falle von pgrep, den Puffer-Trugschluss bei „hängt seit Minuten ohne Ausgabe", die Erkennung von rsync mit Systemquelle, und ein Wächter-Skript für lange Sitzungen. Auslösen mit „/prozesse" oder bei „läuft das noch?", „muss das laufen?", „das hängt seit Stunden", „räum die Hintergrundaufgaben auf".
---

# Hintergrundaufgaben prüfen und aufräumen

> Leitsatz: **Eine Kachel ist kein Prozess, und Stille ist kein Hänger.**
> Beides zu verwechseln kostet entweder Stunden Wartezeit oder Stunden Schaden.

Dieser Skill entstand aus einer Sitzung, in der beides gleichzeitig passierte:
vier Warteschleifen schliefen fünf Stunden vor sich hin, während zwei `rsync`
mit `/` als Quelle zwölf Stunden lang die Wurzel eines Macs auf einen
öffentlichen Webserver schaufelten. Gefunden hat sie nicht die Überwachung,
sondern die Rückfrage des Nutzers.

## Die vier Fragen, in dieser Reihenfolge

1. **Läuft es wirklich?** — Kachel in der Oberfläche ≠ laufender Prozess.
2. **Muss es laufen?** — gehört es zur aktuellen Aufgabe, oder ist es Rest?
3. **Kommt es voran?** — oder wartet es auf etwas, das nie eintritt?
4. **Richtet es Schaden an?** — Systempfade, geteilte Ressourcen, fremde Ziele.

---

## 1. Läuft es wirklich?

Erst messen, dann urteilen. **Nach Programmnamen filtern, nicht nach
Kommandozeile:**

```sh
ps -eo pid,etime,comm,args | awk '$3 ~ /(^|\/)(rsync|sshpass|php|dart|flutter|node|find|git)$/'
```

> **⚠️ Selbsttreffer-Falle.** `pgrep -f 'deploy.sh api'` findet **sich selbst**,
> weil die eigene Kommandozeile diesen Text enthält. Eine Warteschleife
> `until ! pgrep -f 'deploy.sh api'; do sleep 20; done` wartet dann bis in alle
> Ewigkeit auf ihr eigenes Ende. Dasselbe gilt für Wächter-Skripte, die per
> `grep` auf die Kommandozeile prüfen — sie melden sich selbst als Fund.
>
> Abhilfe: auf `comm` (Programmname) prüfen, oder auf die **Ausgabedatei**
> statt auf einen Prozessnamen:
> ```sh
> until grep -q '@@@FERTIG@@@' ausgabe.log; do sleep 20; done
> ```
> Dafür muss das überwachte Kommando am Ende eine eindeutige Marke drucken.

Existiert kein Prozess, ist die Kachel eine **Karteileiche**: der Vorgang ist
beendet, nur die Anzeige steht noch. Dem Nutzer sagen, dass er sie wegräumen
kann — nicht so tun, als würde noch gearbeitet.

---

## 2. Muss es laufen?

| Fund | Urteil |
|---|---|
| Gehört zur laufenden Aufgabe | laufen lassen |
| Warteschleife auf etwas längst Fertiges | beenden |
| Entwicklungsserver von gestern (`php -S`, `npm run dev`, `vite`) | beenden, Port prüfen |
| Testlauf, den niemand mehr liest | beenden |
| Wächter/Monitor der aktuellen Sitzung | laufen lassen |

Ports gezielt freiräumen:
```sh
lsof -ti :8199 | xargs -r kill
```
`lsof` auf die **Ressource** findet auch Prozesse, deren Name nicht mehr passt —
`ps | grep phpunit` übersieht einen Prozess, der nur noch `php` heißt.

---

## 3. Kommt es voran?

> **⚠️ Puffer-Trugschluss.** „Seit zehn Minuten keine Ausgabe" heißt oft nur,
> dass die Ausgabe durch `| tail` oder `| head` läuft — die puffern **alles**
> bis zum Ende der Eingabe. Vor dieser Falle wurde in derselben Sitzung ein
> Testlauf nach 6:25 abgebrochen und als „hängt an der Datenbank" diagnostiziert.
> Er brauchte 15 Minuten und war grün.

Prüfreihenfolge bei scheinbarem Stillstand:

```sh
# a) Lebt der Prozess, und in welchem Zustand?
ps -o pid,stat,etime,comm -p <pid>      # STAT "UN" = wartet auf I/O
# b) Waechst die Ausgabedatei?
ls -l ausgabe.log; sleep 20; ls -l ausgabe.log
# c) Haelt jemand die geteilte Ressource?
lsof <pfad-zur-testdatenbank>
```

Erst wenn a, b **und** c nichts liefern, ist es ein echter Hänger.

**Schreib Ausgaben direkt in eine Datei**, nicht durch eine Pipe:
```sh
befehl > ausgabe.log 2>&1        # gut: waechst laufend
befehl 2>&1 | tail -20           # schlecht: erst am Ende sichtbar
```

---

## 4. Richtet es Schaden an?

Diese Funde sind **Notfälle**, nicht Aufräumarbeit:

```sh
# rsync, das einen Systempfad als QUELLE hat
ps -eo etime,comm,args | awk '$2 ~ /(^|\/)rsync$/ {
  for (i=3; i<=NF; i++) if ($i=="/" || $i ~ /^\/(Users|System|Applications|Volumes)\/?$/) { print; break }
}'
```

Ein `rsync … / ziel:/pfad/` kopiert die **gesamte Maschine**. Entsteht meist so:
eine Variable mit dem Quellpfad bleibt leer, weil der vorherige Schritt
fehlschlug und sein Rückgabewert verschluckt wurde. Sofort beenden, dann das
Ziel prüfen und bereinigen — **und die Ursache im Skript beheben**, sonst
startet der nächste Lauf denselben Vorgang wieder.

Weitere Alarmzeichen:
- Prozesse älter als zwei Stunden, die zu keiner offenen Aufgabe gehören
- Zwei Testläufe auf derselben Datenbank oder demselben Cache
- `find /` oder `du /` — durchsucht die ganze Platte, meist versehentlich
- Ein Build auf einem Netzlaufwerk, obwohl ein lokaler Spiegel vereinbart war

---

## Der Wächter für lange Sitzungen

Bei Arbeit über mehrere Stunden **einmal am Anfang** setzen, statt sich auf
gelegentliches Nachsehen zu verlassen. Filtert nach Programmnamen, damit er
sich nicht selbst meldet:

```sh
while true; do
  ps -eo etime=,comm=,args= 2>/dev/null \
    | awk '$2 ~ /(^|\/)rsync$/ { for (i=3;i<=NF;i++) if ($i=="/" || $i ~ /^\/(Users|System|Applications|Volumes)\/?$/) { print; break } }' \
    | cut -c1-160 | sed 's/^/NOTFALL rsync kopiert einen Systempfad: /'

  ps -eo etime=,comm=,args= 2>/dev/null \
    | awk '$2 ~ /(^|\/)(rsync|sshpass|php|dart|flutter|node)$/ && ($1 ~ /-/ || ($1 ~ /^[0-9]{2}:[0-9]{2}:[0-9]{2}$/ && $1+0 >= 2))' \
    | cut -c1-140 | sed 's/^/LANGLAEUFER ueber 2h: /'

  sleep 600
done
```

In Claude Code als `Monitor` mit `persistent: true` starten. Jede Ausgabezeile
wird zu einer Meldung — deshalb muss der Filter eng sein.

---

## Beenden: sanft vor hart

```sh
kill <pid>          # zuerst
sleep 3
kill -9 <pid>       # nur wenn er dann noch lebt
```

Vor dem Beenden fragen, ob der Vorgang **Zustand hinterlässt**: ein
abgebrochener Testlauf lässt halbe Tabellen und offene Sperren zurück. Dann die
Testdatenbank beiseiteschieben und neu aufsetzen, statt den nächsten Lauf auf
den Trümmern zu starten.

**Nicht beenden**, ohne es zu sagen: Ein getöteter Prozess ist ein Ergebnis und
gehört in den Bericht, nicht ins Betriebsgeräusch.

---

## Abschlussbericht

Nach jedem Durchgang knapp berichten:

- was **wirklich lief** (mit Laufzeit)
- was **Karteileiche** war (Kachel ohne Prozess)
- was **beendet** wurde und warum
- welche **Ursache** dahinterstand, wenn etwas Unerwartetes lief
- was **weiterläuft** und warum es weiterlaufen soll
