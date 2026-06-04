# Kapitel 4: Online-Ticket-Protokoll

Dieses Kapitel definiert die Ausstellungs-, Abfrage-, Konvertierungs- und Degradationsabläufe von `Trusted_Ticket`. Der Ablauf in diesem Kapitel entspricht der Designabsicht in §2.2 des Architekturplans.

## 4.1 Anwendungsvoraussetzungen

Trusted_Ticket wird nur verwendet, wenn das Endgerät online auf Ticket_Issuer oder einen Online-Widerrufsdienst zugreifen kann. Wenn der Online-Dienst nicht verfügbar ist, fällt das Endgerät automatisch gemäß §4.5 auf die Offline-Autorisierung zurück.

Das Endgerät MUSS gleichzeitig sowohl Authorization_Descriptor (Kapitel 3) als auch Trusted_Ticket unterstützen. Das Endgerät DARF die Annahme legitim signierter Trusted_Tickets NICHT ablehnen, selbst wenn seine Richtlinie die Offline-Autorisierung bevorzugt.

## 4.2 Ausstellungsablauf

### 4.2.1 Ausstellungsanfrage

Der Autorisierer initiiert eine Ausstellungsanfrage über einen CAP-unterstützenden Client (z. B. mobile App, Web-Konsole) an Ticket_Issuer. Die Interaktion zwischen dem Client und Ticket_Issuer liegt außerhalb des Geltungsbereichs dieser Spezifikation, aber Ticket_Issuer MUSS:

1. Verifizieren, dass die Anfrage von einem authentifizierten Autorisierer stammt
2. Verifizieren, dass der Autorisierer berechtigt ist, Zugriff auf den Ziel-Fay und die Ressource zu gewähren
3. Lokale Richtlinie anwenden, um den Autorisierungsbereich zu validieren (z. B. maximale Gültigkeit, Ressourcen-Whitelist)

### 4.2.2 Ausstellungsschritte

Ticket_Issuer MUSS Trusted_Ticket in den folgenden Schritten erzeugen:

1. **Header konstruieren**: `alg`, `typ`, `kid` gemäß §2.4.2 setzen
2. **Payload konstruieren**: TicketPayload gemäß §2.4.3 ausfüllen; alle erforderlichen Felder MÜSSEN gesetzt sein
3. **Beschränkungen validieren**: `exp - nbf ≤ 7 Tage` lokal validieren
4. **JSON-Serialisierung**: Header und Payload in UTF-8-JSON-Bytes serialisieren
5. **Base64URL-Codierung**: Die Header- und Payload-Bytes separat Base64URL-codieren
6. **Signieren**: Die Signatureingabe `base64url(header) + "." + base64url(payload)` gemäß RFC 7515 berechnen
7. **Compact Serialization erzeugen**: Als `header.payload.signature`-Zeichenfolge verketten
8. **Aufzeichnung registrieren**: Den ausgestellten Ticketstatus intern in Ticket_Issuer pflegen (für Widerrufsabfragen)

### 4.2.3 Lieferung nach der Ausstellung

Trusted_Ticket wird über einen verschlüsselten Kanal wie HTTPS an den Ziel-Fay oder seine iFay_Runtime geliefert. Die spezifische Liefermethode liegt außerhalb des Geltungsbereichs dieser Spezifikation.

## 4.3 Endgerät-Validierungsablauf

Die Unterschiede zwischen der Validierung von Trusted_Ticket und Authorization_Descriptor durch das Endgerät sind:

1. Trusted_Ticket wird zum Anfragezeitpunkt vollständig von iFay_Runtime an das Endgerät übertragen (nicht vorab gespeichert)
2. Das Endgerät KANN den Widerrufsstatus von Ticket_Issuer in Echtzeit abfragen, wenn es online ist
3. Fehlercodes bei Validierungsfehler tragen das Präfix `E_TICKET_*`

### 4.3.1 Validierungsanfrage

