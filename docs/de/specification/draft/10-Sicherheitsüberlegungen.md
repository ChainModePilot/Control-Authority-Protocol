# Kapitel 10: Sicherheitsüberlegungen

Dieses Kapitel beschreibt das Bedrohungsmodell, bekannte Risiken und Maßnahmen des CAP-Protokolls. Dieses Kapitel ist nicht-normativer Inhalt (verwendet keine Schlüsselwörter wie MUST/SHOULD/MAY), aber die in diesem Kapitel beschriebenen Risiken und Maßnahmen SOLLTEN von Implementierern ernst genommen werden.

## 10.1 Bedrohungsmodell

Das Bedrohungsmodell des CAP-Protokolls basiert auf den folgenden Annahmen und Überlegungen.

### 10.1.1 Vertrauensannahmen

Das CAP-Protokoll geht davon aus, dass die folgenden Entitäten vertrauenswürdig sind:

1. **Registration_Authority**: Vertrauensanker; seine Schlüsselverteilungsfähigkeit ist die Grundlage der Protokollsicherheit
2. **Vorinstallierte Vertrauensanker-Schlüssel**: RA-öffentliche Schlüssel, die bei der Endgerätefertigung oder -bereitstellung vorinstalliert sind
3. **Sicherer Speicher des Endgeräts**: Hardware-/Softwaresicherheitsmodule, die zur Speicherung von Verification_Keys und lokalen Signierschlüsseln verwendet werden

Das CAP-Protokoll geht **nicht** davon aus, dass die folgenden Entitäten vertrauenswürdig sind:

1. **Fay-Instanzen**: Fay kann böswillig programmiert oder von einem Angreifer kontrolliert werden
2. **iFay_Runtime**: Kann Bugs enthalten oder ersetzt werden
3. **Netzwerkkanäle**: Können abgehört, manipuliert oder blockiert werden
4. **Unautorisierte Anwendungsprozesse**: Andere Prozesse auf dem Endgerät können versuchen, die CAP-Kontrolle zu umgehen

### 10.1.2 Angreiferfähigkeiten

In Betracht gezogene Angreifer mit den folgenden Fähigkeiten:

| Angreifertyp | Fähigkeit |
|-----------|------|
| Netzwerk-Angreifer | Netzwerknachrichten abhören, manipulieren, blockieren, wiederholen |
| Schurken-Fay | Hält legitime Anmeldeinformationen, zeigt aber abnormales Verhalten |
| Anmeldeinformations-Dieb | Erhält über andere Kanäle eine Kopie der Anmeldeinformation eines anderen |
| Insider-Bedrohung | Descriptor_Issuer selbst kompromittiert |
| Physischer Angreifer | Physischer Kontakt mit Endgerät |

Das CAP-Protokoll bietet Schutz gegen die ersten 4 Angreifertypen; die Verteidigung gegen physische Angreifer hängt von den Hardwaresicherheitsmechanismen des Endgeräts ab (wird nicht auf Protokollebene angegangen).

## 10.2 Bekannte Risiken und Maßnahmen

### 10.2.1 Anmeldeinformationsleckage-Risiko

**Risiko**: Nachdem eine Authorization_Descriptor-Datei von einem Angreifer gestohlen wurde, kann sie von einer anderen Entität als dem Fay, der die Anmeldeinformation hält, an das Endgerät übermittelt werden.

**Maßnahmen**:

- Authorization_Descriptors `subject_fay_id` ist an einen bestimmten Fay gebunden; das Endgerät verifiziert die Subjektübereinstimmung in Schritt 4 von §3.3.2
- Der Angreifer muss gleichzeitig die Laufzeit des Ziel-Fay kontrollieren (d. h. Fay selbst angreifen); Anmeldeinformationsleckage allein reicht nicht aus, um einen Angriff zu konstituieren
- Anmeldeinformationsspeicherung MUSS verschlüsselt sein (§3.2.3), um Offline-Extraktion zu verhindern
- Trusted_Ticket begrenzt das Schadensfenster durch kurze Gültigkeitsdauern (max. 7 Tage) und Echtzeit-Widerrufsabfragen

