* **Läuft portabel als Windows-EXE (USB, ohne Installation, offline).**

* **Liest deine DOCX-Vorlagen** (Rechentest A/B/C) **und die zugehörigen DOCX-Lösungen** ein.

* **Segmentiert den Test** in Aufgabenblöcke:

  * **Aufgabe 1 & 2** werden als **parametrisierte Platzhalter** erkannt.
  * **Aufgabe 3–7** werden als **Blöcke mit Text + Bildern + Optionen a–e** erkannt (je Block **genau eine** richtige Option).

* **Ermittelt die korrekten Lösungen** für 3–7 aus der **Lösungs-DOCX** (z. B. „Aufgabe 4: c“).

* **Erzeugt neue Inhalte für Aufgabe 1 & 2** mit **gleichem Schwierigkeitsgrad**:

  * Zahlengrößen/Operatoren wie in den Beispielen.
  * **Divisionen nur mit ganzzahligem Ergebnis.**
  * **Umwandlungsaufgaben** mit denselben Einheitentypen/Komplexität.
  * Regeln sind **konfigurierbar** (YAML), initial **aus den Beispielen gelernt**.

* **Mischt ab Aufgabe 3**:

  * **Zufällige Reihenfolge der Aufgaben 3–7**.
  * **Zufällige Reihenfolge der Optionen a–e** innerhalb jeder Aufgabe.
  * **Richtige Antwort wandert mit**; nach dem Mischen werden die sichtbaren **Nummern wieder 3–7** gesetzt.

* **Bilderbehandlung 1:1 zum Original**:

  * Jedes Bild ist **genau einer Aufgabe zugeordnet** und **wandert mit**.
  * **Original-Größen (Extents) aus der Vorlage** werden übernommen.
  * **Kein Upscaling**; falls nötig, **proportional verkleinern** (Seitenverhältnis bleibt).

* **RNG/Varianten-ID**:

  * Jede Variante erhält eine **Varianten-ID** (auch auf dem Deckblatt).
  * Die ID dient als **Seed** → **reproduzierbare** Permutationen & Zahlen.

* **Rendert die Ergebnisse**:

  * **Neue Test-DOCX** mit aktualisierten A1/A2, gemischten 3–7, korrekten Bildern, Deckblatt unverändert.
  * **Neue Lösungs-DOCX** (A1/A2-Ergebnisse + richtige Buchstaben 3–7).
  * **Optional PDF-Export**, wenn MS Word vorhanden; sonst nur DOCX.

* **Fehlertoleranz & Checks**:

  * Prüft auf fehlende Bilder/Regeln.
  * Verhindert „unschöne“ Fälle (z. B. nicht-ganzzahlige Division).
  * Verständliche Fehlermeldungen.

**Kurzablauf pro Variante**

1. Vorlagen + Regeln laden → Seed aus Varianten-ID setzen
2. DOCX parsen (A1/A2, Blöcke 3–7, Bilder, richtige Option)
3. A1/A2 neu generieren (gleiches Level)
4. 3–7 und deren Optionen mischen (mit Lösungsmapping)
5. Bilder an den Blöcken belassen, Größen beibehalten/ggf. verkleinern
6. Test-DOCX + Lösungs-DOCX schreiben (Deckblatt/Nummerierung/Lösungstabelle konsistent)
7. Optional DOCX→PDF (falls Word vorhanden)

# SPEC-1-Testgenerator (Single-User)

## Background

Eine Lehrkraft möchte aus vorhandenen Tests (inkl. Bildern und Layout/Deckblatt) automatisch neue Varianten erzeugen. Die Varianten sollen:

- **Deckblatt & Layout 1:1 beibehalten** (Name/Prozente/… Felder bleiben an exakt gleicher Stelle).
- **Ab Aufgabe 3** die Reihenfolge der Aufgaben zufällig mischen.
- **Antwortmöglichkeiten** innerhalb einer Aufgabe zufällig anordnen.
- **Aufgaben 1 & 2** als **parametrisierte Platzhalter** erzeugen, sodass bei jeder Generierung neue, zufällige Zahlen entstehen — **mit gleichem Schwierigkeitsgrad** wie in den Beispielen (Zahlengrößen, keine unendlichen Divisionen; bei den Umwandlungsaufgaben aus „Beispiel 2“ zufällige, aber gleich schwere Werte).
- **Automatisch** sowohl **Test** als auch **Lösung** ausgeben.
- **Bilder** aus dem gelieferten Pool **unverändert** nutzen.