Der von iFay_Runtime gesendete AuthRequest trägt das vollständige Trusted_Ticket:

```
AuthRequest (body of ProtocolMessage) {
  required fay_id        : Fay_ID
  required resource_id   : Resource_ID
  required access_mode   : AccessMode
  required credential    : CredentialContent
}

CredentialContent {
  required type : enum["descriptor_ref", "ticket"]
  oneof:
    descriptor_id : string                  // type == "descriptor_ref"
    ticket        : string                  // type == "ticket", JWS Compact-Zeichenfolge
}
```

### 4.3.2 Validierungsschritte (in Reihenfolge ausgeführt)

Das Endgerät MUSS die Validierung in der folgenden Reihenfolge durchführen. Ein Fehler bei einem Schritt gibt sofort einen Fehler zurück.

#### Schritt 1: JWS-Parsing

Das Endgerät parst die JWS Compact-Zeichenfolge:

- In die drei Teile `header.payload.signature` aufteilen
- Header und Payload Base64URL-decodieren
- Verifizieren, dass `header.typ == "cap-ticket+jws"`
- Verifizieren, dass `header.alg` im durch §8 erlaubten Algorithmus-Set enthalten ist

Fehler bei einem Schritt → `E_TICKET_MALFORMED` zurückgeben

#### Schritt 2: Signaturvalidierung

Das Endgerät verifiziert die JWS-Signatur unter Verwendung des Verification_Key, der `header.kid` entspricht (gemäß RFC 7515).

- Signaturvalidierung fehlgeschlagen → `E_INVALID_SIGNATURE` zurückgeben
- Der Schlüssel, der `kid` entspricht, ist nicht registriert oder widerrufen → `E_VERIFICATION_KEY_INVALID` zurückgeben

#### Schritt 3: Gültigkeitsdauer

`nbf` und `exp` gemäß den Regeln in Schritt 3 von §3.3.2 validieren.

#### Schritt 4: Subjekt-, Endgerät-, Ressourcen- und Modusübereinstimmung

Übereinstimmung gemäß den Regeln in Schritten 4–6 von §3.3.2 durchführen. Fehlercodes verwenden das Präfix `E_TICKET_*` (siehe §9).

#### Schritt 5: Online-Widerrufsabfrage (Optional)

Wenn die Endgeräterichtlinie eine Online-Widerrufsverifizierung erfordert, KANN das Endgerät eine Widerrufsstatusabfrage an Ticket_Issuer initiieren:

```
TicketRevocationQuery {
  required jti : uuid
}

TicketRevocationResponse {
  required jti      : uuid
  required revoked  : boolean
  optional revoked_at : timestamp
}
```

- Abfrage-Timeout (Standard 2 Sekunden) → das Endgerät SOLLTE die Anfrage ablehnen und `E_REVOCATION_QUERY_TIMEOUT` zurückgeben, KANN jedoch so konfiguriert werden, dass es durchgelassen wird (Akzeptanz des Widerrufsverzögerungsrisikos)
- Abfrage gibt `revoked == true` zurück → `E_TICKET_REVOKED` zurückgeben
- Abfragefehler (z. B. Dienst nicht erreichbar) → das Endgerät KANN basierend auf dem lokal zwischengespeicherten Widerrufsstatus entscheiden; bei Überschreitung der Cache-Gültigkeit als Timeout behandeln

Das Endgerät SOLLTE Widerrufsabfrageergebnisse zwischenspeichern, mit einer Cache-Gültigkeit von höchstens 5 Minuten.

### 4.3.3 Nachdem die Validierung besteht

Die Verarbeitung nach bestandener Validierung ist konsistent mit §3.3.3, wobei eine Session erstellt und AuthResult zurückgegeben wird.

## 4.4 Konvertierung in einen Offline-Authorization_Descriptor

Wenn `TicketPayload.convertible == true` (Standard) ist, kann das Endgerät das Trusted_Ticket in einen lokalen Authorization_Descriptor für die Offline-Verwendung konvertieren, nachdem die Validierung bestanden ist.

