## Capítulo 2: Matriz de Capacidades Centrales

Este capítulo proporciona una descripción a nivel estratégico de las cinco capacidades centrales del protocolo CAP, abarcando la autorización sin conexión, los tickets en línea, la gestión de sesiones, la estrategia de transferencia de control y los modos de acceso a recursos. Cada capacidad se desarrolla desde las dimensiones de motivación de diseño, ciclo de vida, mecanismos clave y configuración de estrategias, proporcionando orientación de alto nivel para la elaboración posterior de las especificaciones técnicas del protocolo.

### 2.1 Autorización Sin Conexión (Authorization_Descriptor)

#### Motivación de Diseño

Authorization_Descriptor es el mecanismo de autorización central del protocolo CAP. Su motivación de diseño surge de un juicio fundamental: **cuando el terminal está sin conexión, no se debería privar completamente al Fay de su derecho de control, ya que el Fay puede haber sido previamente autorizado por su Human Prime**.

En escenarios reales, los dispositivos terminales frecuentemente se encuentran en estados donde la red no está disponible o es inestable — modo avión, espacios subterráneos, áreas remotas, fallos de red, etc. Si la verificación de autorización dependiera completamente de servicios en línea, una vez que la red se interrumpa, el Fay perdería inmediatamente todo el control sobre los recursos del terminal, incluso si el Human Prime (Natural_Person) o el puesto oficial (Official_Post) hubiera autorizado explícitamente con anterioridad. Por ejemplo, el iFay de un fotógrafo de vida silvestre está ayudando a operar un dron para tomas aéreas cuando la señal del teléfono se pierde al volar hacia un valle montañoso — sin un mecanismo de autorización sin conexión, el iFay perdería inmediatamente el control del dron, pudiendo causar su caída. Este diseño haría que la disponibilidad del Fay dependiera severamente del estado de la red, lo cual contradice la visión del ecosistema iFay de "los agentes inteligentes asumen el control del terminal de forma transparente".

Authorization_Descriptor, al persistir la relación de autorización en forma de archivo cifrado localmente en el terminal, permite que el terminal complete la verificación de autorización de forma independiente en estado sin conexión. Este archivo contiene el alcance de los recursos, los tipos de permisos y el período de validez para los que el Fay está autorizado. El Descriptor_Validator del lado del terminal puede verificar su legitimidad y validez sin necesidad de conexión a la red.

#### Visión General del Ciclo de Vida

El ciclo de vida de Authorization_Descriptor comprende las siguientes 5 fases:

1. **Emisión (Issuance)**: El Descriptor_Issuer genera y emite el Authorization_Descriptor tras obtener la autorización explícita del autorizante (Natural_Person u Official_Post). El proceso de emisión incluye la determinación del alcance de autorización, el establecimiento del período de validez, la generación de la firma digital, entre otros pasos. Una vez completada la emisión, el Authorization_Descriptor se transfiere a la entidad Fay correspondiente. Por ejemplo, el usuario Zhang San autoriza a su iFay a través de la interfaz de gestión de autorizaciones a usar la cámara y el micrófono del portátil durante los próximos 30 días, y el Descriptor_Issuer genera un Authorization_Descriptor con estos permisos y lo entrega al iFay.

2. **Almacenamiento Local (Local Storage)**: El Fay envía el Authorization_Descriptor obtenido al terminal de destino, y el terminal lo almacena en forma cifrada en un área de almacenamiento seguro local. El almacenamiento local garantiza que el terminal pueda acceder a la información de autorización en estado sin conexión, siendo la base física del mecanismo de autorización sin conexión. Por ejemplo, el iFay envía el archivo de autorización al portátil de Zhang San, que lo cifra y almacena en el chip de seguridad local.

3. **Verificación (Validation)**: Cuando el Fay solicita acceso a recursos del terminal, el Descriptor_Validator del lado del terminal verifica el Authorization_Descriptor almacenado localmente. El contenido de la verificación incluye: la legitimidad de la firma digital (verificada usando Verification_Key), si el período de validez ha expirado, si el alcance de autorización cubre el recurso y tipo de operación solicitados, y si ha sido revocado. Por ejemplo, cuando el iFay solicita activar la cámara, el terminal verifica si la firma del archivo de autorización es legítima, si está dentro del período de validez de 30 días y si incluye el permiso de uso de la cámara.