Als Referenz liegen **3 Tests**, die dazugehörigen **3 Lösungen** und alle benötigten **Bilder** vor (bereitgestellt in einer ZIP). Ziel ist ein **Einzelplatz-Programm**, das lokal läuft (kein Server), mit dem Lehrkräfte per Klick eine neue Test-/Lösungs-PDF generieren können.


## Requirements

**MUST (müssen)**
- **Portable Windows-EXE** (Win 10/11), lauffähig von **USB-Stick**, **ohne Installation/Adminrechte**, vollständig offline.
- **Eingaben**: 3 Testvorlagen + 3 zugehörige Lösungen (bestehende Layouts/Deckblätter) sowie Bild-Assets. Aus diesen werden neue Varianten erzeugt.
- **Layout-Treue 1:1**: Seitenformat, Ränder, Schriften (sofern möglich), Deckblatt-Felder (Name/Prozente/…) exakt erhalten.
- **Aufgabenreihenfolge ab Aufgabe 3**: zufällige Permutation je Generierung.
- **Antwortoptionen je Aufgabe**: zufällig anordnen; korrekte Lösung wird mitgeführt/aktualisiert.
- **Aufgaben 1 & 2**: parametrisierte Platzhalter mit **gleichem Schwierigkeitsgrad** (Zahlenbereiche, Operatoren; **keine unendlichen Divisionen**; Umwandlungsaufgaben behalten Einheitentyp & Komplexität bei).
- **Ausgabe**: automatisch **Test-PDF** *und* **Lösungs-PDF** (inkl. eindeutiger Varianten-ID).
- **Bildnutzung**: vorhandene Bilder unverändert verwenden; Qualität erhalten; korrekte Positionierung/Beschneidung.
- **Deterministische Zufälligkeit**: Seed/Variant-ID zur Reproduzierbarkeit; optional Eingabe eines Seeds.
- **Einfache GUI**: Auswahl Vorlage, Anzahl Varianten, (optional) Seed, Zielordner.
- **Konfigurierbare Regeln**: Schwierigkeitsgrade (Zahlenbereiche, erlaubte Einheiten/Operatoren) in **JSON/YAML**.
- **Fehlerrobustheit**: verständliche Fehlermeldungen (fehlende Bilder, unpassende Regeln, Layout-Konflikte).
- **Leistung**: 1 Variante in < ~5 s auf Standard-Office-PC; Speicherverbrauch moderat.
- **Sicherheit**: keine Internetzugriffe, keine Registry-Änderungen, keine externen Installer.

**SHOULD (sollen)**
- **Batch-Generierung** mehrerer Varianten in einem Lauf; fortlaufende Nummerierung.
- **Vorschau** vor Export.
- **Antwortschlüssel-Export** (CSV) zusätzlich zur Lösungs-PDF.
- **Wasserzeichen/QR** mit Varianten-ID (reproduzierbar, optional).
- **Einfache Template-Erweiterung**: neue Vorlagen via Konfig-Datei (Layout-Zonen, Aufgabenbeschreibung).

**COULD (können)**
- **Kommandozeilenmodus** (ohne GUI) für Power-User.
- **Sitzplan-Generator** (Verteilung der Varianten) als CSV/PDF.

**WON’T (vorerst nicht)**
- Mehrbenutzer-/Serverbetrieb, zentrale Datenbank.
- Vollautomatische OCR/PDF-Verständnis-Pipeline für beliebige Fremdvorlagen (nur „unser“ Template-Format).


## Method

### Architektur (Überblick)

Die Lösung ist ein **portables Windows-Programm (EXE)**, gebaut mit **Python 3.11 + PyInstaller (onefile/onedir)**. Es arbeitet **offline**, liest **DOCX-Vorlagen/Lösungen**, generiert neue **DOCX** und – falls **MS Word** vorhanden – zusätzlich **PDF** via Word-Automation (COM) bzw. docx2pdf. Layouttreue wird dadurch gewahrt, weil wir direkt auf DOCX-Objekte arbeiten.