### 4.4.1 Konvertierungsauslöser

Die Konvertierung SOLLTE in den folgenden Situationen automatisch durchgeführt werden:

1. Trusted_Ticket-Validierung bestanden und `convertible == true`
2. Die Endgeräterichtlinie erlaubt nachfolgenden Offline-Zugriff
3. `TicketPayload.exp` ist ausreichend weit von der aktuellen Zeit entfernt (empfohlen > 1 Stunde)

### 4.4.2 Konvertierungsschritte

1. **Feldzuordnung**: TicketPayload-Felder gemäß der §2.4.4-Tabelle DescriptorPayload zuordnen
2. **Gültigkeitsdauerbeschränkung**: Die konvertierte `not_after` DARF `min(TicketPayload.exp, Konvertierungszeit + 7 Tage)` NICHT überschreiten
3. **Metadaten-Beibehaltung**: Wichtige Audit-Informationen des ursprünglichen Trusted_Ticket in Metadaten speichern:
   - `metadata["origin"] = "converted_from_ticket"`
   - `metadata["origin_jti"] = TicketPayload.jti`
   - `metadata["origin_iss"] = TicketPayload.iss`
   - `metadata["origin_kid"] = TicketHeader.kid`
4. **Lokale Signatur**: Das Endgerät signiert die konvertierte Nutzlast unter Verwendung seines lokal gespeicherten Schlüssels neu (siehe §4.4.3)
5. **Lokale Speicherung**: Den konvertierten Authorization_Descriptor gemäß §3.2.3 verschlüsselt speichern

### 4.4.3 Lokale Signierschlüssel des Endgeräts

Um den Konvertierungsablauf zu unterstützen, MUSS das Endgerät ein Paar lokaler Signierschlüssel halten:

- Privater Schlüssel: Im sicheren Speicher des Endgeräts gespeichert, nur für den lokalen Konvertierungsablauf verwendet
- Öffentlicher Schlüssel: Als Verification_Key vom Typ `source == "pre-installed"` in seinem eigenen Schlüsselspeicher registriert

Die `signature.key_id` des konvertierten Authorization_Descriptor MUSS diesen lokalen Schlüssel identifizieren, und `payload.issuer_id` MUSS auf `"local-conversion:" + terminal_id` gesetzt sein.

Beim Validieren eines lokal konvertierten Authorization_Descriptor verifiziert das Endgerät die Signatur unter Verwendung des lokalen öffentlichen Schlüssels. Dieses Design stellt sicher:

- Die konvertierte Anmeldeinformation kann unabhängig offline validiert werden, ohne vom ursprünglichen Ticket_Issuer abhängig zu sein
- Die Konvertierungsvertrauenskette ist an die eigene Schlüsselsicherheit des Endgeräts verankert

### 4.4.4 Konvertierungsbeschränkungen

Ein lokal konvertierter Authorization_Descriptor DARF NICHT:

1. Von anderen Endgeräten vertraut werden (seine Signatur-key_id ist lokal)
2. Für die endgeräteübergreifende Verwendung migriert werden
3. Über den Widerrufsmechanismus von Descriptor_Issuer widerrufen werden (muss lokal gelöscht werden)

Das Endgerät SOLLTE in den folgenden Situationen lokal konvertierte Authorization_Descriptors proaktiv löschen:

1. Erhalt einer Widerrufsbenachrichtigung für das ursprüngliche Trusted_Ticket (über jti korreliert)
2. Die Anmeldeinformation ist seit mehr als 24 Stunden abgelaufen

## 4.5 Online-zu-Offline-Degradation

### 4.5.1 Degradationsauslöser

Das Endgerät überwacht kontinuierlich die Verfügbarkeit von Online-Diensten. Wenn eine der folgenden Bedingungen erfüllt ist, wird die Degradation ausgelöst:

1. Die Verbindung mit Ticket_Issuer ist mehr als 30 Sekunden lang nicht erreichbar
2. Widerrufsabfrageanfragen schlagen 3 aufeinanderfolgende Male mit Timeout fehl
3. Der Netzwerkstack meldet globale Netzwerkunverfügbarkeit

### 4.5.2 Degradationsverhalten

Nach der Degradation behandelt das Endgerät die Dinge gemäß den folgenden Regeln:

1. Annahme neuer Trusted_Tickets ablehnen (es sei denn, für Offline-Annahme konfiguriert) — um zu vermeiden, dass der Widerrufsstatus nicht online verifiziert werden kann
2. Tickets, die bereits in lokale Authorization_Descriptors konvertiert wurden, werden weiterhin gemäß §3-Regeln verwendet
3. Autorisierungsvalidierungsanfragen, die direkt Authorization_Descriptor verwenden, sind von der Degradation nicht betroffen
4. Das Endgerät KANN iFay_Runtime hinweisen, dass es sich derzeit im Offline-Modus befindet, sodass Fay Verhaltensrichtlinien anpassen kann

### 4.5.3 Wiederherstellung

Nachdem das Endgerät die Wiederherstellung des Online-Dienstes erkennt, MUSS es:

1. Priorisiert die Widerrufsliste synchronisieren und alle Widerrufserklärungen, die während des Offline-Zeitraums möglicherweise verpasst wurden, auf den lokalen Status anwenden
2. Aktive Sessions erneut überprüfen; für Sessions, die lokal konvertierte Authorization_Descriptor verwenden, deren ursprüngliches Ticket widerrufen wurde, gemäß den Regeln in Kapitel 5 zwangsweise beenden
3. Die normale Trusted_Ticket-Annahmerichtlinie wiederherstellen

Der Wiederherstellungsprozess ist transparent für iFay_Runtime, aber das Endgerät KANN Modusänderungen über `SessionStateChanged` benachrichtigen.

## 4.6 Priorität in Doppel-Anmeldeinformation-Szenarien

Wenn ein Fay gleichzeitig sowohl ein Trusted_Ticket als auch ein Authorization_Descriptor für dieselbe Ressource hält, ist die iFay_Runtime-Auswahlrichtlinie:

| Netzwerkstatus | Empfohlene Wahl | Grund |
|---------|---------|------|
| Online und Ticket_Issuer erreichbar | Trusted_Ticket | Stärkere Aktualität, Echtzeit-Widerruf möglich |
| Offline | Authorization_Descriptor oder konvertierte lokale Anmeldeinformation | Trusted_Ticket kann den Widerruf nicht online verifizieren |

iFay_Runtime KANN eine Fallback-Anmeldeinformation in AuthRequest bereitstellen, sodass das Endgerät eine Backup-Anmeldeinformation versuchen kann, wenn die Validierung der primären Anmeldeinformation fehlschlägt. Diese Spezifikation schreibt kein spezifisches Format für den Fallback-Mechanismus vor, SOLLTE jedoch durch zwei unabhängige AuthRequests implementiert werden, um Protokollkomplexität zu vermeiden.

## 4.7 Semantische Konsistenz zwischen Trusted_Ticket und Authorization_Descriptor

Wenn iFay_Runtime beide Anmeldeinformationstypen verwendet, MUSS das Endgerät garantieren:

1. **Validierungsergebnis-Konsistenz**: Derselbe Autorisierungsbereich erzeugt unter beiden Anmeldeinformationen konsistente Bestanden-/Abgelehnt-Beurteilungen
2. **Fehlercode-Semantikkonsistenz**: Äquivalente Fehlerursachen (z. B. Gültigkeitsdauer, Signatur, Widerruf) verwenden semantisch entsprechende Fehlercodes (siehe Kapitel 9)
3. **Session-Erstellungskonsistenz**: Es gibt keinen Unterschied im Lebenszyklusmanagement von Sessions, die über die beiden Anmeldeinformationen erstellt wurden
