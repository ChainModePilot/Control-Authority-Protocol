## Capítulo 3: Hoja de Ruta de Evolución de Versiones

Este capítulo define la hoja de ruta de evolución de versiones del protocolo CAP, establece el alcance funcional y las exclusiones de la versión v1, define las reglas de nomenclatura de versiones y la estrategia de compatibilidad, y describe el flujo de publicación desde el borrador hasta la versión oficial. La hoja de ruta de versiones proporciona una guía clara para el desarrollo iterativo del protocolo y la colaboración comunitaria.

### 3.1 Alcance Funcional de la Versión v1

La versión v1 del protocolo CAP se centra en establecer un marco base completo de gestión de permisos de control de terminales, e incluye las siguientes 6 capacidades centrales:

1. **Autorización sin conexión (Authorization_Descriptor)**: El mecanismo central de la versión v1. Implementa la gestión completa del ciclo de vida de Authorization_Descriptor, incluyendo emisión, almacenamiento local, verificación, revocación y renovación. El terminal puede completar la verificación de autorización de forma independiente en estado sin conexión a través del Authorization_Descriptor almacenado localmente, garantizando la disponibilidad básica del Fay en entornos sin red. La versión v1 implementará el modelo completo de cadena de confianza y el mecanismo de lista de revocación local.

2. **Ticket en línea (Trusted_Ticket)**: El mecanismo complementario de la versión v1. Implementa las capacidades de emisión de autorización en tiempo real y consulta del estado de revocación en entornos conectados, soporta la conversión de Trusted_Ticket al formato local de Authorization_Descriptor, así como la estrategia de degradación fluida de en línea a sin conexión. La versión v1 asegura el funcionamiento coordinado entre el ticket en línea y la autorización sin conexión.

3. **Gestión de sesiones (Session lifecycle)**: Implementa la gestión completa del ciclo de vida de las sesiones de control, cubriendo las tres fases de creación, actividad y terminación. La versión v1 soporta la relación de vinculación uno a uno entre Session y Terminal_Resource, e implementa tres formas de terminación de sesión: liberación voluntaria, terminación por tiempo de espera y terminación por revocación.

4. **Estrategia de transferencia de control (Handover_Policy)**: Implementa la capacidad de transferencia de control entre múltiples controladores, cubriendo los escenarios de transferencia entre Fays y entre Fays y usuarios humanos. La versión v1 soporta tres tipos de estrategia (script de reglas de prioridad, determinación en tiempo real por modelo de IA, decisión del usuario humano), garantiza la atomicidad del proceso de transferencia e implementa el mecanismo de retroceso por tiempo de espera.

5. **Detección de actividad (Liveness_Detection)**: Implementa el mecanismo de detección de actividad de sesiones basado en la combinación de conexiones persistentes y latidos a nivel de aplicación. La versión v1 soporta la lógica de determinación dual (la conexión persistente se interrumpe y el latido expira simultáneamente), intervalos de latido y umbrales de tiempo de espera configurables, así como la terminación automática de sesiones inactivas y la recuperación de recursos.

6. **Modo de acceso a recursos (Resource_Access_Mode)**: Implementa la gestión escalonada del acceso a recursos por tipo de operación, soportando los cuatro modos de acceso: read, write, execute y configure. La versión v1 implementa el modelo completo de bloqueo de lectura-escritura, soportando el acceso concurrente multipartito en modo read y el control exclusivo en los modos write, execute y configure.

Las 6 capacidades centrales anteriores constituyen el conjunto funcional completo de la versión v1, proporcionando un marco base de permisos de control para que los agentes inteligentes del ecosistema iFay asuman de forma segura el control de los terminales.

### 3.2 Funcionalidades Explícitamente Excluidas de v1

Las siguientes funcionalidades no están incluidas explícitamente en la versión v1 y se marcan como candidatas para versiones futuras:

| Funcionalidad Excluida | Descripción | Versión Candidata |
|------------------------|-------------|-------------------|
| **Migración de sesiones entre terminales** | Migrar una Session activa de un terminal a otro, manteniendo la continuidad del estado de la sesión | v2+ |
| **Autorización colaborativa multi-terminal** | Mecanismo de sincronización del estado de autorización y verificación colaborativa entre múltiples terminales | v2+ |
| **Cadena de delegación de autorización (Delegation Chain)** | Mecanismo de autorización en cascada donde un Fay delega parte de su autorización obtenida a otros Fays | v2+ |
| **Control de acceso a recursos de grano fino (ABAC)** | Estrategia de control de acceso basada en atributos, soportando expresiones más complejas de condiciones de acceso a recursos | v2+ |
| **Formato estandarizado de registros de auditoría** | Formato de salida unificado de registros de auditoría (Audit_Logger) y especificación de interfaz de consulta | v2+ |
| **Especificación de transmisión cifrada de mensajes del protocolo** | Definición estandarizada del método de cifrado y el proceso de negociación de claves para los mensajes del protocolo CAP en la capa de transporte de red | v2+ |
| **Consenso distribuido de revocación de autorización sin conexión** | Mecanismo de consenso para alcanzar acuerdo sobre el estado de revocación entre múltiples terminales en entornos sin conexión | v3+ |
| **Elevación dinámica de permisos (dentro de Session)** | Elevar dinámicamente los permisos de acceso durante el ciclo de vida de una Session activa sin necesidad de crear una nueva Session | v3+ |

