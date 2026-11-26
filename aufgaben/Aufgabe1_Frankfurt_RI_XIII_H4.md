# Aufgabe 1: Testfragen und regionale Ergänzung
**Region:** Frankfurt am Main · **Heft:** RI XIII H. 4 – *Regesten Kaiser Friedrichs III. (1440–1493), Heft 4: Die Urkunden und Briefe aus dem Stadtarchiv Frankfurt am Main*, bearb. von Paul-Joachim Heinig (1986)

> **Stand:** Alle Angaben zum Schema und alle Zahlen stammen aus Abfragen gegen die vollständig importierte lokale Kopie von RI-alles (Neo4j im Docker-Container, 25.11.2026). Die Cypher-Abfragen der Fragen 1, 2, 5, 6, 8 und 9 sowie die Grundlagen der übrigen Fragen wurden dort ausgeführt; Abfragen auf die geplante Ergänzung (Fragen 3, 7, 10 und Teile von 4, 6) sind Entwürfe.

---

## Schritt 1: Das Modell erkunden

### 1.1 Aufbau von RI-alles

RI-alles ist ein Text-als-Graph-Modell mit **Standoff-Annotationen**: Beziehungen zwischen Regesten, Entities und Textfeldern werden als eigene `Annotation`-Knoten modelliert. Deren Eigenschaft `type` sagt, was sie bedeuten. Es gibt nur drei Beziehungstypen: `PART_OF`, `HAS_ANNOTATION`, `REFERS_TO`.

```
// Gliederung
(:Collection:Regesta) -[:PART_OF]-> (:Volume) -[:PART_OF]-> (:Collection:Department)
(:Content:Text:<Feld>) -[:PART_OF]-> (:Collection:Regesta)

// Register: Entity kommt in Regest vor, mit Rolle
(:Entity) -[:HAS_ANNOTATION]-> (:Annotation {type:'ri:appearsIn', role, sourceType}) -[:REFERS_TO]-> (:Regesta)

// Normierter Ausstellungsort
(:Regesta) -[:HAS_ANNOTATION]-> (:Annotation {type:'ri:placeOfIssue', original}) -[:REFERS_TO]-> (:Entity:Place)

// Registerhierarchie (Untereinträge)
(:Entity) -[:HAS_ANNOTATION]-> (:Annotation {type:'ri:isSubOf'}) -[:REFERS_TO]-> (:Entity)

// Reihenfolge der Textfelder
(:Content) -[:HAS_ANNOTATION]-> (:Annotation:Next {type:'next'}) -[:REFERS_TO]-> (:Content)
```

**Knoten und Eigenschaften**

