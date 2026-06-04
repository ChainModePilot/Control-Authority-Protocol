# Kapitel 9: Fehlercodes und Konformitätsstufen

Dieses Kapitel definiert die standardisierte Fehlercodetabelle des CAP-Protokolls und die Art und Weise, wie Implementierungen Konformitätsstufen deklarieren.

## 9.1 Fehlercode-Format

Fehlercodes verwenden das Präfix `E_` mit Großbuchstaben-Unterstrich-Benennung. Die Fehlercode-Zeichenfolge selbst ist normativ — Implementierungen dieser Spezifikation MÜSSEN die entsprechende Fehlercode-Zeichenfolge gemäß den Definitionen in diesem Kapitel zurückgeben, um eine konsistente Diagnose zwischen Implementierungen zu ermöglichen.

Fehlerantwort-Nachrichtenstruktur:

```
ErrorBody {
  required error_code : string
  required message    : string
  optional details    : map<string, string>
  optional retry_after_ms : uint32
}
```

| Feld | Beschreibung |
|------|------|
| `error_code` | In diesem Kapitel definierter Standard-Fehlercode |
| `message` | Menschenlesbare Fehlerbeschreibung (Sprache nicht festgelegt; Englisch oder Deutsch empfohlen) |
| `details` | Zusätzlicher Kontext für den Fehler (z. B. das spezifische verletzte Feld) |
| `retry_after_ms` | Empfohlene Wartezeit für die Wiederholung des Clients; nur für vorübergehende Fehler sinnvoll |

## 9.2 Anmeldeinformations-bezogene Fehlercodes

Gilt für Validierungsfehlerszenarien für Authorization_Descriptor und Trusted_Ticket.

| Fehlercode | HTTP-Äquivalent | Auslösebedingung | Konformität |
|-------|----------|---------|--------|
| `E_DESCRIPTOR_NOT_FOUND` | 404 | Die referenzierte descriptor_id wurde lokal auf dem Endgerät nicht gefunden | Endgerät MUSS |
| `E_INVALID_STRUCTURE` | 400 | Anmeldeinformationsstruktur entspricht nicht §2.3 / §2.4 | Endgerät/Aussteller MUSS |
| `E_INVALID_SIGNATURE` | 401 | Signaturvalidierung fehlgeschlagen | Endgerät MUSS |
| `E_VERIFICATION_KEY_INVALID` | 401 | Der Verification_Key, der der Signatur-key_id entspricht, ist nicht registriert oder widerrufen | Endgerät MUSS |
| `E_UNKNOWN_ISSUER` | 401 | Die issuer_id der Anmeldeinformation befindet sich nicht in der Vertrauensliste des Endgeräts | Endgerät MUSS |
| `E_DESCRIPTOR_REVOKED` | 403 | Authorization_Descriptor wurde widerrufen | Endgerät MUSS |
| `E_TICKET_REVOKED` | 403 | Trusted_Ticket wurde widerrufen | Endgerät MUSS |
| `E_DESCRIPTOR_EXPIRED` | 403 | Anmeldeinformation hat `not_after` überschritten | Endgerät MUSS |
| `E_DESCRIPTOR_NOT_YET_VALID` | 403 | Aktuelle Zeit früher als `not_before` | Endgerät MUSS |
| `E_TICKET_MALFORMED` | 400 | JWS-Parsing fehlgeschlagen oder typ falsch | Endgerät MUSS |
| `E_DUPLICATE_DESCRIPTOR_ID` | 409 | Übermittelte Anmeldeinformation steht im Konflikt mit gespeicherter Anmeldeinformations-ID mit nicht übereinstimmendem Inhalt | Endgerät MUSS |
| `E_VALIDITY_OUT_OF_RANGE` | 400 | `not_after - not_before` der Anmeldeinformation überschreitet das Limit | Endgerät/Aussteller MUSS |
| `E_REVOCATION_QUERY_TIMEOUT` | 504 | Online-Widerrufsabfrage-Timeout | Endgerät SOLLTE |

## 9.3 Autorisierungsbereich-Fehlercodes

Gilt für Szenarien, in denen die Anmeldeinformation legitim ist, der Autorisierungsbereich jedoch nicht übereinstimmt.