4. **Revocación (Revocation)**: Cuando el autorizante decide retirar la autorización, emite una declaración de revocación que invalida el Authorization_Descriptor correspondiente. La declaración de revocación se distribuye al terminal a través de múltiples canales (véase "Estrategia de distribución de declaraciones de revocación" más adelante). El terminal rechazará los Authorization_Descriptors revocados en las verificaciones posteriores. Por ejemplo, Zhang San descubre que el comportamiento del iFay no cumple con sus expectativas y revoca inmediatamente su autorización de uso de la cámara; el terminal rechazará la solicitud de acceso a la cámara del iFay en la siguiente verificación.

5. **Renovación (Renewal)**: Cuando un Authorization_Descriptor está a punto de expirar o el alcance de autorización necesita ajustarse, el Descriptor_Issuer emite un nuevo Authorization_Descriptor para reemplazar la versión anterior. El proceso de renovación es esencialmente una nueva emisión; la versión anterior se invalida automáticamente cuando la nueva versión entra en vigor. Por ejemplo, el período de validez de 30 días está a punto de expirar, Zhang San decide renovar y además autorizar al iFay a usar dispositivos de almacenamiento; el Descriptor_Issuer emite un nuevo archivo de autorización para reemplazar la versión anterior.

#### Modelo de Cadena de Confianza

La fiabilidad de la autorización sin conexión depende de una cadena de confianza completa. La ruta de transmisión de confianza desde el autorizante hasta el verificador del lado del terminal es la siguiente:

```
Autorizante (Natural_Person / Official_Post)
    │
    │ Delegación de autorización
    ▼
Descriptor_Issuer (Emisor de descripción de autorización)
    │
    │ Emite Authorization_Descriptor (con firma digital)
    ▼
Fay (iFay / coFay)
    │
    │ Presenta Authorization_Descriptor
    ▼
Descriptor_Validator del lado del terminal
    │
    │ Verifica firma usando Verification_Key
    ▼
Resultado de verificación (Aprobado / Rechazado)
```

Eslabones clave de la cadena de confianza:

- **Autorizante → Descriptor_Issuer**: El autorizante delega la autoridad de emisión al Descriptor_Issuer a través de un mecanismo seguro de delegación de autorización, asegurando que solo los emisores autorizados puedan generar Authorization_Descriptors legítimos
- **Descriptor_Issuer → Fay**: El Descriptor_Issuer firma digitalmente el Authorization_Descriptor con su clave privada, y el Fay obtiene el archivo de autorización firmado
- **Fay → Descriptor_Validator**: El Fay presenta el Authorization_Descriptor al terminal, y el Descriptor_Validator del lado del terminal verifica la legitimidad de la firma usando la Verification_Key distribuida por Registration_Authority
- **Registration_Authority → Terminal**: Registration_Authority, como infraestructura de confianza, es responsable de distribuir Verification_Keys a los terminales, asegurando que los terminales puedan verificar el origen de la firma del Authorization_Descriptor

#### Estrategia de Distribución de Declaraciones de Revocación

En escenarios sin conexión, la distribución oportuna de declaraciones de revocación es un desafío central. El terminal necesita obtener la información de revocación más reciente posible sin poder consultar en tiempo real el servicio de revocación en línea. El protocolo CAP adopta las siguientes estrategias de distribución:

- **Lista de Revocación Local (Local Revocation List)**: El terminal mantiene una lista de revocación local que registra los identificadores de Authorization_Descriptors revocados conocidos. Cada vez que el terminal se conecta a la red, sincroniza automáticamente la lista de revocación más reciente desde el servicio de revocación
- **Sincronización activa al conectarse**: Cuando el terminal pasa del estado sin conexión al estado conectado, prioriza la sincronización de la lista de revocación para asegurar que la información de revocación local esté lo más actualizada posible
- **Respaldo por período de validez**: El Authorization_Descriptor lleva incorporado un período de validez. Incluso si la declaración de revocación no llega a tiempo, un Authorization_Descriptor expirado se invalidará automáticamente, limitando la ventana máxima de impacto del retraso en la revocación
- **Efectivo en la siguiente verificación**: El terminal comprueba la lista de revocación local en cada verificación de Authorization_Descriptor. Una vez que la declaración de revocación llega al terminal, las solicitudes de verificación posteriores rechazarán inmediatamente la autorización revocada

