# Kapitel 2: Datenmodell

Dieses Kapitel definiert die Kerndatenstrukturen im CAP-Protokoll, einschließlich Feldnamen, Typen, Beschränkungen und Standardwerten. Die Felddefinitionen in diesem Kapitel sind normativ — jede Implementierung, die dem CAP-Protokoll entspricht, MUSS diese Datenstrukturen gemäß den Definitionen in diesem Kapitel erzeugen und parsen.

`schema/{version}/schema.json` bietet eine formale Ergänzung zu den Datenstrukturen in diesem Kapitel. Wenn schema.json mit der Beschreibung in diesem Kapitel in Konflikt steht, hat schema.json Vorrang.

## 2.1 Datentyp-Konventionen

Diese Spezifikation verwendet die folgenden Basistypen zur Beschreibung von Feldern:

| Typ | Beschreibung | Codierung |
|------|------|------|
| `string` | UTF-8-Zeichenfolge | UTF-8-Bytefolge |
| `bytes` | Bytefolge | Rohbytes |
| `uint32` / `uint64` | Vorzeichenlose Ganzzahl | Big-Endian |
| `timestamp` | Unix-Zeitstempel (Sekunden) | uint64 |
| `uuid` | RFC 4122 UUID v7 | 16 Bytes |
| `enum` | Aufzählungswert | Zeichenfolgenliteral |
| `array<T>` | Geordnete Sammlung von Elementen des Typs T | Array |
| `map<K,V>` | Zuordnung von K zu V | Objekt |

Feldbeschränkungen verwenden die folgende Notation:

- `required`: Das Feld MUSS vorhanden sein, mit einem Wert ungleich null
- `optional`: Das Feld KANN vorhanden sein; Abwesenheit wird als nicht gesetzt behandelt
- `unique`: Der Feldwert ist innerhalb des Systembereichs eindeutig
- `len(N..M)`: Zeichenfolgen-/Bytelänge liegt zwischen N und M (einschließlich)
- `regex(...)`: Der Feldwert MUSS mit dem angegebenen regulären Ausdruck übereinstimmen

## 2.2 Bezeichner

Kernbezeichner im CAP-Protokoll MÜSSEN globale Eindeutigkeit erfüllen. Dieser Abschnitt definiert das Format und die Generierungsregeln verschiedener Bezeichner.

### 2.2.1 Fay_ID

`Fay_ID` identifiziert eindeutig eine Fay-Instanz.

| Attribut | Wert |
|------|------|
| Typ | `string` |
| Format | `"fay:" + uuid_v7` |
| Länge | 40 Zeichen (einschließlich Präfix) |
| Eindeutigkeit | Global eindeutig |
| Generator | Identitätsverwaltungs-Subsystem (außerhalb des CAP-Protokollbereichs) |

Beispiel: `fay:01927b34-7e21-7c4d-a89f-1234567890ab`

### 2.2.2 Terminal_ID

`Terminal_ID` identifiziert eindeutig ein Endgerät.

| Attribut | Wert |
|------|------|
| Typ | `string` |
| Format | `"terminal:" + uuid_v7` |
| Länge | 45 Zeichen (einschließlich Präfix) |
| Eindeutigkeit | Global eindeutig |
| Generator | Registration_Authority |

### 2.2.3 Resource_ID

`Resource_ID` identifiziert eine spezifische Ressource auf dem Endgerät.

| Attribut | Wert |
|------|------|
| Typ | `string` |
| Format | `terminal_id + "/" + resource_path` |
| Länge | Höchstens 256 Zeichen |
| Eindeutigkeit | Eindeutig im Endgerätebereich |
| Generator | Endgerät-Betriebssystem |

`resource_path` MUSS dem regulären Ausdruck `^[a-zA-Z0-9._\-/]+$` entsprechen.

Beispiel: `terminal:01927b34-.../device/camera/front`

### 2.2.4 Descriptor_ID

`Descriptor_ID` identifiziert eindeutig einen Authorization_Descriptor.

| Attribut | Wert |
|------|------|
| Typ | `uuid` |
| Format | UUID v7 |
| Eindeutigkeit | Global eindeutig (über alle Descriptor_Issuer hinweg) |
| Generator | Descriptor_Issuer |

Die Widerrufsliste verwendet Descriptor_ID, um widerrufene Anmeldeinformationen zu identifizieren. Descriptor_Issuer DARF eine zuvor verwendete Descriptor_ID NICHT wiederverwenden.

### 2.2.5 Session_ID

`Session_ID` identifiziert eindeutig eine aktive Sitzung.

