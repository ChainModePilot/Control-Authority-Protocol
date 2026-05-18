## Capítulo 4: Puntos de Integración Externos

Este capítulo describe los puntos de integración del protocolo CAP con sistemas externos, definiendo los límites de responsabilidad, la dirección de interacción y los requisitos de protocolo de comunicación de cada punto de integración. El protocolo CAP no opera de forma aislada; necesita colaborar con iFay_Runtime, el sistema operativo del terminal, los controladores de hardware y Registration_Authority, entre otros sistemas externos, para completar conjuntamente la gestión de permisos de control del terminal. Comprender los límites de interfaz y los modos de interacción de estos puntos de integración es un requisito previo para que los integradores de sistemas implementen correctamente el protocolo CAP.

### 4.1 Integración con iFay_Runtime

iFay_Runtime es el entorno de ejecución de Fay (iFay o coFay), responsable de la gestión del ciclo de vida y la planificación de las instancias Fay. Desde la perspectiva del protocolo CAP, iFay_Runtime es el iniciador de las solicitudes de control — cuando un Fay necesita acceder a recursos del terminal, iFay_Runtime inicia la solicitud de verificación de autorización ante la capa del protocolo CAP en su nombre.

**Responsabilidades de integración:**

- **Iniciar solicitudes de control**: iFay_Runtime envía solicitudes de verificación de autorización a Protocol_Engine en nombre del Fay, incluyendo en la solicitud el identificador de identidad del Fay, el Terminal_Resource de destino y las credenciales de autorización (Authorization_Descriptor o Trusted_Ticket)
- **Gestionar el ciclo de vida del Fay**: iFay_Runtime es responsable del inicio, pausa, reanudación y terminación de las instancias Fay. Cuando una instancia Fay se termina, iFay_Runtime notifica a Protocol_Engine para que libere todas las Sessions activas que posee ese Fay
- **Mantener el canal de detección de actividad**: iFay_Runtime es responsable de mantener la conexión persistente entre el Fay y el terminal, y de enviar mensajes de latido a nivel de aplicación según el intervalo configurado, soportando el funcionamiento normal del mecanismo de Liveness_Detection
- **Recibir notificaciones de estado de sesión**: Protocol_Engine notifica los cambios de estado de la Session (creación exitosa, solicitud de transferencia, terminación forzosa, etc.) a iFay_Runtime, que los transmite a la instancia Fay correspondiente

**Dirección de interacción:** Bidireccional (Bidirectional)

iFay_Runtime inicia solicitudes hacia la capa del protocolo CAP (verificación de autorización, liberación de sesión, envío de latidos), y la capa del protocolo CAP devuelve respuestas y envía notificaciones push a iFay_Runtime (resultados de verificación, cambios de estado de sesión, solicitudes de transferencia).

**Requisitos de protocolo de comunicación:**

- La comunicación entre iFay_Runtime y Protocol_Engine adopta un modelo de solicitud-respuesta basado en mensajes, cuyo formato de mensaje sigue la definición del Schema del protocolo CAP
- El canal de detección de actividad requiere soporte de conexión persistente para implementar latidos a nivel de aplicación y notificaciones push de estado en tiempo real
- El canal de comunicación debe soportar cifrado TLS, asegurando la confidencialidad e integridad de las credenciales de autorización y la información de sesión durante la transmisión
- El formato de serialización de mensajes debe ser consistente con el archivo de definición del Schema (schema.json), asegurando la interoperabilidad entre implementaciones

### 4.2 Integración con el Sistema Operativo del Terminal

El sistema operativo del terminal es el gestor de Terminal_Resource, responsable de la gestión unificada de los recursos de software y hardware del terminal. El protocolo CAP, a través de la integración con el sistema operativo, convierte los resultados de la verificación de autorización en control de acceso real a los recursos — el sistema operativo, según las instrucciones de Protocol_Engine, permite o deniega el acceso del Fay a recursos específicos.

**Responsabilidades de integración:**

