# Kapitel 1: Architektur und Rollen

Dieses Kapitel definiert die Rollen im CAP-Protokoll, die Vertrauensbeziehungen zwischen ihnen und die Schnittstellenverträge mit externen Systemen. Der Inhalt dieses Kapitels bietet die gemeinsame Terminologiebasis und den architektonischen Kontext für nachfolgende Kapitel.

## 1.1 Protokollrollen

Das CAP-Protokoll umfasst die folgenden normativen Rollen. Jede Rolle übernimmt klare Verantwortlichkeiten in Protokollinteraktionen und interagiert mit anderen Rollen über eingeschränkte Schnittstellen.

### 1.1.1 Autorisierer-Rolle

**Natural_Person** und **Official_Post** sind die ultimativen Quellen der Autorisierung. Autorisierer nehmen nicht direkt an Protokollnachrichteninteraktionen teil — Autorisierer delegieren die Ausstellungsbefugnis über einen Autorisierungsdelegationsmechanismus an `Descriptor_Issuer`.

Autorisierer MÜSSEN ihre Identität durch einen sicheren Identitätsverifizierungsmechanismus nachweisen. Der spezifische Verifizierungsmechanismus für die Autorisierer-Identität liegt außerhalb des Geltungsbereichs dieser Spezifikation, ein solcher Mechanismus MUSS jedoch die folgenden Anforderungen erfüllen:

- Nichtabstreitbarkeit: Autorisierer können die von ihnen getroffenen Autorisierungsentscheidungen nicht abstreiten
- Fälschungssicherheit: Dritte können die Autorisierungsentscheidung eines Autorisierers nicht fälschen

### 1.1.2 Aussteller-Rolle

**Descriptor_Issuer** ist verantwortlich für die Generierung und Ausstellung von `Authorization_Descriptor`. **Ticket_Issuer** ist verantwortlich für die Ausstellung von `Trusted_Ticket`. Eine einzelne Entität KANN beide Aussteller-Rollen gleichzeitig übernehmen.

Aussteller MÜSSEN:

1. Anmeldeinformationen nur ausstellen, nachdem sie eine ausdrückliche Autorisierung von einem legitimen Autorisierer erhalten haben
2. Digitale Signaturen auf Anmeldeinformationen unter Verwendung der in Kapitel 8 definierten Signaturalgorithmen anwenden
3. Statusaufzeichnungen ausgestellter Anmeldeinformationen pflegen (mindestens einschließlich Ausstellungszeit, Gültigkeitsdauer und aktuellem Status)
4. Bei Erhalt von Widerrufsanfragen umgehend den Widerrufsstatus aktualisieren und Widerrufserklärungen veröffentlichen

Aussteller DÜRFEN NICHT:

1. Anmeldeinformationen ohne Autorisierung ausstellen
2. Anmeldeinformationen außerhalb des Autorisierungsbereichs des Autorisierers ausstellen
3. Bezeichner widerrufener Anmeldeinformationen wiederverwenden

### 1.1.3 Inhaber-Rolle

**Fay** (einschließlich `iFay` und `coFay`) ist der Inhaber von Anmeldeinformationen und der Anforderer von Ressourcenzugriffen. Fay nimmt indirekt durch `iFay_Runtime` an Protokollinteraktionen teil — Fay selbst sendet keine Protokollnachrichten direkt.

Fay MUSS alle Protokollinteraktionen mit dem Endgerät über sein zugehöriges `iFay_Runtime` abschließen. Von Fay gehaltene Anmeldeinformationen MÜSSEN über iFay_Runtime an das Endgerät übermittelt werden.

### 1.1.4 Laufzeit-Rolle

**iFay_Runtime** ist die Laufzeitumgebung für Fay-Instanzen und übernimmt das Senden und Empfangen von Protokollnachrichten. Eine einzelne iFay_Runtime-Instanz KANN mehrere Fay-Instanzen verwalten.

iFay_Runtime MUSS die in Abschnitt 0.5.3 definierten Anforderungen der Laufzeit-Konformitätsstufe erfüllen.

### 1.1.5 Endgerät-Rolle

Das **Endgerät** (Terminal) ist der Inhaber von `Terminal_Resource` und der Ausführer der Zugriffskontrolle. Das Endgerät integriert die folgenden normativen Komponenten:

- **Protocol_Engine**: Systemkomponente, die die Kernlogik des CAP-Protokolls ausführt
- **Descriptor_Validator**: Komponente, die für die Verifizierung der Legitimität und Gültigkeit von `Authorization_Descriptor` verantwortlich ist
- **Session-Manager**: Pflegt die Session-Zustandsmaschine und den Liveness_Detection-Mechanismus
- **Lokale Widerrufsliste**: Speichert bekannte widerrufene Anmeldeinformationsbezeichner

Das Endgerät MUSS die in Abschnitt 0.5.1 definierten Anforderungen der Endgerät-Konformitätsstufe erfüllen.

