# Capítulo 2: Modelo de Datos

Este capítulo define las estructuras de datos centrales en el protocolo CAP, incluyendo nombres de campos, tipos, restricciones y valores predeterminados. Las definiciones de campos en este capítulo son normativas — cualquier implementación que se ajuste al protocolo CAP DEBE generar y analizar estas estructuras de datos según las definiciones en este capítulo.

`schema/{version}/schema.json` proporciona un complemento formal a las estructuras de datos en este capítulo. Cuando schema.json entre en conflicto con la descripción en este capítulo, schema.json prevalecerá.

## 2.1 Convenciones de Tipos de Datos

Esta especificación utiliza los siguientes tipos base para describir campos:

| Tipo | Descripción | Codificación |
|------|------|------|
| `string` | Cadena UTF-8 | Secuencia de bytes UTF-8 |
| `bytes` | Secuencia de bytes | Bytes en bruto |
| `uint32` / `uint64` | Entero sin signo | Big-endian |
| `timestamp` | Marca de tiempo Unix (segundos) | uint64 |
| `uuid` | RFC 4122 UUID v7 | 16 bytes |
| `enum` | Valor enumerado | Literal de cadena |
| `array<T>` | Colección ordenada de elementos del tipo T | Array |
| `map<K,V>` | Mapeo de K a V | Objeto |

Las restricciones de campo usan la siguiente notación:

- `required`: El campo DEBE estar presente, con un valor no nulo
- `optional`: El campo PUEDE estar presente; la ausencia se trata como no establecido
- `unique`: El valor del campo es único dentro del alcance del sistema
- `len(N..M)`: La longitud de cadena/byte está entre N y M (inclusive)
- `regex(...)`: El valor del campo DEBE coincidir con la expresión regular especificada

## 2.2 Identificadores

Los identificadores centrales en el protocolo CAP DEBEN satisfacer la unicidad global. Esta sección define el formato y las reglas de generación de varios identificadores.

### 2.2.1 Fay_ID

`Fay_ID` identifica de forma única una instancia de Fay.

| Atributo | Valor |
|------|------|
| Tipo | `string` |
| Formato | `"fay:" + uuid_v7` |
| Longitud | 40 caracteres (incluyendo prefijo) |
| Unicidad | Globalmente única |
| Generador | Subsistema de gestión de identidad (fuera del alcance del protocolo CAP) |

Ejemplo: `fay:01927b34-7e21-7c4d-a89f-1234567890ab`

### 2.2.2 Terminal_ID

`Terminal_ID` identifica de forma única un dispositivo terminal.

| Atributo | Valor |
|------|------|
| Tipo | `string` |
| Formato | `"terminal:" + uuid_v7` |
| Longitud | 45 caracteres (incluyendo prefijo) |
| Unicidad | Globalmente única |
| Generador | Registration_Authority |

### 2.2.3 Resource_ID

`Resource_ID` identifica un recurso específico en el terminal.

| Atributo | Valor |
|------|------|
| Tipo | `string` |
| Formato | `terminal_id + "/" + resource_path` |
| Longitud | A lo sumo 256 caracteres |
| Unicidad | Única dentro del alcance del terminal |
| Generador | Sistema operativo del terminal |

`resource_path` DEBE satisfacer la expresión regular `^[a-zA-Z0-9._\-/]+$`.

Ejemplo: `terminal:01927b34-.../device/camera/front`

### 2.2.4 Descriptor_ID

`Descriptor_ID` identifica de forma única un Authorization_Descriptor.

| Atributo | Valor |
|------|------|
| Tipo | `uuid` |
| Formato | UUID v7 |
| Unicidad | Globalmente única (a través de todos los Descriptor_Issuer) |
| Generador | Descriptor_Issuer |

La lista de revocación usa Descriptor_ID para identificar credenciales revocadas. Descriptor_Issuer NO DEBE reutilizar un Descriptor_ID previamente usado.

### 2.2.5 Session_ID

`Session_ID` identifica de forma única una sesión activa.

| Atributo | Valor |
|------|------|
| Tipo | `uuid` |
| Formato | UUID v7 |
| Unicidad | Única dentro del alcance del terminal |
| Generador | Protocol_Engine del terminal |
| Ciclo de vida | Válido solo mientras la sesión esté activa; el ID no se reutiliza después de que la sesión termine |