- **Ejecutar el control de acceso a recursos**: El sistema operativo, según las instrucciones de control de acceso emitidas por Protocol_Engine, permite o bloquea a nivel de sistema el acceso del Fay al Terminal_Resource. Esto incluye el control de acceso al sistema de archivos, la gestión de permisos de procesos, los permisos de acceso a dispositivos, etc.
- **Reportar el estado de los recursos**: El sistema operativo reporta a Protocol_Engine el estado actual del Terminal_Resource (disponible, ocupado, no disponible), permitiendo que Protocol_Engine tome decisiones precisas de asignación de recursos
- **Aislar el entorno de ejecución**: El sistema operativo proporciona aislamiento a nivel de proceso o sandbox para las operaciones de diferentes Fays, asegurando que las operaciones de un Fay no afecten el acceso a recursos de otros Fays o usuarios humanos
- **Reenviar eventos de recursos**: Cuando el estado de un Terminal_Resource cambia (como desconexión de hardware, fallo de software), el sistema operativo reenvía la notificación del evento a Protocol_Engine, activando el proceso correspondiente de terminación de Session o recuperación de recursos

**Dirección de interacción:** Bidireccional (Bidirectional)

La capa del protocolo CAP emite instrucciones de control de acceso y solicitudes de consulta de recursos al sistema operativo, y el sistema operativo reporta el estado de los recursos y reenvía eventos de recursos a la capa del protocolo CAP.

**Requisitos de protocolo de comunicación:**

- La comunicación entre la capa del protocolo CAP y el sistema operativo adopta mecanismos de comunicación entre procesos locales (IPC), cuyo método específico depende de la plataforma del sistema operativo (como Unix Domain Socket, Named Pipe, D-Bus, etc.)
- Las instrucciones de control de acceso deben ejecutarse mediante llamadas síncronas, asegurando que Protocol_Engine pueda conocer inmediatamente el resultado de la ejecución de la instrucción
- Las notificaciones de eventos de recursos adoptan un modo de push asíncrono, donde el sistema operativo notifica proactivamente a Protocol_Engine cuando cambia el estado de los recursos
- La interfaz de comunicación debe seguir el modelo de seguridad nativo del sistema operativo, asegurando que solo los procesos de Protocol_Engine autorizados puedan emitir instrucciones de control de acceso

### 4.3 Integración con Controladores de Hardware

Los controladores de hardware son la interfaz de acceso a los Terminal_Resources de tipo hardware (como cámaras, micrófonos, módulos GPS, dispositivos de almacenamiento, etc.). El protocolo CAP, a través de la integración con los controladores de hardware, implementa la gestión refinada de los permisos de control sobre los recursos de hardware. Los controladores de hardware se sitúan por debajo del sistema operativo en el sistema de integración del protocolo CAP, proporcionando capacidades de acceso de bajo nivel a los recursos de hardware.

**Responsabilidades de integración:**

- **Proporcionar interfaz de acceso a recursos de hardware**: Los controladores de hardware exponen a la capa del protocolo CAP la descripción de capacidades y la interfaz de operación de los recursos de hardware, permitiendo que Protocol_Engine conozca los modos de acceso soportados por los recursos de hardware (read, write, execute, configure)
- **Ejecutar el control de acceso a nivel de hardware**: Los controladores de hardware, según las instrucciones de control transmitidas por Protocol_Engine a través del sistema operativo, ejecutan a nivel de hardware la concesión o denegación de acceso. Por ejemplo, permitir o prohibir que un Fay active la cámara o acceda al micrófono
- **Reportar el estado del hardware**: Los controladores de hardware reportan a las capas superiores el estado de conexión, el estado de funcionamiento y la información de anomalías de los dispositivos de hardware, información que finalmente se transmite a Protocol_Engine para las decisiones de gestión de Sessions
- **Soportar el control exclusivo de recursos de hardware**: Para los recursos de hardware que requieren acceso exclusivo (como la salida de flujo de video de la cámara), los controladores de hardware aseguran a nivel de bajo nivel que solo un controlador pueda usar exclusivamente el recurso en un momento dado

