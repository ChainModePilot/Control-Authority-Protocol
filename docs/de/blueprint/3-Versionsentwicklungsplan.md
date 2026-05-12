## Kapitel 3: Versionsentwicklungsplan

Dieses Kapitel definiert den Versionsentwicklungsplan des CAP-Protokolls, legt den Funktionsumfang und die Ausschlüsse der v1-Version fest, etabliert Versionsnummerierungsregeln und Kompatibilitätsstrategien und beschreibt den Veröffentlichungsprozess vom Entwurf bis zur offiziellen Version. Der Versionsentwicklungsplan bietet einen klaren Pfad für die iterative Entwicklung des Protokolls und die Zusammenarbeit in der Community.

### 3.1 Funktionsumfang der v1-Version

Die v1-Version des CAP-Protokolls konzentriert sich auf die Etablierung eines vollständigen Grundgerüsts für die Verwaltung von Endgerätekontrollberechtigungen und umfasst die folgenden 6 Kernfähigkeiten:

1. **Offline-Autorisierung (Authorization_Descriptor)**: Der Kernmechanismus der v1-Version. Implementierung des vollständigen Lebenszyklusmanagements des Authorization_Descriptor, einschließlich Ausstellung, lokaler Speicherung, Überprüfung, Widerruf und Erneuerung. Das Endgerät kann im Offline-Zustand die Autorisierungsüberprüfung eigenständig über den lokal gespeicherten Authorization_Descriptor durchführen und so die grundlegende Verfügbarkeit des Fay in netzwerklosen Umgebungen gewährleisten. Die v1-Version implementiert das vollständige Vertrauenskettenmodell und den lokalen Widerrufslistenmechanismus.

2. **Online-Tickets (Trusted_Ticket)**: Der ergänzende Mechanismus der v1-Version. Implementierung der Echtzeit-Autorisierungsausstellung und Widerrufsstatus-Abfragefähigkeiten in vernetzten Umgebungen, Unterstützung der Konvertierung von Trusted_Ticket in das lokale Authorization_Descriptor-Format sowie der nahtlosen Degradationsstrategie von Online zu Offline. Die v1-Version stellt die Zusammenarbeit zwischen Online-Tickets und Offline-Autorisierung sicher.

3. **Sitzungsverwaltung (Session lifecycle)**: Implementierung des vollständigen Lebenszyklusmanagements von Kontrollsitzungen, das die drei Phasen Erstellung, Aktivität und Beendigung abdeckt. Die v1-Version unterstützt die Eins-zu-eins-Bindungsbeziehung zwischen Session und Terminal_Resource und implementiert drei Arten der Sitzungsbeendigung: aktive Freigabe, Zeitüberschreitungsbeendigung und Widerrufsbeendigung.

4. **Kontrollübergabestrategie (Handover_Policy)**: Implementierung der Kontrollübertragungsfähigkeit zwischen mehreren Kontrollparteien, die Übergabeszenarien zwischen Fays sowie zwischen Fays und menschlichen Benutzern abdeckt. Die v1-Version unterstützt drei Strategietypen (Prioritätsregelskript, KI-Modell-Echtzeitentscheidung, Entscheidung durch menschlichen Benutzer), garantiert die Atomarität des Übergabeprozesses und implementiert den Zeitüberschreitungs-Rollback-Mechanismus.

5. **Aktivitätserkennung (Liveness_Detection)**: Implementierung des Sitzungsaktivitätserkennungsmechanismus basierend auf der Kombination von dauerhaften Verbindungen und Heartbeats auf Anwendungsebene. Die v1-Version unterstützt die doppelte Bewertungslogik (gleichzeitiges Erfüllen von Verbindungsunterbrechung und Heartbeat-Zeitüberschreitung), konfigurierbare Heartbeat-Intervalle und Zeitüberschreitungsschwellen sowie die automatische Beendigung abgelaufener Sitzungen und Ressourcenrückgewinnung.

6. **Ressourcenzugriffsmodus (Resource_Access_Mode)**: Implementierung der abgestuften Ressourcenzugriffsverwaltung nach Operationstyp mit Unterstützung der vier Zugriffsmodi read, write, execute und configure. Die v1-Version implementiert das vollständige Lese-Schreib-Sperrmodell mit Unterstützung für parallelen Mehrbenutzerzugriff im read-Modus und exklusive Kontrolle in den Modi write, execute und configure.