### 2.2 Ticket En Línea (Trusted_Ticket)

#### Posicionamiento y Escenarios de Uso

Trusted_Ticket es el mecanismo complementario de autorización en línea del protocolo CAP en entornos conectados. Su posicionamiento central es: **proporcionar capacidades de emisión de autorización en tiempo real y consulta del estado de revocación en entornos conectados**, compensando las deficiencias de la autorización sin conexión en cuanto a oportunidad y velocidad de respuesta ante revocaciones.

Los escenarios de uso típicos de Trusted_Ticket incluyen:

- **Autorización temporal**: Cuando el Human Prime necesita otorgar temporalmente al Fay permisos de acceso a un recurso determinado, sin necesidad de pasar por el proceso completo de emisión de Authorization_Descriptor, emitiendo instantáneamente un Trusted_Ticket a través del servicio en línea. Por ejemplo, un usuario necesita temporalmente que su iFay le ayude a imprimir un documento; con una autorización de un solo toque en su teléfono, el servicio en línea emite instantáneamente un ticket temporal limitado al uso de la impresora, sin pasar por el proceso completo de emisión de autorización sin conexión
- **Verificación de revocación en tiempo real**: El terminal en estado conectado puede consultar en tiempo real el estado de revocación de la autorización a través del mecanismo de Trusted_Ticket, obteniendo información de revocación más oportuna que la lista de revocación local. Por ejemplo, un administrador de empresa acaba de revocar el control de un coFay sobre los equipos de la sala de conferencias; el terminal conectado puede conocer inmediatamente esta revocación a través del mecanismo de ticket en línea, en lugar de esperar a la siguiente sincronización de la lista de revocación local
- **Ajuste dinámico de permisos**: En entornos conectados, el alcance de autorización puede ajustarse dinámicamente a través de Trusted_Ticket sin necesidad de volver a emitir un Authorization_Descriptor. Por ejemplo, un iFay originalmente solo tenía permiso de lectura de archivos; el usuario eleva temporalmente su permiso a escritura desde su teléfono, tomando efecto inmediatamente a través de un ticket en línea

#### Relación con Authorization_Descriptor

La relación entre Trusted_Ticket y Authorization_Descriptor es de complementariedad, no de sustitución:

- **Authorization_Descriptor es el núcleo**: La autorización sin conexión es siempre el mecanismo base del protocolo CAP. Trusted_Ticket no puede existir de forma independiente fuera del sistema de Authorization_Descriptor
- **La emisión en línea puede convertirse a formato sin conexión**: Los Trusted_Tickets emitidos en línea pueden convertirse al formato local de Authorization_Descriptor para uso sin conexión. Esto significa que la autorización obtenida a través de Trusted_Ticket no se invalidará inmediatamente por una interrupción de la red
- **Relación de prioridad**: Cuando el terminal posee simultáneamente un Trusted_Ticket y un Authorization_Descriptor, se prioriza la información de autorización en tiempo real proporcionada por Trusted_Ticket, ya que su oportunidad es mayor

#### Estrategia de Degradación de En Línea a Sin Conexión

Cuando los servicios en línea no están disponibles, el protocolo CAP ejecuta una estrategia de degradación fluida:

1. **Detección del estado del servicio en línea**: El terminal monitoriza continuamente el estado de conexión con el servicio de autorización en línea
2. **Retroceso automático**: Cuando el servicio en línea no está disponible, el terminal retrocede automáticamente a la verificación local de Authorization_Descriptor, sin interrumpir el acceso del Fay a los recursos
3. **Conversión de Trusted_Ticket**: Los Trusted_Tickets emitidos durante el período conectado ya se han convertido y almacenado en formato local de Authorization_Descriptor, asegurando que sigan siendo utilizables tras la degradación
4. **Sincronización tras la recuperación**: Cuando el servicio en línea vuelve a estar disponible, el terminal reanuda automáticamente el uso del mecanismo de Trusted_Ticket y sincroniza la información de revocación que pueda haberse perdido durante el período sin conexión

El proceso de degradación es transparente para el Fay — el Fay no necesita percibir si el terminal está utilizando actualmente un ticket en línea o una autorización sin conexión; el resultado de la verificación de autorización se mantiene consistente en ambos modos.

### 2.3 Gestión de Sesiones (Session)

#### Ciclo de Vida

