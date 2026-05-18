## Kapitel 4: Externe Integrationspunkte

Dieses Kapitel beschreibt die Integrationspunkte des CAP-Protokolls mit externen Systemen und definiert die Zuständigkeitsgrenzen, Interaktionsrichtungen und Kommunikationsprotokoll-Anforderungen jedes Integrationspunkts. Das CAP-Protokoll arbeitet nicht isoliert – es muss mit iFay_Runtime, dem Endgerätebetriebssystem, Hardwaretreibern und der Registration_Authority sowie anderen externen Systemen zusammenarbeiten, um die Verwaltung der Endgerätekontrollberechtigungen gemeinsam zu bewältigen. Das Verständnis der Schnittstellengrenzen und Interaktionsmuster dieser Integrationspunkte ist die Voraussetzung für Systemintegratoren, das CAP-Protokoll korrekt zu implementieren.

### 4.1 Integration mit iFay_Runtime

iFay_Runtime ist die Laufzeitumgebung für Fay (iFay oder coFay) und verantwortlich für das Lebenszyklusmanagement und die Planung von Fay-Instanzen. Aus der Perspektive des CAP-Protokolls ist iFay_Runtime der Initiator von Kontrollanfragen – wenn ein Fay auf Endgeräteressourcen zugreifen muss, initiiert iFay_Runtime stellvertretend die Autorisierungsüberprüfungsanfrage an die CAP-Protokollschicht.

**Integrationsverantwortlichkeiten:**

- **Initiierung von Kontrollanfragen**: iFay_Runtime übermittelt stellvertretend für den Fay Autorisierungsüberprüfungsanfragen an die Protocol_Engine. Die Anfrage enthält die Fay-Identitätskennung, die Ziel-Terminal_Resource und den Autorisierungsnachweis (Authorization_Descriptor oder Trusted_Ticket)
- **Verwaltung des Fay-Lebenszyklus**: iFay_Runtime ist für das Starten, Pausieren, Fortsetzen und Beenden von Fay-Instanzen verantwortlich. Wenn eine Fay-Instanz beendet wird, benachrichtigt iFay_Runtime die Protocol_Engine, alle aktiven Sessions dieses Fay freizugeben
- **Aufrechterhaltung des Aktivitätserkennungskanals**: iFay_Runtime ist für die Aufrechterhaltung der dauerhaften Verbindung zwischen Fay und Endgerät verantwortlich und sendet in konfigurierten Intervallen Heartbeat-Nachrichten auf Anwendungsebene, um den normalen Betrieb des Liveness_Detection-Mechanismus zu unterstützen
- **Empfang von Sitzungsstatusbenachrichtigungen**: Die Protocol_Engine benachrichtigt iFay_Runtime über Sitzungsstatusänderungen (erfolgreiche Erstellung, Übergabeanfrage, erzwungene Beendigung usw.), und iFay_Runtime leitet diese an die entsprechende Fay-Instanz weiter

**Interaktionsrichtung:** Bidirektional (Bidirectional)

iFay_Runtime sendet Anfragen an die CAP-Protokollschicht (Autorisierungsüberprüfung, Sitzungsfreigabe, Heartbeat-Versand), und die CAP-Protokollschicht gibt Antworten zurück und sendet Push-Benachrichtigungen an iFay_Runtime (Überprüfungsergebnisse, Sitzungsstatusänderungen, Übergabeanfragen).

**Kommunikationsprotokoll-Anforderungen:**

