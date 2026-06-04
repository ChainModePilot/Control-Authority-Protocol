# Kapitel 5: Sitzungsverwaltung und Aktivitätserkennung

Dieses Kapitel definiert die Zustandsmaschine, Lebenszyklusereignisse und Bindungsregeln von `Session` mit `Terminal_Resource` sowie den `Liveness_Detection`-Mechanismus. Dieses Kapitel entspricht der Designabsicht in §2.3 des Architekturplans.

## 5.1 Session-Zustandsmaschine

Jede Session durchläuft mehrere Zustände einer endlichen Zustandsmaschine innerhalb ihres Lebenszyklus.

### 5.1.1 Zustandsdefinitionen

```
SessionState = enum[
  "creating",
  "active",
  "handover_pending",
  "terminating",
  "terminated"
]
```

| Zustand | Beschreibung |
|------|------|
| `creating` | Autorisierungsvalidierung bestanden; Session wird initialisiert (z. B. Ressourcenzuweisung, OS-Zugriffskontrolle einrichten) |
| `active` | Session ist im aktiven Zustand; Fay kann Operationen innerhalb des autorisierten Bereichs durchführen |
| `handover_pending` | Session nimmt an einer Kontrollübergabe teil (siehe Kapitel 6); das Ergebnis ist unbestimmt |
| `terminating` | Session wird beendet; Ressourcen werden zurückgewonnen; neue Anfragen werden abgelehnt |
| `terminated` | Session ist vollständig beendet; session_id wird in den Verlauf aufgenommen |

### 5.1.2 Zustandsübergangsdiagramm

```
              ┌─────────────┐
              │   (start)   │
              └──────┬──────┘
                     │ Autorisierungsvalidierung bestanden
                     ▼
              ┌─────────────┐
              │  creating   │
              └──────┬──────┘
                     │ Ressourceninitialisierung abgeschlossen
                     ▼
              ┌─────────────┐  Übergabe initiiert  ┌──────────────────┐
              │   active    │─────────────────────→│ handover_pending │
              └──────┬──────┘                      └──────┬───────────┘
                     │  ↑                                 │
              Beend.bed.│  │Übergabe-Timeout-Rollback     │Übergabe abgeschlossen
                     ▼  │                                 ▼
              ┌─────────────┐                       ┌─────────────┐
              │ terminating │←──────────────────────┤ (handed off)│
              └──────┬──────┘                       └─────────────┘
                     │ Ressourcenrückgewinnung abgeschlossen
                     ▼
              ┌─────────────┐
              │ terminated  │
              └─────────────┘
```

### 5.1.3 Zustandsübergangsregeln

Das Endgerät MUSS den Session-Zustand nur über die in der untenstehenden Tabelle erlaubten Übergänge aktualisieren:

| Aktueller Zustand | Zielzustand | Auslösebedingung |
|---------|---------|---------|
| `creating` | `active` | Ressourceninitialisierung abgeschlossen |
| `creating` | `terminating` | Initialisierung fehlgeschlagen (z. B. Ressource von einer anderen Session belegt) |
| `active` | `handover_pending` | Übergabeanfrage für diese Session erhalten |
| `active` | `terminating` | Proaktive Freigabe, Heartbeat-Timeout, Widerrufsbeendigung, Ressource nicht verfügbar |
| `handover_pending` | `terminating` | Übergabe erfolgreich (die ursprüngliche Session MUSS beendet werden) oder Übergabe-Timeout-Stornierung |
| `handover_pending` | `active` | Übergabe vom neuen Controller abgelehnt (auf den ursprünglichen Zustand zurückgesetzt) |
| `terminating` | `terminated` | Ressourcen vollständig freigegeben; Zustandspersistenz abgeschlossen |

Implementierungen DÜRFEN keine nicht aufgelisteten Zustandsübergänge durchführen.

### 5.1.4 Atomarität von Zustandsübergängen

Jeder Zustandsübergang MUSS die folgenden Atomaritätsanforderungen erfüllen:

1. Zustandslesen, -beurteilung und -schreiben MÜSSEN innerhalb eines kritischen Abschnitts abgeschlossen sein; externe Beobachter sehen keine Zwischenzustände
2. Zustandsübergänge und die entsprechenden Ressourcenoperationen (z. B. OS-Zugriffskontrollbereitstellung) MÜSSEN als eine einzige Transaktion ausgeführt werden; entweder alle gelingen oder alle werden zurückgesetzt

## 5.2 Session-Erstellung

### 5.2.1 Erstellungsauslöser

Eine Session wird in den folgenden Szenarien erstellt:

1. iFay_Runtime initiiert einen AuthRequest und die Autorisierungsvalidierung besteht (siehe §3.3, §4.3)
2. Nach erfolgreicher Kontrollübergabe wird eine neue Session für den neuen Controller erstellt (siehe Kapitel 6)

### 5.2.2 Erstellungsschritte

Das Endgerät MUSS eine Session in den folgenden Schritten erstellen:

1. **Ressourcenbelegungsvorprüfung**: Prüfen, ob die Ziel-`resource_id` unter `access_mode` belegt werden kann (siehe §5.3 und die Lese-Schreib-Sperrregeln in Kapitel 7)
2. **Session_ID generieren**: Eine neue UUID v7 zuweisen
3. **Anfangszustand festlegen**: `state = "creating"`
4. **Session-Objekt konstruieren**: Die in §2.5 definierten Felder ausfüllen
5. **OS-Zugriffskontrolle bereitstellen**: Das Betriebssystem über die §1.3.2-Schnittstelle informieren, dem Fay-Prozess den Zugriff auf die Ressource im angegebenen Modus zu erlauben
6. **Zustandsübergang**: `state: creating → active`; `created_at` und `last_heartbeat_at` aufzeichnen
7. **AuthResult zurückgeben**: `session_id` über AuthResult an iFay_Runtime zurückgeben

### 5.2.3 Behandlung von Erstellungsfehlern

Wenn ein Schritt fehlschlägt:

1. Zugewiesene Ressourcen (z. B. OS-Zugriffskontrolleinträge) MÜSSEN zurückgesetzt werden
2. Session-Zustand wechselt zu `terminating` und dann sofort zu `terminated`
3. Den entsprechenden Fehlercode zurückgeben (z. B. `E_RESOURCE_BUSY`, `E_OS_INTEGRATION_FAILED`)

## 5.3 Bindung von Session an Terminal_Resource

### 5.3.1 Eins-zu-eins-Bindung

Jede aktive Session MUSS an genau eine `Resource_ID` gebunden sein. Das Endgerät verwaltet die Ressourcenbelegungsbeziehung über (resource_id, access_mode).

### 5.3.2 Exklusivitäts- und Freigaberegeln

Siehe §7.2 Lese-Schreib-Sperrmatrix in Kapitel 7 für Details. Dieser Abschnitt gibt die vereinfachten Regeln auf der Sitzungsebene:

| Aktuelle aktive Session der Ressource | Neue Session-Anfrage | Behandlung |
|---------------------|----------------|------|
| Keine | Beliebiger Modus | Erstellung erlaubt |
| ≥1 read | read | Erstellung erlaubt |
| ≥1 read | write/execute/configure | Ablehnen (`E_RESOURCE_BUSY`) |
| 1 write/execute/configure | Beliebiger Modus | Ablehnen (`E_RESOURCE_BUSY`) |

### 5.3.3 Kaskadierende Beendigung bei Ressourcenunverfügbarkeit

Wenn die Ressource, die `Resource_ID` entspricht, nicht verfügbar wird (z. B. Hardware-Trennung, Software-Prozessabsturz, Betriebssystem meldet Ressourcenverschwinden):

1. Das Endgerät MUSS sofort alle an diese Ressource gebundenen Sessions auf `terminating` umschalten
2. Das Endgerät MUSS `SessionStateChanged`-Benachrichtigungen an alle betroffenen iFay_Runtimes mit `reason` als `"resource_unavailable"` senden
3. Nachdem die Ressource wiederhergestellt ist, DÜRFEN beendete Sessions NICHT automatisch wiederhergestellt werden; iFay_Runtime muss erneut AuthRequest initiieren, um eine neue Session zu erstellen

## 5.4 Liveness_Detection

Die Aktivitätserkennung bestimmt, ob eine Session noch aktiv ist, durch zwei unabhängige Signale: **Aufrechterhaltung der persistenten Verbindung** und **Anwendungsschicht-Heartbeat**.

### 5.4.1 Aufrechterhaltung der persistenten Verbindung

Zwischen iFay_Runtime und dem Endgerät MUSS ein persistenter Verbindungskanal aufrechterhalten werden (z. B. WebSocket, HTTP/2-Stream, gRPC-Stream).

Das Endgerät MUSS den Zustand dieser persistenten Verbindung überwachen:

- Verbindung hergestellt: Persistentes Verbindungssignal gültig
- Verbindung unterbrochen: Persistentes Verbindungssignal ungültig (einschließlich TCP RST, Timeout ohne Antwort usw.)

Eine persistente Verbindungsunterbrechung KANN das Ergebnis vorübergehender Netzwerkstörungen sein und DARF NICHT allein als Grundlage für die Beendigung einer Session verwendet werden (siehe §5.4.3 Doppelbestimmung).

