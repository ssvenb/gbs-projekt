# Plan für unseren Chatbot – Gruppe 2: Frankfurt am Main (RI XIII H. 4)

**Heft:** RI XIII H. 4 – *Regesten Kaiser Friedrichs III. (1440–1493), Heft 4: Die Urkunden und Briefe aus dem Stadtarchiv Frankfurt am Main*, bearb. von Paul-Joachim Heinig (1986) · 1.054 Regesten, 1440-02-22 bis 1493-06-03
**Datenmodell:** gemeinsames Modell von RI-alles (`riallesatag`), schreibgeschützt auf `bolt://jlu-buster.mni.thm.de:9011`
**Pitch:** 27.11.2026, 5 Minuten, 3 Folien (siehe Abschnitt 6)

> **Stand der Zahlen:** Alle Zahlen zum Basismodell stammen aus Abfragen gegen eine vollständig importierte lokale Kopie von RI-alles (Neo4j 5.26 im Docker-Container, siehe `docker-compose.yml`, 25.11.2026). Zahlen zur Ergänzung (Fragen 3, 4, 6, 7, 10) stammen aus der LLM-Extraktion über alle 1.054 Regesten (`ri13_h4_llm_annotationen/`, 26.11.2026) und sind **ungeprüft**, bis der Goldstandard ausgewertet ist. Ausführliche Herleitungen: `aufgaben/Aufgabe1_…`, `aufgaben/aufgabe2_fallstricke.md`, `aufgaben/Aufgabe3_Systemarchitektur/`.
>
> Die PDF-Fassung dieses Plans liegt in `Abgabe_Chatbot_Plan.pdf`, die drei Pitch-Folien in `Pitch_Folien.pdf` (LaTeX-Quellen in `latex/`).

---

## 1. Wie unser Chatbot das gemeinsame Modell nutzt

RI-alles ist ein Text-als-Graph-Modell mit **Standoff-Annotationen**. Es gibt nur drei Kantentypen (`PART_OF`, `HAS_ANNOTATION`, `REFERS_TO`); was eine Verbindung bedeutet, steht in `Annotation.type`.

```
(:Regesta) -[:PART_OF]-> (:Volume {label:'4'}) -[:PART_OF]-> (:Department {label:'RI13'})
(:Summary|VersoNote|ArchivalHistory|…) -[:PART_OF]-> (:Regesta)                  // Textfelder
(:Entity) -[:HAS_ANNOTATION]-> (:Annotation {type:'ri:appearsIn', role}) -[:REFERS_TO]-> (:Regesta)
(:Regesta) -[:HAS_ANNOTATION]-> (:Annotation {type:'ri:placeOfIssue', original}) -[:REFERS_TO]-> (:Place)
(:Entity) -[:HAS_ANNOTATION]-> (:Annotation {type:'ri:isSubOf'}) -[:REFERS_TO]-> (:Entity)   // Registerhierarchie
```

