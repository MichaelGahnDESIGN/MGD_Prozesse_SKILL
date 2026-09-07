# MGD — Prozesse SKILL

Ein Skill für KI-Agenten (Claude Code & ChatGPT Codex), der **Hintergrundaufgaben prüft, einordnet und beendet** — und dabei die drei Fehlschlüsse vermeidet, an denen solche Prüfungen üblicherweise scheitern.

> **Leitsatz:** Eine Kachel ist kein Prozess, und Stille ist kein Hänger.

## Das Problem

Wer einen Agenten stundenlang arbeiten lässt, sammelt Hintergrundaufgaben an. Die Oberfläche zeigt Kacheln, das Terminal zeigt Ausgaben — beides sagt weniger, als es scheint:

- Eine **Kachel überlebt ihren Prozess**. Sie steht noch da, wenn längst nichts mehr läuft. Wer sie für Arbeit hält, wartet auf ein Ergebnis, das schon da ist.
- **Stille bedeutet nicht Stillstand.** Läuft die Ausgabe durch `| tail`, erscheint sie erst am Ende — vollständig. Zehn Minuten ohne Zeile sind dann kein Symptom, sondern Normalbetrieb.
- **Und manchmal läuft etwas, das niemand bestellt hat.** Ein `rsync`, dessen Quellpfad leer blieb, kopiert nicht ein Verzeichnis, sondern die Maschine.

Dieser Skill ist aus einer Sitzung entstanden, in der alle drei gleichzeitig auftraten: vier Warteschleifen schliefen fünf Stunden, während zwei `rsync` zwölf Stunden lang die Wurzel eines Macs auf einen öffentlichen Webserver schaufelten. Gefunden hat das nicht die Überwachung, sondern eine beiläufige Rückfrage.

## Die Lösung

Vier Fragen, in dieser Reihenfolge — jede kann die nächste erübrigen:

| # | Frage | Worum es geht |
|---|-------|---------------|
| 1 | **Läuft es wirklich?** | Kachel ≠ Prozess. Erst messen, dann urteilen. |
| 2 | **Muss es laufen?** | Gehört es zur Aufgabe, oder ist es Rest von gestern? |
| 3 | **Kommt es voran?** | Oder wartet es auf etwas, das nie eintritt? |
| 4 | **Richtet es Schaden an?** | Systempfade, geteilte Ressourcen, fremde Ziele. |

Frage 4 ist kein Aufräumen mehr, sondern ein Notfall. Sie steht am Ende, weil die ersten drei sie meistens schon sichtbar gemacht haben.

## Drei Fallen, die der Skill benennt

**Die Selbsttreffer-Falle.** `pgrep -f 'deploy.sh api'` findet sich selbst — die eigene Kommandozeile enthält den gesuchten Text. Eine Warteschleife `until ! pgrep -f …` wartet damit auf ihr eigenes Ende und läuft ewig. Abhilfe: auf den Programmnamen (`comm`) prüfen, oder auf eine Fertig-Marke in der Ausgabedatei.

**Der Puffer-Trugschluss.** „Seit zehn Minuten keine Ausgabe" heißt oft nur, dass die Ausgabe durch eine Pipe läuft. In der Ursprungssitzung wurde ein Testlauf deshalb nach 6:25 als „hängt an der Datenbank" abgebrochen. Er brauchte 15 Minuten und war grün.

**Die leere Variable.** Ein `rsync … / ziel:/pfad/` entsteht selten durch einen Tippfehler, sondern dadurch, dass ein vorheriger Schritt fehlschlug und sein Rückgabewert verschluckt wurde. Den Prozess zu beenden reicht nicht — solange die Ursache im Skript steht, startet der nächste Lauf denselben Vorgang.

## Installation

### Claude Code

```bash
mkdir -p ~/.claude/skills/prozesse
curl -o ~/.claude/skills/prozesse/SKILL.md \
  https://raw.githubusercontent.com/MichaelGahnDESIGN/MGD_Prozesse_SKILL/main/SKILL.md
```

Oder als Git-Klon, dann sind Updates ein Einzeiler:

```bash
git clone https://github.com/MichaelGahnDESIGN/MGD_Prozesse_SKILL.git ~/.claude/skills/prozesse
git -C ~/.claude/skills/prozesse pull    # spaeter aktualisieren
```

