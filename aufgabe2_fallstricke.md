# Aufgabe 2: Fallstricke des Modells (Gruppe 2)

Unser Bestand: **RI XIII H. 4** (Frankfurt am Main), 1.054 Regesten, Filter
`r.identifier STARTS WITH '[RI XIII] H. 4 '`.
Stand der Messung: 25.11.2026, lokale Neo4j-Datenbank aus `docker-compose.yml`.

> **Einschränkung:** Der Import lief während der Messung noch (Textbereinigung; Datumskorrektur,
> Ortsknoten und Register fehlten noch). Teil 1 und die mit „DB“ markierten Punkte in Teil 2 sind in
> der Datenbank gemessen. Die mit „Importquelle“ markierten Punkte sind aus dem Importskript und
> dessen Quelldateien (`RI-Places.csv`, `RI_013.xml`) abgeleitet und müssen nach Abschluss des
> Imports in der Datenbank nachgeprüft werden.

---

## Teil 1: Die drei bekannten Probleme

Die Spalte „betroffene Fragen“ ist nach Fragetyp gefüllt, weil unsere zehn Fragen hier noch nicht
eingetragen sind.

### 1. Doppelte Regesten (Chmel und Hefte)

**Befund**
- 100 unserer Regesten nennen im `Reg.:`-Teil von `archivalHistory` ein Chmel-Regest als Link
  (`riregesten:<exchangeIdentifier>`). Alle 101 Links lassen sich auflösen und führen auf
  95 verschiedene Chmel-Regesten.
- „Gleiches Datum + gleicher Ort“ taugt nicht als Erkennung: 136 unserer Regesten haben ein
  Chmel-Regest vom selben Tag und Ort, 122 davon ohne Verweis. In der Stichprobe waren das
  durchweg andere Urkunden.
- Umgekehrt weicht bei 19 der 101 Links das Datum ab (H. 4 n. 22 „Mai 31“, Chmel n. 558 „Mai 30“),
  und der Ort heißt bei Chmel teils anders („Neustadt“ statt „Wiener Neustadt“).
- 25 weitere Regesten erwähnen Chmel nur außerhalb des `Reg.:`-Teils. Das ist ein Querverweis,
  keine Dublette.
- Im `Reg.:`-Teil verweisen außerdem 27 Regesten auf H. 3, vier auf H. 1 und zwei auf H. 2.

**Betroffene Fragen:** alle Zählfragen („wie viele …“) und Ranglisten, sobald die Abfrage über
Heft 4 hinaus auch Chmel oder andere Hefte einbezieht.

**Regel**
- Als Dublette gilt nur ein Verweis im `Reg.:`-Teil.
- Das Heft wird bevorzugt: Gezählt wird das Heft-4-Regest, das verlinkte Chmel-Regest wird nicht
  mitgezählt, sondern als Hinweis angezeigt („auch bei Chmel n. 558, dort mit Mai 30“).
- Fragen nur zu Frankfurt filtern auf Heft 4. Dann gibt es keine Chmel-Dubletten.
- Nicht 1:1 zusammenführen: Mehrere Heft-Regesten können auf dasselbe Chmel-Regest zeigen
  (n. 22–25 → Chmel n. 558).

**Offen:** Bei Verweisen wie „Chmel n. 2, 9“ ist nur die erste Nummer verlinkt, und die passt nicht
immer (H. 4 n. 1 → Chmel n. 2 hat anderes Datum und anderen Ort). Die 19 Links mit abweichendem
Datum sind nicht einzeln durchgesehen.

### 2. Datumsangaben als Spanne

**Befund**
- Nur 41 Regesten haben `startDate <> endDate` (28 bis zu einem Monat, 13 ein ganzes Jahr, etwa
  „[1467]“). Mehrjährige Spannen gibt es bei uns nicht.
- Weitere 442 Regesten haben ein erschlossenes Datum in eckigen Klammern, aber
  `startDate = endDate`. Davon sind etwa 154 „vor …“ und 184 „etwa …“.
  Beispiel: „[Vor 1441 Mai 26]“ steht als exakt 1441-05-26 im Modell.
- Insgesamt sind 483 von 1.054 Daten (46 %) unsicher, 571 exakt.

**Betroffene Fragen:** alle Zeitfragen (Zeitraum, „im Jahr X“, frühestes/spätestes, Verlauf).

