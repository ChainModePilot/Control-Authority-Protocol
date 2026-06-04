# Capítulo 1: Arquitectura y Roles

Este capítulo define los roles en el protocolo CAP, las relaciones de confianza entre ellos y los contratos de interfaz con sistemas externos. El contenido de este capítulo proporciona la base terminológica compartida y el contexto arquitectónico para los capítulos subsiguientes.

## 1.1 Roles del Protocolo

El protocolo CAP involucra los siguientes roles normativos. Cada rol asume responsabilidades claras en las interacciones del protocolo e interactúa con otros roles a través de interfaces restringidas.

### 1.1.1 Rol de Autorizador

**Natural_Person** y **Official_Post** son las fuentes últimas de autorización. Los autorizadores no participan directamente en interacciones de mensajes del protocolo — los autorizadores delegan la autoridad de emisión a `Descriptor_Issuer` a través de un mecanismo de delegación de autorización.

Los autorizadores DEBEN demostrar su identidad a través de un mecanismo seguro de verificación de identidad. El mecanismo específico de verificación de identidad del autorizador queda fuera del alcance de esta especificación, pero tal mecanismo DEBE satisfacer los siguientes requisitos:

- No repudio: Los autorizadores no pueden repudiar las decisiones de autorización que han tomado
- Anti-falsificación: Terceros no pueden falsificar la decisión de autorización de un autorizador

### 1.1.2 Rol de Emisor

**Descriptor_Issuer** es responsable de generar y emitir `Authorization_Descriptor`. **Ticket_Issuer** es responsable de emitir `Trusted_Ticket`. Una sola entidad PUEDE asumir ambos roles de emisor simultáneamente.

Los emisores DEBEN:

1. Emitir credenciales solo después de recibir autorización explícita de un autorizador legítimo
2. Aplicar firmas digitales a las credenciales usando los algoritmos de firma definidos en el Capítulo 8
3. Mantener registros de estado de las credenciales emitidas (incluyendo como mínimo tiempo de emisión, período de validez y estado actual)
4. Actualizar prontamente el estado de revocación y publicar declaraciones de revocación al recibir solicitudes de revocación

Los emisores NO DEBEN:

1. Emitir credenciales sin obtener autorización
2. Emitir credenciales más allá del alcance de autorización del autorizador
3. Reutilizar identificadores de credenciales revocadas

### 1.1.3 Rol de Titular

**Fay** (incluyendo `iFay` y `coFay`) es el titular de credenciales y el solicitante del acceso a recursos. Fay participa en interacciones del protocolo indirectamente a través de `iFay_Runtime` — Fay mismo no envía mensajes de protocolo directamente.

Fay DEBE completar todas las interacciones del protocolo con el terminal a través de su `iFay_Runtime` asociado. Las credenciales que Fay posee DEBEN ser enviadas al terminal a través de iFay_Runtime.

### 1.1.4 Rol de Runtime

**iFay_Runtime** es el entorno de ejecución para instancias de Fay y asume el envío y recepción de mensajes del protocolo. Una sola instancia de iFay_Runtime PUEDE gestionar múltiples instancias de Fay.

iFay_Runtime DEBE satisfacer los requisitos de nivel de conformidad de runtime definidos en la Sección 0.5.3.

### 1.1.5 Rol de Terminal

El **terminal** (Terminal) es el titular de `Terminal_Resource` y el ejecutor del control de acceso. El terminal integra los siguientes componentes normativos:

- **Protocol_Engine**: Componente del sistema que ejecuta la lógica central del protocolo CAP
- **Descriptor_Validator**: Componente responsable de verificar la legitimidad y validez de `Authorization_Descriptor`
- **Gestor de sesiones**: Mantiene la máquina de estados de Session y el mecanismo Liveness_Detection
- **Lista de revocación local**: Almacena identificadores conocidos de credenciales revocadas

El terminal DEBE satisfacer los requisitos de nivel de conformidad de terminal definidos en la Sección 0.5.1.

### 1.1.6 Rol de Infraestructura de Confianza