Session (sesión de control) es el ciclo de vida completo desde la aprobación de la verificación de autorización hasta el fin del acceso, y comprende las siguientes 3 fases:

1. **Creación (Creation)**: Cuando la verificación de autorización del Fay es aprobada, Protocol_Engine crea una instancia de Session para él. Al momento de la creación, la Session registra el identificador del Fay asociado, el Terminal_Resource de destino, la lista de permisos autorizados y la hora de creación. Tras la creación exitosa, el Fay obtiene el control sobre el recurso de destino. Por ejemplo, un iFay solicita usar el navegador del portátil; tras la aprobación de la verificación de autorización, el sistema crea una sesión de control vinculada al navegador, y el iFay puede operar el navegador a partir de ese momento.

2. **Activa (Active)**: Después de la creación, la Session entra en estado activo, y el Fay puede ejecutar operaciones sobre el Terminal_Resource vinculado dentro del alcance de los permisos autorizados. Durante el período activo, el mecanismo de Liveness_Detection monitoriza continuamente el estado de actividad de la sesión, asegurando que el Fay sigue utilizando efectivamente dicha sesión. Por ejemplo, el iFay está consultando información de vuelos a través del navegador para el usuario, enviando continuamente latidos durante este tiempo para indicar que sigue en uso activo.

3. **Terminación (Termination)**: La Session puede terminarse de las siguientes maneras:
   - **Liberación voluntaria**: El Fay libera voluntariamente la Session tras completar la operación, devolviendo el control sobre el Terminal_Resource. Por ejemplo, el iFay libera voluntariamente el control del navegador tras completar la consulta de vuelos
   - **Terminación por tiempo de espera**: Liveness_Detection detecta que la sesión del Fay ya no está activa (tiempo de espera del latido), terminando automáticamente la Session y recuperando los recursos. Por ejemplo, el iFay deja de enviar latidos debido a un fallo del entorno de ejecución, y el terminal recupera automáticamente el control del navegador tras el tiempo de espera
   - **Terminación por revocación**: Cuando la autorización es revocada, las Sessions activas asociadas se terminan forzosamente. Por ejemplo, el usuario descubre que el iFay está navegando contenido inapropiado y revoca inmediatamente la autorización; la sesión del navegador se termina forzosamente

Tras la terminación de la Session, el control del Fay sobre ese Terminal_Resource se libera inmediatamente, y el recurso vuelve a un estado en el que puede ser adquirido por otros controladores.

#### Relación de Vinculación con Terminal_Resource

Existe una relación de vinculación estricta entre Session y Terminal_Resource: **una Session activa corresponde al control de un Terminal_Resource**.

Las reglas centrales de esta relación de vinculación:

- **Vinculación uno a uno**: Cada Session activa se vincula a un Terminal_Resource específico, y el Fay ejecuta operaciones sobre ese recurso a través de la Session. Por ejemplo, un iFay obtiene una Session vinculada a la "cámara frontal" — solo puede operar la cámara frontal a través de esta Session y no puede usarla para operar el micrófono
- **Control exclusivo**: Para los modos de operación que requieren acceso exclusivo (write, execute, configure), un mismo Terminal_Resource puede tener como máximo una Session exclusiva activa en un momento dado. Por ejemplo, cuando iFay-A está escribiendo datos en una tarjeta SD en modo write, iFay-B no puede obtener simultáneamente una Session write para la tarjeta SD
- **Concurrencia de lectura múltiple**: Para el modo read, un mismo Terminal_Resource puede tener múltiples Sessions activas de solo lectura simultáneamente. Por ejemplo, iFay-A e iFay-B pueden leer simultáneamente datos de ubicación del módulo GPS en modo read
- **Vinculación del ciclo de vida**: Cuando un Terminal_Resource no está disponible (por ejemplo, hardware desconectado), todas las Sessions activas vinculadas a ese recurso serán terminadas. Por ejemplo, después de que un usuario desconecta una cámara USB, todas las Sessions vinculadas a esa cámara se terminan automáticamente

#### Mecanismo de Detección de Actividad

Liveness_Detection (detección de actividad) detecta si una sesión Fay sigue activa mediante la combinación de conexiones persistentes y latidos a nivel de aplicación. Su intención de diseño es recuperar oportunamente los recursos ocupados por sesiones inactivas.

Mecanismo de funcionamiento de la detección de actividad:

- **Mantenimiento de conexión persistente**: El Fay y el terminal mantienen un canal de comunicación a través de una conexión persistente, cuyo estado sirve como señal base para la detección de actividad
- **Latido a nivel de aplicación**: Sobre la conexión persistente, el Fay envía periódicamente mensajes de latido a nivel de aplicación, indicando que sigue utilizando efectivamente la Session actual. El intervalo de latido y el umbral de tiempo de espera son configurables
- **Determinación dual**: Solo cuando la conexión persistente se interrumpe y el latido a nivel de aplicación expira simultáneamente, se determina que la Session ha fallado. Este mecanismo de determinación dual evita falsos positivos causados por fluctuaciones breves de la red
- **Recuperación de recursos**: Cuando se determina que una Session ha fallado, Protocol_Engine termina automáticamente dicha Session y libera el Terminal_Resource que ocupaba, permitiendo que el recurso sea adquirido por otros controladores

### 2.4 Estrategia de Transferencia de Control (Handover_Policy)

#### Escenarios Centrales

La transferencia de control ocurre en escenarios donde múltiples controladores necesitan utilizar sucesivamente el mismo Terminal_Resource. El protocolo CAP define dos tipos de escenarios centrales de transferencia:

- **Transferencia de control entre Fays**: Un Fay transfiere el control que posee sobre un Terminal_Resource a otro Fay. Por ejemplo, un iFay responsable de la recopilación de datos, tras completar su tarea, transfiere el control de la cámara al iFay responsable de las videollamadas; o en una fábrica inteligente, un coFay responsable del control de calidad, tras completar la fotografía de productos, transfiere el control de la cámara industrial al coFay responsable del embalaje
- **Transferencia de control entre Fay y usuario humano**: El Fay devuelve el control al usuario humano, o el usuario humano delega el control al Fay. Por ejemplo, un dron está siendo pilotado autónomamente por un iFay cuando el piloto nota condiciones meteorológicas anormales y necesita tomar el control manual — el iFay cede inmediatamente el control de vuelo; una vez que el tiempo mejora, el piloto devuelve el control al iFay para continuar el vuelo autónomo. Otro ejemplo: un usuario está editando manualmente un documento y necesita que el iFay le ayude con el formato, transfiriendo temporalmente el control del editor de documentos al iFay; una vez completado el formato, el iFay devuelve el control

#### Capacidad de Configuración Estratégica

Handover_Policy proporciona capacidades de configuración estratégica, soportando los siguientes 3 tipos de estrategia:

1. **Script de reglas de prioridad (Priority Rule Script)**: Determina la prioridad de la transferencia de control mediante scripts de reglas predefinidos. Los scripts de reglas calculan una puntuación de prioridad basada en factores como el rol del Fay, la urgencia de la tarea y el tipo de recurso. El controlador con mayor prioridad obtiene el control del recurso. Esta estrategia es adecuada para escenarios donde las reglas de transferencia son claras y pueden definirse previamente. Por ejemplo, en un escenario médico, el iFay responsable de alertas de emergencia siempre tiene mayor prioridad que el iFay responsable de la recopilación rutinaria de datos; cuando ambos solicitan simultáneamente el módulo de comunicación, el iFay de alerta de emergencia obtiene automáticamente el control.

2. **Determinación en tiempo real por modelo de IA (AI Model Real-time Decision)**: Un modelo de IA determina la asignación del control en tiempo real según el contexto actual. El modelo de IA considera de forma integral el estado de las tareas de múltiples controladores, la urgencia del uso de recursos, los patrones históricos de transferencia y otros factores para tomar decisiones. Esta estrategia es adecuada para escenarios dinámicos donde las reglas de transferencia son complejas y difíciles de cubrir con reglas estáticas. Por ejemplo, en un hogar inteligente, un iFay responsable de la vigilancia de seguridad y un iFay responsable de videollamadas solicitan simultáneamente la cámara; el modelo de IA determina la prioridad en tiempo real basándose en información contextual como si existe una amenaza de seguridad actual y si la llamada es urgente.

3. **Decisión del usuario humano (Human User Decision)**: La autoridad de decisión sobre la transferencia de control se otorga al usuario humano. Cuando múltiples controladores compiten por el mismo recurso, el terminal presenta una interfaz de selección al usuario humano, quien decide a qué parte otorgar el control. Esta estrategia es adecuada para recursos de alta sensibilidad o escenarios que requieren juicio humano. Por ejemplo, dos iFays solicitan simultáneamente usar la aplicación bancaria del usuario; el terminal muestra un aviso para que el usuario decida a cuál autorizar, ya que las operaciones financieras requieren la aprobación final humana.