| Knoten | wichtige Eigenschaften | Bedeutung |
|---|---|---|
| `Collection:Regesta` | `identifier` („[RI XIII] H. 4 n. 21“), `regestennummernorm` („21“), `bandpk` (`13004000`), `date` (Originalangabe), `startDate`/`endDate` (ISO-Text), `origPlaceOfIssue`, `archivalHistory`, `exchangeIdentifier`, `laufendenummer`, `urn` | ein Regest |
| `Volume` | `label` („4“), `type` („volume“) | Heft/Band |
| `Collection:Department` | `label` („RI13“) | Abteilung der RI |
| `Entity` mit Unterlabel `Person`, `Place`, `Thing`, `Event` | `label` (vollständiger Registereintrag, Untereinträge mit „ // “), `uuid`, `pathLength`, `numberOfChilds` | Registereintrag |
| `Annotation` | `type`, `label`, `uuid`; bei `ri:appearsIn` zusätzlich `role`, `sourceType`, `regestaNumber`, `importKey`; bei `ri:placeOfIssue` zusätzlich `original` | Verbindung mit Bedeutung |
| `Content:Text:<Feld>` | `text` (HTML), `origText` | Textfeld: `Summary`, `Date`, `PlaceOfIssue`, `VersoNote`, `ArchivalHistory`, `OriginalDate`, `Footnotes` … |

Weitere `Annotation.type`-Werte sind Textauszeichnungen aus dem HTML (`a`, `em`, `sup`, `spaced`, `smallcaps` …). Für den Chatbot sind sie nicht relevant.

**Werte von `role` bei `ri:appearsIn`** (gesamt; häufigste): `nennung` (genannt), `empfaenger` (Empfänger), `austOrt` (Ausstellungsort laut Register), `zeuge`, `datOrt`, `p_tot`, `p_komment`, `notar`, `adressat`, `aussteller`.
`sourceType` ist bei einem Teil gesetzt (`U`, `H`, `Dep`); die genaue Bedeutung bitte bei den Betreuenden erfragen.

### 1.2 Heft 4 im Modell

```cypher
MATCH (:Department {label:'RI13'})<-[:PART_OF]-(v:Volume {label:'4'})<-[:PART_OF]-(r:Regesta)
RETURN count(r);   // 1054
```
`Volume.label` allein ist nicht eindeutig (jede Abteilung hat ein Heft „4“), daher immer zusammen mit `Department` filtern. Gleichwertig: `r.bandpk = '13004000'`.

| Merkmal | Wert |
|---|---|
| Regesten | **1054**, 1440-02-22 bis 1493-06-03 |
| Rollen in Heft 4 | `nennung` (Personen 2274 / Orte 881 / Ereignisse 119 / Sachen 72), `empfaenger` (Orte 798 / Personen 789), `austOrt` (490), vereinzelt `p_tot`, `p_komment`. **Nicht** vorhanden: `zeuge`, `adressat`, `aussteller`, `notar` |
| `ri:placeOfIssue` | 1053 Regesten, davon **441 auf den Platzhalter-Ort „-“** (Original „[-]“, „o. O.“) |
| Ausstellungsorte | Wiener Neustadt 132 · Graz 110 · Wien 100 · Linz an der Donau 89 · **Frankfurt am Main 41** · Nürnberg 27 · Köln 23 · Andernach 22 · Augsburg 14 |
| Datum erschlossen („[…]“) | 483 · echte Spannen (`startDate ≠ endDate`): 41 |
| Chmel-Dubletten | 124 Regesten ↔ 136 Chmel-Regesten (Chmel: `bandpk = '13000000'`) |
| pro Jahrzehnt | 1440er 171 · 1450er 124 · 1460er 198 · **1470er 328** · 1480er 189 · 1490er 44 |

**Zwei Ortssysteme:** `ri:placeOfIssue` verweist auf normierte Orte („Frankfurt am Main“), die Register-Rolle `austOrt` auf Registereinträge („Frankfurt (Hessen), Stadt // AO“). Für Fragen nach dem Ausstellungsort nehmen wir `ri:placeOfIssue` (fast vollständig), für Empfänger und Nennungen das Register.

### 1.3 Beispiel: H. 4 n. 21

| Bestandteil | Inhalt |
|---|---|
| `Regesta` | identifier „[RI XIII] H. 4 n. 21“, date „1442 Mai 30“, startDate = endDate „1442-05-30“ |
| `ri:placeOfIssue` | original „Frankfurt“ → Place „Frankfurt am Main“ |
| `ri:appearsIn` | `austOrt`: Frankfurt (Hessen), Stadt // AO · `empfaenger`: Frankfurt (Hessen), Stadt · `nennung`: Heinrich Leubing (Protonotar), Jakob Widerl (Registrator), Kg. Sigmund u. a. |
| `Summary` | Kg. F. quittiert Bürgermeistern, Schöffen und Rat der Stadt Frankfurt die Bezahlung der Stadtsteuer des Jahres 1437 … |
| `VersoNote` | KVr: *A. rel. m(a)g(ist)ri camere Heinric(us) Leubing doctor prothon(otarius).* – KVv: *Rta Jacobus Widerl* |
| `ArchivalHistory` | Org. im StadtA Frankfurt/M. … · Verweis auf Chmel n. 558 |

```cypher
// Regest mit allen Registereinträgen und Rollen
MATCH (r:Regesta {identifier:'[RI XIII] H. 4 n. 21'})<-[:REFERS_TO]-(a:Annotation {type:'ri:appearsIn'})
      <-[:HAS_ANNOTATION]-(e:Entity)
RETURN a.role, [l IN labels(e) WHERE l <> 'Entity'][0] AS art, e.label;
```

### 1.4 Was dem Register fehlt (Ansatzpunkte für die Ergänzung)

Die Prüfung an Beispielregesten zeigt, dass das Register nur grob zwischen „Empfänger“ und „genannt“ unterscheidet:

| Regest | Inhalt | Register sagt | eigentlich |
|---|---|---|---|
| n. 110 | Bitte an den Rat, Johannes Snyder ein Gotteslehen zu geben | Snyder: `nennung`, Frankfurt: `empfaenger` | Snyder ist **Begünstigter** |
| n. 32 | Befehl an den Schultheißen Hermann von Hoch-Weisel, Katharina von Ingelheim zu schützen | Hoch-Weisel: `empfaenger`, Katharina: `nennung` | Katharina ist **Begünstigte**, Hoch-Weisel **Beauftragter** |
| n. 35 | Gebot, die Acht gegen Lüttich, Huy, St. Truiden, Maastricht, Tongern u. a. zu vollziehen | Städte: `nennung` | Städte sind **Betroffene** |
| n. 21 | Quittung; Kanzleivermerk mit Heinrich Leubing | Leubing: `nennung` | Leubing ist **Relator/Protonotar** der Urkunde |

Außerdem gibt es **keine Rechtsakte** (Quittung, Privileg, Befehl …) im Modell.

---

## Schritte 2–4: Zehn Fragen, Abbildung aufs Modell, Lücken

### 2.1 Die MCP-Tools des Chatbots

Der Chatbot schreibt kein freies Cypher, sondern ruft feste Tools auf. Alle Tools filtern Chmel-Dubletten, schließen den Platzhalter-Ort „-“ aus und behandeln Datumsangaben nach den Regeln aus Aufgabe 2. Jedes Ergebnis enthält `identifier` und `urn` als Beleg.

| Tool | Parameter | Zweck |
|---|---|---|
| `search_regesta` | `text?`, `heft?` (z. B. „RI XIII H. 4“), `date_from?`, `date_to?`, `place?` | Regesten filtern |
| `get_regest` | `identifier` | ein Regest mit Feldern, Ort und allen Registereinträgen samt Rollen |
| `find_entity` | `name`, `type?` (`Person`/`Place`/`Thing`/`Event`) | Registereintrag suchen (Volltext auf `Entity.label`), Kandidaten zur Auswahl |
| `regesta_by_entity` | `entity_uuid`, `role?`, `heft?`, `source?` | Regesten zu einer Entity, optional nach Rolle |
| `itinerary` | `heft?`, `date_from?`, `date_to?` | Ausstellungsorte (`ri:placeOfIssue`) mit Anzahl |
| `count_regesta` | `group_by` (`year`/`decade`/`place`/`role`), Filter | Zählungen |
| `co_recipients` | `entity_uuid`, `heft?` | weitere Empfänger derselben Regesten |
| `legal_acts` *(Ergänzung)* | `act?`, `heft?`, `date_from?`, `date_to?`, `source?` | Regesten mit bestimmtem Rechtsakt |

### 2.2 Übersicht

| # | Frage der Lehrkraft | Knoten, Annotationen, Rollen | Tool(s) | Status |
|---|---|---|---|---|
| 1 | Welche Urkunden stellte Friedrich III. 1442 in Frankfurt aus? | Regesta, Volume, `ri:placeOfIssue` → Place | `search_regesta` | Basis |
| 2 | Von wo aus schrieb Friedrich an Frankfurt? | Regesta, Volume, `ri:placeOfIssue` → Place | `itinerary` | Basis |
| 3 | Welche Privilegien bestätigte oder verlieh Friedrich der Stadt? | Regesta + **LegalAct** | `legal_acts` | **Ergänzung** |
| 4 | Welche Steuern zahlte Frankfurt an den König? | Regesta, `Summary`; + **LegalAct** `receipt` | `search_regesta`, `legal_acts` | Basis, genauer mit Ergänzung |
| 5 | Was regelte der Frankfurter Landfrieden von 1442? | Regesta n. 41, `Summary`, `ri:appearsIn` | `get_regest` | Basis |
| 6 | Welche Kanzleibeamten wirkten an den Frankfurter Urkunden mit? | Person, `ri:appearsIn {role:'nennung'}`; + **Rolle `relator`** | `regesta_by_entity` | Basis, genauer mit Ergänzung |
| 7 | Welche Befehle erteilte der König dem Rat, und gegen wen richteten sie sich? | Regesta + **LegalAct** + **Rolle `affected`** | `legal_acts`, `regesta_by_entity` | **Ergänzung** |
| 8 | Wie entwickelte sich der Kontakt über die Jahrzehnte? | Regesta (`startDate`/`endDate`), Volume | `count_regesta(decade)` | Basis |
| 9 | Welche anderen Städte und Herren wurden zusammen mit Frankfurt angeschrieben? | Place/Person, `ri:appearsIn {role:'empfaenger'}` | `find_entity`, `co_recipients` | Basis |
| 10 | Welche einzelnen Personen begünstigte der König in Frankfurt? | Person + **Rolle `beneficiary`** + LegalAct | `regesta_by_entity(role:'beneficiary')` | **Ergänzung** |

**Bilanz:** 5 Fragen gehen mit dem Basismodell (1, 2, 5, 8, 9), 2 werden mit der Ergänzung genauer (4, 6), 3 brauchen sie (3, 7, 10).

### 2.3 Die Fragen im Einzelnen

#### Frage 1: Welche Urkunden stellte Friedrich III. 1442 in Frankfurt aus?
- **Modell:** Regesta → Volume „4“ → Department „RI13“; `ri:placeOfIssue` → Place „Frankfurt am Main“; Datum
- **Tool:** `search_regesta(place:"Frankfurt am Main", heft:"RI XIII H. 4", date_from:"1442-01-01", date_to:"1442-12-31")`
- **Cypher:**
  ```cypher
  MATCH (:Department {label:'RI13'})<-[:PART_OF]-(:Volume {label:'4'})<-[:PART_OF]-(r:Regesta)
        -[:HAS_ANNOTATION]->(a:Annotation {type:'ri:placeOfIssue'})-[:REFERS_TO]->(:Place {label:'Frankfurt am Main'})
  WHERE r.startDate <= '1442-12-31' AND r.endDate >= '1442-01-01'
  MATCH (r)<-[:PART_OF]-(s:Summary)
  RETURN r.identifier, r.date, a.original AS ortImOriginal,
         apoc.text.regreplace(s.text, '<[^>]+>', '') AS text
  ORDER BY r.startDate;
  ```
- **Ergebnis (geprüft):** **19 Regesten** (30. Mai – 14. August 1442), davon 2 mit erschlossenem Ort („[Frankfurt]“). Darunter n. 21 (Quittung Stadtsteuer 1437), n. 31 (Bestätigung der beiden Messen und der Privilegien), n. 32 (Schutzbefehl), n. 35 (Gebot zum Achtvollzug), n. 41 (Landfrieden).
- **Variante:** Über alle Jahre: 1442: 19 · 1474: 2 · 1485: 3 · **1486: 17** (Aufenthalt zur Königswahl Maximilians).

#### Frage 2: Von wo aus schrieb Friedrich an Frankfurt?
- **Modell:** Regesta, Volume, `ri:placeOfIssue` → Place
- **Tool:** `itinerary(heft:"RI XIII H. 4")`
- **Cypher:**
  ```cypher
  MATCH (:Department {label:'RI13'})<-[:PART_OF]-(:Volume {label:'4'})<-[:PART_OF]-(r:Regesta)
        -[:HAS_ANNOTATION]->(a:Annotation {type:'ri:placeOfIssue'})-[:REFERS_TO]->(p:Place)
  RETURN CASE p.label WHEN '-' THEN '[ohne Ort]' ELSE p.label END AS ort,
         count(*) AS anzahl,
         sum(CASE WHEN a.original STARTS WITH '[' AND p.label <> '-' THEN 1 ELSE 0 END) AS erschlossen,
         min(r.startDate) AS von, max(r.endDate) AS bis
  ORDER BY anzahl DESC;
  ```
- **Ergebnis (geprüft):** ohne Ort 441 · Wiener Neustadt 132 · Graz 110 · Wien 100 · Linz an der Donau 89 · Frankfurt am Main 41 · Nürnberg 27 · Köln 23 · Andernach 22 · Augsburg 14
- **Fallstrick:** Der Platzhalter-Ort „-“ darf nicht als Ort ausgegeben werden. Die Antwort nennt die Lücke (42 %) ausdrücklich.

#### Frage 3: Welche Privilegien bestätigte oder verlieh Friedrich der Stadt? *(Ergänzung)*
- **Basismodell:** nur Volltext auf `Summary` („bestätigt“, „Privileg“). Das ist unzuverlässig, weil „bestätigt“ auch bei Belehnungen u. a. steht.
- **Mit Ergänzung:** `(:Regesta)-[:HAS_ANNOTATION]->(:Annotation {type:'ri:legalAct', act:'privilege_confirmation' | 'privilege_grant'})` (Modell in 5.2)
- **Tool:** `legal_acts(act:["privilege_confirmation","privilege_grant"], heft:"RI XIII H. 4")`
- **Cypher:**
  ```cypher
  MATCH (:Department {label:'RI13'})<-[:PART_OF]-(:Volume {label:'4'})<-[:PART_OF]-(r:Regesta)
        -[:HAS_ANNOTATION]->(a:Annotation {type:'ri:legalAct'})
  WHERE a.act IN ['privilege_confirmation','privilege_grant']
  RETURN r.identifier, r.date, a.act, a.object, a.source, a.status
  ORDER BY r.startDate;
  ```
- **Erwartung:** u. a. n. 31 (15. Juli 1442: Bestätigung der beiden Frankfurter Messen und aller Privilegien der Vorgänger)

#### Frage 4: Welche Steuern zahlte Frankfurt an den König? *(Basis, genauer mit Ergänzung)*
- **Basismodell:** Volltext auf das Verb am Anfang des Regests: **68 Regesten** mit „quittiert“ in Heft 4. Frankfurt ist dort meist `empfaenger`.
- **Cypher:**
  ```cypher
  MATCH (:Department {label:'RI13'})<-[:PART_OF]-(:Volume {label:'4'})<-[:PART_OF]-(r:Regesta)<-[:PART_OF]-(s:Summary)
  WITH r, apoc.text.regreplace(s.text,'<[^>]+>','') AS t
  WHERE t =~ '^(Kg\\.|K\\.) F\\. quittiert.*'
  RETURN r.identifier, r.date, left(t, 200) AS text ORDER BY r.startDate;
  ```
- **Erwartung:** n. 21 (Stadtsteuer 1437, ausgestellt 1442), n. 101 (Stadtsteuer 1446, „[1446 etwa Juli 30]“, ohne Ort)
- **Mit Ergänzung:** `ri:legalAct {act:'receipt', object:'Stadtsteuer', year_ref:1437}`, damit Steuerjahr und Ausstellungsjahr getrennt abfragbar sind.

#### Frage 5: Was regelte der Frankfurter Landfrieden von 1442?
- **Modell:** einzelnes Regest; `Summary`, `Footnotes`; Registereinträge über `ri:appearsIn`
- **Tool:** `get_regest("[RI XIII] H. 4 n. 41")`
- **Cypher:**
  ```cypher
  MATCH (r:Regesta {identifier:'[RI XIII] H. 4 n. 41'})<-[:PART_OF]-(c:Content)
  RETURN [l IN labels(c) WHERE NOT l IN ['Content','Text']][0] AS feld,
         apoc.text.regreplace(c.text,'<[^>]+>','') AS text;
  ```
- **Antwort:** Zusammenfassung des langen Regesttextes (Schutz der Bauern während der Ernte, von Geistlichen, Pilgern, Kaufleuten, Fuhrleuten, Kirchen; Verbot unrechtmäßiger Pfändung …) mit Verweis auf n. 41
- **Zweck im Testsatz:** prüft, ob der Chatbot lange Texte korrekt zusammenfasst, ohne etwas zu erfinden.

#### Frage 6: Welche Kanzleibeamten wirkten an den Frankfurter Urkunden mit? *(Basis, genauer mit Ergänzung)*
- **Basismodell:** Personen mit `nennung`, deren Registereintrag ein Kanzleiamt nennt
- **Tool:** `regesta_by_entity` / `count_regesta(group_by:'entity')` mit Filter auf das Amt
- **Cypher:**
  ```cypher
  MATCH (:Department {label:'RI13'})<-[:PART_OF]-(:Volume {label:'4'})<-[:PART_OF]-(r:Regesta)
        <-[:REFERS_TO]-(a:Annotation {type:'ri:appearsIn'})<-[:HAS_ANNOTATION]-(e:Person)
  WHERE e.label =~ '(?i).*(kanzlei|protonotar|kammermeister|registrator|sekretär).*'
  RETURN e.label, count(DISTINCT r) AS anzahl ORDER BY anzahl DESC LIMIT 15;
  ```
- **Ergebnis (geprüft, Auszug):** Ulrich Weltzli (Kanzleischreiber, später Kanzler) 25 · Jakob Widerl (Sekretär/Registrator) 19 · Johann Ungnad (Kammermeister) 12 · Peter Gamp (Notar) 11 · Lukas Snitzer (Registrator) 11 …
- **Grenze:** Der Filter trifft auch Fürsten mit Kanzleramt im Eintrag (z. B. den Mainzer Erzbischof als Erzkanzler, 39). Ob jemand die Urkunde **bearbeitet** hat (Kanzleivermerk) oder nur im Text vorkommt, sagt `nennung` nicht.
- **Mit Ergänzung:** Rolle `relator` bzw. `registrator` aus dem Kanzleivermerk in `VersoNote` (515 Regesten), verknüpft mit den vorhandenen Register-Personen.

#### Frage 7: Welche Befehle erteilte der König dem Rat, und gegen wen richteten sie sich? *(Ergänzung)*
- **Basismodell:** Verben „befiehlt“ (118), „gebietet“ (98), „lädt“ (78), „fordert“ (42), „verbietet“ (16). Die Betroffenen stehen nur als `nennung` im Register.
- **Mit Ergänzung:** `ri:legalAct {act:'mandate' | 'summons' | 'ban_enforcement' | 'prohibition'}` + `ri:appearsIn {role:'affected'}`
- **Tool:** `legal_acts(act:[…], heft:"RI XIII H. 4")`, danach `regesta_by_entity(role:'affected')`
- **Cypher:**
  ```cypher
  MATCH (:Department {label:'RI13'})<-[:PART_OF]-(:Volume {label:'4'})<-[:PART_OF]-(r:Regesta)
        -[:HAS_ANNOTATION]->(la:Annotation {type:'ri:legalAct'})
  WHERE la.act IN ['mandate','summons','ban_enforcement','prohibition']
  OPTIONAL MATCH (r)<-[:REFERS_TO]-(:Annotation {type:'ri:appearsIn', role:'affected'})<-[:HAS_ANNOTATION]-(e:Entity)
  RETURN la.act, r.identifier, r.date, la.object, collect(e.label) AS betroffen, la.source
  ORDER BY r.startDate;
  ```
- **Erwartung:** n. 35 (Vollzug der von Kg. Sigmund verhängten Acht gegen Lüttich, Huy, St. Truiden, Maastricht, Tongern u. a.), n. 107 (Konrad von Steeg nicht von der Münze verdrängen lassen; 14. Dezember 1446, Graz)

#### Frage 8: Wie entwickelte sich der Kontakt über die Jahrzehnte?
- **Modell:** Regesta (`startDate`/`endDate`), Volume
- **Tool:** `count_regesta(group_by:"decade", heft:"RI XIII H. 4")`
- **Cypher:**
  ```cypher
  MATCH (:Department {label:'RI13'})<-[:PART_OF]-(:Volume {label:'4'})<-[:PART_OF]-(r:Regesta)
  WITH r, toInteger(left(r.startDate,3))*10 AS a, toInteger(left(r.endDate,3))*10 AS b
  RETURN CASE WHEN a = b THEN toString(a) + 'er' ELSE 'unsicher ' + a + '–' + b END AS jahrzehnt,
         count(*) AS anzahl
  ORDER BY jahrzehnt;
  ```
- **Ergebnis (geprüft):** 1440er 171 · 1450er 124 · 1460er 198 · **1470er 328** · 1480er 189 · 1490er 44
- **Fallstrick:** „[Vor 1442 April 7]“ ist als `startDate = endDate = 1442-04-07` gespeichert, das „vor“ steht nur noch in `date` (siehe Aufgabe 2).

#### Frage 9: Welche anderen Städte und Herren wurden zusammen mit Frankfurt angeschrieben?
- **Modell:** Entity (Place/Person), `ri:appearsIn {role:'empfaenger'}`, Volume
- **Tool:** `find_entity("Frankfurt", type:"Place")` → Kandidat „Frankfurt (Hessen), Stadt“ → `co_recipients(entity_uuid, heft:"RI XIII H. 4")`
- **Cypher:**
  ```cypher
  MATCH (f:Place {label:'Frankfurt (Hessen), Stadt'})-[:HAS_ANNOTATION]->(:Annotation {type:'ri:appearsIn', role:'empfaenger'})
        -[:REFERS_TO]->(r:Regesta)-[:PART_OF]->(:Volume {label:'4'})-[:PART_OF]->(:Department {label:'RI13'})
  MATCH (r)<-[:REFERS_TO]-(:Annotation {type:'ri:appearsIn', role:'empfaenger'})<-[:HAS_ANNOTATION]-(o:Entity)
  WHERE o <> f
  RETURN o.label AS mitEmpfaenger, count(DISTINCT r) AS gemeinsam
  ORDER BY gemeinsam DESC LIMIT 15;
  ```
- **Ergebnis (geprüft):** Herren von Eppstein 8 · Grafen von Isenburg 8 · Erzbischof von Mainz 8 · Worms 7 · Wetzlar 6 · Friedberg 6 · Speyer 6 · Gelnhausen 6 · Köln 5 · Stadt Mainz 5
- **Fallstrick:** „Frankfurt“ gibt es als Registereintrag mit vielen Untereinträgen („// AO“, „// Kirchen und Klöster // Bartholomäusstift“ …) und zusätzlich als normierten Ort „Frankfurt am Main“. `find_entity` muss die Kandidaten zur Auswahl anbieten.

#### Frage 10: Welche einzelnen Personen begünstigte der König in Frankfurt? *(Ergänzung)*
- **Basismodell:** Begünstigte stehen nur als `nennung` im Register und sind nicht von anderen Erwähnten zu unterscheiden (siehe 1.4).
- **Mit Ergänzung:** `ri:appearsIn {role:'beneficiary', source:'llm'}` auf die vorhandenen Register-Personen, dazu `ri:legalAct`
- **Tool:** `regesta_by_entity(role:"beneficiary", heft:"RI XIII H. 4")`
- **Cypher:**
  ```cypher
  MATCH (:Department {label:'RI13'})<-[:PART_OF]-(:Volume {label:'4'})<-[:PART_OF]-(r:Regesta)
        <-[:REFERS_TO]-(b:Annotation {type:'ri:appearsIn', role:'beneficiary'})<-[:HAS_ANNOTATION]-(e:Person)
  OPTIONAL MATCH (r)-[:HAS_ANNOTATION]->(la:Annotation {type:'ri:legalAct'})
  RETURN e.label, la.act, la.object, r.identifier, r.date, b.source, b.status
  ORDER BY r.startDate;
  ```
- **Erwartung:** Johannes Snyder (n. 110, Bitte um ein Gotteslehen) · Katharina von Ingelheim (n. 32, Schutz) · Konrad von Steeg (n. 107, Frankfurter Münze)

---

## Schritt 5: Die regionale Ergänzung

### 5.1 Was wir per LLM extrahieren

Wir ergänzen das Register dort, wo es nur `nennung` oder `empfaenger` kennt, und fügen Rechtsakte hinzu. **Neue Entities legen wir nur an, wenn eine Person nicht im Register steht.** In der Regel verknüpfen wir die vorhandenen Register-Entities des jeweiligen Regests.

**A) Zusätzliche Rollen** für `ri:appearsIn`

| `role` (neu) | Bedeutung | verfeinert | Beispiel Heft 4 |
|---|---|---|---|
| `beneficiary` | Begünstigter (≠ Empfänger) | `nennung` | Johannes Snyder (n. 110), Katharina von Ingelheim (n. 32) |
| `commissioned` | mit dem Vollzug beauftragt | `empfaenger` | Schultheiß Hermann von Hoch-Weisel (n. 32) |
| `affected` | gegen wen sich die Maßnahme richtet | `nennung` | Lüttich, Huy, St. Truiden, Maastricht, Tongern (n. 35) |
| `petitioner` | auf wessen Bitte gehandelt wird | `nennung` | Bürgermeister und Rat (Bestätigungen „auf Ersuchen“) |
| `payer` | Zahlender | `empfaenger` | Stadt Frankfurt (n. 21) |
| `relator` / `registrator` | Kanzleivermerk: Relator bzw. Registrator | `nennung` | Heinrich Leubing / Jakob Widerl (n. 21) |

Die Register-Rolle bleibt erhalten. Die neue Annotation kommt hinzu, sodass für dieselbe Person z. B. `nennung` (Register) und `beneficiary` (LLM) nebeneinander stehen.

**B) Rechtsakte** (kontrolliertes Vokabular, aus den häufigsten Verben am Anfang der Regesten in Heft 4 abgeleitet)

| `act` | deutsch | Signal im Regest (Anzahl) | Zusatzfelder |
|---|---|---|---|
| `communication` | Mitteilung | „teilt … mit“ (178) | — |
| `mandate` | Befehl/Gebot | „befiehlt“ (118), „gebietet“ (98), „fordert“ (42) | `object` |
| `summons` | Ladung | „lädt“ (78) | `object` |
| `receipt` | Quittung | „quittiert“ (68) | `object`, `year_ref`, `amount` |
| `privilege_confirmation` | Privilegienbestätigung | „bestätigt“ (26) | `object` |
| `privilege_grant` | Privilegienverleihung | „erteilt … Privileg“ | `object` |
| `credential` | Beglaubigung eines Gesandten | „beglaubigt“ (22) | — |
| `request` | Bitte / Erste Bitte | „ersucht“ (21), „bittet“ (14) | `object` |
| `commission` | Auftrag | „beauftragt“ (18) | `object` |
| `prohibition` | Verbot | „verbietet“ (16) | `object` |
| `appointment` | Ernennung | „ernennt“ (13) | `object` |
| `ban_enforcement` | Acht/Aberacht | „Acht“ | — |
| `public_peace` | Landfrieden | „Landfrieden“ | — |
| `other` | sonstiges | — | `label` |

### 5.2 So sieht es im Modell aus

Wir verwenden **dieselben Muster wie RI-alles**, also Annotation-Knoten mit `type`. Damit funktionieren alle Abfragen auf Register und Ergänzung gleich.

```cypher
// Rolle: Johannes Snyder (vorhandener Registereintrag) ist Begünstigter in n. 110
MATCH (r:Regesta {identifier:'[RI XIII] H. 4 n. 110'})
MATCH (e:Person {label:'Snyder, Johannes, Sohn des Wigands'})
CREATE (e)-[:HAS_ANNOTATION]->(:Annotation {
  uuid: randomUUID(), type: 'ri:appearsIn', label: 'ri:appearsIn', role: 'beneficiary',
  evidence: 'dem Johannes, Sohn des Frankfurter Bürgers Wigand Snyder',
  source: 'llm', model: '<Modell + Version>', prompt_version: 'v1',
  confidence: 0.88, status: 'unreviewed', created: datetime()
})-[:REFERS_TO]->(r);

// Rechtsakt: Bitte um ein Gotteslehen (Richtung wie bei ri:placeOfIssue)
MATCH (r:Regesta {identifier:'[RI XIII] H. 4 n. 110'})<-[:PART_OF]-(s:Summary)
CREATE (r)-[:HAS_ANNOTATION]->(la:Annotation {
  uuid: randomUUID(), type: 'ri:legalAct', label: 'ri:legalAct', act: 'request',
  object: 'erstes freiwerdendes Gotteslehen in Frankfurt',
  evidence: 'das erste freiwerdende Gotteslehen in ihrer Stadt zu verleihen',
  source: 'llm', model: '<Modell + Version>', prompt_version: 'v1',
  confidence: 0.91, status: 'unreviewed', created: datetime()
})-[:REFERS_TO]->(s);   // verweist auf das Textfeld mit der Belegstelle
```

**Wo die Daten liegen:** RI-alles auf jlu-buster ist schreibgeschützt. Wir schreiben die Ergänzung in **unsere eigene Neo4j-Instanz** (für die Entwicklung die lokale Vollkopie im Docker-Container). Verknüpft wird über `uuid` von Regest und Entity, die in beiden Datenbanken gleich sind.

### 5.3 Herkunftskennzeichnung

| `source` | Bedeutung | erkennbar an / Pflichtfelder |
|---|---|---|
| `register` | aus dem RI-Register (bestehende Annotationen in RI-alles) | alle `ri:appearsIn` ohne `source`-Eigenschaft (sie haben `importKey`) |
| `llm` | vom Sprachmodell aus `Summary` oder `VersoNote` erschlossen | `model`, `prompt_version`, `created`, `confidence`, `evidence` |
| `manual` | von uns von Hand angelegt oder korrigiert | `reviewed_by`, `reviewed_at`, ggf. `note` |

So filtert ein Tool nach Herkunft:
```cypher
WHERE coalesce(a.source, 'register') IN $erlaubteQuellen
```

Zusätzlich `status` bei `llm`: `unreviewed` → `verified` (bestätigt, `reviewed_by` gesetzt) oder `rejected` (bleibt für die Evaluation gespeichert, wird nicht ausgegeben). Wird eine LLM-Annotation inhaltlich korrigiert, entsteht eine neue mit `source: 'manual'`, und die alte wird `rejected`.

Für Annotationen aus den formelhaften Kanzleivermerken, die per regulärem Ausdruck statt LLM gewonnen werden, setzen wir zusätzlich `method: 'rule'`.

**Im Chatbot sichtbar:** Jede Aussage nennt ihre Grundlage, z. B. „laut Register: genannt“ bzw. „vom Sprachmodell erschlossen: Begünstigter (ungeprüft)“. Über den Tool-Parameter `source` kann die Lehrkraft LLM-Annotationen ausblenden.

### 5.4 Extraktions-Pipeline (offline, einmalig für Heft 4)

1. **Daten holen:** alle 1054 Regesten mit `Summary`, `VersoNote` und **ihren Register-Entities samt Rollen** (HTML entfernt, nach `laufendenummer` sortiert).
2. **Kontext:** Bei den 22 „desgleichen“-Regesten das vorhergehende Regest mitgeben.
3. **Kanzleivermerke per Regel:** Formeln wie „A. rel.“, „Rta“, Ämter wie „prothon.“, „vicecanc.“ per regulärem Ausdruck; Namen gegen die bereits als `nennung` verknüpften Personen abgleichen.
4. **LLM-Aufruf pro Regest:** Prompt mit Regesttext, Liste der Register-Entities (Label + uuid) und Vokabular aus 5.1. Das LLM ordnet **nur vorhandenen Entities** neue Rollen zu und gibt Rechtsakte als JSON zurück, jeweils mit wörtlicher Belegstelle.
5. **Automatische Prüfung:** `evidence` muss wörtlich im bereinigten Text vorkommen; `role`/`act` nur aus dem Vokabular; `entity_uuid` muss zu diesem Regest gehören.
6. **Stichprobe von Hand:** ca. 50 Regesten als Goldstandard annotieren, Präzision und Trefferquote pro Rolle und Rechtsakt messen. Die Zahlen gehören in die Abgabe.
7. **Import** mit `source: 'llm'`, `status: 'unreviewed'`.

Beispiel einer LLM-Ausgabe für n. 110:

```json
{
  "regest": "[RI XIII] H. 4 n. 110",
  "legal_acts": [
    {"act": "request", "object": "erstes freiwerdendes Gotteslehen in Frankfurt",
     "evidence": "das erste freiwerdende Gotteslehen in ihrer Stadt zu verleihen", "confidence": 0.91}
  ],
  "roles": [
    {"entity_uuid": "<uuid von 'Snyder, Johannes, Sohn des Wigands'>", "role": "beneficiary",
     "evidence": "dem Johannes, Sohn des Frankfurter Bürgers Wigand Snyder", "confidence": 0.88}
  ]
}
```

---

## Befunde für Aufgabe 2 (Fallstricke)

Bei der Erkundung schon gefunden; ausgearbeitet wird es in Aufgabe 2:

- **Chmel-Dubletten:** erkennbar über `<link riregesten:…>` in `archivalHistory`, das auf die `exchangeIdentifier` des Chmel-Regests zeigt. 124 Regesten aus Heft 4 ↔ 136 Chmel-Regesten (z. B. n. 21 = Chmel n. 558).
  ```cypher
  MATCH (h:Regesta {bandpk:'13004000'}) WHERE h.archivalHistory CONTAINS 'riregesten:'
  WITH h, [x IN split(h.archivalHistory,'riregesten:')[1..] | split(split(x,'>')[0],' ')[0]] AS ziele
  UNWIND ziele AS z
  MATCH (c:Regesta {exchangeIdentifier: z}) WHERE c.bandpk IN ['13000000','13000001']
  RETURN h.identifier, c.identifier;
  ```
- **Datumsangaben:** 483 erschlossen, 41 echte Spannen. „[Vor 1440 November 7]“ ist als ein einzelner Tag gespeichert; unpräzise Angaben wie „[Vor 1442 Herbst]“ werden zum ganzen Jahr.
- **Ausstellungsorte:** 441 Regesten zeigen per `ri:placeOfIssue` auf den **Platzhalter-Ort „-“**. Erschlossene Orte sind nur an `original` („[Frankfurt]“) erkennbar. Register-`austOrt` und `ri:placeOfIssue` sind zwei verschiedene Ortssysteme; bei n. 32 nennt das Register zwei `austOrt` (Köln und Frankfurt).
- **„desgleichen“-Regesten:** nur zusammen mit dem Vorgänger verständlich.