- Die Kommunikation zwischen iFay_Runtime und Protocol_Engine erfolgt über ein nachrichtenbasiertes Anfrage-Antwort-Muster, wobei das Nachrichtenformat der CAP-Protokoll-Schema-Definition folgt
- Der Aktivitätserkennungskanal erfordert Unterstützung für dauerhafte Verbindungen, um Heartbeats auf Anwendungsebene und Echtzeit-Status-Push zu ermöglichen
- Der Kommunikationskanal sollte TLS-Verschlüsselung unterstützen, um die Vertraulichkeit und Integrität von Autorisierungsnachweisen und Sitzungsinformationen während der Übertragung zu gewährleisten
- Das Nachrichtenserialisierungsformat sollte mit der Schema-Definitionsdatei (schema.json) konsistent sein, um die Interoperabilität zwischen verschiedenen Implementierungen sicherzustellen

### 4.2 Integration mit dem Endgerätebetriebssystem

Das Endgerätebetriebssystem ist der Verwalter der Terminal_Resource und verantwortlich für die einheitliche Verwaltung der Software- und Hardwareressourcen auf dem Endgerät. Das CAP-Protokoll wandelt durch die Integration mit dem Betriebssystem die Ergebnisse der Autorisierungsüberprüfung in tatsächliche Ressourcenzugriffskontrolle um – das Betriebssystem erlaubt oder verweigert den Zugriff eines Fay auf bestimmte Ressourcen gemäß den Anweisungen der Protocol_Engine.

**Integrationsverantwortlichkeiten:**

- **Durchführung der Ressourcenzugriffskontrolle**: Das Betriebssystem erlaubt oder blockiert auf Systemebene den Zugriff eines Fay auf Terminal_Resources gemäß den von der Protocol_Engine erteilten Zugriffskontrollanweisungen. Dies umfasst Dateisystemzugriffskontrolle, Prozessberechtigungsverwaltung, Gerätezugriffserlaubnis usw.
- **Meldung des Ressourcenstatus**: Das Betriebssystem meldet der Protocol_Engine den aktuellen Status der Terminal_Resource (verfügbar, belegt, nicht verfügbar), damit die Protocol_Engine genaue Ressourcenzuweisungsentscheidungen treffen kann
- **Isolierung der Ausführungsumgebung**: Das Betriebssystem bietet Prozess- oder Sandbox-Isolierung für die Operationen verschiedener Fays, um sicherzustellen, dass die Operationen eines Fay den Ressourcenzugriff anderer Fays oder menschlicher Benutzer nicht beeinträchtigen
- **Weiterleitung von Ressourcenereignissen**: Wenn sich der Status einer Terminal_Resource ändert (z. B. Hardware getrennt, Software abgestürzt), leitet das Betriebssystem die Ereignisbenachrichtigung an die Protocol_Engine weiter und löst den entsprechenden Sitzungsbeendigungs- oder Ressourcenrückgewinnungsprozess aus

**Interaktionsrichtung:** Bidirektional (Bidirectional)

Die CAP-Protokollschicht sendet Zugriffskontrollanweisungen und Ressourcenabfragen an das Betriebssystem, und das Betriebssystem meldet den Ressourcenstatus und leitet Ressourcenereignisse an die CAP-Protokollschicht weiter.

**Kommunikationsprotokoll-Anforderungen:**

- Die Kommunikation zwischen CAP-Protokollschicht und Betriebssystem erfolgt über lokale Interprozesskommunikation (IPC), wobei die spezifische Methode von der Betriebssystemplattform abhängt (z. B. Unix Domain Socket, Named Pipe, D-Bus usw.)
- Zugriffskontrollanweisungen sollten synchron ausgeführt werden, um sicherzustellen, dass die Protocol_Engine das Ausführungsergebnis sofort erfährt
- Ressourcenereignisbenachrichtigungen erfolgen asynchron per Push – das Betriebssystem benachrichtigt die Protocol_Engine proaktiv bei Ressourcenstatusänderungen
- Die Kommunikationsschnittstelle sollte dem nativen Sicherheitsmodell des Betriebssystems folgen, um sicherzustellen, dass nur autorisierte Protocol_Engine-Prozesse Zugriffskontrollanweisungen erteilen können

### 4.3 Integration mit Hardwaretreibern

