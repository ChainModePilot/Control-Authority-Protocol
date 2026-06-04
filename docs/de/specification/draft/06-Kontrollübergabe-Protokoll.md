# Kapitel 6: Kontrollübergabe-Protokoll

Dieses Kapitel definiert den Protokollablauf von `Handover_Policy` (Kontrollübergabestrategie), einschließlich der Semantik von drei Richtlinientypen, Atomaritätsgarantien der Übergabe, Timeout-Rollback und Benachrichtigungsmechanismen. Dieses Kapitel entspricht der Designabsicht in §2.4 des Architekturplans.

## 6.1 Anwendungsszenarien

Kontrollübergabe tritt in Szenarien auf, in denen mehrere Controller dieselbe `Terminal_Resource` nacheinander verwenden müssen. Diese Spezifikation definiert zwei Kern-Übergabetypen:

1. **Fay-zu-Fay**: Ein Fay überträgt seine gehaltene Session-Kontrollbefugnis an einen anderen Fay
2. **Fay-zu-Mensch**: Ein Fay gibt die Kontrollbefugnis an einen menschlichen Benutzer zurück, oder ein menschlicher Benutzer delegiert die Kontrollbefugnis an einen Fay

Diese Spezifikation beschreibt hauptsächlich Fay-zu-Fay. Fay-zu-Mensch-Übergabe verwendet denselben Ablauf wieder und modelliert den „menschlichen Benutzer" als Entität, die direkt mit dem Endgerät-OS über eine spezielle Schnittstelle interagiert.

## 6.2 Übergabeinitiierung

### 6.2.1 Auslösebedingungen

Eine Übergabe wird durch eines der folgenden Szenarien ausgelöst:

1. **Neuer Fay fordert Ressource aktiv an**: Fay-B fordert über AuthRequest Zugriff auf eine von Fay-A belegte Ressource an
2. **Aktueller Fay tritt aktiv ab**: Fay-A fordert aktiv über `SessionTransferRequest` an, die Kontrollbefugnis an einen designierten Fay-B zu übertragen
3. **Verwaltungsauslöser**: Ein menschlicher Benutzer oder eine Verwaltungsschnittstelle fordert eine Übertragung der Kontrollbefugnis an

### 6.2.2 Übergabeanfrage-Nachricht

```
SessionTransferRequest (body of ProtocolMessage) {
  required source_session_id : Session_ID                  // Session, die derzeit die Kontrollbefugnis hält
  required target_fay_id     : Fay_ID                      // Zielübernehmer
  required target_credential : CredentialContent           // Autorisierungs-Anmeldeinformation des Ziels
  required handover_reason   : string
  optional handover_metadata : map<string, string>
}
```

Nach Erhalt der Übergabeanfrage initiiert das Endgerät den Übergabe-Bewertungsablauf.

## 6.3 Übergabebewertung

Das Endgerät bewertet, ob die Übergabe in den folgenden Schritten erlaubt wird:

### 6.3.1 Schritt 1: Quell-Session-Validierung

- Verifizieren, dass die Session, die `source_session_id` entspricht, existiert und im Zustand `active` ist
- Verifizieren, dass der Initiator das Recht hat, diese Übergabe anzufordern (z. B. von der iFay_Runtime der Session oder einer Entität mit Verwaltungsbefugnis)

Fehler → `E_HANDOVER_INVALID_SOURCE` zurückgeben

### 6.3.2 Schritt 2: Ziel-Autorisierungsvalidierung

- `target_credential` auf Legitimität validieren (vollständige Validierung gemäß Kapitel 3/4-Regeln)
- Validieren, dass `target_fay_id` für die Ressource im `access_mode` der ursprünglichen Session autorisiert ist

Fehler → `E_HANDOVER_INVALID_TARGET` zurückgeben

### 6.3.3 Schritt 3: Richtlinienbewertung

Das Endgerät wendet `Handover_Policy` gemäß §6.4 an, um zu entscheiden, ob die Übergabe erlaubt wird.

Fehler → `E_HANDOVER_REJECTED_BY_POLICY` zurückgeben

### 6.3.4 Schritt 4: Atomare Vorbelegung

Nachdem die Richtlinienbewertung bestanden ist, schaltet das Endgerät die ursprüngliche Session sofort auf `handover_pending` um und tritt in den Vorbelegungszustand ein:

- Die Ressource akzeptiert keine anderen Kontrollbefugnisanforderungen, bis die Übergabe abgeschlossen ist
- Die ursprüngliche Session kann weiterhin Heartbeat aufrechterhalten (mit aktiven äquivalenten Heartbeat-Anforderungen), kann aber keine neuen Ressourcenoperationen initiieren