Die oben genannten 6 Kernfähigkeiten bilden den vollständigen Funktionsumfang der v1-Version und bieten einen grundlegenden Rahmen für die Kontrollberechtigung, der es intelligenten Agenten im iFay-Ökosystem ermöglicht, Endgeräte sicher zu übernehmen.

### 3.2 Explizit ausgeschlossene Funktionen der v1-Version

Die folgenden Funktionen sind explizit nicht in der v1-Version enthalten und als Kandidaten für zukünftige Versionen gekennzeichnet:

| Ausgeschlossene Funktion | Beschreibung | Kandidatenversion |
|--------------------------|-------------|-------------------|
| **Sitzungsmigration zwischen Endgeräten** | Migration einer aktiven Session von einem Endgerät zu einem anderen unter Beibehaltung der Sitzungsstatuskontinuität | v2+ |
| **Kooperative Autorisierung mehrerer Endgeräte** | Synchronisation des Autorisierungsstatus und kooperativer Überprüfungsmechanismus zwischen mehreren Endgeräten | v2+ |
| **Autorisierungsdelegationskette (Delegation Chain)** | Kaskadierender Autorisierungsmechanismus, bei dem ein Fay einen Teil seiner erhaltenen Autorisierung an andere Fays delegiert | v2+ |
| **Feingranulare Ressourcenzugriffskontrolle (ABAC)** | Attributbasierte Zugriffskontrollstrategie mit Unterstützung für komplexere Ressourcenzugriffsbedingungen | v2+ |
| **Standardisiertes Audit-Protokollformat** | Einheitliches Ausgabeformat und Abfrageschnittstellenspezifikation für Audit-Protokolle (Audit_Logger) | v2+ |
| **Verschlüsselungstransportspezifikation für Protokollnachrichten** | Standardisierte Definition der Verschlüsselungsmethoden und Schlüsselverhandlungsprozesse für CAP-Protokollnachrichten auf der Netzwerktransportschicht | v2+ |
| **Verteilter Widerrufskonsens für Offline-Autorisierung** | Konsensmechanismus für die Einigung auf den Widerrufsstatus zwischen mehreren Endgeräten in Offline-Umgebungen | v3+ |
| **Dynamische Berechtigungserhöhung (innerhalb der Session)** | Dynamische Erhöhung der Zugriffsberechtigungen während des aktiven Session-Lebenszyklus ohne Erstellung einer neuen Session | v3+ |

Die v1-Version folgt dem Prinzip „Erst die Grundlage schaffen, dann die Fähigkeiten erweitern" und priorisiert die Vollständigkeit und Stabilität der 6 Kernfähigkeiten, wobei erweiterte Funktionen für nachfolgende Versionsiterationen vorbehalten bleiben.

### 3.3 Versionsnummerierungsregeln und Kompatibilitätsstrategie

#### Versionsnummerierungsregeln

Das CAP-Protokoll verwendet das Format der semantischen Versionierung (Semantic Versioning): **`MAJOR.MINOR.PATCH`**

- **MAJOR (Hauptversionsnummer)**: Wird erhöht, wenn nicht abwärtskompatible wesentliche Änderungen am Protokoll vorgenommen werden. Beispiele: strukturelle Anpassungen des Kernnachrichtenformats, grundlegende Änderungen des Autorisierungsüberprüfungsprozesses. Eine Änderung der Hauptversionsnummer bedeutet, dass bestehende Implementierungen angepasst werden müssen
- **MINOR (Nebenversionsnummer)**: Wird erhöht, wenn dem Protokoll abwärtskompatible Funktionen oder Fähigkeiten hinzugefügt werden. Beispiele: Hinzufügen eines neuen Ressourcenzugriffsmodus, Erweiterung der Strategietypen der Handover_Policy. Eine Änderung der Nebenversionsnummer beeinträchtigt nicht den normalen Betrieb bestehender Implementierungen
- **PATCH (Revisionsnummer)**: Wird erhöht, wenn abwärtskompatible Fehlerkorrekturen oder Dokumentationsklarstellungen am Protokoll vorgenommen werden. Beispiele: Korrektur mehrdeutiger Beschreibungen in der Spezifikationsdokumentation, Ergänzung von Behandlungshinweisen für Grenzfälle

Entwurfsversionen verwenden die Kennzeichnung **`draft`** ohne Versionsnummer. Entwurfsversionen geben keine Kompatibilitätsgarantien und können jederzeit nicht abwärtskompatible Änderungen erfahren.

Beispiele für offiziell veröffentlichte Versionsnummern: `1.0.0`, `1.1.0`, `1.1.1`, `2.0.0`

#### Kompatibilitätsstrategie

