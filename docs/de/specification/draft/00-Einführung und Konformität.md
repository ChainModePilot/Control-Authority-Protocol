# Kapitel 0: Einführung und Konformität

## 0.1 Dokumentstatus

Dieses Dokument ist die Entwurfsversion der normativen technischen Spezifikation des Control Authority Protocol (CAP). Die Entwurfsversion verspricht keine Abwärtskompatibilität und kann vor der formalen Veröffentlichung beliebige Änderungen erfahren. Sobald dieser Entwurf stabilisiert ist und vollständige Testverifizierung besteht, wird er als erste formale Version unter `specification/2025-10-25/` veröffentlicht.

Dieses Dokument ist zur gemeinsamen Verwendung mit dem CAP-Architekturplan (`docs/de/blueprint/`) bestimmt:

- Der **Architekturplan** beantwortet „was, warum und wofür" — er definiert die Designabsicht, Fähigkeitsgrenzen und Kernmechanismen des Protokolls
- Die **Spezifikation** beantwortet „wie und wie zu verifizieren" — sie definiert die Nachrichtenformate, Ablaufschritte, Fehlerbehandlung und Konformitätsanforderungen des Protokolls

Wenn nicht-normative Beschreibungen im Architekturplan mit normativen Bestimmungen in dieser Spezifikation in Konflikt stehen, hat diese Spezifikation Vorrang.

## 0.2 Geltungsbereich

Diese Spezifikation definiert die technischen Details des CAP-Protokolls v1 und deckt die 6 Kernfähigkeiten ab, die in Abschnitt 3.1 von Kapitel 3 des Architekturplans aufgelistet sind:

1. Ausstellung, Speicherung, Validierung, Widerruf und Erneuerung der Offline-Autorisierung (Authorization_Descriptor)
2. Ausstellung, Abfrage und Konvertierung in Offline-Autorisierung von Online-Tickets (Trusted_Ticket)
3. Vollständiger Lebenszyklus der Sitzungsverwaltung (Session)
4. Drei Richtlinientypen und Atomaritätsgarantien der Kontrollübergabestrategie (Handover_Policy)
5. Doppelbestimmungsmechanismus der Aktivitätserkennung (Liveness_Detection)
6. Lese-Schreib-Sperrmodell des Ressourcenzugriffsmodus (Resource_Access_Mode)

Diese Spezifikation schließt explizit die in Abschnitt 3.2 von Kapitel 3 des Architekturplans aufgelisteten Funktionen aus, einschließlich endgeräteübergreifender Sitzungsmigration, Mehrgerät-kollaborative Autorisierung, Autorisierungsdelegationsketten, ABAC, Audit-Log-Standardformat, Spezifikation für die verschlüsselte Übertragung von Protokollnachrichten, verteilter Widerrufskonsens und dynamische Berechtigungserhöhung innerhalb einer Session.

## 0.3 Konformitätsterminologie

Diese Spezifikation folgt den Schlüsselwortkonventionen von RFC 2119 und RFC 8174. Die folgenden Schlüsselwörter haben normative Bedeutung, wenn sie in **Großbuchstaben** erscheinen:

- **MUST / muss**: Absolute Anforderung. Implementierungen, die diese Anforderung nicht erfüllen, entsprechen nicht dieser Spezifikation
- **MUST NOT / darf nicht**: Absolutes Verbot. Implementierungen, die dieses Verbot verletzen, entsprechen nicht dieser Spezifikation
- **SHOULD / sollte**: Starke Empfehlung. Kann aus triftigen Gründen bei vollem Verständnis der Konsequenzen abweichen
- **SHOULD NOT / sollte nicht**: Starke Abratung. Kann aus triftigen Gründen bei vollem Verständnis der Konsequenzen abweichen
- **MAY / kann**: Optional. Implementierungen können selbst entscheiden, ob sie dies bereitstellen

Dieselben Vokabeln, die nicht in Großbuchstaben erscheinen, tragen nur ihre wörtliche Bedeutung und haben keine normative Kraft.

## 0.4 Terminologieabgleich

Die in dieser Spezifikation verwendete Terminologie ist vollständig konsistent mit dem Architekturplan-Glossar `00-Glossar.md`. Wenn diese Spezifikation auf einen Begriff verweist, verwendet sie den im Architekturplan definierten Bezeichner (z. B. `Authorization_Descriptor`, `Fay`, `Terminal_Resource`).

Zur Vereinfachung der Referenz markiert diese Spezifikation Schlüsselbegriffe in **Fettschrift** beim ersten Auftreten in jedem Kapitel mit kurzen Definitionen aus dem Architekturplan-Glossar. Die vollständigen Definitionen der Begriffe richten sich nach dem Architekturplan.

## 0.5 Konformitätsstufen

Diese Spezifikation definiert 3 Implementierungskonformitätsstufen. Eine Implementierung MUSS mindestens die „Endgerät-Konformitätsstufe" erfüllen, um Konformität mit CAP v1 zu beanspruchen.

### 0.5.1 Endgerät-Konformität (Terminal Conformance)

Gilt für Endgeräte, die `Descriptor_Validator`, `Protocol_Engine` und Sitzungsverwaltungslogik implementieren. Endgeräteimplementierungen MÜSSEN:

1. Den in Kapitel 3 definierten Authorization_Descriptor-Validierungsablauf vollständig implementieren
2. Den in Kapitel 5 definierten Session-Lebenszyklus und den Liveness_Detection-Mechanismus vollständig implementieren
3. Die in Kapitel 7 definierte Resource_Access_Mode-Lese-Schreib-Sperrsemantik vollständig implementieren
4. Alle Anfragen ablehnen, die die Validierung nicht bestehen, und standardisierte Fehlercodes gemäß Kapitel 9 zurückgeben
5. Eine lokale Widerrufsliste pflegen und sie gemäß der in Kapitel 3 definierten Strategie synchronisieren, wenn sie online ist

### 0.5.2 Aussteller-Konformität (Issuer Conformance)

Gilt für vertrauenswürdige Entitäten, die `Descriptor_Issuer` oder `Ticket_Issuer` implementieren. Aussteller-Implementierungen MÜSSEN:

1. Legitime Authorization_Descriptor und Trusted_Ticket gemäß den in Kapitel 2 definierten Feldbeschränkungen erzeugen
2. Digitale Signaturen auf Anmeldeinformationen gemäß den in Kapitel 8 definierten kryptografischen Anforderungen anwenden
3. Statusaufzeichnungen ausgestellter Anmeldeinformationen pflegen und Widerrufsoperationen unterstützen
4. Die in Kapitel 4 definierte Konvertierung von Trusted_Ticket zu Authorization_Descriptor implementieren

### 0.5.3 Laufzeit-Konformität (Runtime Conformance)

Gilt für Fay-Laufzeitumgebungen, die `iFay_Runtime` implementieren. Laufzeit-Implementierungen MÜSSEN:

1. Autorisierungsvalidierungsanfragen gemäß dem in Kapitel 1 definierten Schnittstellenvertrag an Protocol_Engine übermitteln
2. Persistente Verbindungen aufrechterhalten und Anwendungsschicht-Heartbeats mit der in Kapitel 5 definierten Frequenz senden
3. Sitzungszustandsänderungs-Benachrichtigungen empfangen und weiterleiten, die von Protocol_Engine gepusht werden
4. Protocol_Engine proaktiv benachrichtigen, zugehörige Sessions freizugeben, wenn eine Fay-Instanz beendet wird

Eine Implementierung kann mehrere Konformitätsstufen gleichzeitig erfüllen. Beispielsweise kann ein integriertes Endgerät sowohl als Endgeräte-Implementierung als auch als Laufzeit-Implementierung dienen.

## 0.6 Normative Verweise

Diese Spezifikation verweist normativ auf die folgenden Dokumente:

- RFC 2119 / RFC 8174: In dieser Spezifikation verwendete Schlüsselwortkonventionen
- RFC 8949 (CBOR): Kompakte binäre Serialisierung für Authorization_Descriptor (siehe Kapitel 2)
- RFC 8032 (EdDSA): Standardalgorithmus für digitale Signaturen (siehe Kapitel 8)
- RFC 7515 (JWS): JSON-Serialisierung und Signatur für Trusted_Ticket (siehe Kapitel 4)
- RFC 5280 (X.509): Optionales Zertifikatformat (siehe Kapitel 8)

Die CAP-Schema-Definitionsdatei (`schema/{version}/schema.json`) dient als formale Ergänzung zu Kapitel 2 dieser Spezifikation mit normativer Kraft, die dieser Spezifikation entspricht. Wenn schema.json mit der textuellen Beschreibung in dieser Spezifikation in Konflikt steht, hat schema.json Vorrang.

## 0.7 Dokumentstruktur

Diese Spezifikation ist in der folgenden Reihenfolge organisiert:

| Kapitel | Thema | Hauptinhalt |
|------|------|---------|
| Kapitel 1 | Architektur und Rollen | Protokollrollen, Vertrauenskette, externe Schnittstellenverträge |
| Kapitel 2 | Datenmodell | Felddefinitionen der Kerndatenstrukturen |
| Kapitel 3 | Offline-Autorisierungsprotokoll | Vollständiger Authorization_Descriptor-Ablauf |
| Kapitel 4 | Online-Ticket-Protokoll | Vollständiger Trusted_Ticket-Ablauf und Degradation |
| Kapitel 5 | Sitzungsverwaltung und Aktivitätserkennung | Session-Zustandsmaschine und Heartbeat |
| Kapitel 6 | Kontrollübergabe-Protokoll | Drei Handover_Policy-Richtlinientypen |
| Kapitel 7 | Ressourcenzugriffsmodus | Semantik von read/write/execute/configure |
| Kapitel 8 | Kryptografie und Signaturen | Algorithmussatz, Schlüsselformate, Verteilung |
| Kapitel 9 | Fehlercodes und Konformitätsstufen | Standardfehlercodes, Stufendeklaration |
| Kapitel 10 | Sicherheitsüberlegungen | Bedrohungsmodell, bekannte Risiken |

Lesern wird empfohlen, die Kapitel 0–2 in Reihenfolge zu lesen, um eine Grundlage zu schaffen, und dann je nach Implementierungsschwerpunkt zu relevanten Kapiteln zu springen.
