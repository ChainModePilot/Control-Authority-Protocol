# Kapitel 3: Offline-Autorisierungsprotokoll

Dieses Kapitel definiert den vollständigen Lebenszyklus-Protokollablauf des `Authorization_Descriptor`, einschließlich der fünf Phasen Ausstellung, lokale Speicherung, Validierung, Widerruf und Erneuerung. Der Ablauf in diesem Kapitel entspricht der Designabsicht in §2.1 des Architekturplans.

## 3.1 Ausstellungsablauf

Der Ausstellungsablauf wird von Descriptor_Issuer ausgeführt, nachdem eine ausdrückliche Autorisierung vom Autorisierer erhalten wurde. Diese Spezifikation definiert die Formatbeschränkungen des Ausstellungsergebnisses, schreibt jedoch nicht die spezifische Interaktion zwischen dem Autorisierer und Descriptor_Issuer vor (verschiedene Bereitstellungen können unterschiedliche Formen wie Webformulare, mobile App-Autorisierung, Unternehmensautorisierungs-Verwaltungssysteme usw. annehmen).

### 3.1.1 Ausstellungseingaben

Zum Zeitpunkt der Ausstellung MUSS Descriptor_Issuer die folgenden Informationen bereits bestätigt haben:

1. Die Identität des Autorisierers (`grantor_id`) hat den eigenen Identitätsverifizierungsmechanismus von Descriptor_Issuer bestanden
2. Der Autorisierungsbereich (Ziel-`subject_fay_id`, `terminal_id`, `grants`) ist explizit vom Autorisierer angegeben
3. Die Gültigkeitsdauer (`not_before`, `not_after`) ist explizit vom Autorisierer angegeben oder folgt einer Standardrichtlinie

### 3.1.2 Ausstellungsschritte

Descriptor_Issuer MUSS Authorization_Descriptor in den folgenden Schritten erzeugen:

1. **Bezeichner generieren**: Eine neue `descriptor_id` (UUID v7) zuweisen; eine zuvor verwendete ID DARF NICHT wiederverwendet werden
2. **Nutzlast konstruieren**: `DescriptorPayload` gemäß §2.3.2 ausfüllen; alle erforderlichen Felder MÜSSEN gesetzt sein
3. **Beschränkungen validieren**: Beschränkungen wie `not_after - not_before ≤ 90 Tage` lokal validieren (siehe §2.3.2)
4. **CBOR-Serialisierung**: Die Nutzlast unter Verwendung der RFC 8949 deterministischen Codierung serialisieren
5. **Digitale Signatur**: Die Nutzlast-Bytes mit dem privaten Schlüssel von Descriptor_Issuer unter Verwendung des in §8 definierten Algorithmus signieren
6. **Struktur zusammenstellen**: Den vollständigen `AuthorizationDescriptor` konstruieren, einschließlich version, payload und signature
7. **Aufzeichnung registrieren**: Die Anmeldeinformation im internen Statusspeicher von Descriptor_Issuer registrieren (für nachfolgende Widerrufsverwaltung)

### 3.1.3 Lieferung nach der Ausstellung

Nach Abschluss der Ausstellung liefert Descriptor_Issuer Authorization_Descriptor an den Ziel-Fay. Die Liefermethode liegt außerhalb des Geltungsbereichs dieser Spezifikation, SOLLTE jedoch erfüllen:

1. Lieferung über einen verschlüsselten Kanal, um das Abfangen der Anmeldeinformation während der Übertragung zu vermeiden
2. Lieferung an die `iFay_Runtime`, zu der der Fay gehört, wobei iFay_Runtime sie im Auftrag des Fay hält

## 3.2 Lokaler Speicherablauf

Fay übermittelt Authorization_Descriptor über iFay_Runtime an das Ziel-Endgerät zur lokalen Speicherung.

### 3.2.1 Übermittlungsnachricht

iFay_Runtime sendet eine `DescriptorSubmit`-Nachricht an Protocol_Engine:

```
DescriptorSubmit (body of ProtocolMessage) {
  required descriptor : AuthorizationDescriptor
}
```

### 3.2.2 Endgeräteverarbeitung

Nach Erhalt von `DescriptorSubmit` MUSS das Endgerät es in der folgenden Reihenfolge verarbeiten:

1. **Strukturvalidierung**: Verifizieren, dass `descriptor` der in §2.3 definierten Struktur entspricht
2. **Signaturvalidierung**: Die Signatur unter Verwendung des Verification_Key verifizieren, der `key_id` entspricht (siehe §3.3.4)
3. **Duplikatprüfung**: Wenn lokal bereits eine Anmeldeinformation mit derselben `descriptor_id` gespeichert ist und der neu übermittelte Inhalt mit dem gespeicherten Inhalt byteweise übereinstimmt, Erfolg zurückgeben; andernfalls mit einem Duplikat-ID-Fehler ablehnen
4. **Speicherung**: Den Authorization_Descriptor verschlüsselt in einem lokalen sicheren Speicherbereich speichern (siehe §3.2.3)
5. **Ergebnis zurückgeben**: Eine `DescriptorSubmitResult`-Nachricht an iFay_Runtime zurückgeben, mit Erfolg oder einem Fehlercode

### 3.2.3 Speicheranforderungen

Das Endgerät MUSS:

1. Authorization_Descriptor verschlüsselt speichern; der Verschlüsselungsschlüssel DARF von unautorisierten Prozessen NICHT lesbar sein
2. Das Speichermedium SOLLTE über Anti-physische-Manipulation-Fähigkeit verfügen (z. B. Sicherheitschip, TEE)
3. Die Speicherkapazitätsobergrenze SOLLTE nicht weniger als 1024 Anmeldeinformationen betragen; bei Überschreitung der Obergrenze werden abgelaufene Anmeldeinformationen mit einer LRU-Richtlinie verdrängt

Das Endgerät DARF NICHT:

1. Authorization_Descriptor im Klartext-Format speichern
2. Vor dem Speichern irgendein Feld des descriptor ändern (einschließlich metadata)

### 3.2.4 Fehlercodes

| Fehlercode | Auslöserbedingung |
|-------|---------|
| `E_INVALID_STRUCTURE` | Struktur entspricht nicht §2.3 |
| `E_INVALID_SIGNATURE` | Signaturvalidierung fehlgeschlagen |
| `E_UNKNOWN_ISSUER` | Der Verification_Key, der `key_id` entspricht, ist auf dem Endgerät nicht registriert |
| `E_DUPLICATE_DESCRIPTOR_ID` | Konflikt mit gespeicherter descriptor_id mit nicht übereinstimmendem Inhalt |
| `E_STORAGE_FULL` | Speicherkapazität ist voll und keine Verdrängung ist möglich |
| `E_VALIDITY_OUT_OF_RANGE` | `not_after - not_before` überschreitet das Limit |

## 3.3 Validierungsablauf

Der Validierungsablauf wird bei jeder Ressourcenzugriffsanfrage ausgeführt. Dieser Abschnitt definiert die vollständigen Schritte und Entscheidungsregeln für die Validierung.

### 3.3.1 Validierungsauslöser

Wenn iFay_Runtime einen `AuthRequest` sendet, der Ressourcenzugriff anfordert, löst die Endgerät-Protocol_Engine den Validierungsablauf aus:

```
AuthRequest (body of ProtocolMessage) {
  required fay_id        : Fay_ID
  required resource_id   : Resource_ID
  required access_mode   : AccessMode
  required credential    : CredentialReference
}

CredentialReference {
  required type : enum["descriptor", "ticket"]
  required id   : string                      // descriptor_id oder jti
}
```

Hinweis: Im Offline-Autorisierungsszenario dieser Spezifikation verweist `credential` auf einen Authorization_Descriptor, der bereits auf dem Endgerät gespeichert ist; die vollständige Anmeldeinformation muss nicht in der Anfrage übertragen werden.

### 3.3.2 Validierungsschritte (in Reihenfolge ausgeführt)

Das Endgerät MUSS die Validierung in der folgenden Reihenfolge durchführen. Ein Fehler bei einem Schritt gibt sofort den entsprechenden Fehlercode zurück, ohne mit nachfolgenden Schritten fortzufahren.

#### Schritt 1: Existenz der Anmeldeinformation

Das Endgerät sucht im lokalen Speicher nach dem Authorization_Descriptor, der `credential.id` entspricht.

- Nicht gefunden → `E_DESCRIPTOR_NOT_FOUND` zurückgeben

#### Schritt 2: Widerrufsstatus

Das Endgerät überprüft die lokale Widerrufsliste, um zu bestätigen, dass die `descriptor_id` nicht widerrufen wurde.

- Widerrufen → `E_DESCRIPTOR_REVOKED` zurückgeben