## 6.4 Übergabe-Richtlinientypen

`Handover_Policy` wird auf der Resource_ID-Granularität konfiguriert; jede Ressource KANN eine andere Richtlinie verwenden. Diese Spezifikation definiert drei Richtlinientypen; alle Endgeräte MÜSSEN mindestens das §6.4.1-Prioritätsregel-Skript implementieren.

### 6.4.1 Prioritätsregel-Skript

Das Endgerät berechnet Prioritätspunkte für die Quell-Session und den Ziel-Fay gemäß einem vordefinierten Regelskript; der höhere Punktestand gewinnt die Kontrollbefugnis.

**Konfigurationsstruktur**:

```
PriorityPolicyConfig {
  required policy_type : "priority_script"
  required script_id   : string                      // Skript-Bezeichner, der auf dem Endgerät vorkonfiguriert ist
  optional parameters  : map<string, string>
}
```

**Bewertungsablauf**:

1. Das Endgerät lädt das Regelskript, das `script_id` entspricht
2. Eingabe: Quell-Session-Info, Ziel-Fay-Bezeichner, Ziel-Anmeldeinformation grants, aktuelle Zeit, Resource_ID-Metadaten
3. Ausgabe: Quell-Prioritätspunktestand `S_source`, Ziel-Prioritätspunktestand `S_target`
4. Entscheidung: `S_target > S_source` → Übergabe erlaubt; sonst abgelehnt

Die spezifische Sprache des Regelskripts (z. B. CEL, Lua-Subset, JSON Logic) wird durch die Implementierung gewählt, MUSS aber:

- Deterministisch sein (gleiche Eingabe erzeugt gleiche Ausgabe)
- Keinen Zugriff auf das Netzwerk oder das Dateisystem haben
- Eine Ausführungszeit von höchstens 100 ms haben

### 6.4.2 KI-Modell-Echtzeitentscheidung

Das Endgerät ruft ein vorintegriertes KI-Modell auf, um eine Echtzeitentscheidung basierend auf dem aktuellen Kontext zu treffen.

**Konfigurationsstruktur**:

```
AIPolicyConfig {
  required policy_type   : "ai_model"
  required model_endpoint : string                    // Modell-Inferenz-Endpunkt, der für das Endgerät zugänglich ist
  optional context_fields : array<string>
  required decision_timeout_ms : uint32 (default 500)
}
```

**Bewertungsablauf**:

1. Das Endgerät konstruiert eine Entscheidungsanfrage, einschließlich Quell-/Zielinformationen und durch `context_fields` spezifizierten Kontexts
2. Ruft den KI-Modell-Inferenz-Endpunkt auf
3. Das Modell gibt eine Entscheidung zurück: `allow` / `reject` / `require_human`
4. Bei `require_human` Rückfall auf §6.4.3 menschliche Benutzerentscheidung

Das Endgerät MUSS:

- Einen harten Timeout setzen (Standard 500 ms); bei Timeout als `reject` behandeln
- Aktuelle Entscheidungsergebnisse zwischenspeichern, um hochfrequente Schwankungen zu vermeiden (Cache-Gültigkeit ≤ 5 Sekunden)
- Bei Nichtverfügbarkeit des KI-Modells auf das Prioritätsregel-Skript zurückfallen

### 6.4.3 Menschliche Benutzerentscheidung

Das Endgerät fordert eine Entscheidung von einem menschlichen Benutzer über die Benutzerschnittstelle an.

**Konfigurationsstruktur**:

```
HumanPolicyConfig {
  required policy_type     : "human_decision"
  required ui_channel      : enum["system_dialog", "companion_app", "external_panel"]
  required decision_timeout_seconds : uint32 (default 30)
  required default_action  : enum["allow", "reject"]    // Standardaktion bei Timeout
}
```

**Bewertungsablauf**:

1. Das Endgerät präsentiert die Übergabeanfragedetails dem menschlichen Benutzer über `ui_channel`
2. Wartet auf Benutzereingabe: erlauben / ablehnen
3. Bei Timeout → gemäß `default_action` behandeln

Der menschliche Entscheidungsmodus MUSS:

- In der Benutzerschnittstelle klar anzeigen: Quell-Fay-Bezeichner, Ziel-Fay-Bezeichner, Ressourcen-Bezeichner, Zugriffsmodus, Übergabegrund
- KEINE sensiblen Anmeldeinformationsdetails (z. B. Signatur, Schlüssel-ID) in der Benutzerschnittstelle anzeigen
- Empfehlen, die Standardaktion auf `reject` (konservative Sicherheit) zu setzen