**Regel**
- Es zählt die Überlappung: `r.startDate <= $ende AND r.endDate >= $anfang`.
- Ein Datum gilt als unsicher, wenn `r.startDate <> r.endDate` oder `r.date CONTAINS '['`.
- Der Chatbot nennt die Anzahl der unsicheren Treffer („37 Regesten, davon 12 mit erschlossenem
  Datum“). Bei „vor …“ sagt er ausdrücklich, dass das gespeicherte Datum nur der späteste
  mögliche Tag ist.

### 3. Fehlende Ausstellungsorte

**Befund**
- Der Ort steht in `origPlaceOfIssue` und ist fast nie leer (1 × NULL). Stattdessen steht
  432 × „[-]“ (mit Varianten) und 9 × „o. O.“. Damit haben 442 Regesten (42 %) keinen Ort,
  und ein `IS NULL`-Test findet sie nicht.
- 52 Orte sind erschlossen und stehen in Klammern („[Frankfurt]“ 10 × neben „Frankfurt“ 30 ×, einmal mit Fußnotenzeichen).
- 35 Werte enthalten Fußnoten-Markup („[Graz]<sup>1</sup>“).
- 427 der 442 ortlosen Regesten sind Stücke, deren Urkunde nicht erhalten ist
  („Org. und Kop. noch nicht aufgetaucht“).

**Betroffene Fragen:** alle Ortsfragen („wo ausgestellt“, „wie viele in Frankfurt“, Itinerar, Karte).

**Regel**
- Lücke offen benennen, kein Erschließen per LLM: Bei verlorenen Stücken ist der Ort wirklich
  unbekannt, ein LLM würde raten.
- Vor dem Vergleich `<sup>…</sup>` und eckige Klammern entfernen. Geklammerte Orte zählen mit,
  mit dem Hinweis „erschlossen“.
- Jede Ortsantwort endet mit der Lücke („bei 442 von 1.054 Regesten ist kein Ort angegeben“).
- „Kein Ort“ heißt: NULL, beginnt mit `[-]` oder beginnt mit `o. O.`.

---

## Teil 2: Weitere Probleme aus der Datenbank (unabhängig von unseren Fragen)

### A. Ortlose Regesten landen an einem Schein-Ort „-“ (Importquelle)

Das Importskript verknüpft `origPlaceOfIssue` per exaktem Textvergleich mit `RI-Places.csv`.
Dort sind „[-]“, „o. O.“ und ihre Varianten auf den normalisierten Namen „-“ abgebildet, und das
Skript legt dafür einen echten `Place`-Knoten an. Folge: 441 unserer 442 ortlosen Regesten fallen bei
Ortsfragen nicht heraus, sondern hängen über `ri:placeOfIssue` alle an einem Ort namens „-“ und
bekommen `r.placeOfIssue = '-'`. In einer Rangliste wäre „-“ der häufigste Ausstellungsort, weit
vor Wiener Neustadt (125).

**Regel:** Bei jeder Ortsabfrage `p.normalizedGerman <> '-'` ausschließen und die Zahl getrennt
als Lücke ausweisen.

Positiv: Die übrigen 612 Regesten werden vollständig zugeordnet, auch die Schreibweisen mit
Klammern und `<sup>`. Die Normalisierung („[Frankfurt]“ → „Frankfurt am Main“,
„[Linz]<sup>1</sup>“ → „Linz an der Donau“) verschluckt dabei aber die Information
„erschlossen“. Sie bleibt nur in `origPlaceOfIssue` bzw. `a.original` erhalten.

### B. „Frankfurt“ hat vier verschiedene Bedeutungen und Zahlen (DB + Importquelle)

| Lesart | Regesten in Heft 4 |
|---|---|
| Aus dem Frankfurter Bestand (ganzes Heft) | 1.054 |
| „Frankfurt“ kommt im Regestentext vor | 638 |
| Registerlemma „Frankfurt (Hessen), Stadt“ samt Unterlemmata | 460 |
| Nur das Hauptlemma | 389 |
| Ausstellungsort Frankfurt (genannt oder erschlossen) | 41 |

Dazu kommen zwei getrennte Ortssysteme: der Ausstellungsort aus `RI-Places.csv`
(„Frankfurt am Main“) und das Registerlemma („Frankfurt (Hessen), Stadt“). Beide werden
voraussichtlich eigene Knoten für dieselbe Stadt. Im Register gibt es außerdem
„Frankfurt a. d. Oder“ und Personen „von Frankfurt“.

