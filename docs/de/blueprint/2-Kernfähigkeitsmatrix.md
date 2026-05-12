## Kapitel 2: Kernfähigkeitsmatrix

Dieses Kapitel beschreibt die fünf Kernfähigkeiten des CAP-Protokolls auf strategischer Ebene: Offline-Autorisierung, Online-Tickets, Sitzungsverwaltung, Kontrollübergabestrategie und Ressourcenzugriffsmodi. Jede Fähigkeit wird hinsichtlich Designmotivation, Lebenszyklus, Schlüsselmechanismen und Strategiekonfiguration erläutert und bietet eine übergeordnete Orientierung für die nachfolgende Erstellung der technischen Protokollspezifikation.

### 2.1 Offline-Autorisierung (Authorization_Descriptor)

#### Designmotivation

Der Authorization_Descriptor ist der Kern-Autorisierungsmechanismus des CAP-Protokolls. Seine Designmotivation basiert auf einer grundlegenden Einschätzung: **Wenn ein Endgerät offline ist, sollte einem Fay nicht vollständig das Übernahmerecht entzogen werden, da der Fay möglicherweise zuvor von seinem menschlichen Wirt autorisiert wurde**.

In realen Szenarien befinden sich Endgeräte häufig in einem Zustand, in dem das Netzwerk nicht verfügbar oder instabil ist – Flugmodus, unterirdische Räume, abgelegene Gebiete, Netzwerkausfälle usw. Wenn die Autorisierungsüberprüfung vollständig von Online-Diensten abhängt, verliert der Fay bei einer Netzwerkunterbrechung sofort alle Kontrollrechte über Endgeräteressourcen, selbst wenn der menschliche Wirt (Natural_Person) oder der offizielle Posten (Official_Post) zuvor ausdrücklich autorisiert hat. Beispielsweise hilft der iFay eines Wildtierfotografen beim Steuern einer Drohne für Luftaufnahmen, als das Telefonsignal nach dem Flug in ein Bergtal verloren geht — ohne Offline-Autorisierungsmechanismus würde der iFay sofort die Kontrolle über die Drohne verlieren, was möglicherweise zum Absturz führen könnte. Ein solches Design würde die Verfügbarkeit des Fay stark von der Netzwerkverfügbarkeit abhängig machen, was der Vision des iFay-Ökosystems einer „nahtlosen Übernahme von Endgeräten durch intelligente Agenten" widerspricht.

Der Authorization_Descriptor persistiert die Autorisierungsbeziehung in Form einer verschlüsselten Datei lokal auf dem Endgerät, sodass das Endgerät die Autorisierungsüberprüfung auch im Offline-Zustand eigenständig durchführen kann. Diese Datei enthält den Ressourcenumfang, die Berechtigungstypen und die Gültigkeitsdauer, für die der Fay autorisiert ist. Der Descriptor_Validator auf der Endgeräteseite kann die Legitimität und Gültigkeit ohne Netzwerkverbindung überprüfen.

#### Lebenszyklusübersicht

Der Lebenszyklus des Authorization_Descriptor umfasst die folgenden 5 Phasen:

1. **Ausstellung (Issuance)**: Der Descriptor_Issuer erstellt und stellt den Authorization_Descriptor aus, nachdem er die ausdrückliche Autorisierung des Autorisierenden (Natural_Person oder Official_Post) erhalten hat. Der Ausstellungsprozess umfasst die Festlegung des Autorisierungsumfangs, die Einstellung der Gültigkeitsdauer, die Erstellung der digitalen Signatur und weitere Schritte. Nach Abschluss der Ausstellung wird der Authorization_Descriptor an die entsprechende Fay-Entität übergeben. Beispielsweise autorisiert der Benutzer Zhang San seinen iFay über die Autorisierungsverwaltungsoberfläche, die Kamera und das Mikrofon des Laptops für die nächsten 30 Tage zu nutzen, und der Descriptor_Issuer erstellt einen Authorization_Descriptor mit diesen Berechtigungen und übergibt ihn dem iFay.

2. **Lokale Speicherung (Local Storage)**: Der Fay übermittelt den erhaltenen Authorization_Descriptor an das Zielendgerät, das ihn in verschlüsselter Form in einem lokalen sicheren Speicherbereich ablegt. Die lokale Speicherung stellt sicher, dass das Endgerät auch im Offline-Zustand auf die Autorisierungsinformationen zugreifen kann, und bildet die physische Grundlage des Offline-Autorisierungsmechanismus. Beispielsweise übermittelt der iFay die Autorisierungsdatei an Zhang Sans Laptop, der sie verschlüsselt im lokalen Sicherheitschip speichert.

