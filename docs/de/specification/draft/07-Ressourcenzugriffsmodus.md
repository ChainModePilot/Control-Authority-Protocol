# Kapitel 7: Ressourcenzugriffsmodus

Dieses Kapitel definiert die Semantik von `Resource_Access_Mode`, die Lese-Schreib-Sperrmatrix, die Anwendung von Beschränkungsbedingungen und seine Beziehung zu Session-Berechtigungen. Dieses Kapitel entspricht der Designabsicht in §2.5 des Architekturplans.

## 7.1 Modusdefinitionen

Das CAP-Protokoll definiert 4 Zugriffsmodi:

```
AccessMode = enum["read", "write", "execute", "configure"]
```

### 7.1.1 read

**Semantik**: Nur-Lese-Zugriff auf Terminal_Resource ohne Modifikation des Ressourcenzustands.

Typische Operationen:

- Lesen von Sensor-Echtzeitdaten (Temperatur, Feuchtigkeit, Standort)
- Anzeigen von Mediendateien (Fotos, Videos)
- Abfragen von Geräte-Zustandsattributen (Akkustand, Verbindungsstatus)

**Nebenläufigkeit**: Teilbar. Mehrere Sessions KÖNNEN gleichzeitig Lesemoduszugriff auf dieselbe Ressource halten.

### 7.1.2 write

**Semantik**: Datenschreibung oder Zustandsänderung an Terminal_Resource.

Typische Operationen:

- Schreiben von Dateien auf Speichergeräte
- Ändern von Geräte-Betriebsparametern (Lautstärke, Helligkeit, Temperatureinstellung)
- Senden von Inhaltseingaben an Anwendungsfenster

**Nebenläufigkeit**: Exklusiv. Zu jedem Zeitpunkt hält höchstens eine Session den write-Modus für dieselbe Ressource.

### 7.1.3 execute

**Semantik**: Auslösen von Operationsanweisungen an Terminal_Resource.

Typische Operationen:

- Starten von Anwendungsprogrammen
- Auslösen von Hardware-Operationsanweisungen (Foto aufnehmen, Start, Drucken)
- Aufrufen von RPC-Schnittstellen

**Nebenläufigkeit**: Exklusiv.

Die Unterscheidung zwischen `write` und `execute`: write modifiziert den Datenzustand; execute löst Aktionen aus (auch wenn beide letztendlich den Zustand ändern). Die Zugriffskontrollgranularität des Endgeräts kann die beiden möglicherweise nicht streng unterscheiden; in solchen Fällen KÖNNEN Implementierungen execute als Unterklasse von write behandeln (das Halten der write-Berechtigung gewährt automatisch die execute-Fähigkeit). Die Standardsemantik dieser Spezifikation ist jedoch, dass die beiden unabhängig sind — das Halten der execute-Berechtigung gewährt **NICHT** automatisch die write-Berechtigung und umgekehrt.

### 7.1.4 configure

**Semantik**: Systemkonfigurationsänderungen auf Systemebene an Terminal_Resource.

Typische Operationen:

- Modifizieren von Geräte-Hardwareparametern (Kameraauflösung, Netzwerk-SSID)
- Anpassen von Sicherheitsrichtlinien (Firewall-Regeln, Pairing-Einstellungen)
- Ändern von Ressourcen-Zugriffsrichtlinien (z. B. `Handover_Policy`-Konfiguration)

**Nebenläufigkeit**: Exklusiv und mit hohen Privilegien. configure erfordert typischerweise eine höhere Autorisierungsstufe (siehe §7.4.3).

## 7.2 Lese-Schreib-Sperrmatrix

Die Lese-Schreib-Sperrmatrix definiert die Kompatibilität verschiedener Modi für dieselbe Ressource. Die Kompatibilität von **neuen Anfragen mit bestehender Belegung** ist wie folgt:

| Bestehende Belegung \ Neue Anfrage | read | write | execute | configure |
|------------------|:----:|:-----:|:-------:|:---------:|
| Keine            |  ✓   |   ✓   |    ✓    |     ✓     |
| read (≥1)        |  ✓   |   ✗   |    ✗    |     ✗     |
| write            |  ✗   |   ✗   |    ✗    |     ✗     |
| execute          |  ✗   |   ✗   |    ✗    |     ✗     |
| configure        |  ✗   |   ✗   |    ✗    |     ✗     |