| Baustein | Wofür der Chatbot ihn braucht |
|---|---|
| `Regesta` (`identifier`, `date`, `startDate`/`endDate`, `origPlaceOfIssue`, `archivalHistory`, `urn`) | Treffer, Datum, Beleg (jede Antwort nennt `identifier` und Link) |
| `Volume` + `Department` | Heftfilter: `Volume.label = '4'` ist nur zusammen mit `Department 'RI13'` eindeutig (gleichwertig `bandpk = '13004000'`) |
| `Summary`, `VersoNote`, `ArchivalHistory` | Regestentext, Kanzleivermerk, Überlieferung und Chmel-Verweis |
| `Entity` (`Person`, `Place`, `Thing`, `Event`) | Registereinträge; Untereinträge tragen den vollen Pfad („Frankfurt (Hessen), Stadt // AO“) |
| `ri:appearsIn` mit `role` | in Heft 4 nur `nennung` (3.359), `empfaenger` (1.606), `austOrt` (490), vereinzelt `p_tot`, `p_komment` |
| `ri:placeOfIssue` → `Place` | normierter Ausstellungsort; `original` zeigt, ob erschlossen („[Frankfurt]“) |

**Grundsatz:** Das LLM schreibt kein freies Cypher. Es ruft **feste MCP-Tools** auf, deren Cypher wir geschrieben und getestet haben. Die Tools setzen immer den Heftfilter, schließen den Schein-Ort „-“ aus, zählen `DISTINCT` Regesten, rechnen mit Datumsüberlappung und geben zu jedem Treffer `identifier` und `urn` als Beleg zurück.

### MCP-Tools

| Tool | Parameter | Zweck |
|---|---|---|
| `search_regesta` | `text?`, `heft?`, `date_from?`, `date_to?`, `place?` | Regesten filtern (Datum, Ausstellungsort, Volltext) |
| `get_regest` | `identifier` | ein Regest mit allen Textfeldern, Ort und Registereinträgen samt Rollen |
| `find_entity` | `name`, `type?` | Registereintrag suchen, Kandidaten zur Auswahl anbieten |
| `regesta_by_entity` | `entity_uuid`, `role?`, `heft?`, `source?`, `include_sub?` | Regesten zu einer Entity, optional nach Rolle und mit Unterlemmata |
| `itinerary` | `heft?`, `date_from?`, `date_to?` | Ausstellungsorte mit Anzahl, Zeitraum und Lücke |
| `count_regesta` | `group_by` (`year`/`decade`/`place`/`role`/`entity`), Filter | Zählungen und Verläufe |
| `co_recipients` | `entity_uuid`, `heft?` | weitere Empfänger derselben Regesten |
| `legal_acts` *(Ergänzung)* | `act?`, `heft?`, `date_from?`, `date_to?`, `source?` | Regesten mit einem bestimmten Rechtsakt |

---

## 2. Zehn Fragen einer Lehrkraft (Grundstock für den Testsatz)

| # | Frage | Knoten, Annotationen, Rollen | Tool(s) | Basis/Ergänzung |
|---|---|---|---|---|
| 1 | Welche Urkunden stellte Friedrich III. 1442 in Frankfurt aus? | Regesta, Volume, `Summary`, `ri:placeOfIssue` → Place „Frankfurt am Main“ | `search_regesta` | Basis |
| 2 | Von wo aus schrieb Friedrich an Frankfurt? | Regesta, Volume, `ri:placeOfIssue` → Place | `itinerary` | Basis |
| 3 | Welche Privilegien bestätigte oder verlieh Friedrich der Stadt? | Regesta, **`ri:legalAct`** (`privilege_*`), `ri:appearsIn` (`beneficiary`/`empfaenger`) | `legal_acts` | **Ergänzung** |
| 4 | Welche Steuern zahlte Frankfurt an den König? | Regesta, `Summary`; **`ri:legalAct {act:'receipt', year_ref}`** | `search_regesta`, `legal_acts` | Basis, genauer mit Ergänzung |
| 5 | Was regelte der Frankfurter Landfrieden von 1442? | Regesta n. 41, `Summary`, `Footnotes`, `ri:appearsIn` | `get_regest` | Basis |
| 6 | Welche Kanzleibeamten wirkten an den Frankfurter Urkunden mit? | Person, `ri:appearsIn {role:'nennung'}`, `VersoNote`; **Rollen `relator`/`registrator`/`chancery`** | `regesta_by_entity`, `count_regesta(entity)` | Basis, genauer mit Ergänzung |
| 7 | Welche Befehle erteilte der König dem Rat, und gegen wen richteten sie sich? | Regesta, **`ri:legalAct`** (`mandate`, `summons`, …), **Rolle `affected`** | `legal_acts`, `regesta_by_entity` | **Ergänzung** |
| 8 | Wie entwickelte sich der Kontakt über die Jahrzehnte? | Regesta (`startDate`/`endDate`, `date`), Volume | `count_regesta(decade)` | Basis |
| 9 | Welche anderen Städte und Herren wurden zusammen mit Frankfurt angeschrieben? | Place/Person, `ri:appearsIn {role:'empfaenger'}`, `ri:isSubOf` | `find_entity`, `co_recipients` | Basis |
| 10 | Welche einzelnen Personen begünstigte der König in Frankfurt? | Person, **Rolle `beneficiary`**, `ri:legalAct` | `regesta_by_entity(role:'beneficiary')` | **Ergänzung** |

**Bilanz:** 5 Fragen gehen mit dem Basismodell (1, 2, 5, 8, 9), 2 werden mit der Ergänzung genauer (4, 6), 3 brauchen sie (3, 7, 10).

### Die Fragen im Einzelnen

Alle Abfragen beginnen mit dem Heftfilter
`MATCH (:Department {label:'RI13'})<-[:PART_OF]-(:Volume {label:'4'})<-[:PART_OF]-(r:Regesta)` (unten als `H4(r)` abgekürzt).

**Frage 1 – Urkunden 1442 in Frankfurt**
- Modell: Regesta → Volume/Department; `ri:placeOfIssue` → Place „Frankfurt am Main“; Datum über `startDate`/`endDate`
- Tool: `search_regesta(place:"Frankfurt am Main", heft:"RI XIII H. 4", date_from:"1442-01-01", date_to:"1442-12-31")`
- Abfrage:
  ```cypher
  H4(r) MATCH (r)-[:HAS_ANNOTATION]->(a:Annotation {type:'ri:placeOfIssue'})-[:REFERS_TO]->(:Place {label:'Frankfurt am Main'})
  WHERE r.startDate <= '1442-12-31' AND r.endDate >= '1442-01-01'
  MATCH (r)<-[:PART_OF]-(s:Summary)
  RETURN DISTINCT r.identifier, r.date, a.original, s.text ORDER BY r.startDate
  ```
- Erwartung (geprüft): **19 Regesten** (30. Mai – 14. August 1442), 2 davon mit erschlossenem Ort „[Frankfurt]“; u. a. n. 21 (Quittung Stadtsteuer 1437), n. 31 (Bestätigung der Messen und Privilegien), n. 41 (Landfrieden).

**Frage 2 – Von wo aus schrieb Friedrich?**
- Modell: `ri:placeOfIssue` → Place, `a.original` für „erschlossen“
- Tool: `itinerary(heft:"RI XIII H. 4")`
- Abfrage: `H4(r) MATCH (r)-[:HAS_ANNOTATION]->(a {type:'ri:placeOfIssue'})-[:REFERS_TO]->(p:Place) WHERE p.label <> '-' RETURN p.label, count(DISTINCT r) …`
- Erwartung (geprüft): Wiener Neustadt 132 · Graz 110 · Wien 100 · Linz 46 · Frankfurt am Main 41 · Nürnberg 27 · Köln 23 · Andernach 22 · Augsburg 14; **ohne Ort: 442 von 1.054 (42 %)** – wird immer mit genannt.

**Frage 3 – Privilegien für die Stadt** *(Ergänzung)*
- Basismodell: nur Volltext („bestätigt“, „Privileg“), unzuverlässig (auch Belehnungen).
- Modell mit Ergänzung: `(r)-[:HAS_ANNOTATION]->(:Annotation {type:'ri:legalAct', act:'privilege_confirmation'|'privilege_grant'})`, dazu Frankfurt (`Place` „Frankfurt (Hessen), Stadt“) mit `role` `beneficiary` oder `empfaenger`
- Tool: `legal_acts(act:["privilege_confirmation","privilege_grant"], heft:"RI XIII H. 4")`
- Erwartung (LLM, ungeprüft): **25 Akte in 24 Regesten** (15 Verleihungen, 10 Bestätigungen), z. B. n. 31 (Messen 1442), n. 214/233 (Messeprivilegien), n. 861 (Sodener Salzquelle 1483), n. 862 (Mainfischerei).

**Frage 4 – Steuern an den König** *(Basis, genauer mit Ergänzung)*
- Basismodell: Regestenanfang „K./Kg. F. quittiert“ im `Summary` (68 Regesten), Frankfurt als `empfaenger`
- Mit Ergänzung: `ri:legalAct {act:'receipt', object:'Stadtsteuer', year_ref:1437, amount}` – Steuerjahr und Ausstellungsjahr getrennt abfragbar
- Tool: `legal_acts(act:"receipt")`, Rückfall `search_regesta(text:"quittiert")`
- Erwartung (LLM, ungeprüft): 66 Quittungen für Frankfurt, 59 über die Stadtsteuer (meist 928 fl. rh.); bei 24 weicht das Steuerjahr vom Ausstellungsjahr ab (n. 21–24: Steuern 1437–1441, quittiert 1442).

**Frage 5 – Landfrieden 1442**
- Modell: ein Regest; alle `Content`-Knoten (`Summary`, `Footnotes`, …), Registereinträge über `ri:appearsIn`
- Tool: `get_regest("[RI XIII] H. 4 n. 41")`
- Zweck im Testsatz: prüft, ob ein sehr langer Text korrekt und ohne Erfindungen zusammengefasst wird (Schutz von Bauern, Geistlichen, Pilgern, Kaufleuten; Verbot unrechtmäßiger Pfändung …).

**Frage 6 – Kanzleibeamte** *(Basis, genauer mit Ergänzung)*
- Basismodell: Personen mit `nennung`, deren Registerlabel ein Kanzleiamt nennt (`kanzlei|protonotar|kammermeister|registrator|sekretär`)
- Tool: `count_regesta(group_by:"entity", role:"nennung", heft:…)` mit Amtsfilter
- Erwartung (geprüft): Ulrich Weltzli 25 · Jakob Widerl 19 · Johann Ungnad 12 · Peter Gamp 11 · Lukas Snitzer 11 – aber auch Fürsten mit Ehrenamt (Erzbischof von Mainz als Erzkanzler).
- Mit Ergänzung: Rollen `relator`, `registrator`, `chancery` aus dem Kanzleivermerk (`VersoNote`, 515 Regesten): 125 Rollen (LLM, ungeprüft). Ab ca. 1470 steht nur noch „A.m.p.d.i.“ ohne Namen – das muss der Chatbot sagen.

**Frage 7 – Befehle an den Rat und Betroffene** *(Ergänzung)*
- Modell: `ri:legalAct {act IN ['mandate','summons','ban_enforcement','prohibition']}` + `ri:appearsIn {role:'affected'}`
- Tool: `legal_acts(act:[…])`, dann `regesta_by_entity(role:"affected")`
- Erwartung (LLM, ungeprüft): 169 Akte in 167 Regesten an Frankfurt (108 Befehle, 42 Ladungen, 10 Verbote, 9 Achtsachen); häufigste Betroffene: Türken (13), Hz. Karl von Burgund (8), Hz. Ludwig von Bayern-Landshut (7). Beispiel n. 35 (Acht gegen Lüttich, Huy, St. Truiden …).
- Fallstrick: 4-mal ist Frankfurt selbst `affected` (n. 842, Ladung durch den Fiskal) – „Befehl an den Rat“ und „Maßnahme gegen den Rat“ trennen.

**Frage 8 – Kontakt über die Jahrzehnte**
- Modell: Regesta `startDate`/`endDate`, `date`
- Tool: `count_regesta(group_by:"decade", heft:"RI XIII H. 4")`
- Erwartung (geprüft): 1440er 171 · 1450er 124 · 1460er 198 · **1470er 328** · 1480er 189 · 1490er 44; dazu Hinweis auf 483 unsichere Daten.

**Frage 9 – Mitempfänger Frankfurts**
- Modell: Place „Frankfurt (Hessen), Stadt“, `ri:appearsIn {role:'empfaenger'}`, andere Entities mit derselben Rolle im selben Regest
- Tool: `find_entity("Frankfurt", type:"Place")` → Auswahl → `co_recipients(entity_uuid, heft:…)`
- Erwartung (geprüft): Herren von Eppstein 8 · Grafen von Isenburg 8 · Erzbischof von Mainz 8 · Worms 7 · Wetzlar, Friedberg, Speyer, Gelnhausen je 6

**Frage 10 – Begünstigte Personen** *(Ergänzung)*
- Basismodell: Begünstigte stehen nur als `nennung` im Register (n. 110: Johannes Snyder = `nennung`).
- Modell mit Ergänzung: `(:Person)-[:HAS_ANNOTATION]->(:Annotation {type:'ri:appearsIn', role:'beneficiary', source:'llm'})-[:REFERS_TO]->(r)` + `ri:legalAct`
- Tool: `regesta_by_entity(role:"beneficiary", heft:…)`
- Erwartung (LLM, ungeprüft): 55 Personen, 70-mal begünstigt; z. B. Johannes Snyder (n. 110, Gotteslehen), Katharina von Ingelheim (n. 32, Schutz), Konrad von Steeg (n. 107, Münze). Fürsten und Herrscher werden getrennt ausgegeben.

---

## 3. Fallstricke des Modells und unser Umgang damit

| Fallstrick | Befund in Heft 4 | Unsere Regel (fest in den Tools) | Was der Chatbot sagt |
|---|---|---|---|
| **Doppelte Regesten (Chmel und Hefte)** | 100 Regesten verweisen im `Reg.:`-Teil von `archivalHistory` auf ein Chmel-Regest (94 verschiedene; Chmel: `bandpk '13000000'`). Gleiches Datum + Ort taugt nicht als Erkennung; 18 Paare haben sogar verschiedene Daten (n. 22 „Mai 31“ ↔ Chmel n. 558 „Mai 30“). Mehrere Heft-Regesten zeigen auf dasselbe Chmel-Regest. | Heft vor Chmel: gezählt wird das Heft-4-Regest. Dublette nur bei Link im `Reg.:`-Teil, sonst Querverweis. Fragen zu Frankfurt filtern auf Heft 4. | „auch bei Chmel n. 558 (dort 30. Mai)“ als Hinweis, nicht als zweiter Treffer |
| **Datumsangaben als Spanne** | 41 echte Spannen (`startDate ≠ endDate`), weitere 442 erschlossene Daten mit `startDate = endDate`: „[Vor 1441 Mai 26]“ steht als exakt 1441-05-26 im Modell. Zusammen 483 von 1.054 (46 %) unsicher. | Überlappung: `r.startDate <= $ende AND r.endDate >= $anfang`. Unsicher, wenn Spanne oder `date` enthält „[“. Ausgabe immer mit Originalangabe `date`. | „19 Regesten, davon 3 mit erschlossenem Datum“; bei „vor …“: gespeichertes Datum ist der späteste mögliche Tag |
| **Fehlende Ausstellungsorte** | 442 Regesten (42 %) ohne Ort – aber nicht als NULL: 441 hängen per `ri:placeOfIssue` an einem **Schein-Ort „-“** (Original „[-]“, „o. O.“). 427 davon sind verlorene Stücke. 52 Orte sind erschlossen („[Frankfurt]“). | `p.label <> '-'` in jeder Ortsabfrage; Lücke getrennt zählen. Kein Erschließen per LLM (würde raten). Erschlossene Orte zählen mit, markiert. | „bei 442 von 1.054 Regesten ist kein Ort überliefert“; „[Frankfurt] = erschlossen“ |
| Doppelte Annotationen | 44 Regesten mit doppeltem `ri:placeOfIssue` (Linz: 89 Annotationen, 46 Regesten); 44 Regesten mit doppelter Registerverknüpfung | immer `count(DISTINCT r)` | – |
| „Frankfurt“ ist mehrdeutig | Ausstellungsort „Frankfurt am Main“ (41) ≠ Registerlemma „Frankfurt (Hessen), Stadt“ (389, mit Unterlemmata 460) ≠ Textvorkommen (638) ≠ ganzes Heft (1.054) | `find_entity` bietet Kandidaten an; Tools geben an, welche Lesart gezählt wurde; Teilbaum über `ri:isSubOf` | „Gezählt: Frankfurt als Empfänger laut Register“ |
| Aussteller nicht im Text | Texte beginnen mit „K. F.“ / „Kg. F.“, „Friedrich“ steht nur in 42 Texten | Aussteller über Heftfilter, nie Volltext; Abkürzungen im Systemprompt erklärt | – |
| Kanzleivermerke ohne Namen | ab ca. 1470 meist nur „A.m.p.d.i.“ | keine Erfindung von Bearbeitern | „Für diese Urkunde nennt der Vermerk keinen Bearbeiter.“ |

---

## 4. Unsere Ergänzung für die Region: Rollen und Rechtsakte per LLM

Das Register unterscheidet nur grob zwischen `empfaenger` und `nennung`, und Rechtsakte gibt es gar nicht. Beispiel n. 32: Befehl an den Schultheißen Hermann von Hoch-Weisel, Katharina von Ingelheim zu schützen – im Register ist Hoch-Weisel `empfaenger` und Katharina nur `nennung`; eigentlich ist sie **Begünstigte** und er **Beauftragter**.

### 4.1 Was wir aus den Regestentexten gewinnen

**A) Zusätzliche Rollen** für `ri:appearsIn` (nur auf **vorhandene** Register-Entities desselben Regests)

| `role` | Bedeutung | verfeinert | Beispiel | Anzahl (LLM) |
|---|---|---|---|---|
| `beneficiary` | Begünstigter | `nennung` | Johannes Snyder (n. 110) | 300 |
| `commissioned` | mit dem Vollzug beauftragt | `empfaenger` | Hermann von Hoch-Weisel (n. 32) | 241 |
| `affected` | gegen wen sich die Maßnahme richtet | `nennung` | Lüttich, Huy, Maastricht (n. 35) | 515 |
| `petitioner` | auf wessen Bitte gehandelt wird | `nennung` | Bürgermeister und Rat | 123 |
| `payer` | Zahlender | `empfaenger` | Stadt Frankfurt (n. 21) | 152 |
| `relator` / `registrator` / `chancery` | Kanzleivermerk (`VersoNote`) | `nennung` | Heinrich Leubing / Jakob Widerl (n. 21) | 27 / 5 / 93 |

**B) Rechtsakte** (kontrolliertes Vokabular aus den häufigsten Verben am Regestenanfang)

