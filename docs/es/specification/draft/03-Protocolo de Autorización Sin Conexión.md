# Capítulo 3: Protocolo de Autorización Sin Conexión

Este capítulo define el flujo completo del protocolo del ciclo de vida de `Authorization_Descriptor`, incluyendo las cinco etapas de emisión, almacenamiento local, validación, revocación y renovación. El flujo en este capítulo corresponde a la intención de diseño en §2.1 del plan arquitectónico.

## 3.1 Flujo de Emisión

El flujo de emisión es ejecutado por Descriptor_Issuer después de recibir autorización explícita del autorizador. Esta especificación define las restricciones de formato del resultado de emisión pero no prescribe la interacción específica entre el autorizador y Descriptor_Issuer (diferentes despliegues pueden adoptar diferentes formas como formularios web, autorización de App móvil, sistemas de gestión de autorización empresarial, etc.).

### 3.1.1 Entradas de Emisión

En el momento de la emisión, Descriptor_Issuer DEBE haber confirmado ya la siguiente información:

1. La identidad del autorizador (`grantor_id`) ha pasado el propio mecanismo de verificación de identidad de Descriptor_Issuer
2. El alcance de autorización (`subject_fay_id`, `terminal_id`, `grants` objetivo) está explícitamente especificado por el autorizador
3. El período de validez (`not_before`, `not_after`) está explícitamente especificado por el autorizador o sigue una política predeterminada

### 3.1.2 Pasos de Emisión

Descriptor_Issuer DEBE generar Authorization_Descriptor siguiendo estos pasos:

1. **Generar identificador**: Asignar un nuevo `descriptor_id` (UUID v7); NO DEBE reutilizar un ID previamente usado
2. **Construir carga útil**: Llenar `DescriptorPayload` según §2.3.2; todos los campos requeridos DEBEN estar establecidos
3. **Validar restricciones**: Validar localmente restricciones como `not_after - not_before ≤ 90 días` (ver §2.3.2)
4. **Serialización CBOR**: Serializar la carga útil usando codificación determinista RFC 8949
5. **Firma digital**: Firmar los bytes de carga útil usando la clave privada de Descriptor_Issuer con el algoritmo definido en §8
6. **Ensamblar estructura**: Construir el `AuthorizationDescriptor` completo, incluyendo version, payload y signature
7. **Registrar registro**: Registrar la credencial en el almacén de estado interno de Descriptor_Issuer (para gestión de revocación subsiguiente)

### 3.1.3 Entrega Post-Emisión

Después de que la emisión esté completa, Descriptor_Issuer entrega Authorization_Descriptor al Fay objetivo. El método de entrega queda fuera del alcance de esta especificación, pero DEBERÍA satisfacer:

1. Entrega vía un canal cifrado para evitar la interceptación de la credencial durante la transmisión
2. Entrega al `iFay_Runtime` al cual pertenece el Fay, con iFay_Runtime poseyéndola en nombre del Fay

## 3.2 Flujo de Almacenamiento Local

Fay envía Authorization_Descriptor al terminal objetivo a través de iFay_Runtime para almacenamiento local.

### 3.2.1 Mensaje de Envío

iFay_Runtime envía un mensaje `DescriptorSubmit` a Protocol_Engine:

```
DescriptorSubmit (body of ProtocolMessage) {
  required descriptor : AuthorizationDescriptor
}
```

### 3.2.2 Procesamiento del Terminal

Después de recibir `DescriptorSubmit`, el terminal DEBE procesarlo en el siguiente orden:

1. **Validación estructural**: Verificar que `descriptor` se ajuste a la estructura definida en §2.3
2. **Validación de firma**: Verificar la firma usando el Verification_Key correspondiente a `key_id` (ver §3.3.4)
3. **Verificación de duplicados**: Si una credencial con el mismo `descriptor_id` ya está almacenada localmente y el contenido recién enviado coincide con el contenido almacenado byte por byte, devolver éxito; de lo contrario rechazar con un error de ID duplicado
4. **Almacenamiento**: Cifrar y almacenar el Authorization_Descriptor en el área de almacenamiento seguro local (ver §3.2.3)
5. **Devolver resultado**: Devolver un mensaje `DescriptorSubmitResult` a iFay_Runtime, conteniendo éxito o un código de error

### 3.2.3 Requisitos de Almacenamiento

El terminal DEBE:

