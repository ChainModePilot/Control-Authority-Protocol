# CAP-Architekturplan

Das Control Authority Protocol (CAP) definiert, wie Endgeräte überprüfen, ob ein Fay (iFay oder coFay) von seinem menschlichen Wirt autorisiert wurde, um rechtmäßig auf Endgeräteressourcen zuzugreifen. Das Protokoll basiert auf einer offline gespeicherten Autorisierungsbeschreibungsdatei (Authorization_Descriptor) als Kernmechanismus, ergänzt durch ein Online-Vertrauensticket (Trusted_Ticket), und deckt Autorisierungsüberprüfung, Sitzungsverwaltung, Kontrollübergabe, Ressourcenzugriffsmodi und Aktivitätserkennung ab. Es bietet einen standardisierten Rahmen für die Kontrollberechtigung, der es intelligenten Agenten im iFay-Ökosystem ermöglicht, die Software und Hardware von Endgeräten sicher zu übernehmen.

## Glossar

- **CAP (Control Authority Protocol)**: Kontrollberechtigungsprotokoll, das definiert, wie Endgeräte überprüfen, ob ein Fay autorisiert wurde, um rechtmäßig auf Endgeräteressourcen zuzugreifen
- **iFay**: Unabhängige Fay-Entität, ein intelligenter Agent, der einer natürlichen Person (Natural_Person) zugeordnet ist
- **coFay**: Kollaborative Fay-Entität, ein kollaborativer intelligenter Agent, der einem offiziellen Posten (Official_Post) zugeordnet ist
- **Natural_Person**: Natürliche Person, das menschliche Individuum, dem ein iFay zugeordnet ist, und die ultimative Quelle der Autorisierung
- **Official_Post**: Offizieller Posten, die organisatorische Position, der ein coFay zugeordnet ist, und die ultimative Quelle der Autorisierung
- **iFay_Runtime**: iFay-Laufzeitumgebung, verantwortlich für das Lebenszyklusmanagement von Fay-Instanzen, die Planung und die Initiierung von Kontrollanfragen
- **Authorization_Descriptor**: Autorisierungsbeschreibungsdatei, eine verschlüsselte Datei, die lokal auf dem Endgerät gespeichert ist und den Ressourcenumfang, die Berechtigungstypen und die Gültigkeitsdauer beschreibt, für die ein Fay autorisiert ist – der Kernmechanismus der Offline-Autorisierung
- **Trusted_Ticket**: Vertrauensticket, ein Online-Nachweis, der im vernetzten Szenario von einem Ticket_Issuer ausgestellt wird und als ergänzender Mechanismus zur Offline-Autorisierung dient
- **Terminal_Resource**: Endgeräteressource, Hardware-Geräte oder Client-Software auf dem Endgerät, die zugänglich und bedienbar sind
- **Descriptor_Issuer**: Aussteller der Autorisierungsbeschreibung, eine vertrauenswürdige Entität, die nach Autorisierung durch eine Natural_Person oder einen Official_Post für die Erstellung und Ausstellung von Authorization_Descriptors verantwortlich ist
- **Descriptor_Validator**: Beschreibungsdatei-Validator, eine Komponente auf der Endgeräteseite, die für die Überprüfung der Legitimität und Gültigkeit von Authorization_Descriptors verantwortlich ist
- **Registration_Authority**: Registrierungsstelle, eine vertrauenswürdige Entität, die für die Verwaltung der Registrierung von Endgerätehardware, -software und -betriebssystemen sowie für die Verteilung von Verification_Keys verantwortlich ist
- **Verification_Key**: Überprüfungsschlüssel, ein Schlüssel, den das Endgerät durch Registrierung erhält und der zur Überprüfung der digitalen Signatur von Authorization_Descriptors verwendet wird
- **Protocol_Engine**: Protokoll-Engine, die Systemkomponente, die die Kernlogik des Control Authority Protocol ausführt
- **Session**: Kontrollsitzung, der vollständige Lebenszyklus von der bestandenen Autorisierungsüberprüfung bis zum Ende des Zugriffs
- **Handover_Policy**: Kontrollübergabestrategie, die die Entscheidungsregeln für die Übertragung der Kontrolle zwischen mehreren Fays oder zwischen Fays und Menschen definiert
- **Resource_Access_Mode**: Ressourcenzugriffsmodus, ein Mechanismus zur abgestuften Verwaltung des Ressourcenzugriffs nach Operationstyp (Lese-Schreib-Sperrmodell)
- **Liveness_Detection**: Aktivitätserkennung, die Erkennung, ob eine Fay-Sitzung noch aktiv ist, durch die Kombination von dauerhaften Verbindungen und Heartbeats auf Anwendungsebene
- **Capability_Matrix**: Fähigkeitsmatrix, die strukturierte Beschreibung der Kernfähigkeiten des CAP-Protokolls im Architekturplan
- **Audit_Logger**: Audit-Protokollierer, die Komponente, die für die Aufzeichnung aller Autorisierungsüberprüfungs- und Ressourcenzugriffsoperationen verantwortlich ist