`mandate` (Befehl, 290) · `summons` (Ladung, 192) · `communication` (89) · `commission` (Auftrag, 89) · `request` (Bitte/Erste Bitte, 74) · `receipt` (Quittung, 72) · `prohibition` (62) · `privilege_grant` (36) · `credential` (Beglaubigung, 31) · `ban_enforcement` (Acht, 29) · `privilege_confirmation` (24) · `enfeoffment` (Belehnung, 17) · `appointment` (13) · `consent` (7) · `public_peace` (Landfrieden, 5) · `ban_release` (2) · `other` (69). Zusatzfelder je nach Akt: `object`, `year_ref`, `amount`.

Ergebnis über alle 1.054 Regesten: **1.101 Rechtsakte und 1.456 Rollen**; die automatische Prüfung meldet 0 Fehler.

### 4.2 So sieht es im Modell aus

Wir verwenden **dieselben Muster wie RI-alles** (Annotation-Knoten mit `type`), damit Register und Ergänzung mit denselben Abfragen und Tools gelesen werden.

```cypher
// Rolle: vorhandener Registereintrag Johannes Snyder ist Begünstigter in n. 110
(:Person {label:'Snyder, Johannes, Sohn des Wigands'})
  -[:HAS_ANNOTATION]->(:Annotation {type:'ri:appearsIn', role:'beneficiary',
       evidence:'dem Johannes, Sohn des Frankfurter Bürgers Wigand Snyder',
       source:'llm', model:'…', prompt_version:'v1', confidence:0.88,
       status:'unreviewed', created:datetime()})
  -[:REFERS_TO]->(:Regesta {identifier:'[RI XIII] H. 4 n. 110'})

// Rechtsakt: Bitte um ein Gotteslehen – verweist auf das Textfeld mit der Belegstelle
(:Regesta {identifier:'[RI XIII] H. 4 n. 110'})
  -[:HAS_ANNOTATION]->(:Annotation {type:'ri:legalAct', act:'request',
       object:'erstes freiwerdendes Gotteslehen in Frankfurt',
       evidence:'das erste freiwerdende Gotteslehen in ihrer Stadt zu verleihen',
       source:'llm', model:'…', prompt_version:'v1', confidence:0.91, status:'unreviewed'})
  -[:REFERS_TO]->(:Summary)
```