Los tres tipos de estrategia pueden configurarse independientemente a nivel de granularidad de Terminal_Resource; diferentes recursos en el mismo terminal pueden adoptar diferentes estrategias de transferencia.

#### Garantía de Atomicidad

El proceso de transferencia de control debe cumplir con el requisito de atomicidad: **en cualquier momento dado, un Terminal_Resource tiene como máximo un controlador activo**.

Principios de implementación de la garantía de atomicidad:

- **Primero liberar, luego adquirir**: Durante el proceso de transferencia, la Session del controlador original se termina primero, y luego se crea la Session del nuevo controlador, asegurando que no se produzca un estado en el que dos controladores posean simultáneamente el control del mismo recurso. Por ejemplo, al transferir el control de vuelo de un dron de iFay-A a iFay-B, la sesión de control de iFay-A debe terminarse primero antes de que se pueda crear la sesión de control de iFay-B — nunca habrá una situación en la que dos iFays controlen simultáneamente el vuelo del dron
- **Indivisible**: La operación de transferencia se ejecuta como un todo; o tiene éxito completamente (liberación del controlador original + adquisición del nuevo controlador), o falla completamente (retroceso al estado anterior a la transferencia)
- **Sin exposición de estados intermedios**: Un observador externo ve un estado de control de recursos consistente en cualquier momento dado, sin observar estados intermedios durante el proceso de transferencia

#### Principios de Manejo de Tiempo de Espera

El proceso de transferencia de control puede no completarse dentro del tiempo esperado por diversas razones (latencia de red, falta de respuesta del controlador, etc.). El principio del protocolo CAP para el manejo del tiempo de espera en la transferencia es: **las transferencias que no se completen dentro del tiempo de espera deben retroceder al estado anterior a la transferencia, manteniendo la sesión del controlador original sin cambios**.

Reglas específicas de manejo:

- **Umbral de tiempo de espera configurable**: Cada tipo de estrategia de transferencia puede configurar un umbral de tiempo de espera independiente, adaptándose a los requisitos de oportunidad de diferentes escenarios
- **Protección de retroceso**: Tras activarse el tiempo de espera, la operación de transferencia se retrocede automáticamente, y la Session del controlador original permanece en estado activo sin verse afectada. Por ejemplo, iFay-A está controlando la cámara e intenta transferir el control a iFay-B, pero iFay-B no logra responder dentro del tiempo especificado debido a la latencia de red; la transferencia se retrocede automáticamente e iFay-A continúa manteniendo el control de la cámara
- **Mecanismo de notificación**: Tras el retroceso por tiempo de espera, Protocol_Engine envía una notificación de fallo de transferencia a los controladores relevantes, explicando la razón del tiempo de espera
- **Estrategia de reintento**: Tras el fallo de la transferencia, se puede decidir según la configuración de la estrategia si se reintenta automáticamente o se espera intervención manual

### 2.5 Modo de Acceso a Recursos (Resource_Access_Mode)

#### Modelo de Clasificación

Resource_Access_Mode clasifica el acceso a recursos en 4 modos según el tipo de operación, formando una clasificación de permisos de menor a mayor:

1. **read (lectura)**: Acceso de solo lectura al Terminal_Resource, sin modificar el estado del recurso. Por ejemplo, leer datos en tiempo real de un sensor de temperatura, ver fotos en el álbum del usuario, obtener la información de ubicación actual del módulo GPS. El modo read es compartible, permitiendo que múltiples Fays accedan simultáneamente al mismo recurso en modo read — por ejemplo, múltiples iFays pueden leer simultáneamente datos del mismo sensor de temperatura sin interferirse mutuamente.

2. **write (escritura)**: Escritura de datos o modificación del estado del Terminal_Resource. Por ejemplo, guardar fotos capturadas en un dispositivo de almacenamiento, modificar el valor de temperatura de un dispositivo domótico inteligente, escribir contenido en un documento. El modo write es exclusivo; solo se permite que un controlador acceda al recurso en modo write en un momento dado — por ejemplo, dos iFays no pueden escribir datos simultáneamente en el mismo archivo, ya que esto causaría corrupción de datos.