**Restrisiko**: Wenn der Angreifer den Fay-Prozess (einschließlich seiner Schlüssel) vollständig kontrolliert, kann die Anmeldeinformation immer noch missbraucht werden. Dieses Protokoll kann das Problem eines „vollständig kompromittierten Fay" nicht lösen — dies ist ein höherrangiges Fay-Integritätsschutzproblem.

### 10.2.2 Widerrufsverzögerungs-Risiko

**Risiko**: Es gibt ein Verzögerungsfenster zwischen Anmeldeinformationswiderruf und dem Erreichen der Widerrufserklärung am Endgerät, während dem die widerrufene Anmeldeinformation immer noch die Validierung bestehen kann.

**Maßnahmen**:

- Wenn das Endgerät kontinuierlich online ist, werden Widerrufserklärungen standardmäßig innerhalb von 1 Stunde synchronisiert (§3.4.5)
- Bei Verwendung von Trusted_Ticket KANN das Endgerät die Verzögerung durch Echtzeit-Widerrufsabfragen eliminieren (§4.3.2 Schritt 5)
- Die Anmeldeinformations-Gültigkeitsobergrenze (90 Tage offline / 7 Tage online) begrenzt die maximale Verzögerung
- Aussteller SOLLTEN in hochsensiblen Szenarien kürzere Gültigkeitsdauern verwenden (z. B. 1 Tag)

**Restrisiko**: Langfristig offline befindliche Endgeräte können widerrufene Anmeldeinformationen tagelang oder länger akzeptieren. Dies ist der inhärente Kompromiss des Offline-Autorisierungsmechanismus — die Wahl von Verfügbarkeit über Widerrufsaktualität.

### 10.2.3 Replay-Angriffs-Risiko

**Risiko**: Ein Angreifer fängt legitime Protokollnachrichten ab und spielt sie an das Endgerät zurück.

**Maßnahmen**:

- AuthRequest enthält `message_id` (UUID v7, zeitbezogen); das Endgerät KANN einen kurzfristigen gesehen-ID-Cache pflegen, um Wiederholungen abzulehnen
- Heartbeats `sequence_number` ist monoton steigend; Wiederholungen werden erkannt (§5.4.2)
- TLS-Kanäle bieten grundlegenden Replay-Schutz (jede TLS-Sitzung ist unabhängig)
- Die `revocation_id` der Widerrufserklärung ist eindeutig; Wiederholungen sind ungültig

**Restrisiko**: Innerhalb einer TLS-Sitzung sind Nachrichtenwiederholungen möglich, wenn ein Angreifer bereits die TLS-Verschlüsselung gebrochen hat. Dies ist eine Bedrohung auf TLS-Ebene und liegt nicht im Geltungsbereich des CAP-Protokolls.

### 10.2.4 Uhrenangriffs-Risiko

**Risiko**: Ein Angreifer umgeht Gültigkeitsdauerprüfungen durch Modifikation der Endgerätuhr oder legt uhrbezogene Schwachstellen offen.

**Maßnahmen**:

- Die Endgerät-Uhrenquelle sollte vertrauenswürdig sein (Systemuhr, NTP-Synchronisation)
- §3.6 definiert Uhrendrifterkennung: Validierung wird bei signifikanten Uhrensprüngen abgelehnt
- Endgeräte, die zu lange offline sind, SOLLTEN den Benutzer auffordern, die Uhr neu zu synchronisieren

**Restrisiko**: Ein Angreifer, der das Endgerät physisch kontrolliert, kann möglicherweise die Uhr manipulieren. Physische Sicherheit ist die Verantwortung der Endgerätehardware.

### 10.2.5 Descriptor_Issuer-Kompromittierungs-Risiko

**Risiko**: Descriptor_Issuer wird vollständig von einem Angreifer kontrolliert, der beliebige Anmeldeinformationen ausstellen kann.

**Maßnahmen**:

- §8.6.2 definiert einen Notfall-Schlüsselwiderrufsmechanismus
- Registration_Authority kann alle von Descriptor_Issuer ausgestellten Anmeldeinformationen ungültig machen, indem sie dessen Verification_Key widerruft
- Das Endgerät beendet zugehörige Sessions sofort nach Erhalt des Verification_Key-Widerrufs (§5.5)
- Die Sicherheit der Vertrauenswurzel (Registration_Authority) ist entscheidend — ihre Kompromittierung wird die gesamte Vertrauenskette zum Einsturz bringen

**Restrisiko**: Vor dem Widerruf des Verification_Key ausgestellte Anmeldeinformationen können immer noch missbraucht werden. Das Widerrufsfenster ist unvermeidlich.

### 10.2.6 Ressourcenerschöpfungsangriffs-Risiko

**Risiko**: Ein böswilliger Fay erschöpft Endgerätressourcen (Verbindungen, Sessions, Speicher) durch massive Anfragen.

**Maßnahmen**:

- §5.8 definiert Session-Mengenbegrenzungen
- §3.2.3 definiert die Anmeldeinformations-Speicherobergrenze
- Heartbeat-Timeout fordert ungültige Sessions zurück und gibt Ressourcen frei
- Das Endgerät SOLLTE Ratenbegrenzung basierend auf fay_id implementieren (zusätzlich zu §7.4.2-Frequenzbeschränkungen)

**Restrisiko**: Mehrere legitim ausgestellte Anmeldeinformationen können sich verschwören, um Ressourcen zu erschöpfen. Die Endgeräterichtlinie sollte das Gesamtkontingent eines einzelnen Fay begrenzen.

### 10.2.7 Übergabe-Race-Condition-Risiko