## 2.3 Authorization_Descriptor

Authorization_Descriptor es la estructura de datos central para la autorización sin conexión. Un Authorization_Descriptor consiste en dos partes: **carga útil (payload)** y **firma (signature)**.

### 2.3.1 Estructura de Nivel Superior

```
AuthorizationDescriptor {
  required version       : uint32
  required payload       : DescriptorPayload
  required signature     : DescriptorSignature
}
```

| Campo | Descripción |
|------|------|
| `version` | Número de versión del protocolo; las implementaciones v1 DEBEN establecer en `1` |
| `payload` | Carga útil de información de autorización (ver §2.3.2) |
| `signature` | Firma digital sobre la carga útil (ver §2.3.3) |

### 2.3.2 DescriptorPayload

```
DescriptorPayload {
  required descriptor_id   : Descriptor_ID
  required issuer_id       : string
  required subject_fay_id  : Fay_ID
  required terminal_id     : Terminal_ID
  required grants          : array<Grant> (len 1..256)
  required issued_at       : timestamp
  required not_before      : timestamp
  required not_after       : timestamp
  optional grantor_id      : string
  optional metadata        : map<string, string>
}
```

| Campo | Restricciones | Descripción |
|------|------|------|
| `descriptor_id` | required, unique | Identificador globalmente único de esta credencial |
| `issuer_id` | required | Identificador del emisor, correspondiente a Descriptor_Issuer en la ruta de confianza de claves |
| `subject_fay_id` | required | Identificador de Fay siendo autorizado |
| `terminal_id` | required | Identificador del terminal que limita el alcance de autorización |
| `grants` | required, 1..256 elementos | Lista de elementos de autorización específicos (ver §2.3.4) |
| `issued_at` | required | Tiempo de emisión |
| `not_before` | required, ≥ issued_at | Tiempo de inicio de validez |
| `not_after` | required, > not_before | Tiempo de expiración |
| `grantor_id` | optional | Identificador del autorizador (Natural_Person o Official_Post) |
| `metadata` | optional | Metadatos personalizados definidos por el emisor, sin impacto en la semántica del protocolo |

Las implementaciones DEBEN rechazar Authorization_Descriptors que no satisfagan las siguientes condiciones:

1. `not_after - not_before > 90 días`: Esta especificación limita el período máximo de validez de una sola credencial a 90 días
2. `not_before > tiempo actual + 24 horas`: Está prohibido emitir credenciales con un tiempo de efectividad excesivamente temprano (previene abuso de pre-emisión)
3. El array `grants` está vacío: Una credencial sin elementos de autorización no tiene sentido

### 2.3.3 DescriptorSignature

```
DescriptorSignature {
  required algorithm       : enum["ed25519", "ecdsa-p256-sha256"]
  required key_id          : string
  required signature_value : bytes
}
```

| Campo | Descripción |
|------|------|
| `algorithm` | Algoritmo de firma; ver Capítulo 8 |
| `key_id` | Identificador de la clave usada para firmar, correspondiente al identificador de Verification_Key |
| `signature_value` | Resultado de firmar los bytes de carga útil serializados en CBOR |

Entrada de firma: Serializa `DescriptorPayload` en una secuencia de bytes usando RFC 8949 CBOR Deterministic Encoding y úsala como la entrada al algoritmo de firma.

### 2.3.4 Grant

```
Grant {
  required resource_pattern : string
  required modes            : array<AccessMode> (len 1..4)
  optional constraints      : map<string, string>
}
```

| Campo | Descripción |
|------|------|
| `resource_pattern` | Patrón de coincidencia de recurso (ver §2.3.5) |
| `modes` | Lista de modos de acceso autorizados; tipo de elemento `AccessMode` |
| `constraints` | Restricciones adicionales (por ejemplo, ventanas de tiempo, geocercas); ver Capítulo 7 para impacto en la semántica del protocolo |

### 2.3.5 Patrón de Coincidencia de Recursos

`resource_pattern` soporta la siguiente sintaxis de coincidencia:

- Coincidencia exacta: `terminal:xxx/device/camera/front`
- Coincidencia con comodín: `terminal:xxx/device/camera/*` (coincide con todas las cámaras bajo ese terminal)
- Coincidencia de terminal completo: `terminal:xxx/device/camera/**` (coincide con todos los niveles bajo esa ruta)

Las implementaciones DEBEN:

1. Soportar solo las tres sintaxis anteriores; rechazar patrones que contengan otros caracteres especiales
2. Hacer que el comodín `*` coincida solo con un segmento de ruta de un solo nivel
3. Permitir que el comodín `**` aparezca solo al final del patrón

### 2.3.6 AccessMode

```
AccessMode = enum["read", "write", "execute", "configure"]
```

La semántica de cada modo de acceso se describe en el Capítulo 7.

## 2.4 Trusted_Ticket

Trusted_Ticket es una credencial de autorización en línea. Su estructura está basada en RFC 7515 JWS Compact Serialization.

### 2.4.1 Estructura de Nivel Superior

Un Trusted_Ticket es una cadena JWS que consiste en tres partes separadas por `.`:

```
base64url(header) . base64url(payload) . base64url(signature)
```

### 2.4.2 Header

```
TicketHeader {
  required alg : enum["EdDSA", "ES256"]
  required typ : "cap-ticket+jws"
  required kid : string
}
```

| Campo | Descripción |
|------|------|
| `alg` | Algoritmo de firma (consistente con §8) |
| `typ` | Valor fijo `"cap-ticket+jws"`, usado para distinguir tipos de tickets |
| `kid` | Identificador de clave, usado para verificación de firma |

### 2.4.3 Payload

```
TicketPayload {
  required jti  : uuid                    // ID único del ticket
  required iss  : string                  // Identificador de Ticket_Issuer
  required sub  : Fay_ID                  // Fay autorizado
  required aud  : Terminal_ID             // Terminal objetivo
  required iat  : timestamp               // Tiempo de emisión
  required nbf  : timestamp               // Tiempo de inicio de validez
  required exp  : timestamp               // Tiempo de expiración
  required grants : array<Grant>          // Misma estructura que §2.3.4
  optional convertible : boolean (default true)  // Si es convertible a Authorization_Descriptor
}
```

Las implementaciones DEBEN rechazar Trusted_Tickets donde `exp - nbf > 7 días`. El período máximo de validez de los tickets en línea es más corto que el de la autorización sin conexión, asegurando que el mecanismo de revocación en línea pueda surtir efecto rápidamente.

### 2.4.4 Conversión de Trusted_Ticket a Authorization_Descriptor

Cuando `convertible == true`, el terminal PUEDE convertir un Trusted_Ticket a un formato local de Authorization_Descriptor para uso sin conexión. Reglas de conversión:

| Campo TicketPayload | Mapeado a Campo DescriptorPayload |
|-------------------|-----------------------------|
| `jti` | `descriptor_id` |
| `iss` | `issuer_id` |
| `sub` | `subject_fay_id` |
| `aud` | `terminal_id` |
| `iat` | `issued_at` |
| `nbf` | `not_before` |
| `exp` | `not_after` (pero NO DEBE exceder `iat + 7 días`) |
| `grants` | `grants` |

El Authorization_Descriptor convertido es re-firmado por el terminal usando su clave almacenada localmente, con el `key_id` de la firma marcado como fuente de conversión (ver Capítulo 4). La información de firma del Trusted_Ticket original DEBE ser retenida en metadatos para propósitos de auditoría.

## 2.5 Session

Session es la estructura de estado de sesión interna del terminal; la estructura completa no se transmite en mensajes del protocolo. Esta sección define los campos de Session para estandarizar la máquina de estados y las convenciones de interfaz.

```
Session {
  required session_id          : Session_ID
  required fay_id              : Fay_ID
  required runtime_id          : string
  required resource_id         : Resource_ID
  required access_mode         : AccessMode
  required granted_modes       : array<AccessMode>
  required state               : SessionState
  required created_at          : timestamp
  required last_heartbeat_at   : timestamp
  required credential_ref      : CredentialRef
}

SessionState = enum[
  "creating",
  "active",
  "handover_pending",
  "terminating",
  "terminated"
]

CredentialRef {
  required type   : enum["descriptor", "ticket"]
  required id     : string                     // descriptor_id o jti
  required not_after : timestamp
}
```

