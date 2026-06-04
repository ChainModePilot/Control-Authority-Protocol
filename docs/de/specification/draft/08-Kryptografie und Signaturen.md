# Kapitel 8: Kryptografie und Signaturen

Dieses Kapitel definiert die vom CAP-Protokoll verwendeten Signaturalgorithmen, Schlüsselformate, das Schlüssellebenszyklusmanagement und die Verteilungsanforderungen. Die kryptografischen Anforderungen in diesem Kapitel sind normativ — jede Implementierung, die dem CAP-Protokoll entspricht, MUSS gemäß den Definitionen in diesem Kapitel implementieren.

## 8.1 Algorithmussatz

CAP v1 definiert zwei Sätze obligatorischer Signaturalgorithmen und Schlüsselformate:

| Algorithmus-ID | Algorithmus | Länge des öffentlichen Schlüssels | Signaturlänge | Status |
|---------|------|---------|---------|------|
| `ed25519` | Ed25519 (RFC 8032) | 32 Bytes | 64 Bytes | Obligatorisch |
| `ecdsa-p256-sha256` | ECDSA P-256 + SHA-256 (RFC 6979) | 65 Bytes (unkomprimiert) / 33 Bytes (komprimiert) | 64 Bytes (DER-codiert oder raw) | Obligatorisch |

Alle CAP-Implementierungen MÜSSEN beide Algorithmussätze gleichzeitig unterstützen. Wenn Trusted_Ticket JWS verwendet, sind die entsprechenden Algorithmusnamen:

| CAP-Algorithmus-ID | JWS `alg`-Feld |
|------------|---------------|
| `ed25519` | `EdDSA` |
| `ecdsa-p256-sha256` | `ES256` |

### 8.1.1 Algorithmus-Auswahlempfehlungen

- **Bevorzugt `ed25519`**: Bessere Leistung, kürzere Signaturen, einfachere Schlüsselverwaltung
- **`ecdsa-p256-sha256`**: Für die Verwendung unter regulatorischen Anforderungen wie FIPS 140-3

Aussteller SOLLTEN standardmäßig `ed25519` in allen neu ausgestellten Anmeldeinformationen verwenden, außer wenn explizite Compliance-Anforderungen bestehen.

### 8.1.2 Nicht erlaubte Algorithmen

CAP v1-Implementierungen DÜRFEN die folgenden Algorithmen NICHT verwenden, um neue Anmeldeinformationen auszustellen oder zu akzeptieren:

- Beliebige RSA-Signaturvarianten (schlechte Leistung, größere Schlüssel)
- ECDSA secp256k1 (nicht-FIPS-Standardkurve)
- ECDSA P-384 / P-521 (in v1 nicht obligatorisch, kann in zukünftigen Versionen hinzugefügt werden)
- Beliebige SHA-1-abgeleitete Signaturalgorithmen

## 8.2 Schlüsselformate

### 8.2.1 Ed25519

Öffentliche Schlüssel werden gemäß RFC 8032 §5.1.5 codiert:

- 32-Byte-Little-Endian komprimierter Edwards-Kurvenpunkt

In DescriptorPayload ist `signature.signature_value` eine 64-Byte-Rohsignatur (ohne ASN.1-Codierung).

In JWS werden Signaturen gemäß RFC 8037 codiert, mit der 64-Byte-Rohsignatur base64url-codiert.

### 8.2.2 ECDSA P-256

Öffentliche Schlüssel werden gemäß RFC 5480 codiert:

- Unkomprimiertes Format: 65 Bytes (0x04 || X || Y)
- Komprimiertes Format: 33 Bytes (0x02/0x03 || X)

Implementierungen MÜSSEN sowohl unkomprimierte als auch komprimierte öffentliche Schlüssel gleichzeitig akzeptieren.

Signaturformat:

- In DescriptorSignature: 64-Byte-Rohsignatur (R || S, jeweils 32 Bytes Big-Endian), DER nicht verwendet
- In JWS: codiert gemäß RFC 7518 §3.4 (64 Bytes R || S)

### 8.2.3 Binärdarstellung von Schlüsseln

Das Feld `VerificationKey.key_material` speichert direkt die Bytefolge des obigen Formats:

- ed25519 → 32 Bytes
- ecdsa-p256-sha256 (unkomprimiert) → 65 Bytes
- ecdsa-p256-sha256 (komprimiert) → 33 Bytes

## 8.3 Konstruktion der Signatureingabe

### 8.3.1 Authorization_Descriptor-Signatureingabe

Konstruieren Sie die Signatureingabe in den folgenden Schritten:

1. Nehmen Sie `AuthorizationDescriptor.payload` (DescriptorPayload-Struktur)
2. Serialisieren Sie sie unter Verwendung der RFC 8949 CBOR **Deterministic Encoding** in eine Bytefolge
3. Die Bytefolge wird zur Eingabe für den Signaturalgorithmus

Wichtige Beschränkungen der CBOR Deterministic Encoding (aus RFC 8949 §4.2):

- Map-Schlüssel werden in lexikografischer Reihenfolge sortiert
- Numerische Werte verwenden die kürzeste Codierung
- Zeichenfolgen und Bytefolgen verwenden die kürzeste Längencodierung
- Indefinite-length encoding wird nicht verwendet

Implementierungen MÜSSEN diese Beschränkungen strikt einhalten, um die Konsistenz der Signaturverifizierung zwischen Implementierungen zu gewährleisten.

### 8.3.2 Trusted_Ticket-Signatureingabe

Konstruieren Sie die JWS-Signatureingabe gemäß RFC 7515 §5.1:

```
Signing Input = base64url(UTF8(Header)) + "." + base64url(UTF8(Payload))
```

Wobei:

- Header und Payload gemäß RFC 7159 in UTF-8-JSON serialisiert werden
- JSON-Feldreihenfolge: Implementierungen SOLLTEN Schlüssel in lexikografischer Reihenfolge sortieren, aber die strikte Feldreihenfolgeanforderung wird durch die base64url-codierten Bytes bestimmt

### 8.3.3 RevocationStatement-Signatureingabe

Die Konstruktion der Signatureingabe für Widerrufserklärungen ist die gleiche wie für Authorization_Descriptor:

1. Entfernen Sie das Feld `signature` und behalten Sie die übrigen RevocationStatement-Felder bei
2. CBOR Deterministic Encoding-Serialisierung
3. Verwenden Sie die Bytefolge als Signatureingabe

## 8.4 Schlüsselverteilung

### 8.4.1 Verteilungsmodi

Das Endgerät MUSS die folgenden zwei Verification_Key-Verteilungsmodi unterstützen:

#### Offline-Vorinstallation

Der initiale Verification_Key wird bei der Werks- oder Bereitstellungsinstallation des Endgeräts vorinstalliert. Vorinstallierte Schlüssel:

- Markiert mit `source = "pre-installed"` in der `VerificationKey`-Struktur
- MÜSSEN während der Herstellung oder Bereitstellung über einen physischen oder kontrollierten sicheren Kanal geschrieben werden
- Entsprechen typischerweise Root-Level-Descriptor_Issuern (z. B. Geräteherstellungs-Authentifizierungs-Issuer)

#### Online-Verteilung

Registration_Authority verteilt neue Verification_Keys über Online-Schnittstellen:

- Markiert mit `source = "ra-distributed"` in der `VerificationKey`-Struktur
- MUSS über TLS/mTLS-Kanäle übertragen werden (siehe §1.3.4)
- Das Endgerät MUSS verifizieren, dass der Sender eine vertrauenswürdige Registration_Authority ist

### 8.4.2 Vertrauensanker-Konfiguration

Der öffentliche Schlüssel der Registration_Authority des Endgeräts (zur Verifizierung von Signaturen von RA-Push-Nachrichten verwendet) dient als Vertrauensanker:

- MUSS während der Herstellung oder Erstbereitstellung des Endgeräts vorinstalliert werden
- Darf nur über physische oder kontrollierte Kanäle ersetzt werden
- Wird nicht über CAP-Protokoll-Laufzeitmechanismen aktualisiert

### 8.4.3 Verteilungsnachrichten

Schlüsselverteilungsnachrichten, die von Registration_Authority gepusht werden:

```
VerificationKeyDistribution {
  required version       : uint32 (= 1)
  required key           : VerificationKey
  required ra_signature  : DescriptorSignature       // Von Registration_Authority signiert
}
```

Endgeräteverarbeitungsablauf:

1. Verifizieren, dass `ra_signature` von einer vertrauenswürdigen Registration_Authority ausgestellt wurde
2. Verifizieren, dass `key.algorithm` im durch §8.1 erlaubten Algorithmussatz enthalten ist
3. Verifizieren, dass `key.valid_from` relativ zur aktuellen Zeit des Endgeräts angemessen ist (z. B. nicht über aktuelle Zeit + 24 Stunden hinausgeht)
4. In den Endgeräteschlüsselspeicher schreiben
5. Eine Bestätigung an Registration_Authority zurückgeben (der Bestätigungsmechanismus liegt außerhalb des Geltungsbereichs dieser Spezifikation)

## 8.5 Schlüsselspeicherung