| Attribut | Wert |
|------|------|
| Typ | `uuid` |
| Format | UUID v7 |
| Eindeutigkeit | Eindeutig im Endgerätebereich |
| Generator | Endgerät-Protocol_Engine |
| Lebenszyklus | Nur gültig, solange die Sitzung aktiv ist; die ID wird nach Beendigung der Sitzung nicht wiederverwendet |

## 2.3 Authorization_Descriptor

Authorization_Descriptor ist die Kerndatenstruktur für die Offline-Autorisierung. Ein Authorization_Descriptor besteht aus zwei Teilen: **Nutzlast (payload)** und **Signatur (signature)**.

### 2.3.1 Top-Level-Struktur

```
AuthorizationDescriptor {
  required version       : uint32
  required payload       : DescriptorPayload
  required signature     : DescriptorSignature
}
```

| Feld | Beschreibung |
|------|------|
| `version` | Protokollversionsnummer; v1-Implementierungen MÜSSEN auf `1` setzen |
| `payload` | Autorisierungsinformationsnutzlast (siehe §2.3.2) |
| `signature` | Digitale Signatur über die Nutzlast (siehe §2.3.3) |

### 2.3.2 DescriptorPayload

```
DescriptorPayload {
  required descriptor_id   : Descriptor_ID
  required issuer_id       : string
  required subject_fay_id  : Fay_ID
  required terminal_id     : Terminal_ID
  required grants          : array<Grant> (len 1..256)
  required issued_at       : timestamp
  required not_before      : timestamp
  required not_after       : timestamp
  optional grantor_id      : string
  optional metadata        : map<string, string>
}
```

| Feld | Beschränkungen | Beschreibung |
|------|------|------|
| `descriptor_id` | required, unique | Global eindeutiger Bezeichner dieser Anmeldeinformation |
| `issuer_id` | required | Aussteller-Bezeichner, entspricht Descriptor_Issuer im Schlüsselvertrauenspfad |
| `subject_fay_id` | required | Bezeichner des autorisierten Fay |
| `terminal_id` | required | Endgerät-Bezeichner, der den Autorisierungsbereich begrenzt |
| `grants` | required, 1..256 Elemente | Liste spezifischer Autorisierungselemente (siehe §2.3.4) |
| `issued_at` | required | Ausstellungszeit |
| `not_before` | required, ≥ issued_at | Beginn der Gültigkeitsdauer |
| `not_after` | required, > not_before | Ablaufzeit |
| `grantor_id` | optional | Autorisierer-Bezeichner (Natural_Person oder Official_Post) |
| `metadata` | optional | Aussteller-definierte benutzerdefinierte Metadaten ohne Auswirkungen auf die Protokollsemantik |

Implementierungen MÜSSEN Authorization_Descriptors ablehnen, die die folgenden Bedingungen nicht erfüllen:

1. `not_after - not_before > 90 Tage`: Diese Spezifikation begrenzt die maximale Gültigkeitsdauer einer einzelnen Anmeldeinformation auf 90 Tage
2. `not_before > aktuelle Zeit + 24 Stunden`: Die Ausstellung von Anmeldeinformationen mit übermäßig früher Wirksamkeitszeit ist verboten (verhindert Missbrauch durch Vorausausstellung)
3. `grants`-Array ist leer: Eine Anmeldeinformation ohne Autorisierungselemente ist bedeutungslos

### 2.3.3 DescriptorSignature

```
DescriptorSignature {
  required algorithm       : enum["ed25519", "ecdsa-p256-sha256"]
  required key_id          : string
  required signature_value : bytes
}
```

| Feld | Beschreibung |
|------|------|
| `algorithm` | Signaturalgorithmus; siehe Kapitel 8 |
| `key_id` | Bezeichner des zur Signatur verwendeten Schlüssels, entspricht dem Verification_Key-Bezeichner |
| `signature_value` | Ergebnis der Signatur der CBOR-serialisierten Nutzlast-Bytes |

Signatureingabe: Serialisieren Sie `DescriptorPayload` unter Verwendung der RFC 8949 CBOR Deterministic Encoding in eine Bytefolge und verwenden Sie diese als Eingabe für den Signaturalgorithmus.

### 2.3.4 Grant

```
Grant {
  required resource_pattern : string
  required modes            : array<AccessMode> (len 1..4)
  optional constraints      : map<string, string>
}
```

| Feld | Beschreibung |
|------|------|
| `resource_pattern` | Ressourcenübereinstimmungsmuster (siehe §2.3.5) |
| `modes` | Liste der autorisierten Zugriffsmodi; Elementtyp `AccessMode` |
| `constraints` | Zusätzliche Beschränkungen (z. B. Zeitfenster, Geofences); siehe Kapitel 7 für Auswirkungen auf die Protokollsemantik |

### 2.3.5 Ressourcenübereinstimmungsmuster

`resource_pattern` unterstützt die folgende Übereinstimmungssyntax:

- Genaue Übereinstimmung: `terminal:xxx/device/camera/front`
- Platzhalterübereinstimmung: `terminal:xxx/device/camera/*` (passt zu allen Kameras unter diesem Endgerät)
- Vollständige Endgeräteübereinstimmung: `terminal:xxx/device/camera/**` (passt zu allen Ebenen unter diesem Pfad)

Implementierungen MÜSSEN:

1. Nur die obigen drei Syntaxen unterstützen; Muster mit anderen Sonderzeichen ablehnen
2. Den Platzhalter `*` nur auf ein Pfadsegment einer Ebene abbilden
3. Den Platzhalter `**` nur am Ende des Musters zulassen

### 2.3.6 AccessMode

```
AccessMode = enum["read", "write", "execute", "configure"]
```

Die Semantik der einzelnen Zugriffsmodi wird in Kapitel 7 beschrieben.

## 2.4 Trusted_Ticket

Trusted_Ticket ist eine Online-Autorisierungs-Anmeldeinformation. Ihre Struktur basiert auf der RFC 7515 JWS Compact Serialization.

### 2.4.1 Top-Level-Struktur

Ein Trusted_Ticket ist eine JWS-Zeichenfolge, die aus drei durch `.` getrennten Teilen besteht:

```
base64url(header) . base64url(payload) . base64url(signature)
```

### 2.4.2 Header

```
TicketHeader {
  required alg : enum["EdDSA", "ES256"]
  required typ : "cap-ticket+jws"
  required kid : string
}
```

| Feld | Beschreibung |
|------|------|
| `alg` | Signaturalgorithmus (konsistent mit §8) |
| `typ` | Fester Wert `"cap-ticket+jws"`, zur Unterscheidung von Tickettypen verwendet |
| `kid` | Schlüsselbezeichner, zur Signaturverifizierung verwendet |

### 2.4.3 Payload

```
TicketPayload {
  required jti  : uuid                    // Eindeutige Ticket-ID
  required iss  : string                  // Ticket_Issuer-Bezeichner
  required sub  : Fay_ID                  // Autorisiertes Fay
  required aud  : Terminal_ID             // Ziel-Endgerät
  required iat  : timestamp               // Ausstellungszeit
  required nbf  : timestamp               // Beginn der Gültigkeitsdauer
  required exp  : timestamp               // Ablaufzeit
  required grants : array<Grant>          // Gleiche Struktur wie §2.3.4
  optional convertible : boolean (default true)  // Ob in Authorization_Descriptor konvertierbar
}
```

Implementierungen MÜSSEN Trusted_Tickets ablehnen, bei denen `exp - nbf > 7 Tage`. Die maximale Gültigkeitsdauer von Online-Tickets ist kürzer als die der Offline-Autorisierung, um sicherzustellen, dass der Online-Widerrufsmechanismus rechtzeitig wirksam werden kann.

### 2.4.4 Konvertierung von Trusted_Ticket zu Authorization_Descriptor

Wenn `convertible == true` ist, KANN das Endgerät ein Trusted_Ticket in ein lokales Authorization_Descriptor-Format konvertieren, um es für den Offline-Gebrauch zu verwenden. Konvertierungsregeln:

| TicketPayload-Feld | Zugeordnet zu DescriptorPayload-Feld |
|-------------------|-----------------------------|
| `jti` | `descriptor_id` |
| `iss` | `issuer_id` |
| `sub` | `subject_fay_id` |
| `aud` | `terminal_id` |
| `iat` | `issued_at` |
| `nbf` | `not_before` |
| `exp` | `not_after` (darf jedoch `iat + 7 Tage` NICHT überschreiten) |
| `grants` | `grants` |

Der konvertierte Authorization_Descriptor wird vom Endgerät unter Verwendung seines lokal gespeicherten Schlüssels neu signiert, wobei die Signatur `key_id` als Konvertierungsquelle markiert wird (siehe Kapitel 4). Die Signaturinformationen des ursprünglichen Trusted_Ticket MÜSSEN für Audit-Zwecke in den Metadaten beibehalten werden.

## 2.5 Session

Session ist die interne Sitzungszustandsstruktur des Endgeräts; die vollständige Struktur wird nicht in Protokollnachrichten übertragen. Dieser Abschnitt definiert die Felder von Session, um die Zustandsmaschine und die Schnittstellenkonventionen zu standardisieren.

```
Session {
  required session_id          : Session_ID
  required fay_id              : Fay_ID
  required runtime_id          : string
  required resource_id         : Resource_ID
  required access_mode         : AccessMode
  required granted_modes       : array<AccessMode>
  required state               : SessionState
  required created_at          : timestamp
  required last_heartbeat_at   : timestamp
  required credential_ref      : CredentialRef
}

SessionState = enum[
  "creating",
  "active",
  "handover_pending",
  "terminating",
  "terminated"
]

CredentialRef {
  required type   : enum["descriptor", "ticket"]
  required id     : string                     // descriptor_id oder jti
  required not_after : timestamp
}
```