La versión v1 sigue el principio de "primero establecer la base, luego expandir las capacidades", priorizando la integridad y estabilidad de las 6 capacidades centrales y dejando las funcionalidades avanzadas para iteraciones de versiones posteriores.

### 3.3 Reglas de Nomenclatura de Versiones y Estrategia de Compatibilidad

#### Reglas de Nomenclatura de Versiones

El protocolo CAP adopta el formato de versionado semántico (Semantic Versioning): **`MAJOR.MINOR.PATCH`**

- **MAJOR (versión principal)**: Se incrementa cuando el protocolo sufre cambios importantes no retrocompatibles. Por ejemplo, ajustes estructurales en el formato de mensajes centrales, cambios fundamentales en el flujo de verificación de autorización. Un cambio en la versión principal significa que las implementaciones existentes necesitan modificaciones de adaptación
- **MINOR (versión secundaria)**: Se incrementa cuando el protocolo añade funcionalidades o capacidades retrocompatibles. Por ejemplo, añadir un nuevo modo de acceso a recursos, ampliar los tipos de estrategia de Handover_Policy. Un cambio en la versión secundaria no afecta el funcionamiento normal de las implementaciones existentes
- **PATCH (número de revisión)**: Se incrementa cuando el protocolo realiza correcciones de errores o aclaraciones de documentación retrocompatibles. Por ejemplo, corregir descripciones ambiguas en la documentación de la especificación, complementar instrucciones de manejo de casos límite

Las versiones en borrador utilizan el identificador **`draft`**, sin número de versión adjunto. Las versiones en borrador no garantizan ninguna compatibilidad y pueden sufrir cambios disruptivos en cualquier momento.

Ejemplos de números de versión publicados oficialmente: `1.0.0`, `1.1.0`, `1.1.1`, `2.0.0`

#### Estrategia de Compatibilidad

La estrategia de compatibilidad del protocolo CAP sigue los siguientes principios:

- **Retrocompatibilidad dentro de la misma versión principal**: Dentro de la misma versión MAJOR, las nuevas versiones de la implementación del protocolo deben poder procesar los mensajes del protocolo de versiones anteriores. Por ejemplo, un terminal que implementa v1.2.0 debe poder procesar Authorization_Descriptors en formato v1.0.0
- **Sin garantía de compatibilidad entre versiones principales**: No se garantiza la retrocompatibilidad entre diferentes versiones MAJOR. Cuando se incrementa la versión principal, la especificación del protocolo listará explícitamente todos los cambios disruptivos y la guía de migración
- **Alineación de la versión del Schema con la versión del protocolo**: Cada versión del protocolo corresponde a una versión del Schema, y los archivos del Schema se almacenan en directorios nombrados con el número de versión del protocolo (como `schema/2025-10-25/`), asegurando que la relación de correspondencia de versiones sea clara y rastreable
- **Declaración de versión mínima soportada**: Las implementaciones del protocolo pueden declarar la versión mínima del protocolo que soportan; los mensajes del protocolo inferiores a esa versión serán rechazados

### 3.4 Flujo de Publicación desde Borrador hasta Versión Oficial

La publicación del protocolo CAP desde borrador hasta versión oficial sigue el siguiente flujo:

**Fase uno: Borrador (Draft)**

- Las especificaciones del protocolo y las definiciones del Schema se almacenan en el directorio `draft/` (como `schema/draft/`, `specification/draft/`)
- La fase de borrador permite cambios disruptivos frecuentes, sin compromiso de compatibilidad
- El contenido del borrador acepta retroalimentación y revisión de la comunidad, con iteración y optimización continuas
- Los documentos y el Schema de la fase de borrador pueden actualizarse en cualquier momento, sin necesidad de seguir las reglas de incremento de número de versión

**Fase dos: Candidata a Publicación (Release Candidate)**

- Cuando el contenido del borrador se estabiliza, entra en la fase de candidata a publicación
- Las versiones candidatas a publicación utilizan el sufijo `-rc.N` (como `1.0.0-rc.1`)
- La fase de candidata a publicación solo acepta correcciones de errores y aclaraciones de documentación, no acepta nuevas funcionalidades
- Las versiones candidatas a publicación deben pasar una verificación de pruebas completa, incluyendo validación del Schema, verificación de equivalencia estructural de documentos multilingües y pruebas de propiedades

**Fase tres: Publicación Oficial (Release)**

- Tras la verificación suficiente de la versión candidata a publicación, se elimina el sufijo `-rc.N` y se convierte en versión oficial
- Las especificaciones del protocolo y las definiciones del Schema de la versión oficial se copian del directorio `draft/` a un directorio nombrado con la fecha de la versión (como `schema/2025-10-25/`)
- Tras la publicación de la versión oficial, las especificaciones del protocolo y las definiciones del Schema de esa versión ya no se modifican (inmutables); cualquier cambio se publica a través de una nueva versión
- Al momento de la publicación de la versión oficial, los documentos del plan arquitectónico y las especificaciones del protocolo en los 9 idiomas deben estar completados sincrónicamente y haber pasado la verificación de equivalencia estructural

**Fase cuatro: Mantenimiento (Maintenance)**

- Tras la publicación de la versión oficial, entra en fase de mantenimiento, aceptando solo correcciones de errores a nivel PATCH
- Cuando se publica una nueva versión MAJOR o MINOR, la versión anterior entra en un período de mantenimiento limitado y finalmente se marca como obsoleta (deprecated)
- Los documentos y las definiciones del Schema de las versiones obsoletas se conservan en el repositorio para referencia histórica, pero ya no aceptan ninguna actualización