Das Endgerät MUSS alle Verification_Keys und lokalen Signierschlüssel sicher speichern (siehe §4.4.3).

### 8.5.1 Speicheranforderungen

Das Endgerät MUSS:

1. Den privaten Schlüsselteil aller Schlüssel verschlüsselt speichern (öffentliche Schlüssel können im Klartext gespeichert werden)
2. Der Zugriff auf private Schlüssel ist durch OS-Prozessberechtigungen geschützt; nur Protocol_Engine kann lesen
3. SOLLTE private Schlüssel in Hardware-Sicherheitselementen (z. B. TPM, Secure Enclave, TEE) speichern

Das Endgerät DARF NICHT:

1. Private Schlüssel im Klartext auf persistenten Medien schreiben
2. Private Schlüssel über CAP-Protokollnachrichten übertragen
3. Private Schlüsselinhalte in Debug-Protokollen oder Fehlerausgaben offenlegen

### 8.5.2 Behandlung von Schlüsselleckagen

Wenn das Endgerät erkennt, dass ein Schlüssel möglicherweise geleakt wurde (z. B. sicherer Speicher angegriffen):

1. Sofort die Verwendung dieses Schlüssels einstellen
2. Über den §8.6-Ablauf an Registration_Authority melden
3. Vor der vollständigen Verteilung des neuen Schlüssels alle zugehörigen Anmeldeinformationsvalidierungen ablehnen (konservative Richtlinie)

## 8.6 Schlüsselaktualisierung und -rotation

### 8.6.1 Reibungsloser Übergang

Beim Rotieren von Verification_Keys MUSS Registration_Authority einen reibungslosen Übergang bereitstellen:

1. Den neuen Schlüssel an alle Endgeräte verteilen
2. Während der Übergangszeit (Standard 30 Tage) sind sowohl der neue als auch der alte Schlüssel gültig
3. Nach Ablauf der Übergangszeit wird `valid_until` des alten Schlüssels erreicht und automatisch ungültig

Während der Übergangszeit MUSS das Endgerät sowohl den neuen als auch den alten Schlüssel gleichzeitig pflegen und den entsprechenden Schlüssel zur Validierung gemäß `key_id` auswählen.

### 8.6.2 Notfallwiderruf

```
VerificationKeyRevocation {
  required version          : uint32 (= 1)
  required key_id           : string
  required revocation_time  : timestamp
  required reason           : enum["compromised", "superseded", "ra_decision"]
  required ra_signature     : DescriptorSignature
}
```

Nach Erhalt der Widerrufsnachricht MUSS das Endgerät:

1. Verifizieren, dass die Signatur von einer vertrauenswürdigen Registration_Authority stammt
2. Sofort die key_id als widerrufen markieren
3. In nachfolgenden Validierungsanfragen alle Anmeldeinformationen, die diese key_id verwenden, ablehnen (`E_VERIFICATION_KEY_INVALID` zurückgeben)
4. Alle aktiven Sessions proaktiv prüfen: Sessions, die mit dieser key_id ausgestellte Anmeldeinformationen verwenden, zwangsweise beenden (siehe §5.5)

### 8.6.3 Rotation der lokalen Signierschlüssel des Endgeräts

Lokale Signierschlüssel, die das Endgerät zur Unterstützung des §4.4-Konvertierungsablaufs hält:

- SOLLTEN alle 90 Tage automatisch rotiert werden
- Bei Rotation generiert das Endgerät ein neues Schlüsselpaar; der alte Schlüssel wird weiterhin zur Validierung ausgestellter Anmeldeinformationen verwendet, bis sie alle ablaufen
- Das Endgerät KANN bis zu 4 lokale Signierschlüssel gleichzeitig halten (deckt die maximale Gültigkeitsdauer + reibungslosen Übergang ab)

## 8.7 Zusammenfassung der kryptografischen Anforderungen

| Anforderung | Geltungsbereich |
|------|------|
| Signaturalgorithmus MUSS aus ed25519 / ecdsa-p256-sha256 gewählt werden | Alle Signaturszenarien |
| Private Schlüssel MÜSSEN sicher gespeichert werden | Alle Aussteller und Endgeräte |
| Verteilung öffentlicher Schlüssel MUSS über verschlüsselte Kanäle erfolgen | Registration_Authority → Endgerät |
| Vertrauensanker MÜSSEN über physische oder kontrollierte Kanäle vorinstalliert werden | Alle Endgeräte |
| CBOR Deterministic Encoding für Offline-Anmeldeinformationssignatureingabe | Authorization_Descriptor, RevocationStatement |
| JWS Compact Serialization für Online-Tickets | Trusted_Ticket |