## 6.5 Atomaritätsgarantie

Übergabe MUSS Atomarität erfüllen: **Zu jedem Zeitpunkt hat eine einzelne Resource_ID höchstens einen aktiven Controller**.

### 6.5.1 Atomare Sequenz

Das Endgerät MUSS die Übergabe in der folgenden Reihenfolge ausführen, wobei jeder Schritt innerhalb eines kritischen Abschnitts abgeschlossen wird:

```
[T0] handover_pending betreten: Quell-Session-Zustand wechselt; Ressource tritt in Vorbelegung ein
[T1] Quell-Session beenden: source_session_id-Zustand active → terminating
                            OS-Zugriffskontrollwiderruf ausgelöst
[T2] Ressourcenrückgewinnung abgeschlossen: Quellzustand terminating → terminated
                                            Ressource ist zu diesem Zeitpunkt im Zustand „keine aktive Session"
[T3] Ziel-Session erstellen: Neue Session für Ziel-Fay gemäß §5.2-Ablauf erstellen
                             Zustand tritt in creating ein
[T4] OS-Zugriffskontrolle bereitstellen: Ressourcenzugriff für den Ziel-Fay aktivieren
[T5] Ziel-Session wechselt zu active: Übergabe abgeschlossen
```

Externe Beobachter zwischen [T2] und [T3] beobachten den Ressourcenzustand als „keine aktive Session" und beobachten **niemals**, dass zwei Sessions gleichzeitig aktiv sind.

### 6.5.2 Rollback bei Zwischenschrittfehler

Wenn ein Schritt in [T0]–[T2] fehlschlägt:

- Die Quell-Session MUSS auf `active` zurückgesetzt werden
- Ressourcenrückgewinnungs-Vorbelegung wird storniert
- `E_HANDOVER_FAILED_AT_RELEASE` zurückgeben

Wenn [T3]–[T4] fehlschlägt (Quelle wurde beendet, aber Ziel kann nicht erstellt werden):