| Fehlercode | HTTP-Äquivalent | Auslösebedingung |
|-------|----------|---------|
| `E_SUBJECT_MISMATCH` | 403 | Anmeldeinformations-Subjekt stimmt nicht mit Anforderer-fay_id überein |
| `E_TERMINAL_MISMATCH` | 403 | Anmeldeinformations-terminal_id stimmt nicht mit aktuellem Endgerät überein |
| `E_AUTHORIZATION_INSUFFICIENT` | 403 | Kein passender Grant für (resource, mode) |
| `E_RATE_LIMIT_EXCEEDED` | 429 | Überschreitet das Frequenzlimit in constraints |
| `E_OUT_OF_TIME_WINDOW` | 403 | Aktuelle Zeit liegt außerhalb des constraints-Zeitfensters |
| `E_OUT_OF_GEO_FENCE` | 403 | Endgerätestandort liegt außerhalb des constraints-Geofence |
| `E_UNSUPPORTED_CONSTRAINT` | 400 | Anmeldeinformation enthält nicht erkannten constraint-Schlüssel |

## 9.4 Ressourcen- und Sitzungsfehlercodes

Gilt für Fehler im Zusammenhang mit Ressourcenbelegung und Session-Zustand.

| Fehlercode | HTTP-Äquivalent | Auslösebedingung |
|-------|----------|---------|
| `E_RESOURCE_BUSY` | 409 | Ressource bereits im angeforderten Modus belegt (verletzt Lese-Schreib-Sperrmatrix) |
| `E_RESOURCE_UNAVAILABLE` | 503 | Ressource derzeit nicht verfügbar (Hardware-Trennung, Software läuft nicht usw.) |
| `E_RESOURCE_NOT_FOUND` | 404 | resource_id existiert nicht |
| `E_SESSION_NOT_FOUND` | 404 | session_id existiert nicht |
| `E_SESSION_TERMINATED` | 410 | Session beendet, kann Operation nicht fortsetzen |
| `E_SESSION_LIMIT_EXCEEDED` | 429 | Überschreitet das in §5.8 definierte Session-Mengenlimit |
| `E_OS_INTEGRATION_FAILED` | 500 | OS-Zugriffskontrollbereitstellung fehlgeschlagen |

## 9.5 Übergabe-bezogene Fehlercodes

Gilt für den in Kapitel 6 definierten Übergabeablauf.

| Fehlercode | HTTP-Äquivalent | Auslösebedingung |
|-------|----------|---------|
| `E_HANDOVER_INVALID_SOURCE` | 400 | source_session_id existiert nicht oder Zustand erlaubt keine Übergabe |
| `E_HANDOVER_INVALID_TARGET` | 400 | target_credential-Validierung fehlgeschlagen |
| `E_HANDOVER_REJECTED_BY_POLICY` | 403 | Richtlinienbewertung lehnt Übergabe ab |
| `E_HANDOVER_FAILED_AT_RELEASE` | 500 | Quell-Session-Freigabe während der Übergabe fehlgeschlagen |
| `E_HANDOVER_FAILED_AT_ACQUIRE` | 500 | Ziel-Session-Erstellung während der Übergabe fehlgeschlagen |
| `E_HANDOVER_TIMEOUT` | 504 | Übergabe-Timeout |
| `E_HANDOVER_IN_PROGRESS` | 409 | Ressource hat eine laufende Übergabe; neue Anfrage abgelehnt |
| `E_HANDOVER_RETRY_LIMIT` | 429 | Wiederholungsanzahl überschreitet Limit |

## 9.6 Protokoll- und Systemfehlercodes

| Fehlercode | HTTP-Äquivalent | Auslösebedingung |
|-------|----------|---------|
| `E_PROTOCOL_VERSION_UNSUPPORTED` | 400 | Nachrichten-version-Feld wird von der Implementierung nicht unterstützt |
| `E_INVALID_MESSAGE` | 400 | Protokollnachrichtenformat entspricht nicht §2.6 |
| `E_INTERNAL_ERROR` | 500 | Interner Endgerätefehler (legt keine Details offen) |
| `E_NOT_IMPLEMENTED` | 501 | Implementierung bietet die optionale Fähigkeit nicht |
| `E_STORAGE_FULL` | 507 | Endgeräte-Anmeldeinformationsspeicher voll |