Die Kompatibilitätsstrategie des CAP-Protokolls folgt den folgenden Prinzipien:

- **Abwärtskompatibilität innerhalb derselben Hauptversion**: Innerhalb derselben MAJOR-Version muss eine Implementierung der neuen Protokollversion in der Lage sein, Protokollnachrichten der alten Version zu verarbeiten. Beispielsweise muss ein Endgerät, das v1.2.0 implementiert, in der Lage sein, Authorization_Descriptors im v1.0.0-Format zu verarbeiten
- **Keine Kompatibilitätsgarantie zwischen Hauptversionen**: Zwischen verschiedenen MAJOR-Versionen wird keine Abwärtskompatibilität garantiert. Bei Erhöhung der Hauptversionsnummer listet die Protokollspezifikation alle nicht abwärtskompatiblen Änderungen und Migrationsleitfäden explizit auf
- **Schema-Version und Protokollversion sind aufeinander abgestimmt**: Jede Protokollversion entspricht einer Schema-Version, und die Schema-Dateien werden in einem nach der Protokollversion benannten Verzeichnis gespeichert (z. B. `schema/2025-10-25/`), um eine klare und nachvollziehbare Versionszuordnung zu gewährleisten
- **Deklaration der minimal unterstützten Version**: Protokollimplementierungen können die minimal unterstützte Protokollversion deklarieren. Protokollnachrichten unterhalb dieser Version werden abgelehnt

### 3.4 Veröffentlichungsprozess vom Entwurf zur offiziellen Version

Die Veröffentlichung des CAP-Protokolls vom Entwurf zur offiziellen Version folgt dem folgenden Prozess:

**Phase 1: Entwurf (Draft)**

- Protokollspezifikation und Schema-Definition werden im `draft/`-Verzeichnis gespeichert (z. B. `schema/draft/`, `specification/draft/`)
- In der Entwurfsphase sind häufige nicht abwärtskompatible Änderungen erlaubt, ohne Kompatibilitätsgarantien
- Entwurfsinhalte werden durch Community-Feedback und Reviews kontinuierlich iterativ optimiert
- Dokumente und Schema in der Entwurfsphase können jederzeit aktualisiert werden, ohne den Regeln zur Versionsnummernerhöhung folgen zu müssen

**Phase 2: Release-Kandidat (Release Candidate)**

- Wenn der Entwurfsinhalt stabil wird, tritt er in die Release-Kandidat-Phase ein
- Release-Kandidat-Versionen werden mit dem Suffix `-rc.N` gekennzeichnet (z. B. `1.0.0-rc.1`)
- In der Release-Kandidat-Phase werden nur Fehlerkorrekturen und Dokumentationsklarstellungen akzeptiert, keine neuen Funktionen
- Release-Kandidat-Versionen müssen eine vollständige Testvalidierung bestehen, einschließlich Schema-Validierung, Überprüfung der strukturellen Äquivalenz mehrsprachiger Dokumente und Eigenschaftstests

**Phase 3: Offizielle Veröffentlichung (Release)**

- Nach ausreichender Validierung wird das `-rc.N`-Suffix der Release-Kandidat-Version entfernt, und sie wird zur offiziellen Version
- Die Protokollspezifikation und Schema-Definition der offiziellen Version werden vom `draft/`-Verzeichnis in ein nach dem Veröffentlichungsdatum benanntes Verzeichnis kopiert (z. B. `schema/2025-10-25/`)
- Nach der Veröffentlichung der offiziellen Version werden die Protokollspezifikation und Schema-Definition dieser Version nicht mehr geändert (immutable). Alle Änderungen erfolgen durch die Veröffentlichung neuer Versionen
- Bei der Veröffentlichung der offiziellen Version müssen alle 9 Sprachversionen der Architekturplandokumente und Protokollspezifikationen gleichzeitig fertiggestellt und die strukturelle Äquivalenzvalidierung bestanden haben

**Phase 4: Wartung (Maintenance)**

- Nach der Veröffentlichung der offiziellen Version tritt sie in die Wartungsphase ein und akzeptiert nur Fehlerkorrekturen auf PATCH-Ebene
- Wenn eine neue MAJOR- oder MINOR-Version veröffentlicht wird, tritt die alte Version in eine begrenzte Wartungsperiode ein und wird schließlich als veraltet (deprecated) markiert
- Dokumente und Schema-Definitionen veralteter Versionen bleiben im Repository als historische Referenz erhalten, erhalten jedoch keine weiteren Aktualisierungen