Die Register-Rolle bleibt erhalten; die neue Annotation kommt hinzu. RI-alles auf jlu-buster ist schreibgeschützt, daher liegt die Ergänzung in **unserer eigenen Neo4j-Instanz** (Kopie von RI-alles + Ergänzung), verknüpft über die `uuid` von Regest und Entity. Import: `ri13_h4_llm_annotationen/import_llm_annotationen.cypher` (idempotent über `importKey`, mit Rückgängig-Abfrage).

### 4.3 Herkunft: Register, LLM, manuell

| `source` | Bedeutung | erkennbar an / Pflichtfelder |
|---|---|---|
| `register` | aus dem RI-Register (bestehende Annotationen) | ohne `source`-Eigenschaft, mit `importKey` → Tools lesen `coalesce(a.source,'register')` |
| `llm` | vom Sprachmodell aus `Summary`/`VersoNote` | `model`, `prompt_version`, `created`, `confidence`, wörtliche `evidence` |
| `manual` | von uns angelegt oder korrigiert | `reviewed_by`, `reviewed_at`, ggf. `note` |

LLM-Annotationen haben zusätzlich `status`: `unreviewed` → `verified` (geprüft) oder `rejected` (bleibt für die Evaluation, wird nicht ausgegeben). Eine inhaltliche Korrektur erzeugt eine neue `manual`-Annotation, die alte wird `rejected`.