**Regel:** Der Chatbot muss bei „Frankfurt“ klären oder ansagen, welche Lesart er zählt.

### C. Registerhierarchie: Zählen am Hauptlemma unterschlägt Treffer (Importquelle)

Von 1.562 Registerlemmata mit Bezug auf Heft 4 sind 845 Unterlemmata. Wer nur das Hauptlemma
zählt, verliert Treffer (Frankfurt: 389 statt 460). Unterlemmata haben außerdem Namen ohne
Kontext („AO“, „Juden“, „Gericht“, „Tag (1454)“), die nur zusammen mit dem Oberlemma verständlich
sind.

**Regel:** Entitätsfragen immer über den Teilbaum (`IS_SUB_OF`) auswerten und Unterlemmata mit
dem Namen des Oberlemmas ausgeben.

### D. Rollen: „kommt vor“ ist nicht „Empfänger“ (Importquelle)

Die Registerverweise auf Heft 4 tragen eine Rolle: 3.361 × `nennung`, 1.606 × `empfaenger`,
490 × `austOrt`, 11 × `p_tot`. Ein eigenes Empfänger-Textfeld (`Recipient`) gibt es in Heft 4
nicht.

**Regel:** Fragen nach Empfängern filtern auf `a.role = 'empfaenger'`; Fragen nach „erwähnt“
nehmen alle Rollen. Ohne Rollenfilter werden Empfänger und bloße Erwähnungen vermischt.

### E. Der Aussteller steht nicht im Text (DB)

840 Regestentexte beginnen mit „K. F.“, 212 mit „Kg. F.“. Das Wort „Friedrich“ kommt nur in
42 Texten vor. Eine Volltextsuche nach „Friedrich III.“ findet also fast nichts. Der Aussteller
steht im Feld `Title` (bei uns immer „Friedrich III.“). „Kg.“ und „K.“ unterscheiden die Zeit vor
und nach der Kaiserkrönung 1452.

**Regel:** Aussteller über `Title` bzw. den Heftfilter bestimmen, nie über Textsuche. Dem LLM die
Abkürzungen „K. F.“ und „Kg. F.“ im Prompt erklären.

### F. Markup in den Texten (DB)

570 der 1.054 Regestentexte enthalten Markup, meist Fußnotenzeichen wie `<sup>[1]</sup>` mitten im
Satz. Während des Imports enthielten zudem 1.137 Textknoten unseres Hefts fehlerhaftes HTML
(`<a href=…>…</link>`). Ob das nach der Bereinigung verschwindet, ist nachzuprüfen.

**Regel:** Für Suche und LLM das bereinigte `text` verwenden, nicht `origText`, und Reste von
Markup vor der Ausgabe entfernen.

### G. Die Hälfte der Regesten sind verlorene Stücke (DB)

Bei 517 von 1.054 Regesten (49 %) steht in `archivalHistory` „noch nicht aufgetaucht“. Diese
Stücke sind aus anderen Quellen erschlossen. Sie haben meist kein exaktes Datum und keinen Ort
(427 der 517 sind ortlos). Etwa 371 Regesten beginnen mit „Org.“ (Original erhalten), 158 mit
„Kop.“.

**Regel:** Bei „wie viele Urkunden“ sagen, ob erschlossene Stücke mitgezählt sind. Fragen nach
Überlieferung, Siegel oder Archivsignatur gelten nur für die erhaltenen Stücke.

### H. Chmel ist anders erschlossen als die Hefte (DB)

- Ortsnamen: Chmel schreibt 2.177 × „Neustadt“ und nie „Wiener Neustadt“.
- Text: Chmel-Regesten beginnen ohne Subjekt und klein („erklärt, dass …“), mit anderer
  Rechtschreibung.
- 577 der 8.972 Chmel-Regesten haben eine Datumsspanne. Die 133 Regesten im Chmel-Anhang haben
  gar keinen Ort.

**Regel:** Vergleiche über Hefte und Chmel hinweg nur über normalisierte Orte, nicht über
`origPlaceOfIssue`. Stilunterschiede bei Textsuche und Zusammenfassungen einkalkulieren.

### I. Die Datenbank enthält alle Regesta Imperii, nicht nur Friedrich III. (DB)

