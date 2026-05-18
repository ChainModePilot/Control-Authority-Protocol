## Kapitel 5: Schema und Liefergegenstände

Dieses Kapitel beschreibt die Struktur der Liefergegenstände des CAP-Projekts, definiert die zentrale Stellung der Protokoll-Schema-Definitionsdateien innerhalb der Liefergegenstände sowie die Zuordnung zwischen Schema-Versionen und Protokollversionen. Das Verständnis der hierarchischen Beziehungen und Abhängigkeiten zwischen den Liefergegenständen ist die Grundlage für die Zusammenarbeit von Protokollimplementierern und Community-Beitragenden am Projekt.

### 5.1 Drei Kategorien von Kernliefergegenständen

Das Liefergegenstände-System des CAP-Projekts besteht aus drei Kategorien von Kernliefergegenständen, die von der Dokumentation über die Definition bis zur Implementierung eine vollständige Lieferkette bilden:

| Kategorie der Liefergegenstände | Beschreibung | Beispiel für Repository-Pfad |
|--------------------------------|-------------|------------------------------|
| **Mehrsprachige Protokolldokumentation** | Umfasst den Architekturplan (Blueprint) und die technische Protokollspezifikation (Specification), die in 9 Sprachen äquivalent veröffentlicht werden und eine menschenlesbare Beschreibung der Designabsicht, des Fähigkeitsumfangs und der technischen Details des Protokolls bieten | `docs/{lang}/blueprint/`, `docs/{lang}/specification/` |
| **Protokoll-Schema-Definitionsdateien** | Die autoritative Definition der Protokollnachrichtenformate, bereitgestellt in mehreren maschinenlesbaren und menschenlesbaren Formaten, als Referenz für SDK-Implementierung und Interoperabilitätstests | `schema/{version}/` |
| **SDK-Implementierungen in verschiedenen Sprachen** | Konkrete Programmiersprachen-Implementierungen des Protokolls, entwickelt auf Basis der Schema-Definitionsdateien, die Entwicklern verschiedener Technologie-Stacks sofort einsetzbare Protokollintegrationsfähigkeiten bieten | `sdk/{language}/` |

Hierarchische Beziehungen zwischen den drei Kategorien von Liefergegenständen:

- **Mehrsprachige Protokolldokumentation** befindet sich auf der strategischen und normativen Ebene und definiert das übergeordnete Design dessen, „was das Protokoll tut" und „wie es es tut"
- **Protokoll-Schema-Definitionsdateien** befinden sich auf der Definitionsebene und wandeln die Nachrichtenformate der Protokollspezifikation in präzise, maschinenverarbeitbare formale Definitionen um
- **SDK-Implementierungen in verschiedenen Sprachen** befinden sich auf der Implementierungsebene und wandeln die Protokollfähigkeiten auf Basis der Schema-Definitionsdateien in direkt aufrufbare Programmierschnittstellen um

Die Aktualisierung der drei Kategorien von Liefergegenständen folgt einem Top-Down-Übertragungspfad: Änderungen an der Protokolldokumentation treiben die Aktualisierung der Schema-Definition voran, und die Aktualisierung der Schema-Definition treibt die Anpassung der SDK-Implementierung voran.

### 5.2 Rolle der Schema-Definitionsdateien

Die Schema-Definitionsdateien sind die **autoritative Definition (Authoritative Definition)** der CAP-Protokollnachrichtenformate und übernehmen im Liefergegenstände-System des Projekts die folgenden Kernfunktionen:

- **Einzige Wahrheitsquelle für Nachrichtenformate (Single Source of Truth)**: Die Schema-Definitionsdateien beschreiben präzise die Feldnamen, Datentypen, Pflicht-/Optional-Einschränkungen und verschachtelten Strukturen aller Nachrichtentypen im CAP-Protokoll. Wenn Mehrdeutigkeiten zwischen der Textbeschreibung der Protokolldokumentation und der Schema-Definition bestehen, hat die Schema-Definition Vorrang
- **Entwicklungsreferenz für SDK-Implementierung**: Die Nachrichtenserialisierungs-/Deserialisierungslogik der SDKs in verschiedenen Sprachen muss strikt der Schema-Definition folgen. SDK-Entwickler erfahren über die Schema-Definitionsdateien die genaue Struktur jedes Nachrichtentyps und stellen sicher, dass die SDK-Implementierungen in verschiedenen Sprachen im Nachrichtenformat vollständig konsistent sind
- **Validierungsreferenz für Interoperabilitätstests**: Interoperabilitätstests zwischen SDKs verschiedener Sprachen verwenden die Schema-Definition als Validierungsstandard. Eine von einem SDK serialisierte Nachricht muss von einem anderen SDK korrekt deserialisiert werden können, und die Schema-Definitionsdatei ist die einzige Grundlage für die Bestimmung der „Korrektheit"
- **Grundlage für automatisierte Validierung**: Schema-Definitionsdateien (insbesondere schema.json) können direkt von automatisierten Tools konsumiert werden, für die automatische Validierung von Nachrichtenformaten, Codegenerierung und Dokumentationsgenerierung, wodurch menschliche Fehler reduziert werden