1. Almacenar Authorization_Descriptor cifrado; la clave de cifrado NO DEBE ser legible por procesos no autorizados
2. El medio de almacenamiento DEBERÍA tener capacidad anti-manipulación física (por ejemplo, chip seguro, TEE)
3. El techo de capacidad de almacenamiento DEBERÍA no ser menor a 1024 credenciales; cuando se exceda el techo, las credenciales expiradas son desalojadas usando política LRU

El terminal NO DEBE:

1. Almacenar Authorization_Descriptor en formato de texto plano
2. Modificar cualquier campo del descriptor antes del almacenamiento (incluyendo metadata)

### 3.2.4 Códigos de Error

| Código de Error | Condición de Activación |
|-------|---------|
| `E_INVALID_STRUCTURE` | La estructura no se ajusta a §2.3 |
| `E_INVALID_SIGNATURE` | Validación de firma fallida |
| `E_UNKNOWN_ISSUER` | El Verification_Key correspondiente a `key_id` no está registrado en el terminal |
| `E_DUPLICATE_DESCRIPTOR_ID` | Conflicto con descriptor_id almacenado con contenido no coincidente |
| `E_STORAGE_FULL` | La capacidad de almacenamiento está llena y no es posible el desalojo |
| `E_VALIDITY_OUT_OF_RANGE` | `not_after - not_before` excede el límite |

## 3.3 Flujo de Validación

El flujo de validación se ejecuta en cada solicitud de acceso a recursos. Esta sección define los pasos completos y las reglas de decisión para la validación.

### 3.3.1 Activador de Validación

Cuando iFay_Runtime envía un `AuthRequest` solicitando acceso a recursos, el Protocol_Engine del terminal activa el flujo de validación:

```
AuthRequest (body of ProtocolMessage) {
  required fay_id        : Fay_ID
  required resource_id   : Resource_ID
  required access_mode   : AccessMode
  required credential    : CredentialReference
}

CredentialReference {
  required type : enum["descriptor", "ticket"]
  required id   : string                      // descriptor_id o jti
}
```

Nota: En el escenario de autorización sin conexión de esta especificación, `credential` referencia un Authorization_Descriptor ya almacenado en el terminal; no es necesario transmitir la credencial completa en la solicitud.

### 3.3.2 Pasos de Validación (Ejecutados en Orden)

El terminal DEBE ejecutar la validación en el siguiente orden. Un fallo en cualquier paso devuelve inmediatamente el código de error correspondiente sin proceder a pasos subsiguientes.

#### Paso 1: Existencia de la Credencial

El terminal busca el Authorization_Descriptor correspondiente a `credential.id` en el almacenamiento local.

- No encontrado → devolver `E_DESCRIPTOR_NOT_FOUND`

#### Paso 2: Estado de Revocación

El terminal verifica la lista de revocación local para confirmar que el `descriptor_id` no ha sido revocado.

- Revocado → devolver `E_DESCRIPTOR_REVOKED`

#### Paso 3: Período de Validez

El terminal verifica que el tiempo actual esté dentro del intervalo `[not_before, not_after]`.

- Tiempo actual < `not_before` → devolver `E_DESCRIPTOR_NOT_YET_VALID`
- Tiempo actual ≥ `not_after` → devolver `E_DESCRIPTOR_EXPIRED`

Fuente del reloj del terminal: DEBE usar un reloj de sistema calibrado. El terminal DEBERÍA manejar las solicitudes de validación con cautela cuando no esté en línea por períodos extendidos (ver §3.6 Manejo de Deriva del Reloj).

#### Paso 4: Coincidencia del Sujeto

El terminal verifica que `payload.subject_fay_id == AuthRequest.fay_id`.

- No coincide → devolver `E_SUBJECT_MISMATCH`

#### Paso 5: Coincidencia del Terminal

El terminal verifica que `payload.terminal_id == ID del terminal actual`.

- No coincide → devolver `E_TERMINAL_MISMATCH`

#### Paso 6: Coincidencia de Recurso y Modo

El terminal itera sobre `payload.grants` y busca un `Grant` que satisfaga:

1. `Grant.resource_pattern` coincide con `AuthRequest.resource_id` según las reglas de §2.3.5
2. `Grant.modes` contiene `AuthRequest.access_mode`
3. Si `Grant.constraints` no está vacío, todas las restricciones se satisfacen (ver §7.4 para semántica de restricciones)

- No se encuentra un Grant coincidente → devolver `E_AUTHORIZATION_INSUFFICIENT`

#### Paso 7: Validación de Firma

El terminal re-valida la firma de la carga útil usando el Verification_Key correspondiente a `signature.key_id`.

