# Capítulo 4: Protocolo de Ticket En Línea

Este capítulo define los flujos de emisión, consulta, conversión y degradación de `Trusted_Ticket`. El flujo en este capítulo corresponde a la intención de diseño en §2.2 del plan arquitectónico.

## 4.1 Premisas de Aplicabilidad

Trusted_Ticket se usa solo cuando el terminal puede conectarse a Ticket_Issuer o a un servicio de revocación en línea. Cuando el servicio en línea no está disponible, el terminal cae automáticamente a la autorización sin conexión según §4.5.

El terminal DEBE soportar simultáneamente tanto Authorization_Descriptor (Capítulo 3) como Trusted_Ticket. El terminal NO DEBE rechazar la aceptación de Trusted_Tickets firmados legítimamente, incluso si su política prefiere la autorización sin conexión.

## 4.2 Flujo de Emisión

### 4.2.1 Solicitud de Emisión

El autorizador inicia una solicitud de emisión a Ticket_Issuer a través de un cliente que soporta CAP (por ejemplo, App móvil, consola web). La interacción entre el cliente y Ticket_Issuer queda fuera del alcance de esta especificación, pero Ticket_Issuer DEBE:

1. Verificar que la solicitud provenga de un autorizador autenticado
2. Verificar que el autorizador esté autorizado para conceder acceso al Fay y recurso objetivo
3. Aplicar política local para validar el alcance de autorización (por ejemplo, validez máxima, lista blanca de recursos)

### 4.2.2 Pasos de Emisión

Ticket_Issuer DEBE generar Trusted_Ticket siguiendo estos pasos:

1. **Construir Header**: Establecer `alg`, `typ`, `kid` según §2.4.2
2. **Construir Payload**: Llenar TicketPayload según §2.4.3; todos los campos requeridos DEBEN estar establecidos
3. **Validar restricciones**: Validar localmente `exp - nbf ≤ 7 días`
4. **Serialización JSON**: Serializar Header y Payload en bytes JSON UTF-8
5. **Codificación Base64URL**: Codificar Base64URL los bytes de Header y Payload por separado
6. **Firmar**: Calcular la entrada de firma `base64url(header) + "." + base64url(payload)` según RFC 7515
7. **Generar Compact Serialization**: Concatenar como la cadena `header.payload.signature`
8. **Registrar registro**: Mantener el estado del ticket emitido internamente en Ticket_Issuer (para consultas de revocación)

### 4.2.3 Entrega Post-Emisión

Trusted_Ticket se entrega al Fay objetivo o su iFay_Runtime vía un canal cifrado como HTTPS. El método de entrega específico queda fuera del alcance de esta especificación.

## 4.3 Flujo de Validación del Terminal

Las diferencias entre el terminal validando Trusted_Ticket y validando Authorization_Descriptor son:

1. Trusted_Ticket es transmitido completamente al terminal por iFay_Runtime en el momento de la solicitud (no pre-almacenado)
2. El terminal PUEDE consultar el estado de revocación de Ticket_Issuer en tiempo real cuando esté en línea
3. Los códigos de error tras fallo de validación llevan el prefijo `E_TICKET_*`

### 4.3.1 Solicitud de Validación

El AuthRequest enviado por iFay_Runtime lleva el Trusted_Ticket completo:

```
AuthRequest (body of ProtocolMessage) {
  required fay_id        : Fay_ID
  required resource_id   : Resource_ID
  required access_mode   : AccessMode
  required credential    : CredentialContent
}

CredentialContent {
  required type : enum["descriptor_ref", "ticket"]
  oneof:
    descriptor_id : string                  // type == "descriptor_ref"
    ticket        : string                  // type == "ticket", cadena JWS Compact
}
```

### 4.3.2 Pasos de Validación (Ejecutados en Orden)

El terminal DEBE ejecutar la validación en el siguiente orden. Un fallo en cualquier paso devuelve inmediatamente un error.

#### Paso 1: Análisis de JWS

El terminal analiza la cadena JWS Compact:

- Dividir en las tres partes `header.payload.signature`
- Decodificar Base64URL header y payload
- Verificar que `header.typ == "cap-ticket+jws"`
- Verificar que `header.alg` esté en el conjunto de algoritmos permitido por §8

