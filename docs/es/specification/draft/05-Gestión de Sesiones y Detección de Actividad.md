# Capítulo 5: Gestión de Sesiones y Detección de Actividad

Este capítulo define la máquina de estados, eventos del ciclo de vida y reglas de enlace de `Session` con `Terminal_Resource`, así como el mecanismo `Liveness_Detection`. Este capítulo corresponde a la intención de diseño en §2.3 del plan arquitectónico.

## 5.1 Máquina de Estados de Session

Cada Session pasa por múltiples estados de una máquina de estados finita dentro de su ciclo de vida.

### 5.1.1 Definiciones de Estados

```
SessionState = enum[
  "creating",
  "active",
  "handover_pending",
  "terminating",
  "terminated"
]
```

| Estado | Descripción |
|------|------|
| `creating` | Validación de autorización pasada; Session se está inicializando (por ejemplo, asignación de recursos, configuración de control de acceso del SO) |
| `active` | Session está en estado activo; Fay puede realizar operaciones dentro del alcance autorizado |
| `handover_pending` | Session está participando en una transferencia de control (ver Capítulo 6); el resultado está indeterminado |
| `terminating` | Session se está terminando; los recursos se están recuperando; las nuevas solicitudes son rechazadas |
| `terminated` | Session está completamente terminada; session_id entra en el registro histórico |

### 5.1.2 Diagrama de Transición de Estados

```
              ┌─────────────┐
              │   (start)   │
              └──────┬──────┘
                     │ Validación de autorización pasada
                     ▼
              ┌─────────────┐
              │  creating   │
              └──────┬──────┘
                     │ Inicialización de recursos completa
                     ▼
              ┌─────────────┐  Transferencia iniciada  ┌──────────────────┐
              │   active    │─────────────────────────→│ handover_pending │
              └──────┬──────┘                          └──────┬───────────┘
                     │  ↑                                     │
              Cond.term.│  │Rollback timeout transferencia    │Transferencia completa
                     ▼  │                                     ▼
              ┌─────────────┐                            ┌─────────────┐
              │ terminating │←───────────────────────────┤ (handed off)│
              └──────┬──────┘                            └─────────────┘
                     │ Recuperación de recursos completa
                     ▼
              ┌─────────────┐
              │ terminated  │
              └─────────────┘
```

### 5.1.3 Reglas de Transición de Estados

El terminal DEBE actualizar el estado de Session solo a través de las transiciones permitidas en la siguiente tabla:

| Estado Actual | Estado Objetivo | Condición de Activación |
|---------|---------|---------|
| `creating` | `active` | Inicialización de recursos completa |
| `creating` | `terminating` | Inicialización fallida (por ejemplo, recurso ocupado por otra Session) |
| `active` | `handover_pending` | Recibida solicitud de transferencia para esta Session |
| `active` | `terminating` | Liberación proactiva, timeout de latido, terminación por revocación, recurso no disponible |
| `handover_pending` | `terminating` | Transferencia exitosa (la Session original DEBE terminar) o cancelación por timeout de transferencia |
| `handover_pending` | `active` | Transferencia rechazada por el nuevo controlador (rollback al estado original) |
| `terminating` | `terminated` | Recursos completamente liberados; persistencia de estado completa |

Las implementaciones NO DEBEN realizar transiciones de estado no listadas.

### 5.1.4 Atomicidad de las Transiciones de Estado

Cada transición de estado DEBE satisfacer los siguientes requisitos de atomicidad:

1. La lectura, juicio y escritura del estado DEBEN completarse dentro de una sección crítica; los observadores externos no ven estados intermedios
2. Las transiciones de estado y las operaciones de recursos correspondientes (por ejemplo, entrega de control de acceso del SO) DEBEN ejecutarse como una sola transacción; o todas tienen éxito o todas se revierten

## 5.2 Creación de Session

### 5.2.1 Activador de Creación

Session se crea en los siguientes escenarios:

1. iFay_Runtime inicia un AuthRequest y la validación de autorización pasa (ver §3.3, §4.3)
2. Después de una transferencia de control exitosa, se crea una nueva Session para el nuevo controlador (ver Capítulo 6)

### 5.2.2 Pasos de Creación

El terminal DEBE crear una Session siguiendo estos pasos:

1. **Pre-verificación de ocupación de recursos**: Verificar si el `resource_id` objetivo puede ser ocupado bajo `access_mode` (ver §5.3 y las reglas de bloqueo lectura-escritura en el Capítulo 7)
2. **Generar Session_ID**: Asignar un nuevo UUID v7
3. **Establecer estado inicial**: `state = "creating"`
4. **Construir objeto Session**: Llenar los campos definidos en §2.5
5. **Entregar control de acceso del SO**: Informar al sistema operativo a través de la interfaz §1.3.2 que permita al proceso Fay acceder al recurso en el modo especificado
6. **Transición de estado**: `state: creating → active`; registrar `created_at` y `last_heartbeat_at`
7. **Devolver AuthResult**: Devolver `session_id` a iFay_Runtime vía AuthResult

### 5.2.3 Manejo de Fallo de Creación

Si algún paso falla:

1. Los recursos asignados (por ejemplo, entradas de control de acceso del SO) DEBEN revertirse
2. El estado de Session transiciona a `terminating` y luego inmediatamente a `terminated`
3. Devolver el código de error correspondiente (por ejemplo, `E_RESOURCE_BUSY`, `E_OS_INTEGRATION_FAILED`)

## 5.3 Enlace de Session a Terminal_Resource

### 5.3.1 Enlace Uno-a-Uno

Cada Session activa DEBE estar enlazada a exactamente un `Resource_ID`. El terminal mantiene la relación de ocupación de recursos a través de (resource_id, access_mode).

### 5.3.2 Reglas de Exclusividad y Compartición

Ver §7.2 matriz de bloqueo lectura-escritura en el Capítulo 7 para detalles. Esta sección da las reglas simplificadas a nivel de sesión:

| Session Activa Actual del Recurso | Solicitud de Nueva Session | Manejo |
|---------------------|----------------|------|
| Ninguna | Cualquier modo | Creación permitida |
| ≥1 read | read | Creación permitida |
| ≥1 read | write/execute/configure | Rechazar (`E_RESOURCE_BUSY`) |
| 1 write/execute/configure | Cualquier modo | Rechazar (`E_RESOURCE_BUSY`) |

### 5.3.3 Terminación en Cascada en Indisponibilidad de Recurso

Cuando el recurso correspondiente a `Resource_ID` se vuelve no disponible (por ejemplo, desconexión de hardware, fallo de proceso de software, sistema operativo reporta desaparición de recurso):

1. El terminal DEBE inmediatamente cambiar todas las Sesiones enlazadas a ese recurso a `terminating`
2. El terminal DEBE empujar notificaciones `SessionStateChanged` a todos los iFay_Runtimes afectados, con `reason` como `"resource_unavailable"`
3. Después de que el recurso se recupere, las Sesiones terminadas NO DEBEN restaurarse automáticamente; iFay_Runtime necesita re-iniciar AuthRequest para crear una nueva Session

## 5.4 Liveness_Detection

La detección de actividad determina si una Session todavía está activa a través de dos señales independientes: **mantenimiento de conexión persistente** y **latido de capa de aplicación**.

### 5.4.1 Mantenimiento de Conexión Persistente

Entre iFay_Runtime y el terminal, DEBE mantenerse un canal de conexión persistente (por ejemplo, WebSocket, flujo HTTP/2, flujo gRPC).

El terminal DEBE monitorear el estado de esta conexión persistente:

- Conexión establecida: Señal de conexión persistente válida
- Conexión rota: Señal de conexión persistente inválida (incluyendo TCP RST, timeout sin respuesta, etc.)

