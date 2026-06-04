# Capítulo 9: Códigos de Error y Niveles de Conformidad

Este capítulo define la tabla estandarizada de códigos de error del protocolo CAP y la forma en que las implementaciones declaran niveles de conformidad.

## 9.1 Formato de Código de Error

Los códigos de error usan el prefijo `E_` con nomenclatura en mayúsculas y guiones bajos. La cadena de código de error en sí es normativa — las implementaciones de esta especificación DEBEN devolver la cadena de código de error correspondiente según las definiciones en este capítulo para permitir diagnóstico consistente entre implementaciones.

Estructura del mensaje de respuesta de error:

```
ErrorBody {
  required error_code : string
  required message    : string
  optional details    : map<string, string>
  optional retry_after_ms : uint32
}
```

| Campo | Descripción |
|------|------|
| `error_code` | Código de error estándar definido en este capítulo |
| `message` | Descripción de error legible por humanos (idioma no fijo; inglés o español recomendado) |
| `details` | Contexto adicional para el error (por ejemplo, el campo específico violado) |
| `retry_after_ms` | Tiempo de espera recomendado para reintento del cliente; significativo solo para errores transitorios |

## 9.2 Códigos de Error Relacionados con Credenciales

Aplica a escenarios de fallo de validación para Authorization_Descriptor y Trusted_Ticket.

| Código de Error | Equivalente HTTP | Condición de Activación | Conformidad |
|-------|----------|---------|--------|
| `E_DESCRIPTOR_NOT_FOUND` | 404 | El descriptor_id referenciado no fue encontrado localmente en el terminal | Terminal MUST |
| `E_INVALID_STRUCTURE` | 400 | La estructura de credencial no se ajusta a §2.3 / §2.4 | Terminal/Emisor MUST |
| `E_INVALID_SIGNATURE` | 401 | Validación de firma fallida | Terminal MUST |
| `E_VERIFICATION_KEY_INVALID` | 401 | El Verification_Key correspondiente al key_id de firma no está registrado o está revocado | Terminal MUST |
| `E_UNKNOWN_ISSUER` | 401 | El issuer_id de la credencial no está en la lista de confianza del terminal | Terminal MUST |
| `E_DESCRIPTOR_REVOKED` | 403 | Authorization_Descriptor ha sido revocado | Terminal MUST |
| `E_TICKET_REVOKED` | 403 | Trusted_Ticket ha sido revocado | Terminal MUST |
| `E_DESCRIPTOR_EXPIRED` | 403 | Credencial pasada `not_after` | Terminal MUST |
| `E_DESCRIPTOR_NOT_YET_VALID` | 403 | Tiempo actual antes de `not_before` | Terminal MUST |
| `E_TICKET_MALFORMED` | 400 | Análisis JWS fallido o typ incorrecto | Terminal MUST |
| `E_DUPLICATE_DESCRIPTOR_ID` | 409 | Credencial enviada en conflicto con ID de credencial almacenada con contenido no coincidente | Terminal MUST |
| `E_VALIDITY_OUT_OF_RANGE` | 400 | `not_after - not_before` de la credencial excede el límite | Terminal/Emisor MUST |
| `E_REVOCATION_QUERY_TIMEOUT` | 504 | Timeout de consulta de revocación en línea | Terminal SHOULD |

## 9.3 Códigos de Error de Alcance de Autorización

Aplica a escenarios donde la credencial es legítima pero el alcance de autorización no coincide.

| Código de Error | Equivalente HTTP | Condición de Activación |
|-------|----------|---------|
| `E_SUBJECT_MISMATCH` | 403 | El subject de la credencial no coincide con el fay_id del solicitante |
| `E_TERMINAL_MISMATCH` | 403 | El terminal_id de la credencial no coincide con el terminal actual |
| `E_AUTHORIZATION_INSUFFICIENT` | 403 | No se encuentra Grant coincidente para (resource, mode) |
| `E_RATE_LIMIT_EXCEEDED` | 429 | Excede el límite de frecuencia en constraints |
| `E_OUT_OF_TIME_WINDOW` | 403 | Tiempo actual fuera de la ventana de tiempo de constraints |
| `E_OUT_OF_GEO_FENCE` | 403 | La ubicación del terminal está fuera de la geocerca de constraints |
| `E_UNSUPPORTED_CONSTRAINT` | 400 | La credencial contiene clave constraint no reconocida |

## 9.4 Códigos de Error de Recurso y Sesión

Aplica a errores relacionados con ocupación de recursos y estado de Session.

| Código de Error | Equivalente HTTP | Condición de Activación |
|-------|----------|---------|
| `E_RESOURCE_BUSY` | 409 | Recurso ya ocupado en el modo solicitado (viola la matriz de bloqueo lectura-escritura) |
| `E_RESOURCE_UNAVAILABLE` | 503 | Recurso actualmente no disponible (desconexión de hardware, software no ejecutándose, etc.) |
| `E_RESOURCE_NOT_FOUND` | 404 | resource_id no existe |
| `E_SESSION_NOT_FOUND` | 404 | session_id no existe |
| `E_SESSION_TERMINATED` | 410 | Session terminada, no puede continuar operación |
| `E_SESSION_LIMIT_EXCEEDED` | 429 | Excede el límite de cantidad de Session definido en §5.8 |
| `E_OS_INTEGRATION_FAILED` | 500 | Entrega de control de acceso del SO fallida |