- Das Endgerät MUSS sofort Ressourcen bereinigen (Ressource tritt in den Zustand „keine aktive Session" ein)
- Die ursprüngliche iFay_Runtime (Quell-Session wurde beendet) und die Ziel-iFay_Runtime (Übernahmefehler) benachrichtigen
- `E_HANDOVER_FAILED_AT_ACQUIRE` zurückgeben
- **Ressourcenzustand**: Bleibt im Leerlauf, mit nachfolgenden Anfragen, die darum konkurrieren. **DARF NICHT** die Quell-Session automatisch wiederherstellen

### 6.5.3 Serialisierung gleichzeitiger Übergabeanfragen

Mehrere gleichzeitige Übergabeanfragen für dieselbe Ressource MÜSSEN serialisiert werden:

- Während sich die Ressource in `handover_pending` befindet, werden neue Übergabeanfragen in die Warteschlange gestellt oder abgelehnt
- Implementierungen KÖNNEN Ablehnung (`E_HANDOVER_IN_PROGRESS`) oder Warteschlange (maximale Warteschlangenlänge 8) wählen
- In die Warteschlange gestellte Anfragen werden in FIFO-Reihenfolge verarbeitet; jede Anfrage tritt in die Bewertung ein, nachdem die vorhergehende Übergabe abgeschlossen ist (Erfolg oder Fehler)

## 6.6 Timeout-Behandlung

Jede Übergabe MUSS einen harten Timeout setzen. Timeout-Schwellen werden durch den Richtlinientyp bestimmt:

| Richtlinientyp | Standard-Timeout | Bereich |
|---------|---------|------|
| Prioritätsregel-Skript | 1 Sekunde | 0,5–2 Sekunden |
| KI-Modell-Echtzeitentscheidung | 1 Sekunde | 0,5–3 Sekunden |
| Menschliche Benutzerentscheidung | 30 Sekunden | 5–120 Sekunden |

### 6.6.1 Timeout-Rollback-Prinzip

Wenn eine Übergabe nicht innerhalb des Timeouts abgeschlossen wird (d. h. [T5] nicht erreicht), MUSS das Endgerät:

1. Den derzeit laufenden Übergabeschritt abbrechen
2. Wenn in [T0]–[T2] (Quelle nicht vollständig beendet): Auf den ursprünglichen Session-active-Zustand zurücksetzen
3. Wenn in [T3]–[T4] (Quelle wurde beendet): Gemäß §6.5.2 behandeln (Quelle nicht wiederherstellen; Ressource auf Leerlauf setzen)
4. `HandoverFailedNotification` an die relevanten Parteien senden

### 6.6.2 Timeout-Benachrichtigung

```
HandoverFailedNotification (body of ProtocolMessage) {
  required handover_id   : uuid
  required source_session_id : Session_ID
  required target_fay_id : Fay_ID
  required reason        : enum["timeout", "policy_rejected", "release_failed", "acquire_failed"]
  optional details       : map<string, string>
}
```

## 6.7 Wiederholungsrichtlinie

Die Wiederholungsrichtlinie nach einem Übergabefehler wird vom Initiator bestimmt. Diese Spezifikation schreibt keinen Wiederholungsmechanismus vor, definiert jedoch die Grenzen von Wiederholungen:

- Das Wiederholungsintervall für dieselbe Übergabeanfrage SOLLTE ≥ 1 Sekunde sein
- Die Anzahl der Wiederholungen für dasselbe (source_session, target_fay)-Paar SOLLTE ≤ 3 sein
- Wiederholungen MÜSSEN eine neue handover_id verwenden

Das Endgerät KANN übermäßig häufige Wiederholungsanfragen ablehnen und `E_HANDOVER_RETRY_LIMIT` zurückgeben.

## 6.8 Fay-zu-Mensch-Übergabe

Übergaben, die die Kontrollbefugnis an einen menschlichen Benutzer zurückgeben, folgen dem obigen Ablauf, aber die Zielpartei ist ein menschlicher Benutzer:

- Das `target_fay_id`-Feld verwendet den speziellen Wert `"human:" + terminal_id` (repräsentiert den menschlichen Benutzer des aktuellen Endgeräts)
- Die Ziel-Autorisierungsvalidierung wird übersprungen (der menschliche Benutzer hat standardmäßig vollständige Berechtigungen über die Ressourcen seines eigenen Endgeräts)
- Die Richtlinienbewertung verwendet typischerweise §6.4.3 menschliche Benutzerentscheidung (Übernahmeabsicht bestätigen)
- Bei der Erstellung der Ziel-Session wird die Session nicht an einen Fay gebunden, sondern direkt vom OS-Benutzerprozess gehalten

Der umgekehrte Ablauf der proaktiven Übernahme durch den menschlichen Benutzer ist symmetrisch: Die ursprüngliche Fay-Session wird durch einen vom Menschen gehaltenen OS-Prozess ersetzt.

## 6.9 Beobachtbarkeit der Übergabe

Das Endgerät MUSS Übergabeereignis-Beobachtbarkeit für die folgenden bereitstellen:

| Empfänger | Empfangenes Ereignis |
|--------|-----------|
| Quell-iFay_Runtime | `SessionStateChanged` (active → handover_pending → terminating → terminated) |
| Ziel-iFay_Runtime | `AuthResult` (Erfolgsantwort für neue Session-Erstellung) oder `HandoverFailedNotification` |
| Endgerät-Audit-Protokoll | Vollständiger Übergabeeintrag (einschließlich handover_id, source, target, policy-Entscheidung, Endergebnis) |

## 6.10 Übergabe-Nachrichtensequenz

Vollständige Nachrichtensequenz für eine erfolgreiche Übergabe:

```
[Source Runtime]      [Target Runtime]      [Terminal]
       │                     │                  │
       │── SessionTransferRequest ─────────────→│
       │                     │                  │── Richtlinie bewerten
       │                     │                  │── source: active → handover_pending
       │←─ SessionStateChanged(handover_pending)│
       │                     │                  │── source beenden
       │←─ SessionStateChanged(terminating)─────│
       │←─ SessionStateChanged(terminated)──────│
       │                     │                  │── target erstellen
       │                     │←─ AuthResult ────│
       │                     │                  │── target: creating → active
       │                     │←─ SessionStateChanged(active)
```

Nachrichtensequenz für Fehler-Rollback:

```
[Source Runtime]      [Target Runtime]      [Terminal]
       │                     │                  │
       │── SessionTransferRequest ─────────────→│
       │                     │                  │── Richtlinie bewerten
       │                     │                  │── source: active → handover_pending
       │←─ SessionStateChanged(handover_pending)│
       │                     │                  │── Timeout oder Fehler
       │                     │                  │── source: handover_pending → active
       │←─ SessionStateChanged(active)──────────│
       │←─ HandoverFailedNotification ──────────│
       │                     │←─ HandoverFailedNotification
```