Fallo en cualquier paso → devolver `E_TICKET_MALFORMED`

#### Paso 2: Validación de Firma

El terminal verifica la firma JWS usando el Verification_Key correspondiente a `header.kid` (según RFC 7515).

- Validación de firma fallida → devolver `E_INVALID_SIGNATURE`
- La clave correspondiente a `kid` no está registrada o está revocada → devolver `E_VERIFICATION_KEY_INVALID`

#### Paso 3: Período de Validez

Validar `nbf` y `exp` según las reglas en el Paso 3 de §3.3.2.

#### Paso 4: Coincidencia de Sujeto, Terminal, Recurso y Modo

Coincidir según las reglas en los Pasos 4–6 de §3.3.2. Los códigos de error usan el prefijo `E_TICKET_*` (ver §9).

#### Paso 5: Consulta de Revocación En Línea (Opcional)

Si la política del terminal requiere verificación de revocación en línea, el terminal PUEDE iniciar una consulta de estado de revocación a Ticket_Issuer:

```
TicketRevocationQuery {
  required jti : uuid
}

TicketRevocationResponse {
  required jti      : uuid
  required revoked  : boolean
  optional revoked_at : timestamp
}
```

- Timeout de consulta (predeterminado 2 segundos) → el terminal DEBERÍA rechazar la solicitud y devolver `E_REVOCATION_QUERY_TIMEOUT`, pero PUEDE estar configurado para permitir el paso (aceptando el riesgo de retraso de revocación)
- La consulta devuelve `revoked == true` → devolver `E_TICKET_REVOKED`
- Fallo de consulta (por ejemplo, servicio inalcanzable) → el terminal PUEDE decidir basado en el estado de revocación almacenado localmente; si está más allá de la validez de caché, tratar como timeout

El terminal DEBERÍA almacenar en caché los resultados de consulta de revocación, con validez de caché que no exceda 5 minutos.

### 4.3.3 Después de que la Validación Pasa

El procesamiento después de que la validación pasa es consistente con §3.3.3, creando una Session y devolviendo AuthResult.

## 4.4 Conversión a Authorization_Descriptor Sin Conexión

Cuando `TicketPayload.convertible == true` (predeterminado), el terminal puede convertir el Trusted_Ticket a un Authorization_Descriptor local para uso sin conexión después de que la validación pase.

### 4.4.1 Activador de Conversión

La conversión DEBERÍA realizarse automáticamente en las siguientes situaciones:

1. Validación de Trusted_Ticket pasada y `convertible == true`
2. La política del terminal permite acceso subsiguiente sin conexión
3. `TicketPayload.exp` está suficientemente lejos del tiempo actual (recomendado > 1 hora)

### 4.4.2 Pasos de Conversión

1. **Mapeo de campos**: Mapear campos de TicketPayload a DescriptorPayload según la tabla de §2.4.4
2. **Restricción de período de validez**: El `not_after` convertido NO DEBE exceder `min(TicketPayload.exp, tiempo de conversión + 7 días)`
3. **Retención de metadatos**: Almacenar información clave de auditoría del Trusted_Ticket original en metadatos:
   - `metadata["origin"] = "converted_from_ticket"`
   - `metadata["origin_jti"] = TicketPayload.jti`
   - `metadata["origin_iss"] = TicketPayload.iss`
   - `metadata["origin_kid"] = TicketHeader.kid`
4. **Firma local**: El terminal re-firma la carga útil convertida usando su clave almacenada localmente (ver §4.4.3)
5. **Almacenamiento local**: Cifrar y almacenar el Authorization_Descriptor convertido según §3.2.3

### 4.4.3 Claves de Firma Locales del Terminal

Para soportar el flujo de conversión, el terminal DEBE poseer un par de claves de firma locales:

- Clave privada: Almacenada en el almacenamiento seguro del terminal, usada solo para el flujo de conversión local
- Clave pública: Registrada como Verification_Key del tipo `source == "pre-installed"` en su propio almacén de claves

El `signature.key_id` del Authorization_Descriptor convertido DEBE identificar esta clave local, y `payload.issuer_id` DEBE establecerse a `"local-conversion:" + terminal_id`.

Al validar un Authorization_Descriptor localmente convertido, el terminal verifica la firma usando la clave pública local. Este diseño asegura:

- La credencial convertida puede ser validada independientemente sin conexión, sin depender del Ticket_Issuer original
- La cadena de confianza de conversión está anclada a la propia seguridad de claves del terminal

### 4.4.4 Limitaciones de Conversión

Un Authorization_Descriptor localmente convertido NO DEBE:

1. Ser confiado por otros terminales (su firma key_id es local)
2. Ser migrado para uso entre terminales
3. Ser revocado a través del mecanismo de revocación de Descriptor_Issuer (debe eliminarse localmente)

El terminal DEBERÍA eliminar proactivamente los Authorization_Descriptors localmente convertidos en las siguientes situaciones:

1. Recepción de una notificación de revocación para el Trusted_Ticket original (correlacionado por jti)
2. La credencial ha expirado por más de 24 horas

## 4.5 Degradación En Línea a Sin Conexión

### 4.5.1 Activador de Degradación

El terminal monitorea continuamente la disponibilidad de servicios en línea. Cuando se cumple alguna de las siguientes condiciones, se activa la degradación:

1. La conexión con Ticket_Issuer está inalcanzable por más de 30 segundos
2. Las solicitudes de consulta de revocación expiran 3 veces consecutivas
3. La pila de red reporta indisponibilidad global de red

### 4.5.2 Comportamiento de Degradación

Después de la degradación, el terminal maneja las cosas según las siguientes reglas:

1. Rechazar aceptar nuevos Trusted_Tickets (a menos que esté configurado para aceptación sin conexión) — para evitar no poder verificar el estado de revocación en línea
2. Los tickets ya convertidos a Authorization_Descriptors locales continúan usándose según las reglas de §3
3. Las solicitudes de validación de autorización que usan Authorization_Descriptor directamente no son afectadas por la degradación
4. El terminal PUEDE indicar a iFay_Runtime que actualmente está en modo sin conexión, permitiendo a Fay ajustar políticas de comportamiento

### 4.5.3 Recuperación

Después de que el terminal detecta la recuperación del servicio en línea, DEBE:

1. Priorizar la sincronización de la lista de revocación, aplicando cualquier declaración de revocación potencialmente perdida durante el período sin conexión al estado local
2. Re-verificar Sesiones activas; para Sesiones que usan Authorization_Descriptor localmente convertido cuyo ticket original ha sido revocado, terminarlas forzosamente según las reglas del Capítulo 5
3. Restaurar la política normal de aceptación de Trusted_Ticket

El proceso de recuperación es transparente para iFay_Runtime, pero el terminal PUEDE notificar cambios de modo vía `SessionStateChanged`.

## 4.6 Prioridad en Escenarios de Doble Credencial

Cuando un Fay posee simultáneamente tanto un Trusted_Ticket como un Authorization_Descriptor para el mismo recurso, la política de selección de iFay_Runtime:

| Estado de Red | Elección Recomendada | Razón |
|---------|---------|------|
| En línea y Ticket_Issuer alcanzable | Trusted_Ticket | Mayor temporalidad, revocación en tiempo real posible |
| Sin conexión | Authorization_Descriptor o credencial local convertida | Trusted_Ticket no puede verificar revocación en línea |

iFay_Runtime PUEDE proporcionar una credencial de respaldo en AuthRequest, permitiendo al terminal probar una credencial de respaldo cuando la validación de la credencial primaria falle. Esta especificación no impone un formato específico para el mecanismo de respaldo, pero DEBERÍA implementarse a través de dos AuthRequests independientes para evitar complejidad del protocolo.

## 4.7 Consistencia Semántica entre Trusted_Ticket y Authorization_Descriptor

Cuando iFay_Runtime usa ambos tipos de credenciales, el terminal DEBE garantizar:

1. **Consistencia de resultado de validación**: El mismo alcance de autorización produce juicios consistentes de aprobado/rechazado bajo ambas credenciales
2. **Consistencia semántica de código de error**: Razones de fallo equivalentes (por ejemplo, período de validez, firma, revocación) usan códigos de error semánticamente correspondientes (ver Capítulo 9)
3. **Consistencia de creación de Session**: No hay diferencia en la gestión del ciclo de vida de Sesiones creadas vía las dos credenciales