- **DOCX-Bearbeitung:** `python-docx` zum Öffnen/Modifizieren (Absätze/Runs/Tabellen, Bilder behalten). 
- **Templating (A1/A2-Platzhalter):** `docxtpl` (Jinja2) für parametrisierte Felder/Blöcke in den Vorlagen A1/A2. 
- **PDF-Export (optional):** pywin32/Word SaveAs2(Format=PDF) oder `docx2pdf` (setzt Word voraus). Fallback: nur DOCX. 
- **Packaging:** PyInstaller (keine Adminrechte zum Ausführen nötig), portable auf USB. 

```plantuml
@startuml
skinparam componentStyle rectangle
actor Lehrer
rectangle App {
  component GUI
  component KonfigManager
  component Regel-Engine <<A1/A2>>
  component Parser <<DOCX>>
  component Randomizer
  component Renderer <<DOCX>>
  component PDFExporter <<optional>>
  component LoesungsBuilder
}
Lehrer --> GUI : Vorlage/Anzahl/Seed wählen
GUI --> KonfigManager : YAML laden
GUI --> Parser : DOCX (Test/Lösung) einlesen
Parser --> Randomizer : Aufgabenblöcke 3..7 + Optionen
KonfigManager --> Regel-Engine : Regeln A1/A2
GUI --> Regel-Engine : Seed
Regel-Engine --> Renderer : Inhalte für A1/A2
Randomizer --> Renderer : permutierte Aufgaben/Optionen
Renderer --> LoesungsBuilder : Mapping korrekte Buchstaben
Renderer --> GUI : DOCX ausgeben
PDFExporter <-- GUI : (wenn Word vorhanden) DOCX->PDF
GUI --> Lehrer : Test.pdf + Loesung.pdf
@enduml
```

### Erkennung & Neu-Anordnung in DOCX

**Anker & Blöcke**
- **Aufgabenblöcke 3–7:** Wir erkennen Überschriften mit Regex wie `^Aufgabe 3` bis `^Aufgabe 7` und gruppieren alle Absätze und Bilder bis zur nächsten Aufgabe. Paragraph- und Drawing-Knoten werden als Block verschoben, sodass Formatierung/Bilder exakt mitwandern. Hinweise zu Inline-Bildern und XML-Manipulation sind bekannt; falls nötig arbeiten wir direkt auf den Word-XML-Knoten (w:p, w:drawing). 
- **Optionen a–e (Single-Choice):** Erkennung via Muster `^[a-e][).:] <Leerzeichen>` in den Absätzen des Blocks; die **Reihenfolge der Optionen** wird zufällig permutiert, die **richtige Lösung** wird mitgeführt.
- **Renummerierung:** Nach dem Shuffle werden die sichtbaren Nummern wieder als 3–7 gesetzt; die **Antworttabelle/Deckblatt** (Lösungsbuchstaben) wird konsistent gefüllt.

**Deterministische Zufälligkeit**
- Varianten-ID (z. B. `A-2025-11-03-042`) → RNG-Seed → reproduzierbare Permutation von **Aufgaben** und **Optionen**.

### Platzhalter & Regel-Engine (Aufgabe 1 & 2)

Wir parametrisieren **A1/A2** in den Vorlagen per `docxtpl`-Platzhaltern (z. B. `{{A1_expr_1}}`, `{{A1_result_1}}`, `{{A2_value}}`, `{{A2_unit_from}}`, `{{A2_unit_to}}`). Die **Regel-Engine** erzeugt je Run neue Werte **mit gleichem Schwierigkeitsgrad**:

- **A1 (Rechnen):**
  - Muster aus Beispielen lernen: erkannte Operatoren (±×÷), **Zahlenbereiche** je Operator, **Anzahl Aufgabenzeilen**.
  - **Divisionen** nur mit **ganzzahligem Ergebnis**; Multiplikation/Addition/Subtraktion passend skaliert (Ziffernanzahl wie Vorlage).
  - Konfigurierbar in YAML (siehe unten).

- **A2 (Umwandlungen):**
  - Erkennung von **Dimensionen** (z. B. Länge, Masse, Zeit) aus Beispielen; konfigurierbare **Einheitenpaare** und **Skalen** (z. B. mm↔cm↔m↔km).
  - Zufallswerte im gleichen Größenbereich (z. B. 2–4-stellig), optional Rundungsregeln (keine endlosen Dezimalen, wenn in Beispielen nicht vorhanden).