**Im Chatbot sichtbar:** „laut Register: genannt“ bzw. „vom Sprachmodell erschlossen: Begünstigter (ungeprüft)“. Mit dem Tool-Parameter `source` kann die Lehrkraft LLM-Annotationen ausblenden.

### 4.4 Pipeline (offline, einmalig für Heft 4)

1. Regesten mit `Summary`, `VersoNote` und ihren Register-Entities (Label + uuid) holen; bei „desgleichen“-Regesten den Vorgänger mitgeben.
2. LLM pro Regest: ordnet **nur vorhandenen Entities** neue Rollen zu und gibt Rechtsakte als JSON zurück, jeweils mit wörtlicher Belegstelle.
3. Automatische Prüfung: `evidence` steht wörtlich im Text, `role`/`act` aus dem Vokabular, `entity_uuid` gehört zu diesem Regest.
4. Goldstandard: 50 Regesten (123 Annotationen, `Pruefliste_Goldstandard_H4.xlsx`) von zwei Personen bewerten → Präzision und Trefferquote je Rolle und Rechtsakt.
5. Import mit `source:'llm'`, `status:'unreviewed'`.

---

## 5. Architektur-Skizze

```
┌──────────────────────────┐   REST/JSON   ┌────────────────────────────────────────────┐   OpenAI-kompatible API   ┌─────────────────────────┐
│ Browseroberfläche (React)│ ◄──────────►  │ Backend: Python / FastAPI                  │ ◄──────────────────────►  │ LLM (konfigurierbar)     │
│ Chat · Quellenbelege     │               │  Chatbot = Agentic Harness                 │                           │ THM-KI-Dienst / lokal    │
│ Karte · Zeitleiste       │               │   – Frage + Systemprompt an LLM            │                           │ (z. B. Qwen3 über        │
│ Personennetz · Steckbrief│               │   – Tool-Aufrufe prüfen und ausführen      │                           │  llama.cpp / vLLM)       │
└──────────────────────────┘               │   – Antwort mit Belegen + Visualisierungsdaten│                         └─────────────────────────┘
                                           │  MCP-Client                                │
                                           └───────────────┬────────────────┬───────────┘
                                                      MCP (tools)       MCP (tools)
                                           ┌───────────────▼──────────┐ ┌───▼──────────────────┐
                                           │ RI-MCP-Server (unsere)   │ │ Wikidata-MCP-Server   │
                                           │ search_regesta, get_regest│ │ Steckbriefe zu       │
                                           │ find_entity, itinerary,  │ │ Personen und Orten   │
                                           │ regesta_by_entity,       │ │ (als externe Quelle  │
                                           │ count_regesta,           │ │  gekennzeichnet)     │
                                           │ co_recipients, legal_acts│ └──────────────────────┘
                                           │ → feste, getestete Cypher│
                                           └───────────────┬──────────┘
                                                        Bolt (nur lesend)
                                           ┌───────────────▼──────────────────────────────┐
                                           │ Neo4j: RI-alles (riallesatag)                 │ ◄── Kopie von jlu-buster:9011 (schreibgeschützt)
                                           │  + unsere Ergänzung (ri:legalAct, neue Rollen)│ ◄── LLM-Extraktion (offline, Abschnitt 4.4)
                                           └──────────────────────────────────────────────┘
```