### 5.3 Zuordnung zwischen Schema-Version und Protokollversion

Jede offiziell veröffentlichte CAP-Protokollversion entspricht einer Schema-Version, und die Schema-Dateien werden in einem nach dem Veröffentlichungsdatum der Protokollversion benannten Verzeichnis gespeichert, um eine klare und nachvollziehbare Versionszuordnung zu gewährleisten.

**Verzeichnisbenennungsregeln:**

| Protokollstatus | Schema-Verzeichnis | Beschreibung |
|----------------|-------------------|-------------|
| Entwurf (Draft) | `schema/draft/` | Schema-Definition in Entwicklung, jederzeit änderbar, ohne Kompatibilitätsgarantien |
| Offiziell veröffentlicht | `schema/{JJJJ-MM-TT}/` | Benannt nach dem Veröffentlichungsdatum, z. B. `schema/2025-10-25/`, nach Veröffentlichung unveränderlich (immutable) |

**Versionszuordnungsregeln:**

- **Eins-zu-eins-Zuordnung**: Jede offiziell veröffentlichte Protokollversion entspricht genau einem Schema-Versionsverzeichnis, dessen Name das Veröffentlichungsdatum dieser Protokollversion verwendet
- **Gemeinsamer Entwurf**: Alle Schema-Änderungen in der Entwurfsphase teilen sich das `schema/draft/`-Verzeichnis. In der Entwurfsphase werden keine Versionsnummern unterschieden
- **Unveränderlichkeit**: Die Dateien in offiziell veröffentlichten Schema-Verzeichnissen werden nach der Veröffentlichung nicht mehr geändert. Alle Schema-Änderungen (einschließlich Fehlerkorrekturen) erfolgen durch die Veröffentlichung neuer Versionen
- **Historische Aufbewahrung**: Verzeichnisse alter Schema-Versionen werden dauerhaft im Repository aufbewahrt, als historische Referenz und zur Validierung der Abwärtskompatibilität

**Aktuelle Versionszuordnung:**

| Protokollversion | Schema-Verzeichnis | Status |
|-----------------|-------------------|--------|
| draft | `schema/draft/` | In Entwicklung |
| 2025-10-25 | `schema/2025-10-25/` | Veröffentlicht |

### 5.4 Beschreibung der Schema-Dateiformate

Jedes Schema-Versionsverzeichnis enthält 3 Formate von Schema-Definitionsdateien, die jeweils auf unterschiedliche Anwendungsszenarien und Nutzer ausgerichtet sind:

| Format | Dateiname | Verwendungszweck | Hauptnutzer |
|--------|-----------|-----------------|-------------|
| **JSON Schema** | `schema.json` | Maschinenlesbare Nachrichtenformatdefinition, verwendet für automatisierte Validierung, Codegenerierung und sprachübergreifende Interoperabilitätsüberprüfung | Automatisierungstools, CI/CD-Pipelines, SDK-Codegeneratoren |
| **TypeScript** | `schema.ts` | TypeScript-Typdefinition, bietet native Typunterstützung für das TypeScript/JavaScript-Ökosystem, verwendet für TS/JS-SDK-Entwicklung und IDE-Typprüfung | TypeScript/JavaScript-SDK-Entwickler, Frontend-Integrationsentwickler |
| **MDX** | `schema.mdx` | Renderbares Dokumentformat, das die Schema-Definition auf menschenfreundliche Weise darstellt, verwendet für die Anzeige auf Dokumentationswebsites und zum Erlernen des Protokolls | Protokolllernende, Dokumentationswebsites, technische Prüfer |

**Beziehung zwischen den drei Formaten:**

- **schema.json ist das autoritative Format**: Wenn Unterschiede zwischen den drei Formaten bestehen, hat schema.json Vorrang. schema.ts und schema.mdx sollten semantisch mit schema.json konsistent sein
- **schema.ts ist das Entwicklungskomfortformat**: schema.ts wandelt die Typdefinitionen aus schema.json in native TypeScript-Typen um, sodass TS/JS-Entwickler direkt in der IDE Typhinweise und Kompilierungszeitprüfungen erhalten
- **schema.mdx ist das Anzeigeformat**: schema.mdx wandelt die strukturierten Definitionen aus schema.json in renderbaren Dokumentinhalt um und unterstützt das interaktive Durchsuchen der Schema-Definition auf Dokumentationswebsites

**Beispiel für die Dateiverzeichnisstruktur:**

```
schema/
├── draft/
│   ├── schema.json
│   ├── schema.ts
│   └── schema.mdx
└── 2025-10-25/
    ├── schema.json
    ├── schema.ts
    └── schema.mdx
```
