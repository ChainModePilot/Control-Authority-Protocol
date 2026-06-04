# Capítulo 6: Protocolo de Transferencia de Control

Este capítulo define el flujo del protocolo de `Handover_Policy` (política de transferencia de control), incluyendo la semántica de tres tipos de políticas, garantías de atomicidad de la transferencia, rollback por timeout y mecanismos de notificación. Este capítulo corresponde a la intención de diseño en §2.4 del plan arquitectónico.

## 6.1 Escenarios Aplicables

La transferencia de control ocurre en escenarios donde múltiples controladores necesitan usar el mismo `Terminal_Resource` secuencialmente. Esta especificación define dos tipos de transferencia centrales:

1. **Fay-a-Fay**: Un Fay transfiere su autoridad de control de Session retenida a otro Fay
2. **Fay-a-Humano**: Un Fay devuelve la autoridad de control a un usuario humano, o un usuario humano delega autoridad de control a un Fay

Esta especificación describe principalmente Fay-a-Fay. La transferencia Fay-a-Humano reutiliza el mismo flujo, modelando al "usuario humano" como una entidad que interactúa directamente con el SO del terminal a través de una interfaz especial.

## 6.2 Iniciación de la Transferencia

### 6.2.1 Condiciones de Activación

Una transferencia es activada por uno de los siguientes escenarios:

1. **Nuevo Fay solicita activamente recurso**: Fay-B solicita acceso vía AuthRequest a un recurso ocupado por Fay-A
2. **Fay actual cede activamente**: Fay-A solicita activamente vía `SessionTransferRequest` transferir la autoridad de control a un Fay-B designado
3. **Activador de gestión**: Un usuario humano o interfaz de gestión solicita transferencia de la autoridad de control

### 6.2.2 Mensaje de Solicitud de Transferencia

```
SessionTransferRequest (body of ProtocolMessage) {
  required source_session_id : Session_ID                  // Session que actualmente posee la autoridad de control
  required target_fay_id     : Fay_ID                      // Receptor objetivo
  required target_credential : CredentialContent           // Credencial de autorización del objetivo
  required handover_reason   : string
  optional handover_metadata : map<string, string>
}
```

Después de recibir la solicitud de transferencia, el terminal inicia el flujo de evaluación de transferencia.

## 6.3 Evaluación de la Transferencia

El terminal evalúa si permitir la transferencia siguiendo estos pasos:

### 6.3.1 Paso 1: Validación de Session Origen

- Verificar que la Session correspondiente a `source_session_id` exista y esté en estado `active`
- Verificar que el iniciador tenga el derecho de solicitar esta transferencia (por ejemplo, desde el iFay_Runtime de la Session o una entidad con autoridad de gestión)

Fallo → devolver `E_HANDOVER_INVALID_SOURCE`

### 6.3.2 Paso 2: Validación de Autorización del Objetivo

- Validar `target_credential` para legitimidad (validación completa según las reglas del Capítulo 3/4)
- Validar que `target_fay_id` esté autorizado para el recurso en el `access_mode` de la Session original

Fallo → devolver `E_HANDOVER_INVALID_TARGET`

### 6.3.3 Paso 3: Evaluación de Política

El terminal aplica `Handover_Policy` según §6.4 para decidir si permitir la transferencia.

Fallo → devolver `E_HANDOVER_REJECTED_BY_POLICY`

### 6.3.4 Paso 4: Pre-ocupación Atómica

Después de que la evaluación de política pase, el terminal inmediatamente cambia la Session original a `handover_pending`, entrando en el estado de pre-ocupación:

- El recurso no acepta otras solicitudes de autoridad de control hasta que la transferencia se complete
- La Session original todavía puede mantener latido (reteniendo requisitos de latido equivalentes a active) pero no puede iniciar nuevas operaciones de recursos

## 6.4 Tipos de Política de Transferencia

`Handover_Policy` se configura a granularidad de Resource_ID; cada recurso PUEDE adoptar una política diferente. Esta especificación define tres tipos de políticas; todos los terminales DEBEN implementar al menos el script de regla de prioridad de §6.4.1.

### 6.4.1 Script de Regla de Prioridad

El terminal calcula puntajes de prioridad para la Session origen y el Fay objetivo según un script de reglas predefinido; el puntaje más alto gana la autoridad de control.

**Estructura de configuración**:

```
PriorityPolicyConfig {
  required policy_type : "priority_script"
  required script_id   : string                      // Identificador de script preconfigurado en el terminal
  optional parameters  : map<string, string>
}
```

**Flujo de evaluación**:

1. El terminal carga el script de reglas correspondiente a `script_id`
2. Entrada: información de Session origen, identificador de Fay objetivo, grants de credencial del objetivo, tiempo actual, metadatos de Resource_ID
3. Salida: puntaje de prioridad origen `S_source`, puntaje de prioridad objetivo `S_target`
4. Decisión: `S_target > S_source` → transferencia permitida; de lo contrario rechazada