**Registration_Authority** es responsable de la gestión del registro del terminal y la distribución de `Verification_Key`. Registration_Authority es el ancla raíz de la cadena de confianza del protocolo CAP.

Registration_Authority DEBE:

1. Distribuir Verification_Key solo a terminales que hayan pasado la verificación de identidad
2. Completar la distribución de claves a través de canales seguros (ver Capítulo 8)
3. Proporcionar mecanismos de actualización y rotación de claves
4. Mantener registros autorizados del estado de registro del terminal

## 1.2 Cadena de Confianza

La cadena de confianza del protocolo CAP se propaga del autorizador al `Descriptor_Validator` del terminal, compuesta por dos rutas de confianza independientes pero cooperantes.

### 1.2.1 Ruta de Confianza de Autorización

La ruta de confianza de autorización determina **si un Fay particular está autorizado para acceder a un recurso particular**:

```
Autorizador (Natural_Person / Official_Post)
    │
    │ ① Delegación de autorización (con evidencia no repudiable)
    ▼
Descriptor_Issuer
    │
    │ ② Emite Authorization_Descriptor (con firma digital)
    ▼
Fay
    │
    │ ③ Envía Authorization_Descriptor
    ▼
Descriptor_Validator
```

El terminal DEBE verificar la autenticidad de cada eslabón en la ruta de confianza de autorización:

- La autenticidad del paso ① la verifica Descriptor_Issuer en el momento de la emisión; el terminal no participa directamente
- La autenticidad del paso ② la verifica el terminal verificando la firma digital de Descriptor_Issuer (dependiendo de la ruta de confianza de claves)
- La autenticidad del paso ③ la verifica el terminal verificando la relación de enlace entre Authorization_Descriptor y el identificador de Fay que envía

### 1.2.2 Ruta de Confianza de Claves

La ruta de confianza de claves determina **si el terminal confía en la firma de un Descriptor_Issuer particular**:

```
Registration_Authority (ancla de confianza)
    │
    │ ① Distribuye Verification_Key (vía canal seguro)
    ▼
Almacén de claves local del terminal
    │
    │ ② Proporciona Verification_Key al Descriptor_Validator
    ▼
Descriptor_Validator
    │
    │ ③ Verifica la firma de Authorization_Descriptor usando Verification_Key
    ▼
Validación aprobada / rechazada
```

El terminal DEBE:

1. Confiar solo en Verification_Keys distribuidos por Registration_Authority
2. Almacenar Verification_Keys de forma segura (ver Capítulo 8 para requisitos de almacenamiento de claves)
3. Rechazar la verificación de firma usando claves no distribuidas por Registration_Authority

## 1.3 Contratos de Interfaz Externa

La capa del protocolo CAP interactúa con 4 sistemas externos a través de interfaces claramente definidas. Esta sección define los contratos normativos de estas interfaces.

### 1.3.1 Interfaz con iFay_Runtime

La interfaz entre iFay_Runtime y el Protocol_Engine del terminal es una **interfaz de mensajes bidireccional**.

**Dirección del mensaje: iFay_Runtime → Protocol_Engine**

| Tipo de Mensaje | Propósito | Campos Requeridos |
|---------|------|---------|
| `AuthRequest` | Inicia una solicitud de validación de autorización | fay_id, resource_id, access_mode, credential |
| `SessionRelease` | Libera una sesión proactivamente | session_id |
| `Heartbeat` | Latido de capa de aplicación | session_id, sequence_number |
| `HandoverResponse` | Responde a una solicitud de transferencia | handover_id, accept |

**Dirección del mensaje: Protocol_Engine → iFay_Runtime**

| Tipo de Mensaje | Propósito | Campos Requeridos |
|---------|------|---------|
| `AuthResult` | Resultado de validación de autorización | request_id, status, session_id (en éxito), error_code (en fallo) |
| `SessionStateChanged` | Notificación de cambio de estado de sesión | session_id, new_state, reason |
| `HandoverRequest` | Notificación de solicitud de transferencia | handover_id, target_resource_id, deadline |
| `HeartbeatAck` | Confirmación de latido | session_id, sequence_number |