La máquina de estados `SessionState` se describe en el Capítulo 5.

## 2.6 Encapsulación de Mensajes del Protocolo

Todos los mensajes entre iFay_Runtime y Protocol_Engine comparten la siguiente estructura de encapsulación:

```
ProtocolMessage {
  required version         : uint32 (= 1)
  required message_id      : uuid
  required message_type    : string
  required timestamp       : timestamp
  required sender_id       : string
  required body            : object
  optional correlation_id  : uuid
}
```

| Campo | Descripción |
|------|------|
| `version` | Número de versión del protocolo; v1 establece a `1` |
| `message_id` | Identificador único de este mensaje |
| `message_type` | Literal de tipo de mensaje (por ejemplo, `"AuthRequest"`) |
| `timestamp` | Tiempo de envío del mensaje |
| `sender_id` | Identificador del remitente (runtime_id o terminal_id) |
| `body` | Cuerpo del mensaje, con estructura determinada por message_type |
| `correlation_id` | ID de mensaje de solicitud asociada (los mensajes de respuesta DEBEN establecer este campo) |

La estructura `body` correspondiente a cada `message_type` se define en el capítulo correspondiente.

## 2.7 Verification_Key

Verification_Key es la clave de verificación de firma poseída por el terminal.

```
VerificationKey {
  required key_id        : string
  required algorithm     : enum["ed25519", "ecdsa-p256-sha256"]
  required key_material  : bytes              // Bytes de clave pública
  required issuer_id     : string             // Identificador del emisor correspondiente a esta clave
  required valid_from    : timestamp
  optional valid_until   : timestamp
  required source        : enum["pre-installed", "ra-distributed"]
}
```

| Campo | Descripción |
|------|------|
| `key_id` | Identificador de clave, correspondiente a DescriptorSignature.key_id |
| `algorithm` | Algoritmo de firma soportado por esta clave |
| `key_material` | Bytes en bruto de la clave pública; codificación determinada por algorithm (ver Capítulo 8) |
| `issuer_id` | Identificador de Descriptor_Issuer correspondiente a esta clave |
| `valid_from` | Tiempo de inicio de validez de la clave |
| `valid_until` | Tiempo de expiración de la clave; sin establecer significa válido a largo plazo |
| `source` | Fuente de la clave: `pre-installed` (pre-instalación de fábrica del terminal) o `ra-distributed` (distribución en línea por Registration_Authority) |

El terminal DEBE almacenar de forma segura todos los Verification_Keys (ver Capítulo 8).

## 2.8 Declaración de Revocación

Una declaración de revocación se usa para notificar al terminal que una credencial particular ha sido revocada.

```
RevocationStatement {
  required version              : uint32 (= 1)
  required revocation_id        : uuid
  required target_descriptor_id : Descriptor_ID
  required issuer_id            : string
  required revoked_at           : timestamp
  optional reason               : enum["unspecified", "compromised", "superseded", "no_longer_needed"]
  required signature            : DescriptorSignature
}
```

La declaración de revocación DEBE ser emitida y firmada por el `issuer_id` del Authorization_Descriptor original.

## 2.9 Serialización y Transmisión

El protocolo CAP usa los siguientes formatos de serialización:

| Estructura de Datos | Formato de Serialización | Propósito |
|---------|-----------|------|
| AuthorizationDescriptor | RFC 8949 CBOR (Deterministic Encoding) | Almacenamiento y transmisión sin conexión |
| Trusted_Ticket | RFC 7515 JWS Compact Serialization | Transmisión en línea |
| ProtocolMessage | JSON (UTF-8) | Interacción iFay_Runtime ↔ Protocol_Engine |
| RevocationStatement | RFC 8949 CBOR (Deterministic Encoding) | Distribución de lista de revocación |

Las implementaciones PUEDEN usar CBOR en lugar de JSON para la capa de transporte de ProtocolMessage para reducir la sobrecarga, pero los nombres de campos y la semántica definidos en schema.json DEBEN permanecer consistentes.