## 9.5 Códigos de Error Relacionados con Transferencia

Aplica al flujo de transferencia definido en el Capítulo 6.

| Código de Error | Equivalente HTTP | Condición de Activación |
|-------|----------|---------|
| `E_HANDOVER_INVALID_SOURCE` | 400 | source_session_id no existe o el estado no permite transferencia |
| `E_HANDOVER_INVALID_TARGET` | 400 | Validación de target_credential fallida |
| `E_HANDOVER_REJECTED_BY_POLICY` | 403 | Evaluación de política rechaza transferencia |
| `E_HANDOVER_FAILED_AT_RELEASE` | 500 | Liberación de Session origen fallida durante transferencia |
| `E_HANDOVER_FAILED_AT_ACQUIRE` | 500 | Creación de Session objetivo fallida durante transferencia |
| `E_HANDOVER_TIMEOUT` | 504 | Timeout de transferencia |
| `E_HANDOVER_IN_PROGRESS` | 409 | Recurso tiene una transferencia en progreso; nueva solicitud rechazada |
| `E_HANDOVER_RETRY_LIMIT` | 429 | Conteo de reintentos excede límite |

## 9.6 Códigos de Error de Protocolo y Sistema

| Código de Error | Equivalente HTTP | Condición de Activación |
|-------|----------|---------|
| `E_PROTOCOL_VERSION_UNSUPPORTED` | 400 | Campo version del mensaje no soportado por la implementación |
| `E_INVALID_MESSAGE` | 400 | El formato de mensaje del protocolo no se ajusta a §2.6 |
| `E_INTERNAL_ERROR` | 500 | Error interno del terminal (no expone detalles) |
| `E_NOT_IMPLEMENTED` | 501 | La implementación no proporciona la capacidad opcional |
| `E_STORAGE_FULL` | 507 | Almacenamiento de credenciales del terminal lleno |

## 9.7 Extensión de Código de Error

Las implementaciones PUEDEN definir códigos de error personalizados, pero DEBEN:

1. Usar el formato de nomenclatura `E_VENDOR_<vendor_name>_<code>` (por ejemplo, `E_VENDOR_ACME_QUOTA_EXCEEDED`)
2. No entrar en conflicto con códigos de error estándar definidos en este capítulo
3. Especificar claramente la semántica en la documentación de la implementación

iFay_Runtime DEBERÍA tratar códigos de error personalizados no reconocidos como `E_INTERNAL_ERROR`.

## 9.8 Declaración de Nivel de Conformidad

Las implementaciones DEBEN declarar los niveles de conformidad que satisfacen a través de lo siguiente.

### 9.8.1 Estructura de Declaración de Conformidad

```
ConformanceClaim {
  required protocol_version : string                  // Por ejemplo, "1.0.0" o "draft"
  required levels           : array<ConformanceLevel> (len 1..3)
  optional optional_features : array<string>
  optional vendor_extensions : array<string>
}

ConformanceLevel = enum["terminal", "issuer", "runtime"]
```

| Campo | Descripción |
|------|------|
| `protocol_version` | Versión del protocolo CAP seguida por la implementación |
| `levels` | Niveles de conformidad satisfechos por la implementación (ver §0.5) |
| `optional_features` | Capacidades opcionales soportadas por la implementación (por ejemplo, `aggregated_heartbeat`, `ai_model_handover_policy`) |
| `vendor_extensions` | Capacidades de extensión de proveedor soportadas por la implementación |

### 9.8.2 Canales de Publicación de Declaración

Las implementaciones DEBERÍAN publicar ConformanceClaim a través de los siguientes canales:

1. Declaración explícita en la documentación del producto
2. Devuelta a través de interfaces específicas de consulta (definidas por la implementación)
3. Declarada en metadatos del paquete SDK

### 9.8.3 Pruebas de Conformidad

La Suite de Pruebas de Conformidad (Conformance Test Suite) del protocolo CAP es mantenida por el proyecto como un activo auxiliar, pero queda fuera del alcance normativo de esta especificación. Cualquier implementación DEBERÍA pasar la suite de pruebas para el nivel de conformidad correspondiente, pero NO DEBE depender de la suite de pruebas como la única garantía de corrección.

## 9.9 Recomendaciones de Uso de Códigos de Error

Las implementaciones DEBERÍAN seguir los siguientes principios de uso de códigos de error:

1. **Preferir el código de error más específico**: Por ejemplo, `E_TICKET_REVOKED` se prefiere sobre `E_TICKET_MALFORMED` (incluso si ambos podrían aparecer durante el análisis)
2. **No exponer detalles internos a través de códigos de error**: Evitar exponer huellas de claves, IDs internos y otra información sensible en el campo error message
3. **Proporcionar asistencia diagnóstica vía el campo details**: Por ejemplo, `details["field"] = "not_after"` indica qué campo específico no es conforme
4. **Usar retry_after_ms para errores transitorios**: Por ejemplo, `E_RESOURCE_BUSY`, `E_HANDOVER_IN_PROGRESS`

## 9.10 Sincronización de Códigos de Error con schema.json

`schema/{version}/schema.json` DEBE contener la definición completa de enumeración de códigos de error, mantenida consistente con este capítulo. Cuando la lista de códigos de error en este capítulo se actualice, schema.json DEBE actualizarse sincrónicamente.