3. **execute (ejecución)**: Ejecución de comandos de operación sobre el Terminal_Resource. Por ejemplo, iniciar una aplicación de navegación en el teléfono, activar el comando de despegue de un dron, ejecutar una tarea de impresión en una impresora. El modo execute es exclusivo, asegurando que la ejecución de comandos de operación no sea interferida por otros controladores — por ejemplo, cuando un iFay está controlando un dron para ejecutar un procedimiento de aterrizaje, otro iFay no puede enviar simultáneamente un comando de despegue.

4. **configure (configuración)**: Cambios de configuración a nivel de sistema sobre el Terminal_Resource. Por ejemplo, modificar los parámetros de resolución y frecuencia de cuadros de la cámara, ajustar las reglas del firewall de red, cambiar la configuración de emparejamiento del módulo Bluetooth. El modo configure es exclusivo y de alto privilegio, y generalmente requiere un nivel de autorización más alto para su ejecución — por ejemplo, modificar la política de seguridad de un router requiere un nivel de autorización más alto que simplemente leer el estado de la red.

#### Esencia del Bloqueo de Lectura-Escritura

El control de concurrencia de Resource_Access_Mode es esencialmente un modelo de bloqueo de lectura-escritura (Read-Write Lock):

- **Las operaciones read permiten acceso concurrente de múltiples Fays**: Múltiples Fays pueden mantener simultáneamente Sessions en modo read sobre el mismo Terminal_Resource sin interferirse mutuamente. Esto es análogo al bloqueo compartido (Shared Lock) en el modelo de bloqueo de lectura-escritura. Por ejemplo, tres iFays pueden leer simultáneamente datos de temperatura y humedad del mismo sensor meteorológico sin bloquearse mutuamente
- **Las operaciones write, execute y configure requieren control exclusivo**: Cuando cualquier Fay accede a un recurso en modo write, execute o configure, otros Fays no pueden acceder simultáneamente a ese recurso en ningún modo de escritura. Esto es similar al bloqueo exclusivo (Exclusive Lock) en el modelo de bloqueo de lectura-escritura. Por ejemplo, cuando un iFay está escribiendo un archivo de video en una tarjeta SD, otro iFay no puede escribir fotos simultáneamente en la misma tarjeta SD
- **Exclusión mutua lectura-escritura**: Cuando existe una Session activa de write, execute o configure sobre un recurso, las nuevas solicitudes de read también serán bloqueadas hasta que la Session exclusiva se libere. A la inversa, cuando existen Sessions activas de read sobre un recurso, las solicitudes de write, execute y configure esperarán hasta que todas las Sessions de read finalicen. Por ejemplo, cuando un iFay está modificando un archivo de configuración (modo write), otro iFay que quiera leer ese archivo también debe esperar a que la escritura se complete, para evitar leer un estado intermedio inconsistente

Este modelo de bloqueo de lectura-escritura maximiza el rendimiento de concurrencia de las operaciones read mientras garantiza la consistencia de datos y la seguridad de las operaciones.

#### Relación con los Permisos de Session

Resource_Access_Mode está estrechamente vinculado con los permisos de autorización de Session: **la lista de permisos de autorización de la Session determina los tipos de operación que el Fay puede ejecutar**.

La relación específica es la siguiente:

- **Restricción por lista de permisos**: Cada Session lleva una lista de permisos de autorización al momento de su creación, derivada del alcance de permisos definido en el Authorization_Descriptor o Trusted_Ticket. El Fay solo puede acceder al recurso en los modos incluidos en la lista de permisos
- **Principio de mínimo privilegio**: La lista de permisos de la Session debe seguir el principio de mínimo privilegio, incluyendo solo los modos de permiso mínimos necesarios para que el Fay complete su tarea actual
- **Los permisos no pueden elevarse**: Una vez creada la Session, su lista de permisos no puede elevarse durante el ciclo de vida de la sesión. Si el Fay necesita un modo de acceso de mayor privilegio, debe crear una nueva Session a través de una nueva verificación de autorización
- **Inclusión jerárquica de permisos**: Los modos de mayor privilegio no incluyen automáticamente las capacidades de los modos de menor privilegio. Por ejemplo, poseer el permiso configure no significa poseer automáticamente el permiso read; cada modo de acceso requiere autorización independiente