**Zusammenspiel:**
1. Die Lehrkraft stellt im Browser eine Frage; React schickt sie per REST an FastAPI.
2. Der Chatbot (Agentic Harness) gibt Frage, Systemprompt (Heft, Abkürzungen „K. F.“, Regeln zu Ort/Datum/Herkunft) und Tool-Beschreibungen an das LLM.
3. Das LLM wählt ein Tool; der MCP-Client ruft es beim RI-MCP-Server auf. Das LLM greift **nie direkt** auf die Datenbank zu.
4. Der MCP-Server führt seine feste Cypher-Abfrage auf Neo4j aus (RI-alles + Ergänzung) und liefert JSON mit `identifier`, `urn`, `source`, Lückenzahlen.
5. Das LLM formuliert die Antwort; der Harness prüft, dass jede genannte Regestennummer aus den Tool-Ergebnissen stammt, und hängt Belege und Visualisierungsdaten an.
6. Der Browser zeigt Antwort, Quellenlinks und ggf. Karte/Zeitleiste.

---

## 6. Pitch am 27.11. (3 Folien, 5 Minuten)

**Folie 1 – Frankfurt im Graphen: 10 Fragen, 8 Tools** (ca. 1,5 min)
- Heft 4: 1.054 Regesten aus dem Stadtarchiv Frankfurt, 1440–1493
- Auswahl der 10 Fragen (Tabelle kompakt) – 5 Basis · 2 genauer · 3 nur mit Ergänzung
- Chatbot nutzt das gemeinsame Modell nur über feste MCP-Tools, jede Antwort mit Regestennummer