### 1.1.6 Vertrauensinfrastruktur-Rolle

**Registration_Authority** ist verantwortlich für die Registrierungsverwaltung der Endgeräte und die Verteilung von `Verification_Key`. Registration_Authority ist der Wurzelanker der Vertrauenskette des CAP-Protokolls.

Registration_Authority MUSS:

1. Verification_Keys nur an Endgeräte verteilen, die die Identitätsverifizierung bestanden haben
2. Schlüsselverteilung über sichere Kanäle abschließen (siehe Kapitel 8)
3. Schlüsselaktualisierungs- und Rotationsmechanismen bereitstellen
4. Autoritative Aufzeichnungen über den Endgeräte-Registrierungsstatus pflegen

## 1.2 Vertrauenskette

Die Vertrauenskette des CAP-Protokolls wird vom Autorisierer an den `Descriptor_Validator` des Endgeräts weitergegeben und besteht aus zwei unabhängigen, aber kooperierenden Vertrauenspfaden.

### 1.2.1 Autorisierungsvertrauenspfad

Der Autorisierungsvertrauenspfad bestimmt, **ob ein bestimmter Fay autorisiert ist, auf eine bestimmte Ressource zuzugreifen**:

```
Autorisierer (Natural_Person / Official_Post)
    │
    │ ① Autorisierungsdelegation (mit nicht-abstreitbarem Beweis)
    ▼
Descriptor_Issuer
    │
    │ ② Stellt Authorization_Descriptor aus (mit digitaler Signatur)
    ▼
Fay
    │
    │ ③ Übermittelt Authorization_Descriptor
    ▼
Descriptor_Validator
```

Das Endgerät MUSS die Authentizität jeder Verbindung auf dem Autorisierungsvertrauenspfad verifizieren:

- Die Authentizität von Schritt ① wird von Descriptor_Issuer zum Ausstellungszeitpunkt verifiziert; das Endgerät nimmt nicht direkt teil
- Die Authentizität von Schritt ② wird vom Endgerät durch Verifizierung der digitalen Signatur von Descriptor_Issuer verifiziert (abhängig vom Schlüsselvertrauenspfad)
- Die Authentizität von Schritt ③ wird vom Endgerät durch Verifizierung der Bindungsbeziehung zwischen Authorization_Descriptor und der Bezeichnung des übermittelnden Fay verifiziert

### 1.2.2 Schlüsselvertrauenspfad

Der Schlüsselvertrauenspfad bestimmt, **ob das Endgerät der Signatur eines bestimmten Descriptor_Issuer vertraut**:

```
Registration_Authority (Vertrauensanker)
    │
    │ ① Verteilt Verification_Key (über sicheren Kanal)
    ▼
Lokaler Schlüsselspeicher des Endgeräts
    │
    │ ② Stellt Verification_Key dem Descriptor_Validator zur Verfügung
    ▼
Descriptor_Validator
    │
    │ ③ Verifiziert Authorization_Descriptor-Signatur mit Verification_Key
    ▼
Validierung bestanden / abgelehnt
```

Das Endgerät MUSS:

1. Nur Verification_Keys vertrauen, die von Registration_Authority verteilt wurden
2. Verification_Keys sicher speichern (siehe Kapitel 8 für Anforderungen an die Schlüsselspeicherung)
3. Signaturverifizierung mit Schlüsseln ablehnen, die nicht von Registration_Authority verteilt wurden

## 1.3 Externe Schnittstellenverträge

Die CAP-Protokollschicht interagiert mit 4 externen Systemen über klar definierte Schnittstellen. Dieser Abschnitt definiert die normativen Verträge dieser Schnittstellen.

### 1.3.1 Schnittstelle mit iFay_Runtime

Die Schnittstelle zwischen iFay_Runtime und der Endgerät-Protocol_Engine ist eine **bidirektionale Nachrichtenschnittstelle**.

**Nachrichtenrichtung: iFay_Runtime → Protocol_Engine**

| Nachrichtentyp | Zweck | Erforderliche Felder |
|---------|------|---------|
| `AuthRequest` | Initiiert eine Autorisierungsvalidierungsanfrage | fay_id, resource_id, access_mode, credential |
| `SessionRelease` | Gibt eine Sitzung proaktiv frei | session_id |
| `Heartbeat` | Anwendungsschicht-Heartbeat | session_id, sequence_number |
| `HandoverResponse` | Antwortet auf eine Übergabeanfrage | handover_id, accept |

**Nachrichtenrichtung: Protocol_Engine → iFay_Runtime**

| Nachrichtentyp | Zweck | Erforderliche Felder |
|---------|------|---------|
| `AuthResult` | Autorisierungsvalidierungsergebnis | request_id, status, session_id (bei Erfolg), error_code (bei Fehler) |
| `SessionStateChanged` | Sitzungszustandsänderungsbenachrichtigung | session_id, new_state, reason |
| `HandoverRequest` | Übergabeanfragebenachrichtigung | handover_id, target_resource_id, deadline |
| `HeartbeatAck` | Heartbeat-Bestätigung | session_id, sequence_number |