Nota: Las verificaciones rápidas de los pasos 1–6 se completan antes de la validación de firma, proporcionando una semántica de fallo razonable (por ejemplo, reportar "credencial revocada" antes de "firma fallida"). Sin embargo, antes de que la validación pase y se devuelva éxito, la validación de firma DEBE haberse ejecutado y pasado.

- Validación de firma fallida → devolver `E_INVALID_SIGNATURE`
- Clave de firma revocada o expirada → devolver `E_VERIFICATION_KEY_INVALID`

### 3.3.3 Después de que la Validación Pasa

Después de que los 7 pasos pasen, el terminal crea una Session según el flujo en el Capítulo 5 y devuelve a iFay_Runtime:

```
AuthResult (body of ProtocolMessage, success) {
  required status         : "granted"
  required session_id     : Session_ID
  required granted_modes  : array<AccessMode>
  required session_expires_at : timestamp
}
```

`session_expires_at` DEBERÍA igual a `min(not_after, tiempo actual + duración máxima de sesión predeterminada)`.

### 3.3.4 Detalles de Validación de Firma

Pasos específicos de validación de firma:

1. Extraer `descriptor.signature.key_id` y buscar la clave pública correspondiente en el almacén Verification_Key del terminal
2. Verificar que el Verification_Key sea válido en el tiempo actual (`valid_from ≤ tiempo actual ≤ valid_until`, si valid_until está establecido)
3. Re-serializar `descriptor.payload` en CBOR según las reglas de §2.3.3
4. Verificar la firma de los bytes serializados usando el Verification_Key y `descriptor.signature.algorithm`
5. Después de la validación exitosa, DEBE almacenar en caché el resultado (el mismo descriptor solo requiere validación de firma una vez durante su ciclo de vida)

## 3.4 Flujo de Revocación

### 3.4.1 Iniciación de Revocación

El autorizador inicia una solicitud de revocación a través de Descriptor_Issuer. La forma de interacción específica para solicitudes de revocación queda fuera del alcance de esta especificación.

Después de recibir una solicitud de revocación, Descriptor_Issuer DEBE:

1. Verificar que la solicitud de revocación sea del autorizador original o de una entidad con autoridad de revocación
2. Generar un `RevocationStatement` según §2.8
3. Firmar la declaración de revocación con la misma clave de firma usada para el Authorization_Descriptor original
4. Agregar la declaración de revocación a la lista de revocación mantenida por Descriptor_Issuer

### 3.4.2 Distribución de Revocación

Las declaraciones de revocación se distribuyen a los terminales a través de los siguientes métodos; el terminal DEBE soportar al menos dos de ellos:

1. **Sincronización basada en pull**: Cuando está en línea, el terminal extrae proactivamente listas de revocación incrementales de Descriptor_Issuer o un servicio de revocación. La frecuencia de sincronización está determinada por la política del terminal y DEBERÍA no ser menos de una vez por hora (durante períodos en línea)
2. **Notificación basada en push**: Descriptor_Issuer empuja declaraciones de revocación al terminal vía Registration_Authority o un servicio de distribución de revocación dedicado
3. **Incrustación en banda**: Un resumen de la lista de revocación reciente se transporta en los metadatos de Trusted_Ticket, permitiendo que los tickets obtenidos en línea transporten automáticamente información de revocación

### 3.4.3 Mantenimiento de la Lista de Revocación del Terminal

La lista de revocación local del terminal DEBE:

1. Almacenar permanentemente todas las declaraciones de revocación no expiradas
2. Después de que el tiempo `not_after` de una credencial haya pasado, PUEDE eliminar la declaración de revocación correspondiente (la credencial ha expirado naturalmente)
3. Validar la firma de cada declaración de revocación y rechazar declaraciones de revocación con firmas inválidas

### 3.4.4 Tiempo Efectivo de Revocación

El tiempo efectivo de una declaración de revocación es:

```
Tiempo efectivo = max(tiempo en que la declaración de revocación llega al terminal, RevocationStatement.revoked_at)
```

El terminal DEBE asegurar que las solicitudes de validación subsiguientes después de que una declaración de revocación llegue al terminal rechacen inmediatamente la credencial. El terminal PUEDE verificar proactivamente si todas las Sesiones activas referencian credenciales revocadas y, si es así, terminar forzosamente esas Sesiones según las reglas en el Capítulo 5.

### 3.4.5 Ventana de Retraso de Revocación

Debido al retraso inherente de la distribución sin conexión, existen las siguientes ventanas de retraso de revocación inevitables:

| Escenario | Retraso Máximo |
|------|---------|
| Terminal continuamente en línea | Un ciclo de sincronización (predeterminado ≤ 1 hora) |
| Terminal brevemente sin conexión | Próxima sincronización después de la reconexión |
| Terminal sin conexión a largo plazo | El retraso máximo es `not_after - tiempo de revocación` de la credencial, pero no más de 90 días (el período máximo de validez de la credencial) |

Los emisores DEBERÍAN limitar el retraso máximo de revocación estableciendo valores de `not_after` más cortos (por ejemplo, 7 días).

## 3.5 Flujo de Renovación

La renovación es esencialmente la emisión de un nuevo Authorization_Descriptor para reemplazar la versión antigua, no una modificación de la credencial existente.

### 3.5.1 Política de Renovación

El autorizador o un mecanismo de renovación automática puede activar la renovación en los siguientes escenarios:

1. La credencial se está acercando a `not_after` y la relación de autorización todavía es válida
2. El alcance de autorización necesita ajustarse (agregar/eliminar grants)
3. Las restricciones de autorización necesitan ajustarse (modificar constraints)

### 3.5.2 Flujo de Renovación

Proceso de renovación:

1. Descriptor_Issuer emite un nuevo Authorization_Descriptor siguiendo el flujo de §3.1, usando un nuevo `descriptor_id`
2. La nueva credencial se envía al terminal y se almacena siguiendo el flujo de §3.2
3. La credencial antigua PUEDE ser revocada proactivamente por el emisor (según §3.4) o PUEDE expirar naturalmente

Durante el período en que las credenciales nuevas y antiguas coexisten, las prioridades de procesamiento del terminal son:

- Las solicitudes de validación DEBEN coincidir preferencialmente con credenciales no expiradas
- Cuando múltiples credenciales no expiradas coinciden, se usa la credencial con el `issued_at` más reciente

### 3.5.3 Comportamientos de Renovación No Permitidos

Las implementaciones NO DEBEN:

1. Modificar cualquier campo de una credencial emitida (incluyendo metadata)
2. Reutilizar el `descriptor_id` de una credencial antigua
3. Modificar la semántica de una credencial antigua mientras todavía es válida (debe hacerse a través de revocación + nueva emisión)

## 3.6 Manejo de Deriva del Reloj

Los relojes de los terminales pueden derivar debido a operación sin conexión a largo plazo o problemas de hardware. Esta sección define las reglas de manejo bajo condiciones de deriva del reloj.

### 3.6.1 Tolerancia del Reloj

El terminal PUEDE introducir tolerancia al validar el período de validez:

- Para `not_before`: Puede permitir hasta 5 minutos de tolerancia temprana (acepta `tiempo actual ≥ not_before - 5min`)
- Para `not_after`: NO DEBE introducir tolerancia (expirado es expirado)

La tolerancia se usa solo para compensar desplazamientos de reloj a corto plazo y NO DEBERÍA usarse para evadir restricciones del período de validez.

### 3.6.2 Confiabilidad del Reloj

El terminal DEBERÍA detectar las siguientes anomalías del reloj y tomar medidas protectoras:

- Retroceso significativo del reloj (el reloj del sistema salta al pasado > 1 hora): Rechazar solicitudes de validación hasta la sincronización del reloj
- Avance significativo del reloj (el reloj del sistema salta al futuro > 24 horas): Rechazar solicitudes de validación hasta la sincronización del reloj

## 3.7 Diagrama de Flujo Completo

```
[Descriptor_Issuer]                    [Fay/iFay_Runtime]                   [Terminal/Protocol_Engine]
        │                                       │                                       │
        │── Emitir AuthorizationDescriptor ───→│                                       │
        │   (con firma digital)                 │                                       │
        │                                       │                                       │
        │                                       │── DescriptorSubmit ─────────────────→│
        │                                       │                                       │── Verificar firma + almacenar
        │                                       │←─ DescriptorSubmitResult ──────────── │
        │                                       │                                       │
        │                                       │── AuthRequest (con descriptor_id) ──→│
        │                                       │                                       │── Validación de 7 pasos
        │                                       │←─ AuthResult (granted) ──────────────│
        │                                       │                                       │── Crear Session
        │                                       │                                       │
        │                                       │       【Fay accede continuamente al recurso】│
        │                                       │                                       │
        │── Revocar RevocationStatement ─────────────────────────────────────────────────→│
        │                                       │                                       │── Agregar a lista de revocación
        │                                       │                                       │── Terminar forzosamente Session relacionada
```