Hardwaretreiber sind die Zugriffsschnittstellen für Hardware-Terminal_Resources (wie Kameras, Mikrofone, GPS-Module, Speichergeräte usw.). Das CAP-Protokoll realisiert durch die Integration mit Hardwaretreibern eine feingranulare Kontrollrechteverwaltung für Hardwareressourcen. Hardwaretreiber befinden sich im Integrationssystem des CAP-Protokolls unterhalb des Betriebssystems und bieten grundlegende Zugriffsfähigkeiten für Hardwareressourcen.

**Integrationsverantwortlichkeiten:**

- **Bereitstellung von Hardwareressourcen-Zugriffsschnittstellen**: Hardwaretreiber stellen der CAP-Protokollschicht Fähigkeitsbeschreibungen und Operationsschnittstellen der Hardwareressourcen zur Verfügung, damit die Protocol_Engine die von den Hardwareressourcen unterstützten Zugriffsmodi (read, write, execute, configure) kennt
- **Durchführung der Zugriffskontrolle auf Hardwareebene**: Hardwaretreiber führen auf Hardwareebene Zugriffserlaubnis oder -verweigerung gemäß den über das Betriebssystem weitergeleiteten Kontrollanweisungen der Protocol_Engine durch. Beispielsweise: Erlauben oder Verbieten, dass ein Fay die Kamera einschaltet oder auf das Mikrofon zugreift
- **Meldung des Hardwarestatus**: Hardwaretreiber melden den Verbindungsstatus, Betriebsstatus und Fehlerinformationen der Hardwaregeräte an die obere Schicht. Diese Informationen werden letztendlich an die Protocol_Engine für Sitzungsverwaltungsentscheidungen weitergeleitet
- **Unterstützung der exklusiven Kontrolle von Hardwareressourcen**: Für Hardwareressourcen, die exklusiven Zugriff erfordern (z. B. der Videostream-Ausgang einer Kamera), stellen Hardwaretreiber auf der untersten Ebene sicher, dass zu einem bestimmten Zeitpunkt nur eine Kontrollpartei die exklusive Nutzung hat

**Interaktionsrichtung:** Unidirektional (Unidirectional) – CAP-Protokollschicht → Hardwaretreiber

Die CAP-Protokollschicht sendet Kontrollanweisungen über das Betriebssystem an die Hardwaretreiber, und die Statusinformationen der Hardwaretreiber werden über die Geräteverwaltungsschicht des Betriebssystems nach oben weitergeleitet. Die CAP-Protokollschicht stellt keinen direkten Kommunikationskanal mit Hardwaretreibern her, sondern interagiert indirekt über das Betriebssystem als Vermittler. Der Meldepfad für den Hardwarestatus ist: Hardwaretreiber → Betriebssystem → Protocol_Engine und gehört zum Betriebssystem-Integrationspunkt (4.2).

**Kommunikationsprotokoll-Anforderungen:**

- Die CAP-Protokollschicht kommuniziert nicht direkt mit Hardwaretreibern. Alle Interaktionen erfolgen indirekt über die Geräteverwaltungsschnittstellen des Betriebssystems (z. B. DeviceIoControl, ioctl, sysfs usw.)
- Die Fähigkeitsbeschreibung von Hardwareressourcen sollte einem einheitlichen Ressourcenbeschreibungsformat folgen, damit die Protocol_Engine verschiedene Arten von Hardwareressourcen auf konsistente Weise verwalten kann
- Hardwaretreiber sollten die vom Betriebssystem bereitgestellten Standard-Gerätezugriffskontrollschnittstellen unterstützen, um sicherzustellen, dass die Zugriffskontrollanweisungen des CAP-Protokolls auf Hardwareebene wirksam werden
- Für hochsensible Hardwareressourcen (z. B. Kameras, Mikrofone) sollten Hardwaretreiber Zugriffssperrmechanismen auf Hardwareebene unterstützen, um eine Umgehung der Zugriffskontrolle auf Betriebssystemebene zu verhindern