Die `SessionState`-Zustandsmaschine wird in Kapitel 5 beschrieben.

## 2.6 Protokollnachrichten-Kapselung

Alle Nachrichten zwischen iFay_Runtime und Protocol_Engine teilen die folgende Kapselungsstruktur:

```
ProtocolMessage {
  required version         : uint32 (= 1)
  required message_id      : uuid
  required message_type    : string
  required timestamp       : timestamp
  required sender_id       : string
  required body            : object
  optional correlation_id  : uuid
}
```

| Feld | Beschreibung |
|------|------|
| `version` | Protokollversionsnummer; v1 setzt auf `1` |
| `message_id` | Eindeutiger Bezeichner dieser Nachricht |
| `message_type` | Nachrichtentypliteral (z. B. `"AuthRequest"`) |
| `timestamp` | Nachrichtensendezeit |
| `sender_id` | Sender-Bezeichner (runtime_id oder terminal_id) |
| `body` | Nachrichtenkörper, dessen Struktur durch message_type bestimmt wird |
| `correlation_id` | Zugehörige Anfrage-Nachrichten-ID (Antwortnachrichten MÜSSEN dieses Feld setzen) |

Die `body`-Struktur, die jedem `message_type` entspricht, wird im entsprechenden Kapitel definiert.

## 2.7 Verification_Key

Verification_Key ist der vom Endgerät gehaltene Signaturverifizierungsschlüssel.

```
VerificationKey {
  required key_id        : string
  required algorithm     : enum["ed25519", "ecdsa-p256-sha256"]
  required key_material  : bytes              // Bytes des öffentlichen Schlüssels
  required issuer_id     : string             // Aussteller-Bezeichner, der diesem Schlüssel entspricht
  required valid_from    : timestamp
  optional valid_until   : timestamp
  required source        : enum["pre-installed", "ra-distributed"]
}
```

| Feld | Beschreibung |
|------|------|
| `key_id` | Schlüsselbezeichner, entspricht DescriptorSignature.key_id |
| `algorithm` | Vom Schlüssel unterstützter Signaturalgorithmus |
| `key_material` | Rohbytes des öffentlichen Schlüssels; Codierung durch Algorithmus bestimmt (siehe Kapitel 8) |
| `issuer_id` | Descriptor_Issuer-Bezeichner, der diesem Schlüssel entspricht |
| `valid_from` | Beginn der Schlüsselgültigkeit |
| `valid_until` | Schlüsselablaufzeit; nicht gesetzt bedeutet langfristig gültig |
| `source` | Schlüsselquelle: `pre-installed` (Endgeräte-Werksinstallation) oder `ra-distributed` (Online-Verteilung durch Registration_Authority) |

Das Endgerät MUSS alle Verification_Keys sicher speichern (siehe Kapitel 8).

## 2.8 Widerrufserklärung

Eine Widerrufserklärung wird verwendet, um das Endgerät zu benachrichtigen, dass eine bestimmte Anmeldeinformation widerrufen wurde.

```
RevocationStatement {
  required version              : uint32 (= 1)
  required revocation_id        : uuid
  required target_descriptor_id : Descriptor_ID
  required issuer_id            : string
  required revoked_at           : timestamp
  optional reason               : enum["unspecified", "compromised", "superseded", "no_longer_needed"]
  required signature            : DescriptorSignature
}
```

Die Widerrufserklärung MUSS vom `issuer_id` des ursprünglichen Authorization_Descriptor ausgestellt und signiert werden.

## 2.9 Serialisierung und Übertragung

Das CAP-Protokoll verwendet die folgenden Serialisierungsformate:

| Datenstruktur | Serialisierungsformat | Zweck |
|---------|-----------|------|
| AuthorizationDescriptor | RFC 8949 CBOR (Deterministic Encoding) | Offline-Speicherung und -Übertragung |
| Trusted_Ticket | RFC 7515 JWS Compact Serialization | Online-Übertragung |
| ProtocolMessage | JSON (UTF-8) | iFay_Runtime ↔ Protocol_Engine-Interaktion |
| RevocationStatement | RFC 8949 CBOR (Deterministic Encoding) | Verteilung der Widerrufsliste |

Implementierungen KÖNNEN CBOR anstelle von JSON für die Transportschicht von ProtocolMessage verwenden, um den Overhead zu reduzieren, aber die in schema.json definierten Feldnamen und die Semantik MÜSSEN konsistent bleiben.