**Folie 2 – Fallstricke und unsere Ergänzung** (ca. 2 min)
- Links: Chmel-Dubletten (100) → Heft zählt, Chmel als Hinweis · Datum: 483 unsicher → Überlappung + Hinweis · Ort: 442 ohne Ort am Schein-Ort „-“ → ausschließen, Lücke nennen
- Rechts: Beispiel n. 32 vorher (Register) / nachher (LLM-Rollen + Rechtsakt), Herkunft `register` / `llm` / `manual` mit `status`
- Zahlen: 1.101 Rechtsakte, 1.456 Rollen, Goldstandard läuft

**Folie 3 – Architektur** (ca. 1,5 min)
- Diagramm: Browser ↔ FastAPI (Chatbot/Harness) ↔ LLM; MCP-Client ↔ RI-MCP-Server (8 Tools) ↔ Neo4j (RI-alles + Ergänzung); Wikidata für Steckbriefe
- Kernaussage: Das LLM formuliert, die Tools rechnen und belegen.

---

## 7. Offene Punkte

- Goldstandard auswerten (Präzision/Trefferquote je Rolle und Rechtsakt) und in die Abgabe übernehmen
- Ergänzung in die eigene Neo4j-Instanz importieren und die Abfragen 3, 7, 10 dort gegenprüfen
- Doppelte Registerverknüpfungen (44 Regesten) in der Datenbank klären
- Wikidata-MCP-Server und Bibliotheken für Karte/Zeitleiste auswählen
