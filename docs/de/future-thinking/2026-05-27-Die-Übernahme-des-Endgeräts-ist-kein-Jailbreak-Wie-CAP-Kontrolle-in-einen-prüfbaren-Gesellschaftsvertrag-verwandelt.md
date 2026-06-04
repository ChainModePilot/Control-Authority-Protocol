<!-- Aktualisierungsdatum: 2026-05-27 -->

# Die Übernahme des Endgeräts ist kein Jailbreak: Wie CAP Kontrolle in einen prüfbaren Gesellschaftsvertrag verwandelt

Im öffentlichen Diskurs klingt der Satz „KI übernimmt das Endgerät" wie eine Katastrophenwarnung.
Denn den meisten Menschen kommt nur ein Bild in den Sinn: Du schläfst, und sie behandelt dein Telefon wie ihr eigenes; du siehst nicht hin, und sie behandelt deinen Computer wie ihren eigenen; du hast nicht autorisiert, und sie kann trotzdem „nebenbei ein paar Dinge erledigen".

Diese Furcht ist nicht aus der Luft gegriffen.
In den letzten zwei Jahren haben wir zu viele Vorfälle gesehen: außer Kontrolle geratene Berechtigungen, versehentlich gelöschte Daten, kompetenzüberschreitende Aufrufe, Black-Box-Entscheidungen, fehlende Verantwortungszuweisung.
In dem Moment, in dem KI vom „Beantworten von Fragen" zum „Handeln nach außen" wechselt, fällt die Toleranz der Gesellschaft ihr gegenüber sofort auf null.

Daher betone ich immer wieder: **KI darf nicht sich selbst überlassen werden; sie muss unter menschlicher Vormundschaft handeln.**
Doch wenn dieser Satz nur eine Haltung bleibt, ändert er nichts an der Realität. Man muss ihn in das Protokoll schreiben, in die Laufzeit, in jedes Tor der Endgerätekontrolle.

Das Control Authority Protocol (CAP) ist genau ein solches „Tor-Protokoll".
Es ist nicht dafür verantwortlich, KI klüger zu machen; es ist nur dafür verantwortlich, eine extrem schlichte Frage zu beantworten, die dennoch entscheidet, ob die Gesellschaft KI akzeptieren kann:

**Ist es diesem Fay (iFay oder coFay) erlaubt, diese Sache zu tun?**

---

## 1. CAPs einzelne Verantwortung im iFay-Ökosystem: Autorisierungsverifikation und Kontrollrechtsverwaltung

CAPs Positionierung ist zurückhaltend, aber unnachgiebig:

Es trägt eine einzelne Kernverantwortung — **zu verifizieren, ob ein Fay die Autorisierung einer Human Prime (des natürlichen Verantwortungssubjekts) oder eines Official Post (organisatorische Position / öffentliche Rolle) erhalten hat, um damit rechtmäßig auf Endgeräteressourcen zuzugreifen**.

Dieser Satz lässt sich in fünf technische Aktionen zerlegen:

1) **Autorisierungsverifikation**: Das Endgerät verifiziert, ob die vom Fay mitgeführte Autorisierungs-Anmeldeinformation legitim, gültig und nicht widerrufen ist.
2) **Sitzungsverwaltung**: Nach bestandener Verifikation wird eine Kontrollsitzung etabliert und ihr vollständiger Lebenszyklus verwaltet.
3) **Kontrollrechtskoordination**: Koordination der Übergabe der Ressourcenkontrolle zwischen Mensch und Fay sowie zwischen Fays.
4) **Abgestufter Ressourcenzugriff**: Stratifizierung des Ressourcenzugriffs in read/write/execute/configure, um das Muster „einmal autorisiert, alles offen" zu vermeiden.
5) **Aktivitätserkennung**: Heartbeat-Erkennung und Timeout-Rückgewinnung, um zu verhindern, dass „Zombie-Sitzungen" Endgeräteressourcen langfristig blockieren.

Ich nenne es ein „Gesellschaftsvertrags-Protokoll", weil es „kannst du das tun oder nicht" vom psychologischen Hinweis einer UI in eine sachliche Prüfung auf Systemebene verwandelt.