`✓`: Kompatibel; die neue Anfrage darf sofort eine Session erstellen.
`✗`: Inkompatibel; die neue Anfrage wird abgelehnt (`E_RESOURCE_BUSY` zurückgeben).

### 7.2.1 Erläuterungen zur Matrix-Semantik

- **read gegenseitig kompatibel**: Mehrere Nur-Lese-Sessions können gleichzeitig nebeneinander existieren
- **write/execute/configure paarweise inkompatibel**: Wenn ein exklusiver Modus die Ressource belegt, werden andere exklusive Modusanfragen abgelehnt
- **read und exklusive Modi schließen sich gegenseitig aus**: Wenn read belegt, sind exklusive Anfragen verboten; wenn ein exklusiver Modus belegt, sind read-Anfragen verboten

### 7.2.2 Endgerätebehandlung

Wenn das Endgerät AuthRequest behandelt, MUSS es:

1. Die Kompatibilität gemäß der Matrix prüfen, nachdem die Validierung bestanden ist und bevor die Session in `creating` eintritt
2. Bei Inkompatibilität direkt `E_RESOURCE_BUSY` zurückgeben; **nicht** automatisch in eine Warteschlange stellen und warten
3. iFay_Runtime kann wählen, ob es wiederholt oder eine Übergabeanfrage initiiert (siehe Kapitel 6)

## 7.3 Beziehung zu Session-Berechtigungen

### 7.3.1 Berechtigungsliste

Jede Session führt bei der Erstellung eine `granted_modes`-Liste mit (siehe §2.5). Der Inhalt der Liste stammt aus den passenden `Grant.modes` in `Authorization_Descriptor.grants` oder `Trusted_Ticket.grants`.

- Wenn Fay über diese Session auf die Ressource zugreift, kann es nur die in `granted_modes` enthaltenen Modi verwenden
- Der von iFay_Runtime im AuthRequest angegebene `access_mode` MUSS eine Teilmenge der Grant-Autorisierungsliste sein

### 7.3.2 Prinzip der minimalen Berechtigung

iFay_Runtime SOLLTE im AuthRequest nur den minimalen Berechtigungsmodus anfordern, der für die aktuelle Aufgabe erforderlich ist. Wenn z. B. nur Datenlesen benötigt wird, `read` anfordern, nicht `read` + `write`.

Der Aussteller SOLLTE im Authorization_Descriptor gemäß dem Prinzip der minimalen Berechtigung autorisieren, um unnötige Berechtigungslockerung zu vermeiden.

### 7.3.3 Keine hierarchische Inklusion zwischen Modi

In CAP v1 sind die Zugriffsmodi gegenseitig unabhängig, mit **keiner** hierarchischen Inklusionsbeziehung:

- Das Halten von `configure` DARF NICHT automatisch `read` / `write` / `execute`-Fähigkeiten gewähren
- Das Halten von `write` DARF NICHT automatisch `read`-Fähigkeit gewähren
- Jeder Zugriffsmodus MUSS unabhängig autorisiert werden

Implementierungen DÜRFEN den Autorisierungsbereich NICHT automatisch erweitern.

### 7.3.4 Berechtigungen können nicht erhöht werden

Nachdem eine Session erstellt wurde, DÜRFEN ihre `granted_modes` innerhalb des Lebenszyklus NICHT erhöht werden. Wenn Fay höhere Berechtigungsmodi benötigt:

1. iFay_Runtime gibt die aktuelle Session proaktiv frei
2. Initiiert einen neuen AuthRequest mit einer Anmeldeinformation, die die erforderlichen Modi hat
3. Das Endgerät erstellt eine neue Session gemäß dem vollständigen Ablauf

Implementierungen DÜRFEN keine „Berechtigungserhöhungs"-Laufzeitschnittstelle bereitstellen.

## 7.4 Anwendung von Beschränkungen