Una rotura de conexión persistente PUEDE ser el resultado de fluctuaciones de red temporales, por lo que NO DEBE usarse sola como base para terminar una Session (ver §5.4.3 doble determinación).

### 5.4.2 Latido de Capa de Aplicación

iFay_Runtime DEBE enviar periódicamente mensajes Heartbeat sobre la conexión persistente para cada Session activa:

```
Heartbeat (body of ProtocolMessage) {
  required session_id      : Session_ID
  required sequence_number : uint64
}

HeartbeatAck (body of ProtocolMessage) {
  required session_id      : Session_ID
  required sequence_number : uint64
}
```

| Parámetro | Predeterminado | Rango |
|------|--------|------|
| Intervalo de latido | 10 segundos | 1–60 segundos |
| Umbral de timeout de latido | 30 segundos | Intervalo de latido × 2 a intervalo de latido × 6 |

`sequence_number` es monotónicamente creciente dentro de cada Session, comenzando desde 0.

El terminal DEBE:

1. Devolver inmediatamente HeartbeatAck después de recibir Heartbeat (`sequence_number` coincide con la solicitud)
2. Actualizar el `last_heartbeat_at` de la Session
3. Rechazar Heartbeats con `sequence_number` menor que el máximo registrado (prevención de repetición)

### 5.4.3 Doble Determinación

El terminal DEBE determinar el fallo de Session solo cuando las siguientes dos condiciones se cumplan **simultáneamente**:

1. La conexión persistente ha estado rota por más tiempo que el umbral de timeout de latido
2. La duración desde `last_heartbeat_at` excede el umbral de timeout de latido

La intención de diseño de esta doble determinación es:

- Durante fluctuaciones a corto plazo de la conexión persistente, los latidos de capa de aplicación pueden estar todavía normales (no debería juzgarse erróneamente como fallido)
- Cuando los latidos de capa de aplicación se detienen pero la conexión persistente permanece (por ejemplo, Fay atascado), se necesita el timeout de latido para detección
- Ambas condiciones ocurriendo simultáneamente es una señal de fallo de alta confianza

### 5.4.4 Manejo Después del Fallo

Después de determinar el fallo, el terminal DEBE:

1. Cambiar el estado de Session: `active → terminating`
2. Liberar el Terminal_Resource ocupado
3. Revocar el acceso del Fay al recurso a través de la interfaz del SO
4. Mientras se espera la recuperación de la conexión persistente o el latido, todas las solicitudes para esa Session devuelven `E_SESSION_TERMINATED`
5. El terminal PUEDE empujar una notificación `SessionStateChanged` a iFay_Runtime después de la recuperación de la conexión persistente, pero la Session ya no es recuperable en este punto

### 5.4.5 Modos de Latido Opcionales

Las implementaciones PUEDEN soportar los siguientes modos de optimización de latido (con declaración de soporte explícita requerida):

1. **Latido agregado**: iFay_Runtime reporta múltiples Sesiones en un solo mensaje Heartbeat (aplicable a escenarios donde un runtime gestiona un gran número de Sesiones)
2. **Latido de frecuencia reducida**: Cuando una Session no ha iniciado acceso a recursos por mucho tiempo, el intervalo de latido puede expandirse hasta 60 segundos

El formato específico del latido agregado, como una extensión opcional, se define como campos opcionales en schema.json.

## 5.5 Terminación de Session

### 5.5.1 Condiciones de Activación de Terminación

| Razón de Activación | Valor del Campo reason | Activador |
|---------|---------------|--------|
| Liberación proactiva | `"client_release"` | iFay_Runtime |
| Timeout de latido | `"liveness_timeout"` | Liveness_Detection del terminal |
| Terminación por revocación | `"credential_revoked"` | Actualización de la lista de revocación del terminal |
| Recurso no disponible | `"resource_unavailable"` | Capa de integración del SO del terminal |
| Credencial expirada | `"credential_expired"` | Verificación periódica del terminal |
| Transferencia completa | `"handed_over"` | Flujo de transferencia del terminal |
| Terminación forzada | `"forced"` | Interfaz de gestión del terminal (por ejemplo, operaciones de admin) |