El lenguaje específico del script de reglas (por ejemplo, CEL, subconjunto de Lua, JSON Logic) es elegido por la implementación, pero DEBE:

- Ser determinista (la misma entrada produce la misma salida)
- No tener acceso a la red o al sistema de archivos
- Tener tiempo de ejecución limitado a 100 ms

### 6.4.2 Decisión en Tiempo Real por Modelo de IA

El terminal llama a un modelo de IA pre-integrado para tomar una decisión en tiempo real basada en el contexto actual.

**Estructura de configuración**:

```
AIPolicyConfig {
  required policy_type   : "ai_model"
  required model_endpoint : string                    // Endpoint de inferencia del modelo accesible al terminal
  optional context_fields : array<string>
  required decision_timeout_ms : uint32 (default 500)
}
```

**Flujo de evaluación**:

1. El terminal construye una solicitud de decisión, incluyendo información origen/objetivo y contexto especificado por `context_fields`
2. Llama al endpoint de inferencia del modelo de IA
3. El modelo devuelve una decisión: `allow` / `reject` / `require_human`
4. Al `require_human`, retroceder a la decisión de usuario humano de §6.4.3

El terminal DEBE:

- Establecer un timeout duro (predeterminado 500 ms); en timeout, tratar como `reject`
- Almacenar en caché resultados recientes de decisión para evitar oscilación de alta frecuencia (validez de caché ≤ 5 segundos)
- Retroceder al script de regla de prioridad cuando el modelo de IA no esté disponible

### 6.4.3 Decisión de Usuario Humano

El terminal solicita una decisión de un usuario humano a través de la interfaz de usuario.

**Estructura de configuración**:

```
HumanPolicyConfig {
  required policy_type     : "human_decision"
  required ui_channel      : enum["system_dialog", "companion_app", "external_panel"]
  required decision_timeout_seconds : uint32 (default 30)
  required default_action  : enum["allow", "reject"]    // Acción predeterminada en timeout
}
```

**Flujo de evaluación**:

1. El terminal presenta los detalles de la solicitud de transferencia al usuario humano vía `ui_channel`
2. Espera entrada del usuario: permitir / rechazar
3. En timeout → manejar según `default_action`

El modo de decisión humana DEBE:

- Mostrar claramente en la UI: identificador de Fay origen, identificador de Fay objetivo, identificador de recurso, modo de acceso, razón de transferencia
- NO mostrar detalles sensibles de credencial (por ejemplo, firma, ID de clave) en la UI
- Recomendar establecer la acción predeterminada a `reject` (seguridad conservadora)

## 6.5 Garantía de Atomicidad

La transferencia DEBE satisfacer atomicidad: **en cualquier momento, un solo Resource_ID tiene como máximo un controlador activo**.

### 6.5.1 Secuencia Atómica

El terminal DEBE ejecutar la transferencia en el siguiente orden, con cada paso completado dentro de una sección crítica:

```
[T0] Entrar a handover_pending: Estado de Session origen cambia; recurso entra en pre-ocupación
[T1] Terminar Session origen: estado de source_session_id active → terminating
                              Revocación de control de acceso del SO activada
[T2] Recuperación de recurso completa: estado de origen terminating → terminated
                                       Recurso está en estado "ninguna Session activa" en este punto
[T3] Crear Session objetivo: Crear nueva Session para Fay objetivo según el flujo §5.2
                             El estado entra en creating
[T4] Entregar control de acceso del SO: Habilitar acceso al recurso para el Fay objetivo
[T5] Session objetivo cambia a active: Transferencia completa
```

Los observadores externos entre [T2] y [T3] observan el estado del recurso como "ninguna Session activa" y **nunca** observan dos Sesiones activas simultáneamente.

### 6.5.2 Rollback por Fallo de Paso Intermedio

Si algún paso en [T0]–[T2] falla:

- La Session origen DEBE revertir a `active`
- La pre-ocupación de recuperación de recursos se cancela
- Devolver `E_HANDOVER_FAILED_AT_RELEASE`

Si [T3]–[T4] falla (el origen ha terminado pero el objetivo no puede crearse):

- El terminal DEBE limpiar inmediatamente los recursos (el recurso entra en estado "ninguna Session activa")
- Notificar al iFay_Runtime original (la Session origen ha terminado) y al iFay_Runtime objetivo (toma fallida)
- Devolver `E_HANDOVER_FAILED_AT_ACQUIRE`
- **Estado del recurso**: Permanecer inactivo, con solicitudes subsiguientes compitiendo por él. **NO DEBE** restaurar automáticamente la Session origen

### 6.5.3 Serialización de Solicitudes de Transferencia Concurrentes

Múltiples solicitudes de transferencia concurrentes para el mismo recurso DEBEN serializarse:

