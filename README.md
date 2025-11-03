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