### 4.4 Integration mit Registration_Authority

Die Registration_Authority (Registrierungsstelle) ist ein Kernbestandteil der Vertrauensinfrastruktur des CAP-Protokolls und verantwortlich für die Registrierungsverwaltung von Endgerätehardware, -software und -betriebssystemen sowie für die Verteilung von Überprüfungsschlüsseln (Verification_Key) an Endgeräte. Der über die Registration_Authority erhaltene Verification_Key ist der Vertrauensanker für die Offline-Autorisierungsüberprüfung – ohne einen legitimen Verification_Key kann das Endgerät die digitale Signatur des Authorization_Descriptor nicht überprüfen.

**Integrationsverantwortlichkeiten:**

- **Endgeräteregistrierung**: Endgeräte (einschließlich Hardware, Betriebssystem und Client-Software) schließen die Registrierung über die Registration_Authority ab und erhalten eine eindeutige Endgerätekennung und anfängliche Konfigurationsinformationen
- **Verification_Key-Verteilung**: Die Registration_Authority verteilt Verification_Keys an registrierte Endgeräte. Das Endgerät verwendet diesen Schlüssel zur Überprüfung der digitalen Signatur des Authorization_Descriptor. Der Schlüsselverteilungsprozess muss über einen sicheren Kanal erfolgen, um Manipulation oder Diebstahl des Schlüssels zu verhindern
- **Schlüsselaktualisierung und -rotation**: Wenn der Verification_Key aktualisiert werden muss (z. B. Schlüsselablauf, Änderung des Ausstellers), ist die Registration_Authority für die Übermittlung des neuen Schlüssels an das Endgerät verantwortlich und koordiniert einen reibungslosen Übergang während des Schlüsselrotationsprozesses
- **Registrierungsstatusverwaltung**: Die Registration_Authority pflegt den Registrierungsstatus des Endgeräts, einschließlich Registrierung, Aussetzung und Abmeldung. Der Registrierungsstatus des Endgeräts beeinflusst, ob es Verification_Key-Aktualisierungen empfangen und am Autorisierungsüberprüfungsprozess des CAP-Protokolls teilnehmen kann

**Interaktionsrichtung:** Unidirektional (Unidirectional) – Registration_Authority → Endgerät

Die Registration_Authority verteilt Verification_Keys und Registrierungsinformationen an das Endgerät, und das Endgerät empfängt diese passiv. Das Endgerät sendet keine Autorisierungsüberprüfungsanfragen an die Registration_Authority und meldet keinen Betriebsstatus – die Autorisierungsüberprüfung des Endgeräts wird vollständig lokal durchgeführt (unter Verwendung des bereits verteilten Verification_Key), ohne eine Echtzeitkommunikation mit der Registration_Authority aufrechterhalten zu müssen. Die Registrierungsanfrage des Endgeräts an die Registration_Authority ist ein einmaliger Initialisierungsprozess und gehört nicht zur regulären Laufzeitinteraktion des CAP-Protokolls.

**Kommunikationsprotokoll-Anforderungen:**

- Die Kommunikation zwischen Registration_Authority und Endgerät muss über einen sicheren Kanal (z. B. TLS/mTLS) erfolgen, um die Vertraulichkeit und Integrität des Verification_Key während der Übertragung zu gewährleisten
- Das Schlüsselverteilungsprotokoll sollte zwei Modi unterstützen: Offline-Vorinstallation und Online-Aktualisierung. Endgeräte können bei der Auslieferung mit einem initialen Verification_Key vorinstalliert werden und erhalten anschließend Schlüsselaktualisierungen über den Online-Kanal
- Schlüsselaktualisierungsnachrichten sollten eine Versionsnummer und einen Gültigkeitszeitpunkt enthalten und einen reibungslosen Übergangszeitraum zwischen altem und neuem Schlüssel unterstützen, um Unterbrechungen der Autorisierungsüberprüfung durch Schlüsselrotation zu vermeiden
- Die Registration_Authority sollte einen Bestätigungsmechanismus für die Schlüsselverteilung bereitstellen, um sicherzustellen, dass das Endgerät den neuen Verification_Key erfolgreich empfangen und gespeichert hat