**Risiko**: Ein Angreifer nutzt Zwischenzustände im Übergabeablauf (z. B. den „keine aktive Session"-Zustand bei §6.5.1 [T2]), um die Ressource zu erbeuten.

**Maßnahmen**:

- Während der Übergabe ist die Ressource im Vorbelegungszustand (`handover_pending`) und akzeptiert keine anderen Session-Anfragen (§6.5.3)
- Selbst wenn ein Fehler zwischen [T2] und [T3] auftritt, tritt die Ressource klar in den „leerlauf"-Zustand ein, mit nachfolgenden Anfragen, die fair konkurrieren (§6.5.2)
- Übergabe-Timeout verhindert, dass die Ressource lange im Pending-Zustand verbleibt

**Restrisiko**: Im Extremfall eines [T3]–[T4]-Fehlers ist die Sitzung des ursprünglichen Fay nicht wiederherstellbar — AuthRequest muss neu initiiert werden. Dies sind notwendige Kosten, die die komplexere Sicherheitsanalyse vermeiden, die durch „automatische Wiederherstellung" entsteht.

### 10.2.8 Degradationsangriffs-Risiko

**Risiko**: Ein Angreifer zwingt das Endgerät in den Degradationsmodus (§4.5), indem er die Verbindung zwischen dem Endgerät und Ticket_Issuer blockiert, und umgeht so Echtzeit-Widerrufsprüfungen.

**Maßnahmen**:

- Das Endgerät lehnt im Degradationsmodus standardmäßig die Annahme neuer Trusted_Tickets ab
- Die Gültigkeitsdauer von Tickets, die bereits in lokale Authorization_Descriptors konvertiert wurden, ist kurz (max. 7 Tage)
- Das Endgerät SOLLTE iFay_Runtime hinweisen, dass es sich derzeit im Degradationsmodus befindet, sodass sensible Operationen aktiv abgelehnt werden können

**Restrisiko**: Lokal konvertierte Anmeldeinformationen während des Degradationszeitraums können immer noch verwendet werden. Dies ist der inhärente Kompromiss des Offline-First-Designs.

### 10.2.9 Seitenkanal-Angriffs-Risiko

**Risiko**: Ableiten sensibler Informationen (z. B. Existenz gültiger Anmeldeinformationen) durch Beobachtung der CAP-Protokoll-Antwortzeit oder von Fehlercode-Details.

**Maßnahmen**:

- Fehlercodes legen keine internen Details offen (§9.9)
- Implementierungen SOLLTEN über verschiedene Validierungsfehlerzweige hinweg ähnliche Antwortzeiten beibehalten
- Schlüsselfingerabdrücke, interne IDs usw. nicht in Fehlerantworten preisgeben

**Restrisiko**: Seitenkanalangriffe vollständig zu eliminieren ist äußerst schwierig. Diese Protokollebene bietet grundlegende Maßnahmen; tiefe Verteidigung hängt von der Implementierungsqualität ab.

## 10.3 Empfehlungen für Bereitstellungssicherheit

Bereitsteller, die das CAP-Protokoll implementieren, sollten die folgenden Sicherheitspraktiken berücksichtigen:

### 10.3.1 Schlüsselverwaltung

- Verwenden Sie Hardwaresicherheitsmodule (HSM, TPM, Secure Enclave) für die Speicherung privater Schlüssel
- Schlüsselrotationszeitraum nicht länger als 90 Tage (lokale Endgeräteschlüssel) / 365 Tage (Aussteller-Schlüssel)
- Etablieren Sie Notfallreaktionsverfahren für Schlüsselleckagen
- Kritische Signaturoperationen erfordern Mehrpersonengenehmigung oder Hardware-PIN

### 10.3.2 Überwachung und Auditierung

- Protokollieren Sie alle Autorisierungsvalidierungsfehlerereignisse mit Warnmeldungen
- Überwachen Sie ungewöhnliche Widerrufsanfragefrequenzen (kann auf Descriptor_Issuer-Kompromittierung hinweisen)
- Audit-Logs verwenden unabhängige persistente Speicherung mit Manipulationsschutz
- Auditieren Sie umfassend alle Operationen an hochsensiblen Ressourcen (configure-Modus, `require_human_confirmation`-Beschränkung)

### 10.3.3 Endgerätehärtung

- Aktivieren Sie OS-erzwungene Zugriffskontrolle (SELinux, AppArmor)
- Verwenden Sie Sandboxen, um iFay_Runtime von anderen Prozessen zu isolieren
- Deaktivieren Sie unnötige Systemdienste, um die Angriffsfläche zu reduzieren
- Aktualisieren Sie Systempatches regelmäßig

### 10.3.4 Netzwerksegmentierung

- Isolieren Sie Registration_Authority-Bereitstellungsnetzwerke von öffentlichen Netzwerken
- Stellen Sie Descriptor_Issuer in kontrollierten Umgebungen bereit
- Beschränken Sie die Kommunikation vom Endgerät zum Aussteller auf notwendige Ports/Protokolle
- Implementieren Sie mTLS (gegenseitige Authentifizierung), um kritische Kommunikation zu schützen

## 10.4 Protokolleinschränkungen

Diese Spezifikation behandelt explizit nicht die folgenden Sicherheitsprobleme:

1. **Fay-Integrität**: Das CAP-Protokoll geht davon aus, dass die Integrität des Codes und Verhaltens der Fay-Instanz selbst durch andere Mechanismen (z. B. Code-Signierung, Laufzeit-Integritätsmessung) gewährleistet wird
2. **Physische Sicherheit**: Wenn das Endgerät vollständig von einem physischen Angreifer kontrolliert wird, kann die Softwareebene keinen Schutz bieten
3. **Audit-Logformat**: v1 standardisiert das Audit-Logformat nicht (Architekturplan §3.2-Ausschlusspunkt)
4. **Endgeräteübergreifende Identitätsföderation**: v1 unterstützt keine endgeräteübergreifende einheitliche Identitätsverwaltung; jedes Endgerät vertraut Registration_Authority unabhängig

## 10.5 Sicherheitsentwicklung in nachfolgenden Versionen

Sicherheitsverbesserungen, die für die Einführung in zukünftigen Versionen (v2+) geplant sind:

1. Verteilter Widerrufskonsens (Architekturplan §3.2-Ausschlusspunkt)
2. Audit-Log-Standardformat (Architekturplan §3.2-Ausschlusspunkt)
3. Quantenresistente kryptografische Algorithmen (Verfolgung von NIST PQC-Standards)
4. Tiefere Integration der Zero-Trust-Prinzipien
5. Formaler Beweis der Protokollsicherheit