**Dirección de interacción:** Unidireccional (Unidirectional) — Capa del protocolo CAP → Controladores de hardware

La capa del protocolo CAP emite instrucciones de control a los controladores de hardware a través del sistema operativo, y la información de estado de los controladores de hardware se transmite hacia arriba a través de la capa de gestión de dispositivos del sistema operativo. La capa del protocolo CAP no establece un canal de comunicación directo con los controladores de hardware, sino que interactúa indirectamente a través del sistema operativo como intermediario. La ruta de reporte del estado del hardware es: Controladores de hardware → Sistema operativo → Protocol_Engine, y forma parte del punto de integración del sistema operativo (4.2).

**Requisitos de protocolo de comunicación:**

- La capa del protocolo CAP no se comunica directamente con los controladores de hardware; toda la interacción se completa indirectamente a través de las interfaces de gestión de dispositivos del sistema operativo (como DeviceIoControl, ioctl, sysfs, etc.)
- La descripción de capacidades de los recursos de hardware debe seguir un formato unificado de descripción de recursos, permitiendo que Protocol_Engine gestione de manera consistente diferentes tipos de recursos de hardware
- Los controladores de hardware deben soportar las interfaces estándar de control de acceso a dispositivos proporcionadas por el sistema operativo, asegurando que las instrucciones de control de acceso del protocolo CAP puedan hacerse efectivas a nivel de hardware
- Para los recursos de hardware de alta sensibilidad (como cámaras, micrófonos), los controladores de hardware deben soportar mecanismos de bloqueo de acceso a nivel de hardware, previniendo la elusión del control de acceso a nivel del sistema operativo

### 4.4 Integración con Registration_Authority

Registration_Authority (autoridad de registro) es un componente central de la infraestructura de confianza del protocolo CAP, responsable de la gestión del registro de hardware, software y sistemas operativos de terminales, así como de la distribución de claves de verificación (Verification_Key) a los terminales. La Verification_Key obtenida por el terminal a través de Registration_Authority es el ancla de confianza de la verificación de autorización sin conexión — sin una Verification_Key legítima, el terminal no puede verificar la firma digital del Authorization_Descriptor.

**Responsabilidades de integración:**

- **Registro de terminales**: Los dispositivos terminales (incluyendo hardware, sistema operativo y software cliente) completan el registro a través de Registration_Authority, obteniendo un identificador único de terminal e información de configuración inicial
- **Distribución de Verification_Key**: Registration_Authority distribuye Verification_Keys a los terminales registrados, que los terminales utilizan para verificar la firma digital de los Authorization_Descriptors. El proceso de distribución de claves debe completarse a través de un canal seguro, previniendo la manipulación o el robo de claves
- **Actualización y rotación de claves**: Cuando una Verification_Key necesita actualizarse (como expiración de clave, cambio de emisor), Registration_Authority es responsable de enviar la nueva clave al terminal y coordinar la transición fluida durante el proceso de rotación de claves
- **Gestión del estado de registro**: Registration_Authority mantiene el estado de registro del terminal, incluyendo registrado, suspendido y dado de baja. El estado de registro del terminal afecta si puede recibir actualizaciones de Verification_Key y participar en el flujo de verificación de autorización del protocolo CAP

**Dirección de interacción:** Unidireccional (Unidirectional) — Registration_Authority → Terminal

Registration_Authority distribuye Verification_Keys e información de registro a los terminales, que los reciben pasivamente. Los terminales no inician solicitudes de verificación de autorización ni reportan el estado de ejecución a Registration_Authority — la verificación de autorización del terminal se completa completamente de forma local (usando la Verification_Key ya distribuida), sin necesidad de mantener comunicación en tiempo real con Registration_Authority. Las solicitudes de registro que el terminal inicia hacia Registration_Authority pertenecen a un flujo de inicialización único y no forman parte de la interacción habitual durante la ejecución del protocolo CAP.

