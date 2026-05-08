# Quality Alert Vision Workflow Plan

Status: Konzept nach Review, vor Implementierung  
Ziel: PVA-Quality-Alerts sollen nicht mehr heuristisch oder zufaellig wirken, sondern Bild + Kontext nachvollziehbar bewerten und bei Unsicherheit sauber eskalieren.

## 1. Problem

Der bestehende Quality-Alert-Pfad bewertet im Kern Textsignale:

- `n8n/assess-alert-v2.mjs` nutzt gewichtete Keywords wie `defekt`, `bruch`, `kratzer`.
- `n8n/workflows/quality-alert-ai-evaluation.json` ist bewusst `text-first`; der Prompt sagt explizit: Es stehen keine Bildinhalte zur Verfuegung.
- Fotos erhoehen aktuell nur die Schwere (`photo_count > 0`), werden aber nicht visuell verstanden.

Damit entsteht ein falscher Eindruck von Intelligenz: Ein beliebiges Bild kann hochgeladen werden, ohne dass das System erkennt, ob darauf ein Produkt, ein Defekt, ein unbrauchbares Foto oder gar kein relevanter Inhalt sichtbar ist.

## 2. Zielverhalten

Wenn ein Nutzer in der PVA einen Quality Alert anlegt, soll der n8n-Workflow den Eintrag wie ein Qualitaetsassistent behandeln:

1. Alert aus PVA empfangen: Beschreibung, Produktkontext, Picking-Kontext, Foto(s).
2. Bildqualitaet pruefen:
   - Bild vorhanden?
   - Bild lesbar/scharf/genug belichtet?
   - Relevantes Objekt sichtbar?
3. Produkt-/Kontextabgleich pruefen:
   - Passt das sichtbare Objekt grob zum erwarteten Produkt?
   - Falls Testdaten: z. B. blauer LEGO 2x2 Stein erwartet.
4. Defektbewertung:
   - Defekt sichtbar?
   - Wenn ja: welcher Typ und wie schwer?
   - Wenn nein: klar sagen, dass kein sichtbarer Defekt erkennbar ist.
5. Entscheidung mit Confidence und Begruendung schreiben.
6. Bei Unsicherheit oder schlechter Bildqualitaet: manuelle Pruefung statt automatischer harter Entscheidung.

## 3. Empfohlenes Klassenschema

Das aktuelle Schema `damage | shortage | wrong_item | unclear` reicht fuer visuelle Qualitaetspruefung nicht aus. Empfohlen ist ein zweistufiges Schema.

### 3.1 Bild-/Evidenzstatus

- `no_image`: kein Bild vorhanden.
- `image_unreadable`: Bild unscharf, zu dunkel, verdeckt oder unbrauchbar.
- `irrelevant_image`: Bild zeigt kein relevantes Produkt/Teil.
- `object_visible`: relevantes Objekt ist sichtbar.

### 3.2 Visuelle Bewertung

- `defect_visible`: Defekt ist sichtbar.
- `no_defect_visible`: Objekt sichtbar, aber kein Defekt erkennbar.
- `wrong_item_visible`: sichtbares Objekt passt nicht zum erwarteten Produkt.
- `uncertain`: keine belastbare Entscheidung moeglich.

### 3.3 Disposition

Die bestehende Disposition kann bleiben, aber sie muss aus Evidenz abgeleitet werden:

- `scrap`: schwerer sichtbarer Schaden, eindeutig unbrauchbar.
- `quarantine`: sichtbarer Schaden oder unklare Lage mit Risiko.
- `rework`: leichte sichtbare Abweichung, z. B. Verpackung/Etikett/Nacharbeit.
- `sellable`: nur wenn Objekt sichtbar ist und kein Defekt erkennbar ist; trotzdem ggf. als Vorschlag, nicht als finaler QA-Freigabebeschluss.
- `manual_review`: schlechte Bildqualitaet, irrelevantes Bild, widerspruechlicher Kontext oder niedrige Confidence.

Wichtig: `sellable` darf nicht aus fehlenden Defekthinweisen entstehen. Es darf nur entstehen, wenn das Bild verwertbar ist und kein Defekt sichtbar ist.