La implementación de la interfaz DEBE:

1. Soportar conexiones persistentes para llevar `Heartbeat` y mensajes push asíncronos
2. Soportar transmisión cifrada TLS
3. Usar un formato de serialización que siga la definición `schema.json`

La implementación de la interfaz PUEDE:

1. Elegir un protocolo de transporte específico (WebSocket, gRPC, HTTP/2, etc.)

### 1.3.2 Interfaz con el Sistema Operativo del Terminal

La interfaz entre Protocol_Engine y el sistema operativo del terminal es una **interfaz bidireccional local**.

El sistema operativo DEBE exponer las siguientes capacidades a Protocol_Engine:

1. Entrega de control de acceso a recursos: Protocol_Engine emite instrucciones de "permitir a Fay X acceder al recurso Z en modo Y"
2. Consulta de estado de recursos: Protocol_Engine consulta la disponibilidad actual y el estado de ocupación de los recursos
3. Suscripción a eventos de recursos: Protocol_Engine se suscribe a eventos de cambio de estado de recursos (por ejemplo, desconexión de hardware, fallo de software)

La implementación específica de la interfaz depende de la plataforma del sistema operativo (Unix Domain Socket, Named Pipe, D-Bus, etc.). Esta especificación no prescribe la implementación específica, pero DEBE satisfacer:

1. Las instrucciones de control de acceso se ejecutan sincrónicamente y devuelven resultados de ejecución
2. Los eventos de recursos se notifican vía push asíncrono
3. La interfaz está protegida por el modelo de seguridad nativo del sistema operativo, llamable solo por procesos Protocol_Engine autorizados

### 1.3.3 Interfaz con Controladores de Hardware (Indirecta)

Protocol_Engine **NO DEBE** comunicarse directamente con los controladores de hardware. Todo el control de hardware se reenvía a través del sistema operativo.

Ruta de reporte de estado desde controladores de hardware a Protocol_Engine:

```
Controlador de hardware → Sistema operativo → Protocol_Engine
```

Los controladores de hardware DEBERÍAN implementar mecanismos de bloqueo de acceso a nivel de hardware para asegurar que incluso si los controles de acceso a nivel de sistema operativo son evadidos, el hardware mismo todavía pueda rechazar el acceso no autorizado.

### 1.3.4 Interfaz con Registration_Authority

La interfaz entre Registration_Authority y el terminal es una **interfaz unidireccional** (RA → terminal).

Registration_Authority DEBE empujar los siguientes mensajes al terminal:

| Tipo de Mensaje | Propósito | Campos Requeridos |
|---------|------|---------|
| `VerificationKeyDistribution` | Distribuye una nueva clave de verificación | key_id, key_material, valid_from, issuer_id |
| `VerificationKeyRevocation` | Revoca una clave de verificación | key_id, revocation_time, reason |
| `RegistrationStatusUpdate` | Actualiza el estado de registro del terminal | terminal_id, new_status |

El terminal DEBE:

1. Recibir mensajes sobre un canal seguro TLS/mTLS
2. Verificar que la fuente de la firma del mensaje sea una Registration_Authority de confianza
3. Devolver una confirmación a Registration_Authority después de recibir una nueva clave

El terminal no inicia proactivamente mensajes de protocolo a Registration_Authority (el flujo de registro inicial del terminal no pertenece a las interacciones normales en tiempo de ejecución del protocolo CAP y queda fuera del alcance de esta especificación).

## 1.4 Mapeo de Roles y Niveles de Conformidad

| Rol Implementado | Nivel de Conformidad Requerido |
|-----------|---------------------|
| Terminal (incluyendo Protocol_Engine y Descriptor_Validator) | Nivel de Conformidad de Terminal (§0.5.1) |
| Descriptor_Issuer o Ticket_Issuer | Nivel de Conformidad de Emisor (§0.5.2) |
| iFay_Runtime | Nivel de Conformidad de Runtime (§0.5.3) |
| Registration_Authority | Fuera del alcance de conformidad de esta especificación (el flujo de registro excede el tiempo de ejecución del protocolo CAP) |
