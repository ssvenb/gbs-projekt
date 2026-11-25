# Aufgabe 2: Fallstricke des Modells (Gruppe 2)

Unser Bestand: **RI XIII H. 4** (Frankfurt am Main), 1.054 Regesten, Filter
`r.identifier STARTS WITH '[RI XIII] H. 4 '`.
Stand der Messung: 25.11.2026, lokale Neo4j-Datenbank aus `docker-compose.yml`.

> **Stand:** Der Import ist abgeschlossen. Alle Zahlen sind in der fertigen Datenbank gemessen
> bzw. nachgeprüft. Wo die fertige Datenbank von der ersten Einschätzung abwich, steht das dabei.

---

## Teil 1: Die drei bekannten Probleme

Die Spalte „betroffene Fragen“ ist nach Fragetyp gefüllt, weil unsere zehn Fragen hier noch nicht
eingetragen sind.

### 1. Doppelte Regesten (Chmel und Hefte)

**Befund**
- 100 unserer Regesten nennen im `Reg.:`-Teil von `archivalHistory` ein Chmel-Regest als Link
  (`riregesten:<exchangeIdentifier>`). Alle 100 Links lassen sich auflösen und führen auf
  94 verschiedene Chmel-Regesten.
- „Gleiches Datum + gleicher Ort“ taugt nicht als Erkennung: 136 unserer Regesten haben ein
  Chmel-Regest vom selben Tag und Ort, 122 davon ohne Verweis. In der Stichprobe waren das
  durchweg andere Urkunden.
- Umgekehrt weicht bei 18 der 100 Links das Datum ab (H. 4 n. 22 „Mai 31“, Chmel n. 558 „Mai 30“),
  und auch der normalisierte Ort stimmt nur bei 71 der 100 überein (Chmel: „Neustadt“,
  Heft: „Wiener Neustadt“).
- 25 weitere Regesten erwähnen Chmel nur außerhalb des `Reg.:`-Teils. Das ist ein Querverweis,
  keine Dublette.
- Im `Reg.:`-Teil verweisen außerdem 27 Regesten auf H. 3, vier auf H. 1 und zwei auf H. 2.
- Im Graphen sind die Links als Annotation abfragbar:
  `(:ArchivalHistory)-[:HAS_ANNOTATION]->(a {type:'a'})-[:REFERS_TO]->(:Regesta)`.
  Darüber hängen 124 unserer Regesten an Chmel, also auch die bloßen Querverweise. Die Kante
  allein unterscheidet Dublette und Querverweis nicht; dafür braucht man die Position im Text
  (hinter `Reg.:`). 407 unserer Regesten verweisen so auf andere Regesten in Heft 4.

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
immer (H. 4 n. 1 → Chmel n. 2 hat anderes Datum und anderen Ort). Die 18 Links mit abweichendem
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

### A. Ortlose Regesten hängen an einem Schein-Ort „-“

Der Import verknüpft `origPlaceOfIssue` mit einem Ortsverzeichnis. Dort sind „[-]“, „o. O.“ und
ihre Varianten auf den Namen „-“ abgebildet, und dafür gibt es einen echten `Place`-Knoten.
441 unserer 442 ortlosen Regesten fallen bei Ortsfragen also nicht heraus, sondern hängen über
`ri:placeOfIssue` an einem Ort mit `label = '-'` und haben `r.placeOfIssue = '-'`. In einer
Rangliste ist „-“ der häufigste Ausstellungsort, weit vor Wiener Neustadt (132). Nur ein Regest
hat wirklich keinen Ort (NULL).

Zwei weitere Folgen:
- **Doppelte Ortsannotationen:** 44 Regesten haben zwei `ri:placeOfIssue`-Annotationen auf
  denselben Ort (alle „Linz“ → „Linz an der Donau“). Wer Annotationen statt Regesten zählt,
  bekommt für Linz 89 statt 46.
- **„Erschlossen“ geht verloren:** „[Frankfurt]“ und „Frankfurt“ landen beide bei „Frankfurt am
  Main“. Dass ein Ort erschlossen ist, steht nur noch in `origPlaceOfIssue` bzw. `a.original`.