Danach ist `/prozesse` in jeder Session verfügbar.

### ChatGPT Codex

```bash
mkdir -p .codex/commands
curl -o .codex/commands/prozesse.md \
  https://raw.githubusercontent.com/MichaelGahnDESIGN/MGD_Prozesse_SKILL/main/SKILL.md

codex --instructions .codex/commands/prozesse.md "/prozesse"
```

### Ohne Installation

Die [`SKILL.md`](SKILL.md) ist ein eigenständiger Prompt. Herunterladen, in ChatGPT, Claude, Gemini oder ein anderes Werkzeug kopieren, fertig.

## Verwendung

```
/prozesse
```

Greift außerdem bei Formulierungen wie „läuft das noch?", „muss das laufen?", „das hängt seit Stunden" oder „räum die Hintergrundaufgaben auf".

Für lange Sitzungen enthält der Skill ein **Wächter-Skript**, das alle zehn Minuten nach zwei Dingen sucht: `rsync` mit einem Systempfad als Quelle, und Prozesse über zwei Stunden Laufzeit. Es filtert nach Programmnamen — sonst meldet es sich selbst als Fund. Einmal zu Beginn gesetzt, ersetzt es das gelegentliche Nachsehen.

## Was der Skill ausdrücklich verlangt

**Beenden ist ein Ergebnis, kein Betriebsgeräusch.** Ein getöteter Prozess gehört in den Bericht — mit Laufzeit, Grund und der Ursache dahinter.

**Vor dem Beenden nach hinterlassenem Zustand fragen.** Ein abgebrochener Testlauf hinterlässt halbe Tabellen und offene Sperren. Der nächste Lauf startet sonst auf den Trümmern und schlägt aus einem Grund fehl, der nichts mit dem Code zu tun hat.

**Sanft vor hart.** Erst `kill`, drei Sekunden warten, `kill -9` nur wenn nötig.

## Grenzen

- Die Kommandos sind auf **macOS und Linux** ausgelegt (`ps`, `lsof`, `awk`). Unter Windows braucht es Entsprechungen.
- Der Skill **erkennt** Muster, er kennt aber nicht deine Projekte: ob ein zwei Stunden alter Prozess legitim ist, entscheidet der Kontext.
- Das Wächter-Skript prüft zwei konkrete Gefahren, keine allgemeine Anomalieerkennung. Es ist bewusst eng gefiltert, weil jede Ausgabezeile zu einer Meldung wird.

## Verwandte MGD Projekte

| Projekt | Beschreibung |
|---------|-------------|
| [MGD_Autopilot_SKILL](https://github.com/MichaelGahnDESIGN/MGD_Autopilot_SKILL) | Projektziele eigenständig abarbeiten, mit Abbruchbedingung |
| [MGD_DEV_SKILL](https://github.com/MichaelGahnDESIGN/MGD_DEV_SKILL) | Release, Sync, Backup und Wissensdokumentation |
| [MGD_Todo_SKILL](https://github.com/MichaelGahnDESIGN/MGD_Todo_SKILL) | Aufgabenmanagement direkt im Projekt-Repo |
| [MGD_ProjectClean_SKILL](https://github.com/MichaelGahnDESIGN/MGD_ProjectClean_SKILL) | Abschluss- und Aufräum-Workflow |
| [MGD_AI-PlayTest_SKILL](https://github.com/MichaelGahnDESIGN/MGD_AI-PlayTest_SKILL) | Live-Playtest aus Nutzerperspektive |
| [MGD_BugReport_SKILL](https://github.com/MichaelGahnDESIGN/MGD_BugReport_SKILL) | Feedback-Hub: Bug-Meldung, Ideen und Support |

## Lizenz

MIT — frei verwendbar, anpassbar, weitergeben mit Namensnennung. Siehe [LICENSE](LICENSE).

---

*Entwickelt von [Michael Gahn DESIGN](https://michael-gahn.de) — gepflegt mit Claude Code & ChatGPT Codex.*

---

## Impressum

Angaben gemäß § 5 DDG — Siehe [`IMPRESSUM.md`](IMPRESSUM.md).