## 4. Architekturvorschlag

### 4.1 Kein Training als erster Schritt

Fuer die Bachelorarbeit und den Prototyp ist ein eigenes trainiertes Defekterkennungsmodell wahrscheinlich zu viel Risiko:

- zu wenig echte Defektbilder,
- hoher Labeling-Aufwand,
- unklare Generalisierung,
- schwierige reproduzierbare Evaluation.

Besser: Vision-LLM oder Vision-API als interpretierender Pruefschritt plus strikt validiertes JSON, Guardrails, Schwellenwerte und Human Review.

### 4.2 Warum das wissenschaftlich vertretbar ist

Industrie-Anomalieerkennung wird oft als Problem mit wenig Defektbeispielen beschrieben. MVTec AD ist der Standard-Benchmark: ueber 5000 hochaufloesende Bilder, 15 Objekt-/Texturkategorien, normale Trainingsbilder und Defekt-Testbilder mit pixelgenauen Annotationen. Der Punkt fuer diese Arbeit: echte Defekte sind selten, deshalb sind unsupervised/anomaly-basierte und synthetische Datenansaetze relevant.

Fuer den Prototyp heisst das:

- keine Behauptung: "Wir trainieren ein perfektes QA-Modell";
- bessere Behauptung: "Wir integrieren eine evidenzbasierte visuelle QA-Assistenz in einen mobilen Picking-Prozess und evaluieren Workflow-Nutzen, Robustheit und Eskalationsqualitaet."

## 5. Synthetic-LEGO-Teststrategie

Da keine echten defekten Warenbilder vorhanden sind, wird ein kontrollierter LEGO-Datensatz aufgebaut.

### 5.1 Produktfamilie

Start mit wenigen, klaren Klassen:

- `lego_blue_2x2_brick`
- optional spaeter: `lego_red_2x2_brick`, `lego_blue_2x4_brick`, falsche Farbe/Form als Wrong-Item-Faelle.

### 5.2 Bildklassen

Pro Produkt werden Bilder in diesen Gruppen erzeugt/gesammelt:

1. `normal_good`
   - sauberer blauer 2x2 Stein, verschiedene Winkel/Lichtbedingungen.
2. `visible_defect`
   - Riss, abgebrochene Ecke, tiefer Kratzer, starke Verschmutzung, Verformung.
3. `minor_issue`
   - leichter Kratzer, kleine Verschmutzung, fragliche Abweichung.
4. `wrong_item`
   - falsche Farbe, falsche Groesse, anderes Objekt.
5. `bad_photo`
   - unscharf, zu dunkel, abgeschnitten, verdeckt.
6. `irrelevant`
   - Tisch, Hand, anderes Objekt, leerer Hintergrund.

### 5.3 Datensatzgroesse fuer PoC

Minimum fuer reproduzierbare Tests:

- 20 gute Bilder
- 20 sichtbare Defekte
- 10 leichte Defekte
- 10 falsche Artikel
- 10 schlechte Fotos
- 10 irrelevante Bilder

Total: ca. 80 Bilder. Das reicht nicht fuer Training, aber fuer Workflow- und Prompt-Evaluation.

### 5.4 Generierung

Moegliche Quellen:

- echte Smartphone-Fotos von LEGO-Steinen,
- synthetisch erzeugte Bilder mit kontrollierten Defekten,
- manuell bearbeitete Bilder (Kratzer/Bruch/Masken),
- Kombination aus echten Normalbildern und synthetischen Defektvarianten.

Wichtig: Der Testdatensatz muss Labels haben:

```json
{
  "image_id": "lego_blue_2x2_defect_001",
  "expected_product": "lego_blue_2x2_brick",
  "ground_truth_evidence_status": "object_visible",
  "ground_truth_visual_finding": "defect_visible",
  "ground_truth_defect_type": "broken_corner",
  "ground_truth_disposition": "quarantine"
}
```

## 6. n8n-Workflow-Zielbild

Neuer Workflow: `quality-alert-vision-assessment.json`

### Stufe A: Input normalisieren