Von 212.176 Regesten gehören nur 26.067 zu Friedrich III. (RI XIII und Chmel). Eine Abfrage ohne
Filter zählt alle Herrscher mit. „Frankfurt“ als Ausstellungsort gibt es datenbankweit in
mindestens acht Schreibweisen („Frankfurt“ 2.541, „Frankfurt [a. Main]“ 404, „Frankfurt am Main“
312, „Frankfurt a. Main“ 277 …).

**Regel:** Jede Abfrage beginnt mit dem Heft- oder Herrscherfilter.

### J. Regestennummern (DB + Importquelle)

- Sieben Nummern haben einen Buchstabenzusatz (391a, 399a, 811a, 816a, 816b, 817a, 932a). Die
  höchste Nummer ist 1047, es sind aber 1.054 Regesten.
- `regestennummernorm` ist ein Text. Sortieren danach ergibt „10“ vor „2“.
- Im Register stehen zwei Nummern, die auf kein Regest passen („#4-1241“, „#4-907 (?)“). Diese
  Verweise gehen beim Import verloren. Ein Regest hat gar keinen Registereintrag.

**Regel:** Nach `sorting` sortieren, nie die Anzahl aus der höchsten Nummer ableiten.

### K. Viele Regesten am selben Tag, darunter Serienschreiben (DB)

An 208 Tagen gibt es mehr als ein Regest (zusammen 653 Regesten, bis zu 19 an einem Tag). Darunter
sind gleichlautende Schreiben an verschiedene Empfänger als eigene Regesten (n. 1000 an Nürnberg,
n. 1001 an Speyer, beide 10. Dezember 1489 zum Feldzug).

**Regel:** „Das Regest vom …“ ist oft mehrdeutig; dann alle Treffer des Tages nennen. Bei
Zählfragen zu einem Thema sagen, dass Serienschreiben einzeln zählen.

### L. Datumsfelder sind Text, Fehlwerte werden zu 2021 (Importquelle)

`startDate` und `endDate` sind Zeichenketten. Der Vergleich funktioniert nur, weil sie im
ISO-Format stehen; echte Datumswerte (`isoStartDate`, `isoEndDate`) ergänzt der Import später.
Ungültige Daten setzt das Skript auf `2021-01-01` und merkt das Original in `startDateWrong` vor.
In Heft 4 liegen alle Daten zwischen 1440-02-22 und 1493-06-03, bei uns ist also bisher nichts
betroffen. Bei Abfragen über Heft 4 hinaus können solche Regesten als „jüngste“ auftauchen.

**Regel:** Zeitabfragen über Heft 4 hinaus schließen `startDateWrong IS NOT NULL` aus.

### M. Sehr unterschiedlich lange Texte (DB)

Die Regestentexte sind zwischen 32 und 11.562 Zeichen lang (Mittel 511). Lange Regesten sprengen
ein knappes Kontextfenster und dominieren eine Ähnlichkeitssuche anders als kurze.

**Regel:** Für das LLM eine Längenobergrenze oder Abschnitte vorsehen und Kürzungen ansagen.

### N. Felder, die nur bei einem Teil vorhanden sind (DB)

| Feld | Regesten mit Feld |
|---|---|
| Title, Date, Summary, ArchivalHistory, Identifier | 1.054 |
| PlaceOfIssue | 1.053 |
| OriginalDate (Datierung im Wortlaut) | 540 |
| VersoNote (Rückvermerk) | 515 |
| Footnotes | 428 |
| Recipient, Witnesses, Incipit, Commentary, Literature | 0 |

Fragen zu Zeugen, Incipit oder Kanzlei lassen sich für unser Heft nicht aus eigenen Feldern
beantworten. Der Regestentext steht nur im `Summary`-Knoten, nicht als Eigenschaft am Regest.

---

## Nachprüfen, sobald der Import fertig ist

1. Punkt A: Gibt es den `Place`-Knoten „-“, und hängen 441 unserer Regesten daran?
2. Punkt B: Sind Ausstellungsort und Registerlemma für Frankfurt zwei getrennte Knoten?
3. Punkte C und D: Kommen `IS_SUB_OF` und `a.role` so in der Datenbank an wie in der Quelle?
4. Punkt F: Ist das fehlerhafte HTML aus `text` verschwunden?
5. Alle Zahlen aus Teil 1 einmal neu ausführen (der Import korrigiert noch Datumswerte).