**Regel:** Bei jeder Ortsabfrage `p.label <> '-'` ausschließen und die Zahl getrennt als Lücke
ausweisen. Immer `count(DISTINCT r)` zählen.

### B. „Frankfurt“ hat vier verschiedene Bedeutungen und Zahlen

| Lesart | Regesten in Heft 4 |
|---|---:|
| Aus dem Frankfurter Bestand (ganzes Heft) | 1.054 |
| „Frankfurt“ kommt im Regestentext vor | 638 |
| Registerlemma „Frankfurt (Hessen), Stadt“ samt Unterlemmata | 460 |
| Nur das Hauptlemma | 389 |
| Ausstellungsort Frankfurt (genannt oder erschlossen) | 41 |

Bestätigt: Für dieselbe Stadt gibt es getrennte `Place`-Knoten. Der Ausstellungsort hängt an
„Frankfurt am Main“ (41 Regesten), die Registernennungen hängen an „Frankfurt (Hessen), Stadt“
(389). Zwischen beiden gibt es keine Verbindung. Zusätzlich existiert ein zweiter Knoten
„Frankfurt am Main“ ohne Bezug zu unserem Heft, außerdem „Frankfurt an der Oder“ und
„Frankfurt auf der Oder“. Das Register führt den Ausstellungsort noch einmal als Unterlemma
„Frankfurt (Hessen), Stadt // AO“ (41 Regesten).

**Regel:** Der Chatbot muss bei „Frankfurt“ klären oder ansagen, welche Lesart er zählt.

### C. Registerhierarchie: Zählen am Hauptlemma unterschlägt Treffer

Das Register ist hierarchisch. Die Hierarchie ist keine Kante, sondern eine Annotation:
`(kind)-[:HAS_ANNOTATION]->({type:'ri:isSubOf'})-[:REFERS_TO]->(eltern)`. Am Knoten
„Frankfurt (Hessen), Stadt“ hängen 79 direkte Unterlemmata (laut `numberOfChilds` 1.289 im ganzen
Teilbaum). Wer nur das Hauptlemma zählt, findet 389 unserer Regesten, mit Unterlemmata bis zur
zweiten Ebene sind es 460.