- Durante el período en que el recurso está en `handover_pending`, las nuevas solicitudes de transferencia se ponen en cola o se rechazan
- Las implementaciones PUEDEN elegir rechazo (`E_HANDOVER_IN_PROGRESS`) o cola (longitud máxima de cola 8)
- Las solicitudes en cola se procesan en orden FIFO; cada solicitud entra en evaluación solo después de que la transferencia precedente se complete (éxito o fallo)

## 6.6 Manejo de Timeout

Cada transferencia DEBE establecer un timeout duro. Los umbrales de timeout están determinados por el tipo de política:

| Tipo de Política | Timeout Predeterminado | Rango |
|---------|---------|------|
| Script de regla de prioridad | 1 segundo | 0.5–2 segundos |
| Decisión en tiempo real por modelo de IA | 1 segundo | 0.5–3 segundos |
| Decisión de usuario humano | 30 segundos | 5–120 segundos |

### 6.6.1 Principio de Rollback por Timeout

Si una transferencia no se completa dentro del timeout (es decir, no llega a [T5]), el terminal DEBE:

1. Abortar el paso de transferencia actualmente en progreso
2. Si está en [T0]–[T2] (origen no completamente terminado): Revertir al estado active de la Session original
3. Si está en [T3]–[T4] (origen ha terminado): Manejar según §6.5.2 (no restaurar origen; recurso establecido a inactivo)
4. Enviar `HandoverFailedNotification` a las partes relevantes

### 6.6.2 Notificación de Timeout

```
HandoverFailedNotification (body of ProtocolMessage) {
  required handover_id   : uuid
  required source_session_id : Session_ID
  required target_fay_id : Fay_ID
  required reason        : enum["timeout", "policy_rejected", "release_failed", "acquire_failed"]
  optional details       : map<string, string>
}
```

## 6.7 Política de Reintento

La política de reintento después de un fallo de transferencia es determinada por el iniciador. Esta especificación no impone un mecanismo de reintento pero define los límites de los reintentos:

- El intervalo de reintento para la misma solicitud de transferencia DEBERÍA ser ≥ 1 segundo
- El número de reintentos para el mismo par (source_session, target_fay) DEBERÍA ser ≤ 3
- Los reintentos DEBEN usar un nuevo handover_id

El terminal PUEDE rechazar solicitudes de reintento excesivamente frecuentes y devolver `E_HANDOVER_RETRY_LIMIT`.

## 6.8 Transferencia Fay-a-Humano

Las transferencias que devuelven la autoridad de control a un usuario humano siguen el flujo anterior, pero la parte objetivo es un usuario humano:

- El campo `target_fay_id` usa el valor especial `"human:" + terminal_id` (representando al usuario humano del terminal actual)
- La validación de autorización del objetivo se omite (el usuario humano tiene permisos completos sobre los recursos de su propio terminal por predeterminado)
- La evaluación de política típicamente usa la decisión de usuario humano de §6.4.3 (confirmando intención de toma)
- Al crear la Session objetivo, la Session no está vinculada a ningún Fay sino que es retenida directamente por el proceso de usuario del SO

El flujo inverso de toma proactiva por usuario humano es simétrico: reemplazar la Session de Fay original con un proceso de SO retenido por el humano.

## 6.9 Observabilidad de la Transferencia

El terminal DEBE proporcionar observabilidad de eventos de transferencia a los siguientes:

| Receptor | Evento Recibido |
|--------|-----------|
| iFay_Runtime origen | `SessionStateChanged` (active → handover_pending → terminating → terminated) |
| iFay_Runtime objetivo | `AuthResult` (respuesta exitosa para creación de nueva Session) o `HandoverFailedNotification` |
| Log de auditoría del terminal | Registro completo de transferencia (incluyendo handover_id, source, target, decisión de política, resultado final) |

## 6.10 Secuencia de Mensajes de Transferencia

Secuencia completa de mensajes para una transferencia exitosa:

```
[Source Runtime]      [Target Runtime]      [Terminal]
       │                     │                  │
       │── SessionTransferRequest ─────────────→│
       │                     │                  │── Evaluar política
       │                     │                  │── source: active → handover_pending
       │←─ SessionStateChanged(handover_pending)│
       │                     │                  │── Terminar source
       │←─ SessionStateChanged(terminating)─────│
       │←─ SessionStateChanged(terminated)──────│
       │                     │                  │── Crear target
       │                     │←─ AuthResult ────│
       │                     │                  │── target: creating → active
       │                     │←─ SessionStateChanged(active)
```

Secuencia de mensajes para rollback de fallo:

```
[Source Runtime]      [Target Runtime]      [Terminal]
       │                     │                  │
       │── SessionTransferRequest ─────────────→│
       │                     │                  │── Evaluar política
       │                     │                  │── source: active → handover_pending
       │←─ SessionStateChanged(handover_pending)│
       │                     │                  │── Timeout o fallo
       │                     │                  │── source: handover_pending → active
       │←─ SessionStateChanged(active)──────────│
       │←─ HandoverFailedNotification ──────────│
       │                     │←─ HandoverFailedNotification
```