## 9.7 Fehlercode-Erweiterung

Implementierungen KÖNNEN benutzerdefinierte Fehlercodes definieren, MÜSSEN jedoch:

1. Das Benennungsformat `E_VENDOR_<vendor_name>_<code>` verwenden (z. B. `E_VENDOR_ACME_QUOTA_EXCEEDED`)
2. Nicht mit in diesem Kapitel definierten Standard-Fehlercodes in Konflikt stehen
3. Die Semantik in der Implementierungsdokumentation klar angeben

iFay_Runtime SOLLTE nicht erkannte benutzerdefinierte Fehlercodes als `E_INTERNAL_ERROR` behandeln.

## 9.8 Konformitätsstufen-Deklaration

Implementierungen MÜSSEN die von ihnen erfüllten Konformitätsstufen über das Folgende deklarieren.

### 9.8.1 Konformitätsanspruch-Struktur

```
ConformanceClaim {
  required protocol_version : string                  // Z. B. "1.0.0" oder "draft"
  required levels           : array<ConformanceLevel> (len 1..3)
  optional optional_features : array<string>
  optional vendor_extensions : array<string>
}

ConformanceLevel = enum["terminal", "issuer", "runtime"]
```

| Feld | Beschreibung |
|------|------|
| `protocol_version` | Von der Implementierung befolgte CAP-Protokollversion |
| `levels` | Von der Implementierung erfüllte Konformitätsstufen (siehe §0.5) |
| `optional_features` | Von der Implementierung unterstützte optionale Fähigkeiten (z. B. `aggregated_heartbeat`, `ai_model_handover_policy`) |
| `vendor_extensions` | Von der Implementierung unterstützte Anbietererweiterungsfähigkeiten |

### 9.8.2 Deklarations-Veröffentlichungskanäle

Implementierungen SOLLTEN ConformanceClaim über die folgenden Kanäle veröffentlichen:

1. Explizite Deklaration in der Produktdokumentation
2. Rückgabe über bestimmte Abfrageschnittstellen (definiert durch die Implementierung)
3. Deklariert in den SDK-Paketmetadaten

### 9.8.3 Konformitätstests

Die Konformitätstest-Suite (Conformance Test Suite) des CAP-Protokolls wird vom Projekt als unterstützendes Asset gepflegt, liegt jedoch außerhalb des normativen Geltungsbereichs dieser Spezifikation. Jede Implementierung SOLLTE die Testsuite für die entsprechende Konformitätsstufe bestehen, DARF jedoch NICHT auf die Testsuite als alleinige Garantie für Korrektheit angewiesen sein.

## 9.9 Fehlercode-Verwendungsempfehlungen

Implementierungen SOLLTEN den folgenden Fehlercode-Verwendungsprinzipien folgen:

1. **Bevorzugen Sie den spezifischsten Fehlercode**: Beispielsweise wird `E_TICKET_REVOKED` `E_TICKET_MALFORMED` vorgezogen (auch wenn beide während der Analyse erscheinen könnten)
2. **Legen Sie keine internen Details über Fehlercodes offen**: Vermeiden Sie es, Schlüsselfingerabdrücke, interne IDs und andere sensible Informationen im error message-Feld offenzulegen
3. **Stellen Sie diagnostische Unterstützung über das details-Feld bereit**: Beispielsweise zeigt `details["field"] = "not_after"` an, welches spezifische Feld nicht konform ist
4. **Verwenden Sie retry_after_ms für vorübergehende Fehler**: Beispielsweise `E_RESOURCE_BUSY`, `E_HANDOVER_IN_PROGRESS`

## 9.10 Synchronisierung von Fehlercodes mit schema.json

`schema/{version}/schema.json` MUSS die vollständige Fehlercode-Aufzählungsdefinition enthalten und mit diesem Kapitel konsistent gehalten werden. Wenn die Fehlercodeliste in diesem Kapitel aktualisiert wird, MUSS schema.json synchron aktualisiert werden.
