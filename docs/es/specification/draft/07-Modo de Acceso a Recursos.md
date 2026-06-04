# Capítulo 7: Modo de Acceso a Recursos

Este capítulo define la semántica de `Resource_Access_Mode`, la matriz de bloqueo lectura-escritura, la aplicación de condiciones de restricción y su relación con los permisos de Session. Este capítulo corresponde a la intención de diseño en §2.5 del plan arquitectónico.

## 7.1 Definiciones de Modo

El protocolo CAP define 4 modos de acceso:

```
AccessMode = enum["read", "write", "execute", "configure"]
```

### 7.1.1 read

**Semántica**: Acceso de solo lectura a Terminal_Resource sin modificar el estado del recurso.

Operaciones típicas:

- Leer datos en tiempo real de sensores (temperatura, humedad, ubicación)
- Ver archivos multimedia (fotos, videos)
- Consultar atributos de estado del dispositivo (nivel de batería, estado de conexión)

**Concurrencia**: Compartible. Múltiples Sesiones PUEDEN poseer simultáneamente acceso en modo lectura al mismo recurso.

### 7.1.2 write

**Semántica**: Escritura de datos o modificación de estado a Terminal_Resource.

Operaciones típicas:

- Escribir archivos a dispositivos de almacenamiento
- Modificar parámetros de operación del dispositivo (volumen, brillo, configuración de temperatura)
- Enviar entrada de contenido a ventanas de aplicación

**Concurrencia**: Exclusiva. En cualquier momento, una sola Session como máximo posee modo escritura para el mismo recurso.

### 7.1.3 execute

**Semántica**: Activar instrucciones de operación en Terminal_Resource.

Operaciones típicas:

- Lanzar programas de aplicación
- Activar instrucciones de operación de hardware (tomar fotos, despegue, imprimir)
- Llamar interfaces RPC

**Concurrencia**: Exclusiva.

La distinción entre `write` y `execute`: write modifica el estado de datos; execute activa acciones (incluso si ambos finalmente cambian el estado). La granularidad del control de acceso del terminal puede no distinguir estrictamente entre ambos; en tales casos, las implementaciones PUEDEN tratar execute como una subclase de write (poseer permiso write otorga automáticamente la capacidad execute). Sin embargo, la semántica predeterminada de esta especificación es que ambos son independientes — poseer permiso execute **NO** otorga automáticamente permiso write y viceversa.

### 7.1.4 configure

**Semántica**: Cambios de configuración a nivel de sistema en Terminal_Resource.

Operaciones típicas:

- Modificar parámetros de hardware del dispositivo (resolución de cámara, SSID de red)
- Ajustar políticas de seguridad (reglas de firewall, configuraciones de emparejamiento)
- Cambiar políticas de acceso a recursos (por ejemplo, configuración de `Handover_Policy`)

**Concurrencia**: Exclusiva y de alto privilegio. configure típicamente requiere autorización de nivel superior para ejercerse (ver §7.4.3).

## 7.2 Matriz de Bloqueo Lectura-Escritura

La matriz de bloqueo lectura-escritura define la compatibilidad de diferentes modos para el mismo recurso. La compatibilidad de **nuevas solicitudes con la ocupación existente** es como sigue:

| Ocupación Existente \ Nueva Solicitud | read | write | execute | configure |
|------------------|:----:|:-----:|:-------:|:---------:|
| Ninguna          |  ✓   |   ✓   |    ✓    |     ✓     |
| read (≥1)        |  ✓   |   ✗   |    ✗    |     ✗     |
| write            |  ✗   |   ✗   |    ✗    |     ✗     |
| execute          |  ✗   |   ✗   |    ✗    |     ✗     |
| configure        |  ✗   |   ✗   |    ✗    |     ✗     |

`✓`: Compatible; la nueva solicitud está permitida para crear inmediatamente una Session.
`✗`: Incompatible; la nueva solicitud es rechazada (devolver `E_RESOURCE_BUSY`).

### 7.2.1 Notas Semánticas de la Matriz

- **read mutuamente compatible**: Múltiples Sesiones de solo lectura pueden coexistir concurrentemente
- **write/execute/configure por pares incompatibles**: Cuando cualquier modo exclusivo ocupa el recurso, las solicitudes de otro modo exclusivo son rechazadas
- **read y modos exclusivos son mutuamente exclusivos**: Cuando read ocupa, las solicitudes exclusivas son prohibidas; cuando un modo exclusivo ocupa, las solicitudes read son prohibidas

### 7.2.2 Manejo del Terminal

Cuando el terminal maneja AuthRequest, DEBE:

1. Verificar la compatibilidad según la matriz después de que la validación pase y antes de que la Session entre en `creating`
2. Devolver `E_RESOURCE_BUSY` directamente cuando es incompatible; **NO** poner automáticamente en cola y esperar
3. iFay_Runtime puede elegir reintentar o iniciar una solicitud de transferencia (ver Capítulo 6)

## 7.3 Relación con Permisos de Session

### 7.3.1 Lista de Permisos

Cada Session lleva una lista `granted_modes` en la creación (ver §2.5). Los contenidos de la lista provienen de `Grant.modes` coincidentes en `Authorization_Descriptor.grants` o `Trusted_Ticket.grants`.

- Cuando Fay opera sobre el recurso a través de esta Session, solo puede usar los modos contenidos en `granted_modes`
- El `access_mode` especificado por iFay_Runtime en el AuthRequest DEBE ser un subconjunto de la lista de autorización Grant

### 7.3.2 Principio de Privilegio Mínimo

iFay_Runtime DEBERÍA solo solicitar el modo de privilegio mínimo necesario para la tarea actual en AuthRequest. Por ejemplo, cuando solo se necesita lectura de datos, solicitar `read` en lugar de `read` + `write`.

El emisor DEBERÍA autorizar según el principio de privilegio mínimo dentro de Authorization_Descriptor, evitando relajación innecesaria de privilegios.

### 7.3.3 Sin Inclusión Jerárquica entre Modos

En CAP v1, los modos de acceso son mutuamente independientes, con **ninguna** relación de inclusión jerárquica:

- Poseer `configure` NO DEBE otorgar automáticamente capacidades `read` / `write` / `execute`
- Poseer `write` NO DEBE otorgar automáticamente capacidad `read`
- Cada modo de acceso DEBE ser autorizado independientemente

Las implementaciones NO DEBEN expandir automáticamente el alcance de autorización.

### 7.3.4 Permisos No Pueden ser Elevados

Después de que se crea una Session, sus `granted_modes` NO DEBEN ser elevados dentro del ciclo de vida. Si Fay necesita modos de mayor privilegio:

1. iFay_Runtime libera proactivamente la Session actual
2. Inicia un nuevo AuthRequest usando una credencial que tiene los modos requeridos
3. El terminal crea una nueva Session según el flujo completo

Las implementaciones NO DEBEN proporcionar ninguna interfaz de tiempo de ejecución de "elevación de permiso".

## 7.4 Aplicación de Restricciones

`Grant.constraints` son condiciones de restricción adicionales aplicadas por el emisor a la autorización. Esta especificación define las siguientes claves de restricción estándar, que las implementaciones DEBEN soportar. PUEDEN soportar restricciones personalizadas (la semántica de las restricciones personalizadas se negocia entre emisores y terminales, pero DEBE ser declarada en la documentación del schema).

### 7.4.1 Restricción de Ventana de Tiempo

```
constraints["time_window"] = "08:00-22:00"           // Válido diariamente 8:00-22:00
constraints["time_window_tz"] = "Asia/Shanghai"     // Zona horaria, predeterminado UTC
```

Tiempo de validación: En el Paso 6 de §3.3.2, verificar si cada Grant satisface las restricciones. Si el tiempo actual no está en la ventana → Grant no coincide.

### 7.4.2 Restricción de Límite de Frecuencia

```
constraints["max_calls_per_hour"] = "60"
constraints["max_calls_per_day"]  = "1000"
```

El terminal DEBERÍA mantener conteos de llamadas para cada combinación (descriptor_id, fay_id, resource_id). Cuando se excede el límite → Grant no coincide (devolver `E_RATE_LIMIT_EXCEEDED`).

### 7.4.3 Restricción de Modo de Alta Sensibilidad

```
constraints["require_human_confirmation"] = "true"
```

Cuando `access_mode == "configure"` u otros escenarios que satisfacen esta restricción, el terminal DEBE solicitar una confirmación de una sola vez del usuario humano antes de la creación de Session. La UI de confirmación comparte el mismo canal con §6.4.3.

### 7.4.4 Restricción de Geocerca

```
constraints["geo_fence"] = "geohash:wx4g0bm"        // Válido solo dentro del rango geográfico especificado
constraints["geo_fence_radius_m"] = "1000"
```

La fuente de ubicación del terminal DEBERÍA usar GPS calibrado o posicionamiento de red. Cuando la ubicación no está disponible → Grant no coincide (rechazo conservador).

### 7.4.5 Manejo de Restricciones No Reconocidas

Cuando el terminal encuentra claves de constraints no reconocidas, DEBE:

- Rechazar el Grant por predeterminado (principio conservador)
- Devolver `E_UNSUPPORTED_CONSTRAINT` e incluir el nombre de la clave no reconocida