#### Schritt 3: Gültigkeitsdauer

Das Endgerät überprüft, ob die aktuelle Zeit innerhalb des Intervalls `[not_before, not_after]` liegt.

- Aktuelle Zeit < `not_before` → `E_DESCRIPTOR_NOT_YET_VALID` zurückgeben
- Aktuelle Zeit ≥ `not_after` → `E_DESCRIPTOR_EXPIRED` zurückgeben

Endgerät-Uhrenquelle: MUSS eine kalibrierte Systemuhr verwenden. Das Endgerät SOLLTE Validierungsanfragen vorsichtig behandeln, wenn es längere Zeit nicht online war (siehe §3.6 Behandlung von Uhrendrift).

#### Schritt 4: Subjektübereinstimmung

Das Endgerät verifiziert, dass `payload.subject_fay_id == AuthRequest.fay_id`.

- Nicht übereinstimmend → `E_SUBJECT_MISMATCH` zurückgeben

#### Schritt 5: Endgeräteübereinstimmung

Das Endgerät verifiziert, dass `payload.terminal_id == aktuelle Endgerät-ID`.

- Nicht übereinstimmend → `E_TERMINAL_MISMATCH` zurückgeben

#### Schritt 6: Ressourcen- und Modusübereinstimmung

Das Endgerät iteriert über `payload.grants` und sucht nach einem `Grant`, der erfüllt:

1. `Grant.resource_pattern` passt zu `AuthRequest.resource_id` gemäß §2.3.5-Regeln
2. `Grant.modes` enthält `AuthRequest.access_mode`
3. Wenn `Grant.constraints` nicht leer ist, sind alle Beschränkungen erfüllt (siehe §7.4 für Beschränkungssemantik)

- Kein passender Grant gefunden → `E_AUTHORIZATION_INSUFFICIENT` zurückgeben

#### Schritt 7: Signaturvalidierung

Das Endgerät validiert die Signatur der Nutzlast unter Verwendung des Verification_Key, der `signature.key_id` entspricht, erneut.

Hinweis: Die schnellen Prüfungen der Schritte 1–6 werden vor der Signaturvalidierung abgeschlossen, um eine vernünftige Fehlersemantik bereitzustellen (z. B. „Anmeldeinformation widerrufen" vor „Signatur fehlgeschlagen" zu melden). Bevor die Validierung jedoch besteht und Erfolg zurückgegeben wird, MUSS die Signaturvalidierung ausgeführt worden sein und bestanden haben.

- Signaturvalidierung fehlgeschlagen → `E_INVALID_SIGNATURE` zurückgeben
- Signierschlüssel widerrufen oder abgelaufen → `E_VERIFICATION_KEY_INVALID` zurückgeben

### 3.3.3 Nachdem die Validierung besteht

Nachdem alle 7 Schritte bestanden sind, erstellt das Endgerät eine Session gemäß dem Ablauf in Kapitel 5 und gibt an iFay_Runtime zurück:

```
AuthResult (body of ProtocolMessage, success) {
  required status         : "granted"
  required session_id     : Session_ID
  required granted_modes  : array<AccessMode>
  required session_expires_at : timestamp
}
```

`session_expires_at` SOLLTE gleich `min(not_after, aktuelle Zeit + maximale Standard-Sitzungsdauer)` sein.

### 3.3.4 Details der Signaturvalidierung

Spezifische Schritte der Signaturvalidierung:

1. `descriptor.signature.key_id` extrahieren und den entsprechenden öffentlichen Schlüssel im Verification_Key-Speicher des Endgeräts suchen
2. Verifizieren, dass der Verification_Key zur aktuellen Zeit gültig ist (`valid_from ≤ aktuelle Zeit ≤ valid_until`, falls valid_until gesetzt ist)
3. `descriptor.payload` gemäß §2.3.3-Regeln erneut CBOR-serialisieren
4. Die Signatur der serialisierten Bytes mit dem Verification_Key und `descriptor.signature.algorithm` verifizieren
5. Nach erfolgreicher Validierung MUSS das Ergebnis zwischengespeichert werden (derselbe descriptor erfordert nur einmal eine Signaturvalidierung während seines Lebenszyklus)

## 3.4 Widerrufsablauf

### 3.4.1 Widerrufsinitialisierung