**Beispiel YAML-Schema (Auszug)**
```yaml
meta:
  template: "Rechentest A.docx"
  solution_template: "Rechentest A Lösung.docx"
  variant_prefix: "A"
  deckblatt:
    show_variant_id: true

aufgabe1:
  rows: 10
  patterns:
    - op: add
      a: {min: 10, max: 99}
      b: {min: 10, max: 99}
    - op: div_int
      a: {min: 20, max: 90}
      b: {divisors_of_a: true, min: 2, max: 10}

aufgabe2:
  conversions:
    - from: cm
      to: mm
      value: {digits: 2, min: 10, max: 99}
    - from: m
      to: cm
      value: {digits: 2, min: 1, max: 20}
  rounding: integer

aufgabe3_7:
  heading_patterns: ["Aufgabe 3", "Aufgabe 4", "Aufgabe 5", "Aufgabe 6", "Aufgabe 7"]
  option_pattern: "^[a-e][).:] "
  shuffle_tasks: true
  shuffle_options: true
```

### Algorithmus (vereinfacht)

```text
1) Lade Konfiguration + Vorlagen (Test & Lösung)
2) Erzeuge Varianten-ID → setze RNG-Seed
3) Parser liest DOCX-Test:
   - Segmentiert Blöcke Aufgabe 3..7 (inkl. Bilder)
   - Erfasst Optionen a–e je Block und markiert korrekte Option (aus Vorlage oder Lösung)
4) Regel-Engine generiert neue Inhalte für A1 und A2
5) Randomizer:
   - Permutiere Reihenfolge der Blöcke 3..7
   - Permutiere Optionen a–e je Block; update korrekter Buchstabe
6) Renderer baut neues DOCX:
   - Setzt A1/A2-Platzhalter
   - Fügt Blöcke 3..7 in neuer Reihenfolge ein
   - Aktualisiert Deckblatt + Lösungsfelder (Buchstaben für 3..7)
7) LösungsBuilder erzeugt Lösung (gleiches Layout, markierte/aufgelistete Buchstaben/Ergebnisse)
8) Export: DOCX, optional PDF (wenn Word vorhanden)
```

### PDF-Erzeugung (optional)
- Mit Word: via COM SaveAs2(FileFormat=PDF) oder `docx2pdf`. Falls Word nicht verfügbar → kein PDF, nur DOCX (Layout bleibt intakt). 

### Fehlerfälle & Validierung
- Fehlende Bilder/Assets → Abbruch mit Klartext-Hinweis.
- Parser findet weniger/mehr als 5 Optionen a–e → Warnung, Variante überspringen oder Regeln anpassen.
- A1/A2-Regeln erzeugen „unschöne“ Fälle (z. B. nicht ganzzahlig) → Ersatzziehung bis valide.

---


### Bildskalierung & Platzierung

- **Zuweisung:** Jedes Bild ist genau einer Aufgabe zugeordnet und wandert beim Shuffle **mit der Aufgabe** mit.
- **Skalierung:** Standard **„fit“** in einen vorgegebenen Rahmen (Breite/Höhe), **Seitenverhältnis immer beibehalten**, **kein Zuschnitt** ("fill") – außer explizit in der Konfiguration gewünscht.
- **Größensteuerung:** Maße in **cm** (intern DOCX-EMUs). Wir begrenzen auf `max_width_cm`/`max_height_cm` per Aufgabe. **Kein Upscaling** über die native Auflösung hinaus (um Pixeligkeit zu vermeiden).
- **DPI/Kompression:** Ziel-DPI pro Export (z. B. 200 dpi für Tests, 300 dpi für drucknahe Qualität). Bilder werden – nur wenn nötig – verlustarm neu berechnet.
- **Positionierung:** Bilder bleiben am **ursprünglichen Anker** (inline oder floating) innerhalb des Aufgaben-Blocks; bei floating setzen wir eine **Layout-Grenze** (z. B. keine Überlappung mit Textfeldern) und optional „fixe Position“.

**Beispiel-Konfiguration (Auszug)**
```yaml
images:
  default:
    max_width_cm: 12.5
    max_height_cm: 6.5
    mode: fit        # fit|fill
    target_dpi: 200  # 200|300
    allow_upscale: false
  aufgabe5:
    max_width_cm: 10.0
    max_height_cm: 8.0
    mode: fit
    target_dpi: 300
```