`Grant.constraints` sind zusätzliche Einschränkungsbedingungen, die vom Aussteller auf die Autorisierung angewendet werden. Diese Spezifikation definiert die folgenden Standardbeschränkungsschlüssel, die Implementierungen MÜSSEN unterstützen. KÖNNEN benutzerdefinierte Beschränkungen unterstützen (die Semantik benutzerdefinierter Beschränkungen wird zwischen Ausstellern und Endgeräten ausgehandelt, MUSS jedoch in der Schema-Dokumentation deklariert werden).

### 7.4.1 Zeitfensterbeschränkung

```
constraints["time_window"] = "08:00-22:00"           // Täglich 8:00-22:00 gültig
constraints["time_window_tz"] = "Asia/Shanghai"     // Zeitzone, Standard UTC
```

Validierungszeitpunkt: In Schritt 6 von §3.3.2 prüfen, ob jeder Grant Beschränkungen erfüllt. Wenn die aktuelle Zeit nicht im Fenster ist → Grant stimmt nicht überein.

### 7.4.2 Frequenzbegrenzungsbeschränkung

```
constraints["max_calls_per_hour"] = "60"
constraints["max_calls_per_day"]  = "1000"
```

Das Endgerät SOLLTE Aufrufzähler für jede (descriptor_id, fay_id, resource_id)-Kombination pflegen. Bei Überschreitung des Limits → Grant stimmt nicht überein (`E_RATE_LIMIT_EXCEEDED` zurückgeben).

### 7.4.3 Hochsensible Modusbeschränkung

```
constraints["require_human_confirmation"] = "true"
```

Wenn `access_mode == "configure"` oder andere Szenarien, die diese Beschränkung erfüllen, MUSS das Endgerät vor der Session-Erstellung eine einmalige Bestätigung vom menschlichen Benutzer anfordern. Die Bestätigungs-Benutzerschnittstelle teilt sich denselben Kanal mit §6.4.3.

### 7.4.4 Geofence-Beschränkung

```
constraints["geo_fence"] = "geohash:wx4g0bm"        // Nur innerhalb des angegebenen geografischen Bereichs gültig
constraints["geo_fence_radius_m"] = "1000"
```

Die Standortquelle des Endgeräts SOLLTE kalibrierte GPS- oder Netzwerkpositionierung verwenden. Wenn der Standort nicht verfügbar ist → Grant stimmt nicht überein (konservative Ablehnung).

### 7.4.5 Behandlung nicht erkannter Beschränkungen

Wenn das Endgerät auf nicht erkannte constraints-Schlüssel stößt, MUSS es:

- Standardmäßig den Grant ablehnen (konservatives Prinzip)
- `E_UNSUPPORTED_CONSTRAINT` zurückgeben und den nicht erkannten Schlüsselnamen einschließen

Bei der Verwendung benutzerdefinierter Beschränkungen SOLLTEN Aussteller das Endgerät über andere Kanäle über die Beschränkungssemantik informieren, um Autorisierungsfehler zu vermeiden.

## 7.5 Zuordnung von Modi zur OS-Zugriffskontrolle

Das Endgerät erzwingt die access_mode-Ausführung über die Zugriffskontrollschnittstelle des Betriebssystems. Dieser Abschnitt definiert die empfohlene Zuordnung von Modi zu OS-Steuerungen. Spezifische Implementierungen können sich je nach Plattform unterscheiden, MÜSSEN jedoch die semantische Äquivalenz beibehalten.

| Ressourcentyp | read | write | execute | configure |
|---------|------|-------|---------|-----------|
| Dateien/Verzeichnisse | Dateisystem-Leseberechtigung | Dateisystem-Schreibberechtigung | Ausführungsberechtigung | Metadaten-Modifikationsberechtigung |
| Gerätedateien | Gerätelesen | Geräteschreiben | ioctl/Steuerschnittstelle | Gerätekonfigurationsregister |
| Anwendungsprozesse | Zustandsabfrage | Eingabeinjektion | Starten/Aufrufen | Startparameter modifizieren |
| Netzwerkressourcen | Daten empfangen | Daten senden | Verbindung herstellen | Routing/Firewall modifizieren |
| Kamera | Frames lesen | Nicht zutreffend | Aufnahme-/Aufzeichnungssteuerung | Auflösung/Bildrate anpassen |