3. **Überprüfung (Validation)**: Wenn ein Fay den Zugriff auf Endgeräteressourcen anfordert, überprüft der Descriptor_Validator auf der Endgeräteseite den lokal gespeicherten Authorization_Descriptor. Die Überprüfung umfasst: Legitimität der digitalen Signatur (Verifizierung mittels Verification_Key), ob die Gültigkeitsdauer abgelaufen ist, ob der Autorisierungsumfang die angeforderten Ressourcen und Operationstypen abdeckt und ob er widerrufen wurde. Beispielsweise prüft das Endgerät bei der Anforderung des iFay, die Kamera zu aktivieren, ob die Signatur der Autorisierungsdatei legitim ist, ob sie innerhalb der 30-tägigen Gültigkeitsdauer liegt und ob sie die Berechtigung zur Kameranutzung enthält.

4. **Widerruf (Revocation)**: Wenn der Autorisierende die Autorisierung zurückziehen möchte, wird durch die Ausstellung einer Widerrufserklärung der entsprechende Authorization_Descriptor ungültig gemacht. Die Widerrufserklärung wird über verschiedene Kanäle an das Endgerät verteilt (siehe unten „Verteilungsstrategie für Widerrufserklärungen"). Das Endgerät wird bei nachfolgenden Überprüfungen widerrufene Authorization_Descriptors ablehnen. Beispielsweise stellt Zhang San fest, dass das Verhalten des iFay nicht seinen Erwartungen entspricht, und widerruft sofort die Kameranutzungsautorisierung; das Endgerät wird die Kamerazugriffsanfrage des iFay bei der nächsten Überprüfung ablehnen.

5. **Erneuerung (Renewal)**: Wenn ein Authorization_Descriptor kurz vor dem Ablauf steht oder der Autorisierungsumfang angepasst werden muss, stellt der Descriptor_Issuer einen neuen Authorization_Descriptor aus, der die alte Version ersetzt. Der Erneuerungsprozess ist im Wesentlichen eine neue Ausstellung; die alte Version wird nach Inkrafttreten der neuen Version automatisch ungültig. Beispielsweise steht die 30-tägige Gültigkeitsdauer kurz vor dem Ablauf, Zhang San entscheidet sich für eine Verlängerung und autorisiert den iFay zusätzlich zur Nutzung von Speichergeräten; der Descriptor_Issuer stellt eine neue Autorisierungsdatei aus, die die alte Version ersetzt.

#### Vertrauenskettenmodell

Die Vertrauenswürdigkeit der Offline-Autorisierung basiert auf einer vollständigen Vertrauenskette. Der Vertrauensübertragungspfad vom Autorisierenden zum Validator auf der Endgeräteseite ist wie folgt:

```
Autorisierender (Natural_Person / Official_Post)
    │
    │ Autorisierungsdelegation
    ▼
Descriptor_Issuer (Aussteller der Autorisierungsbeschreibung)
    │
    │ Stellt Authorization_Descriptor aus (mit digitaler Signatur)
    ▼
Fay (iFay / coFay)
    │
    │ Übermittelt Authorization_Descriptor
    ▼
Descriptor_Validator auf der Endgeräteseite
    │
    │ Überprüft Signatur mittels Verification_Key
    ▼
Überprüfungsergebnis (Bestanden / Abgelehnt)
```

Schlüsselelemente der Vertrauenskette:

- **Autorisierender → Descriptor_Issuer**: Der Autorisierende delegiert die Ausstellungsberechtigung über einen sicheren Autorisierungsdelegationsmechanismus an den Descriptor_Issuer, um sicherzustellen, dass nur autorisierte Aussteller legitime Authorization_Descriptors erstellen können
- **Descriptor_Issuer → Fay**: Der Descriptor_Issuer signiert den Authorization_Descriptor mit seinem privaten Schlüssel digital, und der Fay erhält die signierte Autorisierungsdatei
- **Fay → Descriptor_Validator**: Der Fay übermittelt den Authorization_Descriptor an das Endgerät, und der Descriptor_Validator auf der Endgeräteseite überprüft die Legitimität der Signatur mit dem von der Registration_Authority verteilten Verification_Key
- **Registration_Authority → Endgerät**: Die Registration_Authority als Vertrauensinfrastruktur ist für die Verteilung des Verification_Key an das Endgerät verantwortlich und stellt sicher, dass das Endgerät die Signaturquelle des Authorization_Descriptor überprüfen kann

#### Verteilungsstrategie für Widerrufserklärungen

Im Offline-Szenario ist die rechtzeitige Verteilung von Widerrufserklärungen eine zentrale Herausforderung. Das Endgerät muss in der Lage sein, möglichst aktuelle Widerrufsinformationen zu erhalten, ohne den Online-Widerrufsdienst in Echtzeit abfragen zu können. Das CAP-Protokoll verwendet die folgenden Verteilungsstrategien:

- **Lokale Widerrufsliste (Local Revocation List)**: Das Endgerät pflegt eine lokale Widerrufsliste, die die Kennungen bekannter widerrufener Authorization_Descriptors enthält. Bei jeder Netzwerkverbindung synchronisiert das Endgerät automatisch die neueste Widerrufsliste vom Widerrufsdienst
- **Aktive Synchronisation bei Netzwerkverbindung**: Wenn das Endgerät vom Offline- in den Online-Zustand wechselt, wird die Widerrufsliste vorrangig synchronisiert, um sicherzustellen, dass die lokalen Widerrufsinformationen möglichst aktuell sind
- **Gültigkeitsdauer als Sicherheitsnetz**: Der Authorization_Descriptor selbst enthält eine Gültigkeitsdauer. Selbst wenn die Widerrufserklärung nicht rechtzeitig zugestellt wird, wird ein abgelaufener Authorization_Descriptor automatisch ungültig, was das maximale Auswirkungsfenster der Widerrufsverzögerung begrenzt
- **Wirksamkeit bei nächster Überprüfung**: Das Endgerät prüft bei jeder Überprüfung eines Authorization_Descriptor die lokale Widerrufsliste. Sobald die Widerrufserklärung das Endgerät erreicht, werden nachfolgende Überprüfungsanfragen den widerrufenen Authorization_Descriptor sofort ablehnen

### 2.2 Online-Tickets (Trusted_Ticket)

#### Positionierung und Anwendungsszenarien

Das Trusted_Ticket ist der ergänzende Online-Autorisierungsmechanismus des CAP-Protokolls in vernetzten Umgebungen. Seine Kernpositionierung ist: **In vernetzten Umgebungen Echtzeit-Autorisierungsausstellung und Widerrufsstatus-Abfragefähigkeiten bereitzustellen**, um die Defizite der Offline-Autorisierung in Bezug auf Aktualität und Widerrufsreaktionsgeschwindigkeit auszugleichen.

Typische Anwendungsszenarien für Trusted_Ticket umfassen:

- **Temporäre Autorisierung**: Der menschliche Wirt muss einem Fay vorübergehend Zugriff auf eine bestimmte Ressource gewähren, ohne den vollständigen Ausstellungsprozess eines Authorization_Descriptor durchlaufen zu müssen. Über den Online-Dienst wird sofort ein Trusted_Ticket ausgestellt. Beispielsweise benötigt ein Benutzer vorübergehend, dass sein iFay beim Drucken eines Dokuments hilft; mit einem Fingertipp auf dem Telefon stellt der Online-Dienst sofort ein temporäres Ticket aus, das auf die Druckernutzung beschränkt ist, ohne den vollständigen Offline-Autorisierungsausstellungsprozess durchlaufen zu müssen
- **Echtzeit-Widerrufsüberprüfung**: Im vernetzten Zustand kann das Endgerät über den Trusted_Ticket-Mechanismus den Widerrufsstatus der Autorisierung in Echtzeit abfragen und erhält aktuellere Widerrufsinformationen als über die lokale Widerrufsliste. Beispielsweise hat ein Unternehmensadministrator gerade die Kontrolle eines coFay über die Konferenzraumausrüstung widerrufen; das vernetzte Endgerät kann über den Online-Ticket-Mechanismus sofort von diesem Widerruf erfahren, anstatt auf die nächste Synchronisation der lokalen Widerrufsliste warten zu müssen
- **Dynamische Berechtigungsanpassung**: In vernetzten Umgebungen kann der Autorisierungsumfang über Trusted_Ticket dynamisch angepasst werden, ohne einen Authorization_Descriptor neu ausstellen zu müssen. Beispielsweise hatte ein iFay ursprünglich nur Leseberechtigung für Dateien; der Benutzer erhöht vorübergehend seine Berechtigung auf Schreiben über sein Telefon, was sofort über ein Online-Ticket wirksam wird

#### Beziehung zum Authorization_Descriptor

Die Beziehung zwischen Trusted_Ticket und Authorization_Descriptor ist eine Ergänzungsbeziehung, keine Ersetzungsbeziehung:

- **Authorization_Descriptor ist der Kern**: Die Offline-Autorisierung ist stets der grundlegende Mechanismus des CAP-Protokolls. Das Trusted_Ticket kann nicht unabhängig vom Authorization_Descriptor-System existieren
- **Online-Ausstellung kann in Offline-Format konvertiert werden**: Online ausgestellte Trusted_Tickets können in das lokale Authorization_Descriptor-Format konvertiert werden, um offline genutzt zu werden. Dies bedeutet, dass über Trusted_Ticket erhaltene Autorisierungen nicht sofort bei einer Netzwerkunterbrechung ungültig werden
- **Prioritätsbeziehung**: Wenn das Endgerät gleichzeitig ein Trusted_Ticket und einen Authorization_Descriptor besitzt, werden vorrangig die Echtzeit-Autorisierungsinformationen des Trusted_Ticket verwendet, da diese aktueller sind

#### Degradationsstrategie von Online zu Offline

Wenn Online-Dienste nicht verfügbar sind, führt das CAP-Protokoll eine nahtlose Degradationsstrategie durch:

1. **Erkennung des Online-Dienststatus**: Das Endgerät überwacht kontinuierlich den Verbindungsstatus zum Online-Autorisierungsdienst
2. **Automatischer Rückfall**: Wenn der Online-Dienst nicht verfügbar ist, fällt das Endgerät automatisch auf die lokale Authorization_Descriptor-Überprüfung zurück, ohne den Ressourcenzugriff des Fay zu unterbrechen
3. **Trusted_Ticket-Konvertierung**: Während der Netzwerkverbindung ausgestellte Trusted_Tickets wurden bereits in das lokale Authorization_Descriptor-Format konvertiert und gespeichert, um die Nutzung nach der Degradation sicherzustellen
4. **Synchronisation nach Wiederherstellung**: Wenn der Online-Dienst wieder verfügbar ist, stellt das Endgerät automatisch die Nutzung des Trusted_Ticket-Mechanismus wieder her und synchronisiert möglicherweise während der Offline-Phase versäumte Widerrufsinformationen

Der Degradationsprozess ist für den Fay transparent – der Fay muss nicht wahrnehmen, ob das Endgerät derzeit Online-Tickets oder Offline-Autorisierung verwendet. Das Ergebnis der Autorisierungsüberprüfung bleibt in beiden Modi konsistent.

### 2.3 Sitzungsverwaltung (Session)

#### Lebenszyklus

Die Session (Kontrollsitzung) ist der vollständige Lebenszyklus von der bestandenen Autorisierungsüberprüfung bis zum Ende des Zugriffs und umfasst die folgenden 3 Phasen:

1. **Erstellung (Creation)**: Nachdem die Autorisierungsüberprüfung eines Fay bestanden wurde, erstellt die Protocol_Engine eine Session-Instanz. Bei der Erstellung werden die zugehörige Fay-Kennung, die Ziel-Terminal_Resource, die Liste der autorisierten Berechtigungen und der Erstellungszeitpunkt erfasst. Nach erfolgreicher Erstellung erhält der Fay die Kontrolle über die Zielressource. Beispielsweise fordert ein iFay die Nutzung des Browsers auf dem Laptop an; nach bestandener Autorisierungsüberprüfung erstellt das System eine an den Browser gebundene Kontrollsitzung, und der iFay kann ab diesem Zeitpunkt den Browser bedienen.

2. **Aktiv (Active)**: Nach der Erstellung geht die Session in den aktiven Zustand über, und der Fay kann innerhalb des autorisierten Berechtigungsumfangs Operationen an der gebundenen Terminal_Resource durchführen. Während der aktiven Phase überwacht der Liveness_Detection-Mechanismus kontinuierlich den Aktivitätsstatus der Sitzung, um sicherzustellen, dass der Fay die Sitzung noch aktiv nutzt. Beispielsweise sucht der iFay über den Browser Fluginformationen für den Benutzer und sendet dabei kontinuierlich Heartbeats, um anzuzeigen, dass er noch aktiv in Nutzung ist.

3. **Beendigung (Termination)**: Eine Session kann auf folgende Weise beendet werden:
   - **Aktive Freigabe**: Der Fay gibt die Session nach Abschluss der Operationen aktiv frei und gibt die Kontrolle über die Terminal_Resource zurück. Beispielsweise gibt der iFay nach Abschluss der Flugsuche aktiv die Browserkontrolle frei
   - **Zeitüberschreitungsbeendigung**: Die Liveness_Detection erkennt, dass die Fay-Sitzung nicht mehr aktiv ist (Heartbeat-Zeitüberschreitung), und beendet die Session automatisch, wobei die Ressourcen zurückgewonnen werden. Beispielsweise hört der iFay aufgrund eines Laufzeitabsturzes auf, Heartbeats zu senden, und das Endgerät gewinnt nach der Zeitüberschreitung automatisch die Browserkontrolle zurück
   - **Widerrufsbeendigung**: Wenn die Autorisierung widerrufen wird, werden die zugehörigen aktiven Sessions zwangsweise beendet. Beispielsweise entdeckt der Benutzer, dass der iFay unangemessene Inhalte durchsucht, und widerruft sofort die Autorisierung; die Browsersitzung wird zwangsweise beendet

Nach Beendigung der Session wird die Kontrolle des Fay über die Terminal_Resource sofort aufgehoben, und die Ressource kehrt in einen Zustand zurück, in dem sie von anderen Kontrollparteien erworben werden kann.

#### Bindungsbeziehung mit Terminal_Resource

Zwischen Session und Terminal_Resource besteht eine strenge Bindungsbeziehung: **Eine aktive Session entspricht der Kontrolle über eine Terminal_Resource**.

Die Kernregeln dieser Bindungsbeziehung:

- **Eins-zu-eins-Bindung**: Jede aktive Session ist an eine bestimmte Terminal_Resource gebunden, und der Fay führt über die Session Operationen an dieser Ressource durch. Beispielsweise erhält ein iFay eine an die „Frontkamera" gebundene Session — er kann nur die Frontkamera über diese Session bedienen und sie nicht zur Bedienung des Mikrofons verwenden
- **Exklusive Kontrolle**: Für Operationsmodi, die exklusiven Zugriff erfordern (write, execute, configure), kann dieselbe Terminal_Resource zu einem bestimmten Zeitpunkt höchstens eine aktive exklusive Session haben. Beispielsweise kann iFay-B keine write-Session für die SD-Karte erhalten, wenn iFay-A gerade im write-Modus Daten auf die SD-Karte schreibt
- **Paralleles Lesen**: Für den read-Modus können gleichzeitig mehrere aktive Nur-Lese-Sessions für dieselbe Terminal_Resource existieren. Beispielsweise können iFay-A und iFay-B gleichzeitig im read-Modus Positionsdaten vom GPS-Modul lesen
- **Lebenszyklusverknüpfung**: Wenn eine Terminal_Resource nicht verfügbar wird (z. B. Hardware getrennt), werden alle an diese Ressource gebundenen aktiven Sessions beendet. Beispielsweise werden nach dem Abziehen einer USB-Kamera durch den Benutzer alle an diese Kamera gebundenen Sessions automatisch beendet

#### Aktivitätserkennungsmechanismus

Die Liveness_Detection (Aktivitätserkennung) erkennt durch die Kombination von dauerhaften Verbindungen und Heartbeats auf Anwendungsebene, ob eine Fay-Sitzung noch aktiv ist. Ihr Designziel ist die rechtzeitige Rückgewinnung von Ressourcen, die von abgelaufenen Sitzungen belegt werden.

Funktionsweise der Aktivitätserkennung:

- **Aufrechterhaltung der dauerhaften Verbindung**: Zwischen Fay und Endgerät wird über eine dauerhafte Verbindung ein Kommunikationskanal aufrechterhalten, dessen Verbindungsstatus als Basissignal für die Aktivitätserkennung dient
- **Heartbeats auf Anwendungsebene**: Über die dauerhafte Verbindung hinaus sendet der Fay regelmäßig Heartbeat-Nachrichten auf Anwendungsebene, um anzuzeigen, dass er die aktuelle Session noch aktiv nutzt. Heartbeat-Intervall und Zeitüberschreitungsschwelle sind konfigurierbar
- **Doppelte Bewertung**: Erst wenn sowohl die dauerhafte Verbindung unterbrochen als auch der Heartbeat auf Anwendungsebene abgelaufen ist, wird die Session als abgelaufen eingestuft. Dieser doppelte Bewertungsmechanismus vermeidet Fehleinschätzungen durch kurzzeitige Netzwerkschwankungen
- **Ressourcenrückgewinnung**: Wenn eine Session als abgelaufen eingestuft wird, beendet die Protocol_Engine die Session automatisch und gibt die belegte Terminal_Resource frei, sodass die Ressource von anderen Kontrollparteien erworben werden kann

### 2.4 Kontrollübergabestrategie (Handover_Policy)

#### Kernszenarien

Die Kontrollübergabe findet in Szenarien statt, in denen mehrere Kontrollparteien nacheinander dieselbe Terminal_Resource nutzen müssen. Das CAP-Protokoll definiert zwei Kernübergabeszenarien:

- **Kontrollübertragung zwischen Fays**: Ein Fay überträgt seine Kontrolle über eine Terminal_Resource an einen anderen Fay. Beispielsweise übergibt ein für die Datenerfassung zuständiger iFay nach Abschluss der Aufgabe die Kamerakontrolle an einen für Videoanrufe zuständigen iFay; oder in einer intelligenten Fabrik übergibt ein für die Qualitätskontrolle zuständiger coFay nach dem Fotografieren der Produkte die Kontrolle über die Industriekamera an den für die Verpackung zuständigen coFay
- **Kontrollübertragung zwischen Fay und menschlichem Benutzer**: Ein Fay gibt die Kontrolle an den menschlichen Benutzer zurück, oder der menschliche Benutzer delegiert die Kontrolle an einen Fay. Beispielsweise wird eine Drohne autonom von einem iFay gesteuert, als der Pilot abnormales Wetter bemerkt und die manuelle Kontrolle übernehmen muss — der iFay gibt sofort die Flugkontrolle ab; nachdem sich das Wetter gebessert hat, gibt der Pilot die Kontrolle an den iFay zurück, um den autonomen Flug fortzusetzen. Ein weiteres Beispiel: Ein Benutzer bearbeitet manuell ein Dokument und benötigt den iFay zur Hilfe bei der Formatierung, wobei er vorübergehend die Kontrolle über den Dokumenteneditor an den iFay übergibt; nach Abschluss der Formatierung gibt der iFay die Kontrolle zurück

#### Strategische Konfigurationsfähigkeiten

Die Handover_Policy bietet strategische Konfigurationsfähigkeiten und unterstützt die folgenden 3 Strategietypen:

1. **Prioritätsregelskript (Priority Rule Script)**: Bestimmung der Übergabepriorität durch vordefinierte Regelskripte. Die Regelskripte berechnen Prioritätswerte basierend auf Faktoren wie der Rolle des Fay, der Aufgabendringlichkeit und dem Ressourcentyp. Die Kontrollpartei mit der höchsten Priorität erhält die Ressourcenkontrolle. Diese Strategie eignet sich für Szenarien, in denen die Übergaberegeln klar und vorab definierbar sind. Beispielsweise hat in einem medizinischen Szenario der für Notfallalarme zuständige iFay immer eine höhere Priorität als der für die routinemäßige Datenerfassung zuständige iFay; wenn beide gleichzeitig das Kommunikationsmodul anfordern, erhält der Notfallalarm-iFay automatisch die Kontrolle.

2. **KI-Modell-Echtzeitentscheidung (AI Model Real-time Decision)**: Ein KI-Modell entscheidet in Echtzeit über die Kontrollzuweisung basierend auf dem aktuellen Kontext. Das KI-Modell berücksichtigt den Aufgabenstatus mehrerer Kontrollparteien, die Dringlichkeit der Ressourcennutzung, historische Übergabemuster und andere Faktoren. Diese Strategie eignet sich für dynamische Szenarien, in denen die Übergaberegeln komplex sind und sich nicht durch statische Regeln abdecken lassen. Beispielsweise fordern in einem Smart Home ein für die Sicherheitsüberwachung zuständiger iFay und ein für Videoanrufe zuständiger iFay gleichzeitig die Kamera an; das KI-Modell bestimmt die Priorität in Echtzeit basierend auf Kontextinformationen wie dem Vorhandensein einer aktuellen Sicherheitsbedrohung und der Dringlichkeit des Anrufs.

3. **Entscheidung durch menschlichen Benutzer (Human User Decision)**: Die Entscheidungsbefugnis über die Kontrollübergabe wird dem menschlichen Benutzer übertragen. Wenn mehrere Kontrollparteien um dieselbe Ressource konkurrieren, zeigt das Endgerät dem menschlichen Benutzer eine Auswahlschnittstelle an, und der menschliche Benutzer entscheidet, welcher Partei die Kontrolle gewährt wird. Diese Strategie eignet sich für hochsensible Ressourcen oder Szenarien, die menschliches Urteilsvermögen erfordern. Beispielsweise fordern zwei iFays gleichzeitig die Nutzung der Banking-App des Benutzers an; das Endgerät zeigt eine Aufforderung an, damit der Benutzer selbst entscheidet, welchem iFay die Autorisierung erteilt wird, da Finanzoperationen eine menschliche Endkontrolle erfordern.

Die drei Strategietypen können auf der Granularitätsebene der Terminal_Resource unabhängig konfiguriert werden. Verschiedene Ressourcen auf demselben Endgerät können unterschiedliche Übergabestrategien verwenden.

#### Atomaritätsgarantie

Der Kontrollübergabeprozess muss die Atomaritätsanforderung erfüllen: **Zu jedem Zeitpunkt hat eine Terminal_Resource höchstens eine aktive Kontrollpartei**.

Umsetzungsprinzipien der Atomaritätsgarantie:

- **Erst freigeben, dann erwerben**: Während des Übergabeprozesses wird die Session der ursprünglichen Kontrollpartei zuerst beendet, dann wird die Session der neuen Kontrollpartei erstellt. Dies stellt sicher, dass nicht zwei Kontrollparteien gleichzeitig die Kontrolle über dieselbe Ressource halten. Beispielsweise muss bei der Übergabe der Flugkontrolle einer Drohne von iFay-A an iFay-B die Kontrollsitzung von iFay-A zuerst beendet werden, bevor die Kontrollsitzung von iFay-B erstellt werden kann — es wird niemals eine Situation geben, in der zwei iFays gleichzeitig den Flug der Drohne steuern
- **Unteilbarkeit**: Die Übergabeoperation wird als Ganzes ausgeführt – entweder vollständig erfolgreich (Freigabe durch die ursprüngliche Kontrollpartei + Erwerb durch die neue Kontrollpartei) oder vollständig fehlgeschlagen (Rollback auf den Zustand vor der Übergabe)
- **Keine Offenlegung von Zwischenzuständen**: Externe Beobachter sehen zu jedem Zeitpunkt einen konsistenten Ressourcenkontrollstatus und beobachten keine Zwischenzustände während des Übergabeprozesses

#### Zeitüberschreitungsbehandlungsprinzipien

Der Kontrollübergabeprozess kann aus verschiedenen Gründen (Netzwerkverzögerung, keine Antwort der Kontrollpartei usw.) nicht innerhalb der erwarteten Zeit abgeschlossen werden. Das Behandlungsprinzip des CAP-Protokolls für Übergabe-Zeitüberschreitungen lautet: **Eine nicht innerhalb der Frist abgeschlossene Übergabe sollte auf den Zustand vor der Übergabe zurückgesetzt werden, wobei die Session der ursprünglichen Kontrollpartei unverändert bleibt**.

Spezifische Behandlungsregeln:

- **Konfigurierbare Zeitüberschreitungsschwelle**: Jede Übergabestrategie kann eine unabhängige Zeitüberschreitungsschwelle konfigurieren, um den Zeitanforderungen verschiedener Szenarien gerecht zu werden
- **Rollback-Schutz**: Nach Auslösung der Zeitüberschreitung wird die Übergabeoperation automatisch zurückgesetzt, und die Session der ursprünglichen Kontrollpartei bleibt im aktiven Zustand unverändert. Beispielsweise kontrolliert iFay-A die Kamera und versucht, die Kontrolle an iFay-B zu übergeben, aber iFay-B kann aufgrund von Netzwerkverzögerungen nicht innerhalb der vorgegebenen Zeit antworten; die Übergabe wird automatisch zurückgesetzt, und iFay-A behält weiterhin die Kontrolle über die Kamera
- **Benachrichtigungsmechanismus**: Nach dem Zeitüberschreitungs-Rollback sendet die Protocol_Engine eine Benachrichtigung über das Übergabeversagen an die betroffenen Kontrollparteien mit Angabe des Zeitüberschreitungsgrundes
- **Wiederholungsstrategie**: Nach einem Übergabeversagen kann je nach Strategiekonfiguration entschieden werden, ob automatisch ein erneuter Versuch gestartet oder auf manuelles Eingreifen gewartet wird

### 2.5 Ressourcenzugriffsmodus (Resource_Access_Mode)

#### Stufenmodell

Der Resource_Access_Mode unterteilt den Ressourcenzugriff nach Operationstyp in 4 Modi und bildet eine Berechtigungsabstufung von niedrig nach hoch:

1. **read (Lesen)**: Nur-Lese-Zugriff auf die Terminal_Resource ohne Änderung des Ressourcenstatus. Beispiele: Lesen von Echtzeit-Temperatursensordaten, Anzeigen von Fotos im Benutzeralbum, Abrufen der aktuellen Positionsinformationen des GPS-Moduls. Der read-Modus ist teilbar und erlaubt mehreren Fays den gleichzeitigen Zugriff auf dieselbe Ressource im read-Modus — beispielsweise können mehrere iFays gleichzeitig Daten desselben Temperatursensors lesen, ohne sich gegenseitig zu beeinträchtigen.

2. **write (Schreiben)**: Datenschreibung oder Statusänderung an der Terminal_Resource. Beispiele: Speichern aufgenommener Fotos auf einem Speichergerät, Ändern des Temperatureinstellwerts eines Smart-Home-Geräts, Schreiben von Inhalten in ein Dokument. Der write-Modus ist exklusiv – zu einem bestimmten Zeitpunkt darf nur eine Kontrollpartei im write-Modus auf die Ressource zugreifen — beispielsweise können zwei iFays nicht gleichzeitig Daten in dieselbe Datei schreiben, da dies zu Datenbeschädigung führen würde.

3. **execute (Ausführen)**: Ausführung von Operationsbefehlen an der Terminal_Resource. Beispiele: Starten einer Navigations-App auf dem Telefon, Auslösen des Startbefehls einer Drohne, Ausführen eines Druckauftrags auf einem Drucker. Der execute-Modus ist exklusiv und stellt sicher, dass die Ausführung von Operationsbefehlen nicht durch andere Kontrollparteien gestört wird — beispielsweise kann ein anderer iFay nicht gleichzeitig einen Startbefehl senden, wenn ein iFay eine Drohne zur Ausführung eines Landevorgangs steuert.

4. **configure (Konfigurieren)**: Systemebene-Konfigurationsänderungen an der Terminal_Resource. Beispiele: Ändern der Auflösungs- und Bildfrequenzparameter der Kamera, Anpassen der Netzwerk-Firewall-Regeln, Ändern der Bluetooth-Modul-Kopplungseinstellungen. Der configure-Modus ist exklusiv und erfordert höhere Berechtigungen – in der Regel ist eine höhere Autorisierungsstufe für die Ausführung erforderlich — beispielsweise erfordert die Änderung der Sicherheitsrichtlinie eines Routers eine höhere Autorisierungsstufe als das bloße Lesen des Netzwerkstatus.

#### Lese-Schreib-Sperr-Wesen

Die Parallelitätskontrolle des Resource_Access_Mode ist im Wesentlichen ein Lese-Schreib-Sperrmodell (Read-Write Lock):

- **read-Operationen erlauben parallelen Zugriff mehrerer Fays**: Mehrere Fays können gleichzeitig Sessions im read-Modus für dieselbe Terminal_Resource halten, ohne sich gegenseitig zu beeinträchtigen. Dies entspricht der gemeinsamen Sperre (Shared Lock) im Lese-Schreib-Sperrmodell. Beispielsweise können drei iFays gleichzeitig Temperatur- und Feuchtigkeitsdaten desselben Wettersensors lesen, ohne sich gegenseitig zu blockieren
- **write-, execute- und configure-Operationen erfordern exklusive Kontrolle**: Wenn ein Fay im write-, execute- oder configure-Modus auf eine Ressource zugreift, kann kein anderer Fay gleichzeitig in einem Schreibmodus auf diese Ressource zugreifen. Dies entspricht der exklusiven Sperre (Exclusive Lock) im Lese-Schreib-Sperrmodell. Beispielsweise kann ein anderer iFay nicht gleichzeitig Fotos auf dieselbe SD-Karte schreiben, wenn ein iFay gerade eine Videodatei auf die SD-Karte schreibt
- **Lese-Schreib-Gegenseitiger Ausschluss**: Wenn auf einer Ressource eine aktive write-, execute- oder configure-Session existiert, werden neue read-Anfragen ebenfalls blockiert, bis die exklusive Session freigegeben wird. Umgekehrt warten write-, execute- und configure-Anfragen, wenn aktive read-Sessions auf der Ressource existieren, bis alle read-Sessions beendet sind. Beispielsweise muss ein anderer iFay, der eine Konfigurationsdatei lesen möchte, ebenfalls warten, bis der Schreibvorgang abgeschlossen ist, wenn ein iFay gerade diese Konfigurationsdatei ändert (write-Modus), um das Lesen eines inkonsistenten Zwischenzustands zu vermeiden

Dieses Lese-Schreib-Sperrmodell maximiert die Parallelitätsleistung von read-Operationen bei gleichzeitiger Gewährleistung der Datenkonsistenz und Operationssicherheit.

#### Beziehung zu Session-Berechtigungen

Der Resource_Access_Mode ist eng mit den Autorisierungsberechtigungen der Session verknüpft: **Die Autorisierungsberechtigungsliste der Session bestimmt die Operationstypen, die ein Fay ausführen kann**.

Die spezifischen Beziehungen sind wie folgt:

- **Berechtigungslistenbeschränkung**: Jede Session trägt bei der Erstellung eine Autorisierungsberechtigungsliste, die aus dem im Authorization_Descriptor oder Trusted_Ticket definierten Berechtigungsumfang stammt. Der Fay kann nur in den in der Berechtigungsliste enthaltenen Modi auf die Ressource zugreifen
- **Prinzip der minimalen Berechtigung**: Die Berechtigungsliste der Session sollte dem Prinzip der minimalen Berechtigung folgen und nur die niedrigsten Berechtigungsmodi enthalten, die der Fay zur Erfüllung der aktuellen Aufgabe benötigt
- **Keine Berechtigungserhöhung**: Nach der Erstellung der Session kann die Berechtigungsliste während des Sitzungslebenszyklus nicht erhöht werden. Wenn der Fay einen höheren Zugriffsmodus benötigt, muss er durch eine neue Autorisierungsüberprüfung eine neue Session erstellen
- **Keine hierarchische Berechtigungseinbeziehung**: Höhere Berechtigungsmodi beinhalten nicht automatisch die Fähigkeiten niedrigerer Berechtigungsmodi. Beispielsweise bedeutet der Besitz der configure-Berechtigung nicht automatisch den Besitz der read-Berechtigung – jeder Zugriffsmodus erfordert eine unabhängige Autorisierung