## Implementation

### Projektstruktur
```
rechentest-generator/
  app/
    gui.py                 # Tkinter GUI (Vorlage, Anzahl, Seed, Zielordner)
    main.py                # CLI/Startpunkt, Orchestrierung
    parser_docx.py         # DOCX lesen, Aufgabenblöcke 3–7 + Optionen + Bilder erkennen
    randomizer.py          # Deterministisches Mischen (Aufgaben/Optionen) per Seed
    rule_engine.py         # Generator für Aufgabe 1 & 2 (Zahlbereiche, Einheiten, Divisionen)
    renderer_docx.py       # Neues DOCX schreiben (Deckblatt, Platzhalter, Blöcke, Bilder)
    solution_builder.py    # Lösungs-DOCX erzeugen (A1/A2-Ergebnisse, Buchstaben 3–7)
    pdf_export.py          # Optional: DOCX→PDF über Word/COM oder docx2pdf
    config.py              # YAML laden/validieren, Defaults
    validators.py          # YAML-Schema-Prüfungen, Dry-Run-Checks
    learn_from_examples.py # Einmal: Regeln aus A/B/C-Vorlagen extrahieren
  templates/
    A/ {test.docx, solution.docx, config.yaml}
    B/ {test.docx, solution.docx, config.yaml}
    C/ {test.docx, solution.docx, config.yaml}
  dist/                    # Ausgabevarianten
  buildspec/
    pyinstaller.spec       # Portable Build-Konfiguration (onefile/onedir)
```

### Dateien & Module (Aufgaben)
- **gui.py**: Minimalistische Oberfläche (Dateiauswahl Vorlage A/B/C, Anzahl Varianten, Seed optional, Zielordner). Fortschrittsanzeige, Fehlermeldungen.
- **main.py**: Verknüpft GUI/CLI, setzt Varianten-ID (\<Vorlagenpräfix\>-\<YYYYMMDD\>-\<laufende Nr.\>) und RNG-Seed.
- **parser_docx.py**:
  - Liest Test- und Lösungs-DOCX.
  - Findet **Aufgabenblöcke 3–7** (alle Absätze + Bilder bis zur nächsten Überschrift).
  - Extrahiert **Optionen a–e** je Block und ermittelt **korrekte Option** aus der Lösung.
  - Liest **Bild-Extents** (Breite/Höhe in EMUs) + Media-Datei.
- **randomizer.py**: Permutiert Aufgaben 3–7 und die Optionen a–e deterministisch (Seed). Aktualisiert Mapping „Aufgabe → richtiger Buchstabe“.
- **rule_engine.py**: Generiert Inhalte für **Aufgabe 1 & 2** nach YAML-Regeln (Zahlenbereiche, Operatoren, nur ganzzahlige Divisionen; Umwandlungen: Einheiten & Wertebereiche).
- **renderer_docx.py**: Baut neues **Test-DOCX** (Platzhalter A1/A2 füllen, Blöcke 3–7 in neuer Reihenfolge einsetzen, **Bilder mit Original-Extents**, Nummern wieder 3–7, Deckblatt/Varianten-ID). Baut **Lösungs-DOCX** über **solution_builder.py**.
- **solution_builder.py**: Erzeugt Lösungsseiten (A1/A2-Ergebnisse; Buchstaben 3–7 in Tabelle/Listenform).
- **pdf_export.py**: Optionaler Export nach PDF, wenn Word vorhanden. Fallback: nur DOCX.
- **config.py / validators.py**: YAML laden, Schema prüfen, Defaults mergen; Dry-Run (nur A1/A2 + Mapping 3–7 ohne Render).
- **learn_from_examples.py**: Extrahiert initiale Regeln aus A/B/C (Operatoren, Zahlen-/Einheitenbereiche) und schreibt Vorschlags-YAML.

### Regex-Definitionen (Parsing)
> Alle Muster werden auf **Absatztext** angewendet. Backslashes sind hier doppelt geschrieben (\) für Klarheit.

- **Aufgaben-Überschrift**: `^Aufgabe\s+([3-7])\b`
  - Gruppe 1 = Aufgabennummer; der **Block** reicht bis zur nächsten Überschrift oder Dokumentende.