Die Schnittstellenimplementierung MUSS:

1. Persistente Verbindungen unterstützen, um `Heartbeat` und asynchrone Push-Nachrichten zu transportieren
2. TLS-verschlüsselte Übertragung unterstützen
3. Ein Serialisierungsformat verwenden, das der `schema.json`-Definition folgt

Die Schnittstellenimplementierung KANN:

1. Ein bestimmtes Transportprotokoll wählen (WebSocket, gRPC, HTTP/2 usw.)

### 1.3.2 Schnittstelle mit dem Endgerät-Betriebssystem

Die Schnittstelle zwischen Protocol_Engine und dem Endgerät-Betriebssystem ist eine **lokale bidirektionale Schnittstelle**.

Das Betriebssystem MUSS die folgenden Fähigkeiten an Protocol_Engine zugänglich machen:

1. Bereitstellung der Ressourcenzugriffskontrolle: Protocol_Engine erteilt Anweisungen wie „erlaube Fay X, im Modus Y auf Ressource Z zuzugreifen"
2. Ressourcenzustandsabfrage: Protocol_Engine fragt den aktuellen Verfügbarkeits- und Belegungszustand von Ressourcen ab
3. Ressourcenereignis-Abonnement: Protocol_Engine abonniert Ressourcenzustandsänderungsereignisse (z. B. Hardware-Trennung, Software-Absturz)

Die spezifische Implementierung der Schnittstelle hängt von der Betriebssystemplattform ab (Unix Domain Socket, Named Pipe, D-Bus usw.). Diese Spezifikation schreibt keine spezifische Implementierung vor, MUSS jedoch erfüllen:

1. Zugriffskontrollanweisungen werden synchron ausgeführt und liefern Ausführungsergebnisse zurück
2. Ressourcenereignisse werden über asynchrones Push benachrichtigt
3. Die Schnittstelle wird durch das native Sicherheitsmodell des Betriebssystems geschützt und ist nur von autorisierten Protocol_Engine-Prozessen aufrufbar

### 1.3.3 Schnittstelle mit Hardwaretreibern (Indirekt)

Protocol_Engine **DARF NICHT** direkt mit Hardwaretreibern kommunizieren. Alle Hardwaresteuerung wird durch das Betriebssystem weitergeleitet.

Statusberichtspfad von Hardwaretreibern zu Protocol_Engine:

```
Hardwaretreiber → Betriebssystem → Protocol_Engine
```

Hardwaretreiber SOLLTEN Hardware-Level-Zugriffssperrmechanismen implementieren, um sicherzustellen, dass selbst wenn Zugriffskontrollen auf Betriebssystemebene umgangen werden, die Hardware selbst unautorisierten Zugriff weiterhin ablehnen kann.

### 1.3.4 Schnittstelle mit Registration_Authority

Die Schnittstelle zwischen Registration_Authority und dem Endgerät ist eine **unidirektionale Schnittstelle** (RA → Endgerät).

Registration_Authority MUSS die folgenden Nachrichten an das Endgerät pushen:

| Nachrichtentyp | Zweck | Erforderliche Felder |
|---------|------|---------|
| `VerificationKeyDistribution` | Verteilt einen neuen Verifizierungsschlüssel | key_id, key_material, valid_from, issuer_id |
| `VerificationKeyRevocation` | Widerruft einen Verifizierungsschlüssel | key_id, revocation_time, reason |
| `RegistrationStatusUpdate` | Aktualisiert den Endgeräte-Registrierungsstatus | terminal_id, new_status |

Das Endgerät MUSS:

1. Nachrichten über einen TLS/mTLS-sicheren Kanal empfangen
2. Verifizieren, dass die Signaturquelle der Nachricht eine vertrauenswürdige Registration_Authority ist
3. Eine Bestätigung an Registration_Authority zurückgeben, nachdem ein neuer Schlüssel empfangen wurde

Das Endgerät initiiert keine Protokollnachrichten proaktiv an Registration_Authority (der erste Registrierungsablauf des Endgeräts gehört nicht zu den normalen Laufzeitinteraktionen des CAP-Protokolls und liegt außerhalb des Geltungsbereichs dieser Spezifikation).

## 1.4 Zuordnung von Rollen und Konformitätsstufen

| Implementierte Rolle | Erforderliche Konformitätsstufe |
|-----------|---------------------|
| Endgerät (einschließlich Protocol_Engine und Descriptor_Validator) | Endgerät-Konformitätsstufe (§0.5.1) |
| Descriptor_Issuer oder Ticket_Issuer | Aussteller-Konformitätsstufe (§0.5.2) |
| iFay_Runtime | Laufzeit-Konformitätsstufe (§0.5.3) |
| Registration_Authority | Außerhalb des Konformitätsbereichs dieser Spezifikation (Registrierungsablauf überschreitet die CAP-Protokoll-Laufzeit) |