### 5.5.2 Pasos de Terminación

El terminal DEBE ejecutar la terminación siguiendo estos pasos:

1. **Transición de estado**: `{active|handover_pending} → terminating`
2. **Recuperación de recursos del SO**: Revocar el acceso a recursos del proceso Fay a través de la interfaz §1.3.2
3. **Liberar control de concurrencia**: Eliminar (resource_id, access_mode) de la lista de ocupación (ver Capítulo 7 actualización de bloqueo lectura-escritura)
4. **Notificar a iFay_Runtime**: Enviar un mensaje `SessionStateChanged` con estado `terminated`, incluyendo reason
5. **Transición de estado**: `terminating → terminated`
6. **Persistencia**: PUEDE escribir el registro histórico de Session al log de auditoría

### 5.5.3 Idempotencia de la Terminación

El mensaje SessionRelease iniciado por iFay_Runtime DEBE ser idempotente:

- Si la Session ya está en `terminating` o `terminated`, una liberación repetida DEBE devolver éxito (no tratada como un error)
- Si la Session no existe (ya recuperada), devolver `E_SESSION_NOT_FOUND`

### 5.5.4 Observabilidad de la Terminación

El terminal DEBE notificar la terminación de Session al menos a los siguientes:

1. La iFay_Runtime asociada con la Session (vía `SessionStateChanged`)
2. El subsistema local de gestión de recursos del terminal (para actualización del estado de disponibilidad)

El terminal PUEDE elegir si notificar:

- Otros iFay_Runtimes esperando ese recurso (vía el mecanismo de notificación de transferencia §6)
- Log de auditoría (dependiendo de la política de auditoría de la implementación del terminal)

## 5.6 Notificación de Cambio de Estado

```
SessionStateChanged (body of ProtocolMessage) {
  required session_id : Session_ID
  required new_state  : SessionState
  optional reason     : string
  optional details    : map<string, string>
}
```

El terminal DEBE enviar SessionStateChanged a iFay_Runtime cuando ocurran las siguientes transiciones de estado:

1. `creating → active`: Creación de Session exitosa (también puede comunicarse directamente vía AuthResult; cualquiera de los dos)
2. `active → handover_pending`: Session entra en el flujo de transferencia
3. `handover_pending → active`: Transferencia cancelada o rechazada
4. `* → terminating` y `terminating → terminated`: Inicio y fin del proceso de terminación

## 5.7 Relación entre Session y Ciclo de Vida de Credencial

El ciclo de vida de Session es independiente del ciclo de vida de credencial, pero el terminal DEBE monitorear cambios de estado de credencial:

| Evento de Credencial | Impacto en Session |
|---------|-------------|
| `not_after` de credencial alcanzado | Session asociada termina inmediatamente con `credential_expired` |
| Credencial revocada (declaración de revocación llega al terminal) | Session asociada termina inmediatamente con `credential_revoked` |
| Credencial re-emitida con mismo ID (teóricamente imposible, descriptor_id no es reutilizable) | No aplicable |
| Credencial sobrescrita por nueva versión (flujo de renovación) | Session no afectada (todavía validada usando el credential_ref original) |

## 5.8 Límites de Cantidad de Session

El terminal DEBERÍA implementar límites de cantidad de Session para prevenir agotamiento de recursos:

| Dimensión | Límite Predeterminado | Notas |
|------|---------|------|
| Total de Sesiones activas en un solo terminal | 1024 | Ajustable por política del terminal |
| Sesiones activas para un solo Fay | 64 | Previene que un solo Fay monopolice recursos del sistema |
| Sesiones read activas para un solo Resource_ID | 32 | Previene abuso de compartición de read |

Cuando se excede un límite, el terminal DEBE devolver `E_SESSION_LIMIT_EXCEEDED`.