- **Optionenzeile (a–e)**: `^[a-e][)\.:]\s+`
  - Erkennt a) / b: / c. + Leerzeichen. Mehrzeilige Optionen werden bis zur nächsten Option oder Leerzeile zusammengeführt.
- **Lösungszeile (in Lösung-DOCX)**: `^Aufgabe\s*([3-7])\s*[:\-]?\s*([a-e])\b`
  - Gruppe 1 = Aufgabennummer; Gruppe 2 = richtiger Buchstabe.
- **A1/A2-Platzhalter** (mit docxtpl): `{{A1_.*?}}`, `{{A2_.*?}}` (keine Regex-Verarbeitung nötig, hier nur Konvention der Platzhalternamen).

### YAML-Schema (kommentiert)
> Konfig-Datei pro Vorlage (A/B/C) unter `templates/<X>/config.yaml`.

```yaml
meta:
  template: "Rechentest X.docx"        # Pfad zur Testvorlage
  solution_template: "Rechentest X Lösung.docx"  # Pfad zur Lösungsvorlage
  variant_prefix: "X"                   # z. B. A | B | C
  deckblatt:
    show_variant_id: true               # ID sichtbar auf Deckblatt
    variant_id_format: "{prefix}-{date:%Y%m%d}-{n:03d}"  # Formatstring
  rng:
    seed_source: "variant_id"          # variant_id | fixed
    fixed_seed: null                    # optional fester Seed für Tests

aufgabe1:                                # Rechenaufgaben (Zeilenliste)
  rows: 10                               # Anzahl Zeilen
  patterns:                              # Liste erlaubter Muster
    - op: add                            # add|sub|mul|div_int
      a: {min: 10, max: 99, digits: 2}
      b: {min: 10, max: 99, digits: 2}
    - op: div_int                        # Division mit ganzzahligem Ergebnis
      a: {min: 20, max: 90}
      b: {divisors_of_a: true, min: 2, max: 10}
  formatting:
    spaces_around_operator: true
    result_field_placeholder: "______"   # optisch wie Vorlage

aufgabe2:                                # Umwandlungsaufgaben
  conversions:                           # Liste möglicher Paare
    - from: cm
      to: mm
      value: {min: 10, max: 99, digits: 2}
    - from: m
      to: cm
      value: {min: 1, max: 20, digits: 2}
  rounding: integer                      # integer|fixed(n)|none
  allowed_dimensions: [laenge]           # laenge|masse|zeit ... (optional)

aufgabe3_7:
  heading_patterns: ["Aufgabe 3", "Aufgabe 4", "Aufgabe 5", "Aufgabe 6", "Aufgabe 7"]
  option_pattern: "^[a-e][)\.:]\s+"
  shuffle_tasks: true
  shuffle_options: true
  renumber_visible_from: 3               # sichtbare Nummerierung zurücksetzen

images:                                  # Bildhandhabung
  default:
    mode: fit                            # fit|fill
    max_width_cm: 17.38                  # per Vorlage ermittelt oder kleiner
    max_height_cm: 10.32
    allow_upscale: false
    target_dpi: 200
  overrides:                             # optional pro Aufgabe
    5: {max_width_cm: 10.0, target_dpi: 300}

export:
  pdf: auto                              # auto|never  (auto nur, wenn Word vorhanden)
  output_naming: "{prefix}-{date:%Y%m%d}-{n:03d}-{kind}"  # kind=test|loesung
```

### Ablauf (funktional, API der Module)
- **parse(test_docx, solution_docx)** → `ParsedDoc(test_blocks, solution_map, images)`
- **generate_a1a2(config, rng)** → `A1A2Payload` (Listen der Aufgaben + Lösungen)
- **shuffle_blocks(blocks, rng)** → neue Reihenfolge, **shuffle_options** pro Block
- **render_test(parsed, a1a2, order, option_orders, config)** → `test.docx`
- **build_solution(parsed, a1a2, option_orders)** → `loesung.docx`
- **export_pdf(docx_path, mode=auto)** → `pdf_path | None`

> Ergebnis pro Lauf: **Test-DOCX** + **Lösungs-DOCX**, optional **PDF**. Varianten-ID/Seed sichern Reproduzierbarkeit.