Al usar restricciones personalizadas, los emisores DEBERÍAN informar al terminal sobre la semántica de la restricción a través de otros canales para evitar fallo de autorización.

## 7.5 Mapeo de Modos al Control de Acceso del SO

El terminal hace cumplir la ejecución de access_mode a través de la interfaz de control de acceso del sistema operativo. Esta sección define el mapeo recomendado de modos a controles del SO. Las implementaciones específicas pueden diferir entre plataformas pero DEBEN mantener equivalencia semántica.

| Tipo de Recurso | read | write | execute | configure |
|---------|------|-------|---------|-----------|
| Archivos/directorios | Permiso de lectura del sistema de archivos | Permiso de escritura del sistema de archivos | Permiso de ejecución | Permiso de modificación de metadatos |
| Archivos de dispositivo | Lectura de dispositivo | Escritura de dispositivo | ioctl/interfaz de control | Registros de configuración del dispositivo |
| Procesos de aplicación | Consulta de estado | Inyección de entrada | Lanzar/llamar | Modificar parámetros de lanzamiento |
| Recursos de red | Recibir datos | Enviar datos | Establecer conexión | Modificar enrutamiento/firewall |
| Cámara | Leer frames | No aplicable | Control de captura/grabación | Ajustar resolución/tasa de cuadros |

Los mecanismos de control de acceso específicos de la plataforma (por ejemplo, SELinux, AppArmor, macOS Sandbox, Android Permission) son elegidos e integrados por la implementación del terminal.

## 7.6 Restricciones de Cambio de Modo

La misma (Session, Resource) DEBE mantener su access_mode inicial sin cambios dentro del ciclo de vida. Los cambios de modo se realizan a través de nuevas Sesiones:

1. Liberar la Session actual
2. Especificar el nuevo access_mode al crear una nueva Session
3. El terminal re-evalúa la compatibilidad de ocupación de recursos según la matriz

El terminal NO DEBE proporcionar ninguna interfaz de tiempo de ejecución de "cambio de modo".

## 7.7 Observabilidad del Acceso a Recursos

El terminal DEBERÍA registrar información clave para cada acceso a recursos (para propósitos de auditoría):

- session_id
- fay_id
- resource_id
- access_mode
- Tipo de operación (por ejemplo, tamaño de lectura, conteo de bytes de escritura)
- Marca de tiempo

El formato específico de log de auditoría queda fuera del alcance de esta especificación (ver el ítem de exclusión §3.2 del plan arquitectónico "Formato estandarizado de log de auditoría").

## 7.8 Límites de la Semántica de Modo

### 7.8.1 El modo read no impide que el recurso sea modificado concurrentemente

El modo read solo autoriza el permiso "lectura"; no garantiza que el recurso no será modificado por otros controladores durante el proceso de lectura:

- Múltiples Sesiones read no interfieren entre sí, pero si el recurso está en estado "ninguna Session activa" y es operado directamente por un usuario humano (evadiendo el protocolo CAP), las Sesiones read todavía pueden leer datos cambiantes
- El modo read no proporciona garantías de consistencia transaccional

### 7.8.2 Atomicidad del modo write

El modo write solo proporciona la garantía a nivel de protocolo de "como máximo una Session write en cualquier momento". La atomicidad de operaciones de escritura específicas (por ejemplo, fallo parcial de escritura) depende de la semántica subyacente del recurso y queda fuera del alcance del protocolo CAP.

### 7.8.3 Efectos Secundarios del modo execute

Las operaciones activadas por el modo execute PUEDEN cambiar el estado del recurso. Los efectos secundarios activados por Fay a través de execute son responsabilidad de ese Fay. El protocolo CAP no proporciona capacidad de rollback para operaciones execute.

## 7.9 Flujo de Decisión Completo

El flujo de decisión completo del terminal para manejar acceso a recursos:

```
[AuthRequest llega]
        │
        ▼
  ┌─────────────────────┐
  │ §3 / §4 validación de credencial │
  └──────┬──────────────┘
         │ Pasa
         ▼
  ┌─────────────────────┐
  │ §7.4 evaluación de restricción │
  │ (constraints)       │
  └──────┬──────────────┘
         │ Pasa
         ▼
  ┌─────────────────────┐
  │ §7.2 verificación de matriz de bloqueo lectura-escritura │
  └──────┬──────────────┘
         │ Compatible
         ▼
  ┌─────────────────────┐
  │ §5.2 Crear Session  │
  └──────┬──────────────┘
         │
         ▼
  ┌─────────────────────┐
  │ §7.5 entrega de control de acceso del SO │
  │                     │
  └──────┬──────────────┘
         │
         ▼
  [Devolver AuthResult granted]
```

El fallo en cualquier paso devuelve el código de error correspondiente y no entra en pasos subsiguientes.