**Requisitos de protocolo de comunicación:**

- La comunicación entre Registration_Authority y los terminales debe completarse a través de un canal seguro (como TLS/mTLS), asegurando la confidencialidad e integridad de la Verification_Key durante la transmisión
- El protocolo de distribución de claves debe soportar dos modos: preinstalación sin conexión y actualización en línea. Los terminales pueden venir preinstalados con una Verification_Key inicial de fábrica, y posteriormente recibir actualizaciones de claves a través del canal en línea
- Los mensajes de actualización de claves deben incluir número de versión y tiempo de entrada en vigor, soportando un período de transición fluida entre claves antiguas y nuevas, evitando interrupciones en la verificación de autorización causadas por la rotación de claves
- Registration_Authority debe proporcionar un mecanismo de confirmación de distribución de claves, asegurando que el terminal haya recibido y almacenado exitosamente la nueva Verification_Key

### 4.5 Resumen de Direcciones de Interacción y Protocolos de Comunicación de los Puntos de Integración

La siguiente tabla resume la información de los puntos de integración del protocolo CAP con los 4 sistemas externos, incluyendo la dirección de interacción y los requisitos de protocolo de comunicación:

| Punto de Integración | Sistema Externo | Dirección de Interacción | Requisitos de Protocolo de Comunicación |
|---------------------|-----------------|-------------------------|----------------------------------------|
| 4.1 | iFay_Runtime | Bidireccional (Bidirectional) | Modelo de solicitud-respuesta basado en mensajes; soporte de conexión persistente (detección de actividad); cifrado TLS; formato de mensaje según la definición del Schema de CAP |
| 4.2 | Sistema operativo del terminal | Bidireccional (Bidirectional) | Comunicación entre procesos locales (IPC); llamada síncrona + push asíncrono de eventos; siguiendo el modelo de seguridad nativo del sistema operativo |
| 4.3 | Controladores de hardware | Unidireccional (CAP → Controladores de hardware) | Comunicación indirecta a través de la interfaz de gestión de dispositivos del sistema operativo; formato unificado de descripción de recursos; soporte de bloqueo de acceso a nivel de hardware |
| 4.4 | Registration_Authority | Unidireccional (RA → Terminal) | Canal seguro TLS/mTLS; soporte de preinstalación sin conexión y actualización en línea; gestión de versiones de claves y rotación fluida |

**Explicación de las direcciones de interacción:**

- **Bidireccional (Bidirectional)**: Existe interacción bidireccional de solicitud-respuesta o notificación de eventos entre la capa del protocolo CAP y el sistema externo; ambas partes pueden iniciar la comunicación activamente
- **Unidireccional (Unidirectional)**: El flujo de información va solo de una parte a otra. En 4.3, la capa del protocolo CAP emite instrucciones a los controladores de hardware a través del sistema operativo (sin comunicación directa); en 4.4, Registration_Authority distribuye claves e información de registro a los terminales (los terminales reciben pasivamente)

**Principios de diseño del protocolo de comunicación:**

- **Seguridad**: Todos los canales de comunicación que involucran la transmisión de credenciales de autorización y claves deben estar cifrados, previniendo ataques de intermediario y fugas de información
- **Adaptabilidad a la plataforma**: La integración con el sistema operativo y los controladores de hardware adopta interfaces nativas de la plataforma, asegurando la compatibilidad en diferentes sistemas operativos
- **Interoperabilidad**: La integración con iFay_Runtime y Registration_Authority adopta formatos de mensaje estandarizados (basados en el Schema de CAP), asegurando la interoperabilidad entre diferentes implementaciones
- **Tolerancia a fallos**: El protocolo de comunicación debe soportar mecanismos de reintento y degradación, sin afectar el funcionamiento normal de las funciones centrales del protocolo CAP (especialmente la verificación de autorización sin conexión) cuando la comunicación se interrumpe