### 5.4.2 Anwendungsschicht-Heartbeat

iFay_Runtime MUSS für jede aktive Session periodisch Heartbeat-Nachrichten über die persistente Verbindung senden:

```
Heartbeat (body of ProtocolMessage) {
  required session_id      : Session_ID
  required sequence_number : uint64
}

HeartbeatAck (body of ProtocolMessage) {
  required session_id      : Session_ID
  required sequence_number : uint64
}
```

| Parameter | Standard | Bereich |
|------|--------|------|
| Heartbeat-Intervall | 10 Sekunden | 1–60 Sekunden |
| Heartbeat-Timeout-Schwelle | 30 Sekunden | Heartbeat-Intervall × 2 bis Heartbeat-Intervall × 6 |

`sequence_number` ist innerhalb jeder Session monoton steigend, beginnend bei 0.

Das Endgerät MUSS:

1. Sofort nach Erhalt des Heartbeats HeartbeatAck zurückgeben (`sequence_number` stimmt mit der Anfrage überein)
2. `last_heartbeat_at` der Session aktualisieren
3. Heartbeats mit `sequence_number` kleiner als das aufgezeichnete Maximum ablehnen (Replay-Verhinderung)

### 5.4.3 Doppelbestimmung

Das Endgerät MUSS Session-Fehler nur dann bestimmen, wenn die folgenden zwei Bedingungen **gleichzeitig** erfüllt sind:

1. Die persistente Verbindung war länger als die Heartbeat-Timeout-Schwelle unterbrochen
2. Die Dauer seit `last_heartbeat_at` überschreitet die Heartbeat-Timeout-Schwelle

Die Designabsicht dieser Doppelbestimmung ist:

- Während kurzfristiger Störungen der persistenten Verbindung können Anwendungsschicht-Heartbeats noch normal sein (sollte nicht fälschlicherweise als fehlgeschlagen beurteilt werden)
- Wenn Anwendungsschicht-Heartbeats stoppen, aber die persistente Verbindung verbleibt (z. B. Fay hängt), ist Heartbeat-Timeout zur Erkennung erforderlich
- Das gleichzeitige Auftreten beider ist ein hochzuverlässiges Fehlersignal

### 5.4.4 Behandlung nach Fehler

Nach Bestimmung eines Fehlers MUSS das Endgerät:

1. Session-Zustand umschalten: `active → terminating`
2. Belegtes Terminal_Resource freigeben
3. Den Zugriff des Fay auf die Ressource über die OS-Schnittstelle widerrufen
4. Während des Wartens auf Wiederherstellung der persistenten Verbindung oder des Heartbeats geben alle Anfragen für diese Session `E_SESSION_TERMINATED` zurück
5. Das Endgerät KANN eine `SessionStateChanged`-Benachrichtigung an iFay_Runtime nach Wiederherstellung der persistenten Verbindung pushen, aber die Session ist zu diesem Zeitpunkt nicht mehr wiederherstellbar

### 5.4.5 Optionale Heartbeat-Modi

Implementierungen KÖNNEN die folgenden Heartbeat-Optimierungsmodi unterstützen (mit erforderlicher expliziter Unterstützungsdeklaration):

1. **Aggregierter Heartbeat**: iFay_Runtime meldet mehrere Sessions in einer einzigen Heartbeat-Nachricht (anwendbar auf Szenarien, in denen eine Runtime eine große Anzahl von Sessions verwaltet)
2. **Heartbeat mit reduzierter Frequenz**: Wenn eine Session lange Zeit keinen Ressourcenzugriff initiiert hat, kann das Heartbeat-Intervall auf bis zu 60 Sekunden erweitert werden

Das spezifische Format des aggregierten Heartbeats wird als optionale Erweiterung in schema.json als optionale Felder definiert.

## 5.5 Session-Beendigung

### 5.5.1 Beendigungsauslösebedingungen

| Auslösegrund | reason-Feldwert | Auslöser |
|---------|---------------|--------|
| Proaktive Freigabe | `"client_release"` | iFay_Runtime |
| Heartbeat-Timeout | `"liveness_timeout"` | Endgerät-Liveness_Detection |
| Widerrufsbeendigung | `"credential_revoked"` | Aktualisierung der Endgeräte-Widerrufsliste |
| Ressource nicht verfügbar | `"resource_unavailable"` | Endgerät-OS-Integrationsschicht |
| Anmeldeinformation abgelaufen | `"credential_expired"` | Periodische Endgeräteprüfung |
| Übergabe abgeschlossen | `"handed_over"` | Endgerät-Übergabeablauf |
| Erzwungene Beendigung | `"forced"` | Endgerät-Verwaltungsschnittstelle (z. B. Admin-Operationen) |