Input aus PVA/Odoo:

- `correlation_id`
- `alert_id`
- `description`
- `product_id`, `product_name`, `expected_visual_description`
- `photo_urls` oder `photo_binary_refs`
- `priority`, `picker`, `location`

Validierung:

- ohne `alert_id` und `correlation_id`: abbrechen.
- ohne Bild: `no_image` + `manual_review` oder text-only fallback.

### Stufe B: Bild holen und minimieren

- Bild aus Backend/Odoo holen, nicht direkt aus unsicheren externen Quellen.
- Maximalgroesse begrenzen.
- Keine EXIF/Metadaten weitergeben, falls nicht gebraucht.
- Timeout hart setzen.

### Stufe C: Vision-Modell aufrufen

Prompt muss strikt sein:

- Rolle: visueller QA-Assistent, keine finale Rechts-/QA-Freigabe.
- Erwartetes Produkt beschreiben.
- Pruefe zuerst Bildqualitaet, dann Objekt, dann Defekt.
- Wenn nicht sichtbar: nicht raten.
- JSON-only.

Beispiel-Output:

```json
{
  "evidence_status": "object_visible",
  "visual_finding": "defect_visible",
  "defect_type": "broken_corner",
  "severity": "medium",
  "disposition": "quarantine",
  "confidence": 0.82,
  "summary": "Blauer 2x2 LEGO-Stein sichtbar; eine Ecke wirkt abgebrochen.",
  "recommended_action": "Ware sperren und manuell pruefen.",
  "needs_manual_review": true
}
```

### Stufe D: Schema validieren

n8n darf Modelloutput nicht blind schreiben.

Validierung:

- Enum-Werte pruefen.
- Confidence 0..1.
- `sellable` nur erlauben, wenn `evidence_status=object_visible`, `visual_finding=no_defect_visible`, `confidence >= 0.75`.
- `scrap` nur bei `defect_visible` und hoher Schwere/Confidence.
- `image_unreadable`, `irrelevant_image`, `uncertain` immer `manual_review`.

### Stufe E: Backend-Callback

Neuer oder erweiterter Callback:

`POST /api/internal/n8n/quality-vision-assessment`

Payload:

```json
{
  "schema_version": "v1",
  "correlation_id": "...",
  "alert_id": 123,
  "evidence_status": "object_visible",
  "visual_finding": "defect_visible",
  "defect_type": "broken_corner",
  "severity": "medium",
  "ai_disposition": "quarantine",
  "ai_confidence": 0.82,
  "ai_summary": "...",
  "ai_photo_analysis": "...",
  "ai_recommended_action": "...",
  "needs_manual_review": true,
  "ai_provider": "openai",
  "ai_model": "...",
  "latency_tracking": {
    "started_at": "...",
    "total_duration_ms": 1800,
    "stages": {
      "ingest_ms": 50,
      "callback_ms": 30
    },
    "extra_stages": {
      "image_fetch_ms": 120,
      "vision_ms": 1600,
      "validation_ms": 5
    }
  }
}
```

## 7. Evaluation fuer Bachelorarbeit

### 7.1 Evaluationsfragen

- Erkennt der Workflow unbrauchbare Bilder statt zu halluzinieren?
- Erkennt er sichtbare LEGO-Defekte mit ausreichender Trefferquote?
- Reduziert er die Zeit und verbessert die Qualitaet von Quality Reports?
- Sind seine Eskalationen nachvollziehbar?

### 7.2 Metriken

Auf synthetischem/kuratiertem Testdatensatz:

- Accuracy je Hauptklasse: Defekt / kein Defekt / falscher Artikel / schlechtes Bild / irrelevant.
- Precision/Recall fuer `defect_visible`.
- False-Sellable-Rate: defektes oder unbrauchbares Bild als `sellable` bewertet. Das ist die wichtigste Sicherheitsmetrik.
- Manual-Review-Rate.
- Durchschnittliche Latenz.
- JSON-valid-rate.
- Callback-success-rate.

Im Nutzer-/Prozessvergleich:

- Quality-Report-Zeit.
- Vollstaendigkeit des Reports nach Rubrik.
- Nachvollziehbarkeit der Empfehlung.
- Nutzervertrauen, aber getrennt von echter Modellguete.

### 7.3 Akzeptanzkriterien fuer PoC

Nicht: 100% Defekterkennung.  
Sondern:

- 0 kritische Schema-/Callback-Fehler in Tests.
- 0 bekannte Defektbilder werden als sicher `sellable` mit hoher Confidence freigegeben.
- Schlechte/irrelevante Bilder landen in `manual_review`.
- Mindestens 80% der klaren LEGO-Testbilder werden korrekt grob klassifiziert.
- Workflow bleibt unter definiertem Timeout und schreibt nachvollziehbare Begruendungen.

## 8. Implementierungsreihenfolge

### Schritt 1: Vertrag und Datenmodell

- Backend-Modell fuer Vision-Assessment ergaenzen.
- Callback-Route + Tests schreiben.
- n8n-Output-Schema dokumentieren.

### Schritt 2: Testdatensatz lokal anlegen

- Ordner `data/quality-vision-fixtures/` oder `fixtures/quality-vision/`.
- `labels.jsonl` mit Ground Truth.
- Keine privaten echten Kundendaten committen.

### Schritt 3: Offline-Evaluator

Script:

- liest Labels,
- ruft Vision-Assessment-Modul oder Mock auf,
- berechnet Metriken,
- exportiert CSV/JSON fuer Thesis.

### Schritt 4: n8n Shadow Workflow

- Erst Shadow Mode: bewertet, schreibt aber keine operative Entscheidung.
- Vergleich mit bestehender Heuristik.
- Fehler und Latenzen loggen.

### Schritt 5: Operative Nutzung mit Guardrails

- Nur bei hoher Confidence automatische Empfehlung.
- Nie harte Freigabe bei schlechten Bildern.
- Unsicherheit sichtbar machen.

## 9. Technisches Urteil

Der Vorschlag ist richtig und staerkt das Projekt deutlich. Er verschiebt das Feature von einer Text-Heuristik zu einer echten, evaluierbaren Quality-Assistenz. Der kritische Punkt ist aber Guardrail-Design: Das Modell darf nicht so tun, als habe es einen Defekt gesehen, wenn Bild oder Kontext schwach sind.

Der beste Thesis-kompatible Scope ist daher:

> Vision-gestuetzte Quality-Alert-Assistenz mit synthetisch/kuratiertem LEGO-Testdatensatz, striktem JSON-Vertrag, Human-Review-Eskalation und messbarer False-Sellable-Rate.

Das ist klein genug fuer Umsetzung, aber stark genug fuer eine Bachelorarbeit.

## 10. Recherchequellen

- MVTec AD Dataset: industrieller Anomaly-Detection-Benchmark mit ueber 5000 hochaufloesenden Bildern, 15 Objekt-/Texturkategorien, normalen Trainingsbildern, Defekt-Testbildern und pixelgenauen Annotationen. Relevant als wissenschaftlicher Referenzpunkt fuer visuelle Defekterkennung mit wenig Defektbeispielen.
- Bergmann et al. (2021), *The MVTec Anomaly Detection Dataset: A Comprehensive Real-World Dataset for Unsupervised Anomaly Detection*, International Journal of Computer Vision. Kernaussage fuer dieses Projekt: industrielle Defekte sind selten, unsupervised/anomaly detection und robuste Evaluation sind deshalb zentrale Themen.
- Recherche zu synthetischer Defektdatengenerierung bestaetigt: synthetische oder prozedural erzeugte Defektbilder sind ein legitimer Weg, wenn reale Fehlerbilder fehlen. Fuer diese Arbeit aber primaer als Test-/Evaluationsdaten nutzen, nicht als Beweis fuer produktive Modellguete.
- Recherche zu Vision-/LLM-basierter Quality Inspection bestaetigt: Vision-Modelle koennen flexible Inspektionsassistenz liefern, muessen aber durch JSON-Schema, Confidence-Schwellen, Human Review und No-Guessing-Regeln begrenzt werden.