### 4.5 Übersicht der Interaktionsrichtungen und Kommunikationsprotokolle der Integrationspunkte

Die folgende Tabelle fasst die Integrationspunktinformationen des CAP-Protokolls mit 4 externen Systemen zusammen, einschließlich Interaktionsrichtung und Kommunikationsprotokoll-Anforderungen:

| Integrationspunkt | Externes System | Interaktionsrichtung | Kommunikationsprotokoll-Anforderungen |
|-------------------|----------------|---------------------|--------------------------------------|
| 4.1 | iFay_Runtime | Bidirektional (Bidirectional) | Nachrichtenbasiertes Anfrage-Antwort-Muster; Unterstützung für dauerhafte Verbindungen (Aktivitätserkennung); TLS-Verschlüsselung; Nachrichtenformat gemäß CAP-Schema-Definition |
| 4.2 | Endgerätebetriebssystem | Bidirektional (Bidirectional) | Lokale Interprozesskommunikation (IPC); synchrone Aufrufe + asynchrone Ereignis-Push; Einhaltung des nativen Sicherheitsmodells des Betriebssystems |
| 4.3 | Hardwaretreiber | Unidirektional (CAP → Hardwaretreiber) | Indirekte Kommunikation über Geräteverwaltungsschnittstellen des Betriebssystems; einheitliches Ressourcenbeschreibungsformat; Unterstützung für Zugriffssperren auf Hardwareebene |
| 4.4 | Registration_Authority | Unidirektional (RA → Endgerät) | TLS/mTLS-Sicherheitskanal; Unterstützung für Offline-Vorinstallation und Online-Aktualisierung; Schlüsselversionsverwaltung und reibungslose Rotation |

**Erläuterung der Interaktionsrichtungen:**

- **Bidirektional (Bidirectional)**: Zwischen der CAP-Protokollschicht und dem externen System bestehen bidirektionale Anfrage-Antwort- oder Ereignisbenachrichtigungs-Interaktionen, wobei beide Seiten die Kommunikation initiieren können
- **Unidirektional (Unidirectional)**: Der Informationsfluss verläuft nur von einer Seite zur anderen. In 4.3 sendet die CAP-Protokollschicht Anweisungen über das Betriebssystem an die Hardwaretreiber (keine direkte Kommunikation); in 4.4 verteilt die Registration_Authority Schlüssel und Registrierungsinformationen an das Endgerät (das Endgerät empfängt passiv)

**Designprinzipien des Kommunikationsprotokolls:**

- **Sicherheit**: Alle Kommunikationskanäle, die Autorisierungsnachweise und Schlüssel übertragen, müssen verschlüsselt sein, um Man-in-the-Middle-Angriffe und Informationslecks zu verhindern
- **Plattformanpassung**: Die Integration mit Betriebssystem und Hardwaretreibern verwendet plattformnative Schnittstellen, um die Kompatibilität auf verschiedenen Betriebssystemen sicherzustellen
- **Interoperabilität**: Die Integration mit iFay_Runtime und Registration_Authority verwendet standardisierte Nachrichtenformate (basierend auf dem CAP-Schema), um die Interoperabilität zwischen verschiedenen Implementierungen sicherzustellen
- **Fehlertoleranz**: Das Kommunikationsprotokoll sollte Wiederholungs- und Degradationsmechanismen unterstützen, um bei Kommunikationsunterbrechungen die Kernfunktionalität des CAP-Protokolls (insbesondere die Offline-Autorisierungsüberprüfung) nicht zu beeinträchtigen