Der Autorisierer initiiert eine Widerrufsanfrage über Descriptor_Issuer. Die spezifische Interaktionsform für Widerrufsanfragen liegt außerhalb des Geltungsbereichs dieser Spezifikation.

Nach Erhalt einer Widerrufsanfrage MUSS Descriptor_Issuer:

1. Verifizieren, dass die Widerrufsanfrage vom ursprünglichen Autorisierer oder einer Entität mit Widerrufsbefugnis stammt
2. Eine `RevocationStatement` gemäß §2.8 erzeugen
3. Die Widerrufserklärung mit demselben Signierschlüssel signieren, der für den ursprünglichen Authorization_Descriptor verwendet wurde
4. Die Widerrufserklärung der von Descriptor_Issuer gepflegten Widerrufsliste hinzufügen

### 3.4.2 Widerrufsverteilung

Widerrufserklärungen werden über die folgenden Methoden an Endgeräte verteilt; das Endgerät MUSS mindestens zwei davon unterstützen:

1. **Pull-basierte Synchronisation**: Wenn online, ruft das Endgerät proaktiv inkrementelle Widerrufslisten von Descriptor_Issuer oder einem Widerrufsdienst ab. Die Synchronisationsfrequenz wird durch die Endgeräterichtlinie bestimmt und SOLLTE nicht weniger als einmal pro Stunde betragen (während Online-Zeiträumen)
2. **Push-basierte Benachrichtigung**: Descriptor_Issuer pusht Widerrufserklärungen über Registration_Authority oder einen dedizierten Widerrufsverteilungsdienst an das Endgerät
3. **In-Band-Einbettung**: Eine Zusammenfassung der jüngsten Widerrufsliste wird in den Trusted_Ticket-Metadaten transportiert, sodass online erhaltene Tickets automatisch Widerrufsinformationen mitführen

### 3.4.3 Wartung der Endgeräte-Widerrufsliste

Die lokale Widerrufsliste des Endgeräts MUSS:

1. Alle nicht abgelaufenen Widerrufserklärungen dauerhaft speichern
2. Nachdem die `not_after`-Zeit einer Anmeldeinformation verstrichen ist, KANN die entsprechende Widerrufserklärung gelöscht werden (die Anmeldeinformation ist natürlich abgelaufen)
3. Die Signatur jeder Widerrufserklärung validieren und Widerrufserklärungen mit ungültigen Signaturen ablehnen

### 3.4.4 Widerrufswirksamkeitszeit

Die Wirksamkeitszeit einer Widerrufserklärung ist:

```
Wirksamkeitszeit = max(Zeit, zu der die Widerrufserklärung das Endgerät erreicht, RevocationStatement.revoked_at)
```

Das Endgerät MUSS sicherstellen, dass nachfolgende Validierungsanfragen, nachdem eine Widerrufserklärung das Endgerät erreicht hat, die Anmeldeinformation sofort ablehnen. Das Endgerät KANN proaktiv prüfen, ob alle aktiven Sessions auf widerrufene Anmeldeinformationen verweisen, und falls ja, diese Sessions gemäß den Regeln in Kapitel 5 zwangsweise beenden.

### 3.4.5 Widerrufsverzögerungsfenster

Aufgrund der inhärenten Verzögerung der Offline-Verteilung existieren die folgenden unvermeidlichen Widerrufsverzögerungsfenster:

| Szenario | Maximale Verzögerung |
|------|---------|
| Endgerät kontinuierlich online | Ein Synchronisationszyklus (Standard ≤ 1 Stunde) |
| Endgerät kurzzeitig offline | Nächste Synchronisation nach Wiederverbindung |
| Endgerät langfristig offline | Maximale Verzögerung ist `not_after - Widerrufszeit` der Anmeldeinformation, jedoch nicht mehr als 90 Tage (die maximale Anmeldeinformationsgültigkeitsdauer) |

Aussteller SOLLTEN die maximale Widerrufsverzögerung begrenzen, indem sie kürzere `not_after`-Werte (z. B. 7 Tage) festlegen.

## 3.5 Erneuerungsablauf

Erneuerung ist im Wesentlichen die Ausstellung eines neuen Authorization_Descriptor, um die alte Version zu ersetzen, nicht eine Modifikation der bestehenden Anmeldeinformation.

### 3.5.1 Erneuerungsrichtlinie

Der Autorisierer oder ein automatischer Erneuerungsmechanismus kann eine Erneuerung in den folgenden Szenarien auslösen:

1. Die Anmeldeinformation nähert sich `not_after` und die Autorisierungsbeziehung ist immer noch gültig
2. Der Autorisierungsbereich muss angepasst werden (Hinzufügen/Entfernen von grants)
3. Die Autorisierungsbeschränkungen müssen angepasst werden (Modifikation von constraints)

### 3.5.2 Erneuerungsablauf

Erneuerungsprozess:

1. Descriptor_Issuer stellt einen neuen Authorization_Descriptor gemäß dem §3.1-Ablauf aus, unter Verwendung einer neuen `descriptor_id`
2. Die neue Anmeldeinformation wird gemäß dem §3.2-Ablauf an das Endgerät übermittelt und gespeichert
3. Die alte Anmeldeinformation KANN proaktiv vom Aussteller widerrufen werden (gemäß §3.4) oder KANN natürlich ablaufen

Während des Zeitraums, in dem sowohl neue als auch alte Anmeldeinformationen koexistieren, sind die Verarbeitungsprioritäten des Endgeräts:

- Validierungsanfragen MÜSSEN bevorzugt nicht abgelaufene Anmeldeinformationen abgleichen
- Wenn mehrere nicht abgelaufene Anmeldeinformationen alle übereinstimmen, wird die Anmeldeinformation mit dem jüngsten `issued_at` verwendet

### 3.5.3 Nicht erlaubte Erneuerungsverhalten

Implementierungen DÜRFEN NICHT:

1. Irgendein Feld einer ausgegebenen Anmeldeinformation modifizieren (einschließlich metadata)
2. Die `descriptor_id` einer alten Anmeldeinformation wiederverwenden
3. Die Semantik einer alten Anmeldeinformation modifizieren, während sie noch gültig ist (muss durch Widerruf + neue Ausstellung erfolgen)

## 3.6 Behandlung von Uhrendrift

Endgeräteuhren können aufgrund langfristiger Offline-Operation oder Hardwareproblemen driften. Dieser Abschnitt definiert die Behandlungsregeln unter Uhrendriftbedingungen.

### 3.6.1 Uhrentoleranz

Das Endgerät KANN bei der Validierung der Gültigkeitsdauer Toleranz einführen:

- Für `not_before`: Kann bis zu 5 Minuten frühe Toleranz zulassen (akzeptiert `aktuelle Zeit ≥ not_before - 5min`)
- Für `not_after`: DARF KEINE Toleranz einführen (abgelaufen ist abgelaufen)

Toleranz wird nur verwendet, um kurzfristige Uhrenversätze auszugleichen, und SOLLTE NICHT zur Umgehung von Gültigkeitsdauerbeschränkungen verwendet werden.

### 3.6.2 Uhrenvertrauenswürdigkeit

Das Endgerät SOLLTE die folgenden Uhrenanomalien erkennen und Schutzmaßnahmen ergreifen:

- Signifikante Uhrenrückwärtsverschiebung (Systemuhr springt > 1 Stunde in die Vergangenheit): Validierungsanfragen ablehnen, bis die Uhr synchronisiert ist
- Signifikante Uhrenvorwärtsverschiebung (Systemuhr springt > 24 Stunden in die Zukunft): Validierungsanfragen ablehnen, bis die Uhr synchronisiert ist

## 3.7 Vollständiges Ablaufdiagramm

```
[Descriptor_Issuer]                    [Fay/iFay_Runtime]                   [Terminal/Protocol_Engine]
        │                                       │                                       │
        │── AuthorizationDescriptor ausstellen→│                                       │
        │   (mit digitaler Signatur)            │                                       │
        │                                       │                                       │
        │                                       │── DescriptorSubmit ─────────────────→│
        │                                       │                                       │── Signatur verifizieren + speichern
        │                                       │←─ DescriptorSubmitResult ──────────── │
        │                                       │                                       │
        │                                       │── AuthRequest (mit descriptor_id) ──→│
        │                                       │                                       │── 7-Schritt-Validierung
        │                                       │←─ AuthResult (granted) ──────────────│
        │                                       │                                       │── Session erstellen
        │                                       │                                       │
        │                                       │       【Fay greift kontinuierlich auf Ressource zu】│
        │                                       │                                       │
        │── RevocationStatement widerrufen ─────────────────────────────────────────────→│
        │                                       │                                       │── Zur Widerrufsliste hinzufügen
        │                                       │                                       │── Verbundene Session zwangsweise beenden
```
