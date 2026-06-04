# Technische Spezifikation des CAP-Protokolls (Entwurf)

Dieses Verzeichnis enthält die Entwurfsversion der technischen Spezifikation des Control Authority Protocol (CAP) v1. Die Spezifikation wird basierend auf dem Architekturplan in `docs/de/blueprint/` entwickelt und deckt die 6 Kernfähigkeiten ab, die in §3.1 von Kapitel 3 des Architekturplans aufgelistet sind.

## Dokumentstruktur

| Kapitel | Datei | Inhalt |
|------|------|------|
| Kapitel 0 | `00-Einführung und Konformität.md` | Dokumentstatus, Geltungsbereich, RFC 2119-Schlüsselwörter, Konformitätsstufen, normative Verweise |
| Kapitel 1 | `01-Architektur und Rollen.md` | Protokollrollen, Vertrauenskette, externe Schnittstellenverträge |
| Kapitel 2 | `02-Datenmodell.md` | Kerndatenstrukturen (Authorization_Descriptor, Trusted_Ticket, Session, Verification_Key) |
| Kapitel 3 | `03-Offline-Autorisierungsprotokoll.md` | Vollständiger Authorization_Descriptor-Lebenszyklus-Protokollablauf |
| Kapitel 4 | `04-Online-Ticket-Protokoll.md` | Vollständiger Trusted_Ticket-Ablauf und Degradation |
| Kapitel 5 | `05-Sitzungsverwaltung und Aktivitätserkennung.md` | Session-Zustandsmaschine, Bindungsregeln, Heartbeats, Doppelbestimmung, Timeout-Rückgewinnung |
| Kapitel 6 | `06-Kontrollübergabe-Protokoll.md` | Drei Handover_Policy-Richtlinientypen, Atomaritätsgarantien, Timeout-Rollback |
| Kapitel 7 | `07-Ressourcenzugriffsmodus.md` | Semantik von read/write/execute/configure, Lese-Schreib-Sperrmatrix |
| Kapitel 8 | `08-Kryptografie und Signaturen.md` | Algorithmussatz, Schlüsselformate, Verteilung und Rotation |
| Kapitel 9 | `09-Fehlercodes und Konformitätsstufen.md` | Standardfehlercodetabelle, Konformitätsdeklaration |
| Kapitel 10 | `10-Sicherheitsüberlegungen.md` | Bedrohungsmodell, bekannte Risiken und Maßnahmen |

## Empfohlene Lesereihenfolge

1. Erste Lektüre: Kapitel 0 → Kapitel 1 → Kapitel 2 → Kapitel 3
2. Endgerät implementieren: Kapitel 0–3 → Kapitel 5, 7 → Kapitel 8, 9
3. Aussteller implementieren: Kapitel 0–2 → Kapitel 3, 4 → Kapitel 8
4. iFay_Runtime implementieren: Kapitel 0, 1 → Kapitel 5 → Kapitel 9
5. Sicherheitsüberprüfung: Kapitel 10 + Querverweise auf verwandte Kapitel

## Entwurfsstatus

Dieser Entwurf befindet sich in der Diskussionsphase. Vor der formalen Veröffentlichung:

- Können Feldnamen, Fehlercodes und Beschränkungsschwellen angepasst werden
- Kann die Kapitelstruktur reorganisiert werden
- Wird keine Abwärtskompatibilität garantiert

Nachdem sich die Diskussionen stabilisiert haben, werden die Inhalte dieses Verzeichnisses als `docs/de/specification/2025-10-25/`, der ersten formalen Version des CAP-Protokolls, veröffentlicht.

## Verwandte Assets

- Architekturplan: `docs/de/blueprint/`
- Schema-Definitionen (Entwurf): `schema/draft/`
- Andere Sprachversionen: Diese Spezifikation hat derzeit zh-CN-, zh-TW-, ja-, ko- und en-Versionen; verbleibende Sprachen werden vor der formalen Veröffentlichung übersetzt