---

## 2. Was CAP ausdrücklich nicht tut: Fähigkeit, Identität und Intelligenz fallen nicht in seine Zuständigkeit

Ein gesundes Protokoll muss wissen, was es nicht tut; sonst wird es zum Universalkleber, der schließlich außer Kontrolle gerät.

CAP ist ausdrücklich nicht zuständig für:

- Fays Identitätserstellung und -verwaltung (das ist das FayID/Identitätssystem).
- Fays intelligentes Schlussfolgern und Planen (das ist die Ego/Denkschicht).
- Die Geschäftslogik der Endgeräteressourcen (die gehört dem Endgerät selbst).
- Die Implementierung der unteren Netzwerkübertragung (WebSocket/gRPC sind unwichtig).
- Interne Sicherheitsmechanismen des Betriebssystems (das eigene Sandbox-/Berechtigungsmodell des OS wird nicht von CAP ersetzt).

Je klarer CAPs Verantwortungsgrenze ist, desto eher kann es zur prüfbaren Governance-Infrastruktur werden, statt zu einem „Sicherheits-Patch", der von geschäftlichen Gründen erodiert wird.

---

## 3. Warum „Kontrollrecht" zu einem Protokoll gemacht werden muss: Sonst kann Verantwortung niemals durchdrungen werden

Die häufigste Form der Autorisierung heute ist:
Eine App zeigt einen Dialog, du tippst „Erlauben", und dann hat sie „dauerhafte Erlaubnis".

Selbst im Zeitalter, in dem Menschen Software benutzen, war das schon schlimm genug;
im Zeitalter, in dem Fays Endgeräte übernehmen, wird das zur regelrechten Katastrophe.

Denn Fays Handlungen sind kein einzelner Klick, sondern ein kontinuierliches Verhalten:
Eine App öffnen, Daten lesen, Urteile bilden, Aktionen ausführen, Ausnahmen behandeln, weiter ausführen…
Wenn dieses Verhalten keine klare Grenze namens „Kontrollsitzung" hat, verdunstet die Verantwortung mit der Zeit.

Indem CAP das Kontrollrecht zu einem Protokoll macht, liefert es zwei Kernwerte:

1) **Autorisierung von einem psychologischen Gefühl in eine verifizierbare Anmeldeinformation verwandeln**
2) **Aktionen von verstreuten Operationen in prüfbare Sitzungen verwandeln**

Wenn ein Vorfall eintritt, kannst du endlich beantworten:
Wer hat wann und in welchem Autorisierungsumfang diese Sitzung initiiert? Wie lange hat die Sitzung gedauert? An welchen Ressourcen wurde gearbeitet? Wer hat sie schließlich widerrufen / wer hat sie nach Timeout zurückgewonnen?

Dies ist die Voraussetzung für „Verantwortungsdurchdringung".

---

## 4. Offline-zuerst, Online-Hilfe: CAPs Realismus

Ich stimme CAPs Kerndesignprinzip nachdrücklich zu: **offline-zuerst, online-Hilfe**.

Dahinter steht ein realistisches Urteil:
Netzwerkausfälle sollten weder dem Menschen das Kontrollrecht nehmen, noch sollten sie Fays die Verfügbarkeit unter Vormundschaft nehmen.

CAP trägt dieses Urteil mit zwei Mechanismen:

- **Offline-Autorisierung (Authorization_Descriptor)**: Verschlüsselte Datei, lokal verifizierbar.
- **Online-Tickets (Trusted_Ticket)**: Bietet Echtzeit-Widerruf und dynamische Anpassung bei Netzwerkverbindung.

Die Stärke dieser Kombination ist „graziöse Degradation":

Online erhältst du stärkere Echtzeit-Sicherheit;
ohne Netzwerk kann das System unter verifizierbarer Autorisierung weiterlaufen, anstatt den Menschen in das manuelle Zeitalter zurückzustoßen.

Wenn du willst, dass iFay zu einer menschlichen Erweiterung wird, musst du eine Realität akzeptieren:
das menschliche Leben ist nicht immer online.

---

## 5. CAPs Gefahrenpunkt: Je näher es dem Endgerät kommt, desto mehr muss es an ein Vormundschaftssystem gebunden sein