### 5.5.2 Beendigungsschritte

Das Endgerät MUSS die Beendigung in den folgenden Schritten durchführen:

1. **Zustandsübergang**: `{active|handover_pending} → terminating`
2. **OS-Ressourcenrückgewinnung**: Den Ressourcenzugriff des Fay-Prozesses über die §1.3.2-Schnittstelle widerrufen
3. **Nebenläufigkeitssteuerung freigeben**: (resource_id, access_mode) aus der Belegungsliste entfernen (siehe Kapitel 7 Lese-Schreib-Sperraktualisierung)
4. **iFay_Runtime benachrichtigen**: Eine `SessionStateChanged`-Nachricht mit Zustand `terminated` senden, einschließlich reason
5. **Zustandsübergang**: `terminating → terminated`
6. **Persistenz**: KANN den Verlaufseintrag der Session in das Audit-Protokoll schreiben

### 5.5.3 Idempotenz der Beendigung

Die von iFay_Runtime initiierte SessionRelease-Nachricht MUSS idempotent sein:

- Wenn die Session bereits in `terminating` oder `terminated` ist, MUSS eine wiederholte Freigabe Erfolg zurückgeben (nicht als Fehler behandelt)
- Wenn die Session nicht existiert (bereits zurückgewonnen), `E_SESSION_NOT_FOUND` zurückgeben

### 5.5.4 Beobachtbarkeit der Beendigung

Das Endgerät MUSS die Session-Beendigung mindestens an folgende benachrichtigen:

1. Die mit der Session verbundene iFay_Runtime (über `SessionStateChanged`)
2. Das lokale Ressourcenverwaltungs-Subsystem des Endgeräts (zur Aktualisierung des Verfügbarkeitszustands)

Das Endgerät KANN wählen, ob es benachrichtigt:

- Andere iFay_Runtimes, die auf diese Ressource warten (über den §6-Übergabebenachrichtigungsmechanismus)
- Audit-Protokoll (abhängig von der Audit-Richtlinie der Endgeräteimplementierung)

## 5.6 Zustandsänderungs-Benachrichtigung

```
SessionStateChanged (body of ProtocolMessage) {
  required session_id : Session_ID
  required new_state  : SessionState
  optional reason     : string
  optional details    : map<string, string>
}
```

Das Endgerät MUSS SessionStateChanged an iFay_Runtime senden, wenn die folgenden Zustandsübergänge auftreten:

1. `creating → active`: Session-Erstellung erfolgreich (kann auch direkt über AuthResult kommuniziert werden; eines von beiden)
2. `active → handover_pending`: Session tritt in den Übergabeablauf ein
3. `handover_pending → active`: Übergabe storniert oder abgelehnt
4. `* → terminating` und `terminating → terminated`: Beginn und Ende des Beendigungsprozesses

## 5.7 Beziehung zwischen Session und Anmeldeinformations-Lebenszyklus

Der Session-Lebenszyklus ist unabhängig vom Anmeldeinformations-Lebenszyklus, aber das Endgerät MUSS Anmeldeinformations-Zustandsänderungen überwachen:

| Anmeldeinformations-Ereignis | Session-Auswirkung |
|---------|-------------|
| Anmeldeinformations-`not_after` erreicht | Verbundene Session wird sofort mit `credential_expired` beendet |
| Anmeldeinformation widerrufen (Widerrufserklärung erreicht Endgerät) | Verbundene Session wird sofort mit `credential_revoked` beendet |
| Anmeldeinformation mit derselben ID neu ausgestellt (theoretisch unmöglich, descriptor_id ist nicht wiederverwendbar) | Nicht zutreffend |
| Anmeldeinformation durch neue Version überschrieben (Erneuerungsablauf) | Session nicht betroffen (wird weiterhin mit der ursprünglichen credential_ref validiert) |

## 5.8 Session-Mengenbegrenzungen

Das Endgerät SOLLTE Session-Mengenbegrenzungen implementieren, um Ressourcenerschöpfung zu verhindern:

| Dimension | Standardlimit | Hinweise |
|------|---------|------|
| Gesamtzahl aktiver Sessions auf einem einzelnen Endgerät | 1024 | Anpassbar durch Endgeräterichtlinie |
| Aktive Sessions für einen einzelnen Fay | 64 | Verhindert, dass ein einzelner Fay Systemressourcen monopolisiert |
| Aktive read-Sessions für eine einzelne Resource_ID | 32 | Verhindert Missbrauch der read-Freigabe |

Wenn ein Limit überschritten wird, MUSS das Endgerät `E_SESSION_LIMIT_EXCEEDED` zurückgeben.