Plattformspezifische Zugriffskontrollmechanismen (z. B. SELinux, AppArmor, macOS Sandbox, Android Permission) werden von der Endgeräteimplementierung gewählt und integriert.

## 7.6 Beschränkungen der Modusänderung

Dieselbe (Session, Resource) MUSS innerhalb des Lebenszyklus ihren ursprünglichen access_mode unverändert beibehalten. Modusänderungen werden durch neue Sessions realisiert:

1. Aktuelle Session freigeben
2. Beim Erstellen einer neuen Session den neuen access_mode angeben
3. Das Endgerät bewertet die Ressourcenbelegungskompatibilität gemäß der Matrix neu

Das Endgerät DARF keine „Modusumschalt"-Laufzeitschnittstelle bereitstellen.

## 7.7 Beobachtbarkeit des Ressourcenzugriffs

Das Endgerät SOLLTE wichtige Informationen für jeden Ressourcenzugriff aufzeichnen (für Audit-Zwecke):

- session_id
- fay_id
- resource_id
- access_mode
- Operationstyp (z. B. Lesegröße, Schreib-Byte-Anzahl)
- Zeitstempel

Das spezifische Audit-Protokollformat liegt außerhalb des Geltungsbereichs dieser Spezifikation (siehe Architekturplan §3.2 Ausschlusspunkt „Audit-Log-Standardformat").

## 7.8 Grenzen der Modussemantik

### 7.8.1 read-Modus verhindert nicht, dass die Ressource gleichzeitig modifiziert wird

read-Modus autorisiert nur die „Lese"-Berechtigung; er garantiert nicht, dass die Ressource während des Leseprozesses nicht von anderen Controllern modifiziert wird:

- Mehrere read-Sessions stören sich nicht gegenseitig, aber wenn die Ressource im Zustand „keine aktive Session" ist und direkt von einem menschlichen Benutzer bedient wird (Umgehung des CAP-Protokolls), können read-Sessions immer noch sich ändernde Daten lesen
- read-Modus bietet keine Transaktionskonsistenzgarantien

### 7.8.2 Atomarität des write-Modus

write-Modus bietet nur die Garantie auf Protokollebene „höchstens eine write-Session zu jedem Zeitpunkt". Die Atomarität spezifischer Schreiboperationen (z. B. teilweiser Schreibfehler) hängt von der zugrunde liegenden Semantik der Ressource ab und liegt außerhalb des Geltungsbereichs des CAP-Protokolls.

### 7.8.3 Nebenwirkungen des execute-Modus

Die durch den execute-Modus ausgelösten Operationen KÖNNEN den Ressourcenzustand ändern. Nebenwirkungen, die durch Fay über execute ausgelöst werden, liegen in der Verantwortung dieses Fay. Das CAP-Protokoll bietet keine Rollback-Fähigkeit für execute-Operationen.

## 7.9 Vollständiger Entscheidungsablauf

Vollständiger Entscheidungsablauf des Endgeräts für die Behandlung von Ressourcenzugriffen:

```
[AuthRequest trifft ein]
        │
        ▼
  ┌─────────────────────┐
  │ §3 / §4 Anmeldeinformations-Validierung │
  └──────┬──────────────┘
         │ Bestanden
         ▼
  ┌─────────────────────┐
  │ §7.4 Beschränkungs-Bewertung │
  │ (constraints)       │
  └──────┬──────────────┘
         │ Bestanden
         ▼
  ┌─────────────────────┐
  │ §7.2 Lese-Schreib-Sperrmatrix-Prüfung │
  └──────┬──────────────┘
         │ Kompatibel
         ▼
  ┌─────────────────────┐
  │ §5.2 Session erstellen │
  └──────┬──────────────┘
         │
         ▼
  ┌─────────────────────┐
  │ §7.5 OS-Zugriffskontrolle │
  │     bereitstellen   │
  └──────┬──────────────┘
         │
         ▼
  [AuthResult granted zurückgeben]
```

Ein Fehler bei einem Schritt gibt den entsprechenden Fehlercode zurück und tritt nicht in nachfolgende Schritte ein.