CAP selbst ist nicht für „Vormundschaftssemantik" zuständig.
Es beantwortet nur „Bist du erlaubt, diese Sache zu tun".

Aber in dem Moment, in dem du CAP von „Berechtigungsverifikation" in „unbegrenzte Übernahme" verwandelst, wird es zu einem Jailbreak-Werkzeug.
Daher muss CAPs Design an ein höheres Vormundschaftssystem gebunden sein:

- **Human View**: Menschen können jederzeit bestätigen, zurückverfolgen und eingreifen.
- **Faying**: Die Übergabe der Kontrolle muss explizit, bezeugbar und widerrufbar sein.
- **Rogue Fay**: Wenn die Vormundschaftsbeziehung nicht besteht, lieber anhalten als nach außen handeln.

Ich bin nicht gegen die Übernahme des Endgeräts durch KI.
Wogegen ich bin, ist „Übernahme ohne Verantwortungsannahmepunkt".

Die Übernahme des Endgeräts ist kein Jailbreak; sie sollte wie der Abschluss eines Vertrags sein:
mit klaren Grenzen, prüfbar, widerrufbar, nachverfolgbar.

Was CAP tut, ist den „Vertrag" auf der Endgeräteebene ausführbar zu machen.

---

## 6. Das coFay-Szenario: Kontrollrecht für öffentliche Rollen ist sensibler

Wenn iFay dein persönliches Endgerät übernimmt, liegt die Verantwortung bei dir.
Wenn coFay Krankenhaussysteme, Luftfahrtsysteme oder Verwaltungssysteme übernimmt, liegt die Verantwortung bei Organisationen und öffentlichen Posten.

Dieser Unterschied diktiert eine Sache:
Das Kontrollrecht für öffentliche Rollen kann sich nicht allein auf „interne Vorschriften" verlassen — es muss prüfbar, anfechtbar und nach außen erklärbar sein.

Sonst verwandelt sich Effizienz direkt in eine Black-Box der Macht:

- Warum hat die Triage dich ans Ende der Schlange gestellt?
- Warum hat das Risikomanagement dich abgelehnt?
- Warum hat das Bildungssystem dir ein Etikett aufgedrückt?

Wenn solche Entscheidungen einen coFay involvieren, muss CAP garantieren:
klare Autorisierungsquelle, klare Sitzungsgrenze, klar abgestufter Ressourcenzugriff, klarer Widerrufspfad.

Dies ist die Grundvoraussetzung dafür, dass öffentliche Dienste im KI-Zeitalter weiterhin vertrauenswürdig bleiben.

---

## 7. Schluss: Die wahre Schwierigkeit ist nicht „kann sie übernehmen", sondern „trauen wir uns, sie übernehmen zu lassen"

Technisch ist „die Übernahme des Endgeräts" kein Mysterium.
Was wirklich schwer ist: Wie bringt man die Gesellschaft dazu, das Endgerät auszuliefern.

Ich glaube, die Antwort ist weder ein stärkeres Modell noch eine schönere UI, sondern:

1) KI muss unter menschlicher Vormundschaft handeln;
2) Kontrollrecht muss zu einem Protokoll gemacht werden;
3) Autorisierung muss verifizierbar sein;
4) Sitzungen müssen prüfbar sein;
5) Widerruf muss erreichbar sein;
6) Kontaktverlust muss automatische Rückgewinnung auslösen.

CAP trägt im iFay-Ökosystem die schlichteste Verbindung:
„Ich erlaube dir, das zu tun" in eine auf Endgeräteebene ausführbare Tatsache zu verwandeln.

Wenn diese Untergrenze nicht umgangen wird, kann KI langfristig in der Gesellschaft existieren.
Sonst beschleunigen wir nur, die nächste Panik zu erzeugen.

---

## Verwandte Dokumente
- CAP｜Protokollpositionierung und Grenzen (Deutsch): https://ifay.ai/de/docs/Control-Authority-Protocol/blueprint/01-Protokollpositionierung-und-Grenzen
- iFay｜Roadmap (EN): https://ifay.ai/en/docs/iFay/blueprint/04-Roadmap