Anders als zuerst vermutet tragen die Unterlemmata in der Datenbank den vollen Pfad im Namen
(„Frankfurt (Hessen), Stadt // Juden // Synagoge, jüdische Schule“). Sie sind also verständlich,
aber eine Suche nach exakt „Frankfurt (Hessen), Stadt“ trifft sie nicht. Unterlemmata können
außerdem einen anderen Typ haben als das Oberlemma (unter dem Ort Frankfurt hängen Personen wie
„// Schultheiß“ und Sachen wie „// Messe“).

**Regel:** Entitätsfragen über den Teilbaum auswerten (`ri:isSubOf` folgen oder
`label STARTS WITH`), nicht nur über das Hauptlemma.

### D. Rollen: „kommt vor“ ist nicht „Empfänger“

Entitäten hängen über `(e)-[:HAS_ANNOTATION]->(a {type:'ri:appearsIn'})-[:REFERS_TO]->(r)` am
Regest, die Rolle steht in `a.role`. Für Heft 4:

| Rolle | Verweise | davon Personen | davon Orte |
|---|---:|---:|---:|
| `nennung` | 3.359 | 2.274 | 881 |
| `empfaenger` | 1.606 | 789 | 798 |
| `austOrt` | 490 | 0 | 490 |
| `p_tot`, `p_komment` | 12 | 12 | 0 |

- Empfänger sind zur Hälfte Orte (Städte als Empfänger), nicht nur Personen.
- Ein eigenes Empfänger-Textfeld (`Recipient`) gibt es in Heft 4 nicht.
- Der Ausstellungsort ist doppelt modelliert: als `ri:placeOfIssue` (612 Regesten mit echtem Ort)
  und als Registerrolle `austOrt` (486 Regesten). Die beiden Wege liefern verschiedene Zahlen und
  führen auf verschiedene Ortsknoten.

**Regel:** Fragen nach Empfängern filtern auf `a.role = 'empfaenger'` und schließen Orte ein;
Fragen nach „erwähnt“ nehmen alle Rollen. Für den Ausstellungsort gilt nur `ri:placeOfIssue`.

### E. Der Aussteller steht nicht im Text

840 Regestentexte beginnen mit „K. F.“, 212 mit „Kg. F.“. Das Wort „Friedrich“ kommt nur in
42 Texten vor. Eine Volltextsuche nach „Friedrich III.“ findet also fast nichts. Der Aussteller
steht im Feld `Title` (bei uns immer „Friedrich III.“). „Kg.“ und „K.“ unterscheiden die Zeit vor
und nach der Kaiserkrönung 1452.

**Regel:** Aussteller über `Title` bzw. den Heftfilter bestimmen, nie über Textsuche. Dem LLM die
Abkürzungen „K. F.“ und „Kg. F.“ im Prompt erklären.

### F. Markup und Fußnotenziffern in den Texten

Nach dem Import ist `text` bereinigt: Nur noch einer von 7.806 Textknoten unseres Hefts enthält
ein Tag. `origText` enthält weiter das rohe Markup (2.954 Knoten). Übrig bleiben in `text` aber
die Fußnotenziffern: 329 Regestentexte enthalten ein „[1]“ mitten im Satz.

**Regel:** Für Suche und LLM `text` verwenden, nicht `origText`. Fußnotenziffern vor der Ausgabe
entfernen oder als Verweis auf den `Footnotes`-Knoten erklären.

### G. Die Hälfte der Regesten sind verlorene Stücke

Bei 517 von 1.054 Regesten (49 %) steht in `archivalHistory` „noch nicht aufgetaucht“. Diese
Stücke sind aus anderen Quellen erschlossen. Sie haben meist kein exaktes Datum und keinen Ort
(427 der 517 sind ortlos). Etwa 371 Regesten beginnen mit „Org.“ (Original erhalten), 158 mit
„Kop.“.

**Regel:** Bei „wie viele Urkunden“ sagen, ob erschlossene Stücke mitgezählt sind. Fragen nach
Überlieferung, Siegel oder Archivsignatur gelten nur für die erhaltenen Stücke.

### H. Chmel ist anders erschlossen als die Hefte

- Ortsnamen: Chmel schreibt 2.177 × „Neustadt“ und nie „Wiener Neustadt“. Auch der normalisierte
  Ort bleibt „Neustadt“. Chmel und Hefte hängen für dieselbe Stadt an zwei verschiedenen Knoten.
- Text: Chmel-Regesten beginnen ohne Subjekt und klein („erklärt, dass …“), mit anderer
  Rechtschreibung.
- 577 der 8.972 Chmel-Regesten haben eine Datumsspanne. Die 133 Regesten im Chmel-Anhang haben
  gar keinen Ort.

**Regel:** Bei Vergleichen über Hefte und Chmel hinweg reicht auch der normalisierte Ort nicht;
„Neustadt“ und „Wiener Neustadt“ müssen von Hand gleichgesetzt werden. Stilunterschiede bei
Textsuche und Zusammenfassungen einkalkulieren.

### I. Die Datenbank enthält alle Regesta Imperii, nicht nur Friedrich III.

Von 212.176 Regesten gehören nur 26.067 zu Friedrich III. (RI XIII und Chmel). Eine Abfrage ohne
Filter zählt alle Herrscher mit. „Frankfurt“ als Ausstellungsort gibt es datenbankweit in
mindestens acht Schreibweisen („Frankfurt“ 2.541, „Frankfurt [a. Main]“ 404, „Frankfurt am Main“
312, „Frankfurt a. Main“ 277 …).

**Regel:** Jede Abfrage beginnt mit dem Heft- oder Herrscherfilter.

### J. Regestennummern

- Sieben Nummern haben einen Buchstabenzusatz (391a, 399a, 811a, 816a, 816b, 817a, 932a). Die
  höchste Nummer ist 1047, es sind aber 1.054 Regesten.
- `regestennummernorm` ist ein Text. Sortieren danach ergibt „10“ vor „2“.
- Im Register stehen zwei Nummern, die auf kein Regest passen („#4-1241“, „#4-907 (?)“). Diese
  Verweise sind beim Import verloren gegangen. Jedes unserer Regesten hat aber mindestens eine
  Entität.

**Regel:** Nach `sorting` sortieren, nie die Anzahl aus der höchsten Nummer ableiten.

### K. Viele Regesten am selben Tag, darunter Serienschreiben

An 208 Tagen gibt es mehr als ein Regest (zusammen 653 Regesten, bis zu 19 an einem Tag). Darunter
sind gleichlautende Schreiben an verschiedene Empfänger als eigene Regesten (n. 1000 an Nürnberg,
n. 1001 an Speyer, beide 10. Dezember 1489 zum Feldzug).

**Regel:** „Das Regest vom …“ ist oft mehrdeutig; dann alle Treffer des Tages nennen. Bei
Zählfragen zu einem Thema sagen, dass Serienschreiben einzeln zählen.

### L. Datumsfelder: Text und echtes Datum nebeneinander, Fehlwerte werden zu 2021

`startDate` und `endDate` sind Zeichenketten im ISO-Format. Daneben gibt es jetzt echte
Datumswerte in `isoStartDate` und `isoEndDate` (bei allen 1.054 Regesten vorhanden). Ungültige
Daten hat der Import auf `2021-01-01` gesetzt und das Original in `startDateWrong` bzw.
`endDateWrong` abgelegt. Das betrifft datenbankweit 10 Regesten, davon 2 bei Friedrich III. und
keines in Heft 4.

**Regel:** Für Datumsrechnung `isoStartDate`/`isoEndDate` nehmen. Zeitabfragen über Heft 4 hinaus
schließen Regesten mit `startDateWrong` oder `endDateWrong` aus.

### M. Sehr unterschiedlich lange Texte

Die Regestentexte sind zwischen 32 und 11.562 Zeichen lang (Mittel 511). Lange Regesten sprengen
ein knappes Kontextfenster und dominieren eine Ähnlichkeitssuche anders als kurze.

**Regel:** Für das LLM eine Längenobergrenze oder Abschnitte vorsehen und Kürzungen ansagen.

### N. Felder, die nur bei einem Teil vorhanden sind

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

### O. Eigenschaften der Entitäten sind Annotationen, keine Properties

Ein `Entity`-Knoten hat nur `label`, `uuid`, `pathLength` und `numberOfChilds`. Alles andere
hängt als eigene Annotation mit `a.type` und `a.value` daran: `xmlId`, `wikidataId`, `latitude`,
`longitude`, `latLong`, `type`, `normalizedGerman`. Eine Abfrage wie `p.latitude` oder
`p.normalizedGerman` liefert deshalb immer NULL, ohne Fehlermeldung.

Koordinaten gibt es an zwei Stellen:
- am Regest (`r.nLatLong`, `r.latitude`, `r.longitude`) für den Ausstellungsort: bei 612 von
  1.054 Regesten vorhanden;
- als Annotation an Registerorten: nur 172 der 412 in Heft 4 vorkommenden Orte haben welche.

**Regel:** Entitätseigenschaften immer über die Annotation lesen. Kartenantworten nennen, wie
viele Treffer keine Koordinaten haben.

---

## Ergebnis der Nachprüfung nach Abschluss des Imports

| Punkt | Ergebnis |
|---|---|
| A Schein-Ort „-“ | Bestätigt: 441 Regesten hängen daran. Neu: 44 Regesten mit doppelter Ortsannotation. |
| B Frankfurt-Knoten | Bestätigt: getrennte Knoten für Ausstellungsort und Registerlemma, dazu ein zweiter „Frankfurt am Main“. |
| C Hierarchie | Bestätigt (389 gegenüber 460). Korrigiert: Unterlemmata tragen den vollen Pfad im Namen; die Hierarchie ist eine Annotation. |
| D Rollen | Bestätigt, Zahlen aus der Datenbank eingesetzt. |
| F Markup | Entschärft: `text` ist sauber, es bleiben Fußnotenziffern in 329 Texten. |
| J Register | Korrigiert: Jedes Regest hat mindestens eine Entität. |
| L Datum | Bestätigt: 10 Fehlwerte datenbankweit, keiner in Heft 4. |
| Teil 1 | Datums- und Ortszahlen unverändert. Chmel-Links korrigiert auf 100 Links zu 94 Chmel-Regesten, 18 mit abweichendem Datum. |
