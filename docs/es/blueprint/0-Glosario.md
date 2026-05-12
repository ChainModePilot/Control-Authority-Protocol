# Plan Arquitectónico de CAP

El Control Authority Protocol (CAP) define cómo los dispositivos terminales verifican que un Fay (iFay o coFay) ha sido autorizado por su anfitrión humano para acceder legítimamente a los recursos del terminal. El protocolo se basa en un archivo de descripción de autorización almacenado localmente sin conexión (Authorization_Descriptor) como mecanismo central, complementado por un ticket de confianza en línea (Trusted_Ticket), y cubre la verificación de autorización, la gestión de sesiones, la transferencia de control, los modos de acceso a recursos y la detección de actividad. Proporciona un marco estandarizado de permisos de control para que los agentes inteligentes del ecosistema iFay asuman de forma segura el software y hardware de los dispositivos terminales.

## Glosario

- **CAP (Control Authority Protocol)**: Protocolo de Autoridad de Control, que define cómo los dispositivos terminales verifican si un Fay ha sido autorizado para acceder legítimamente a los recursos del terminal
- **iFay**: Entidad Fay independiente, un agente inteligente asociado a una persona natural (Natural_Person)
- **coFay**: Entidad Fay colaborativa, un agente inteligente colaborativo asociado a un puesto oficial (Official_Post)
- **Natural_Person**: Persona natural, el individuo humano al que está asociado un iFay, y la fuente última de autorización
- **Official_Post**: Puesto oficial, la posición organizacional a la que está asociado un coFay, y la fuente última de autorización
- **iFay_Runtime**: Entorno de ejecución de iFay, responsable de la gestión del ciclo de vida de las instancias Fay, la planificación y la iniciación de solicitudes de control
- **Authorization_Descriptor**: Archivo de descripción de autorización, un archivo cifrado almacenado localmente en el terminal que describe el alcance de los recursos, los tipos de permisos y el período de validez para los que un Fay está autorizado — el mecanismo central de la autorización sin conexión
- **Trusted_Ticket**: Ticket de confianza, una credencial en línea emitida por un Ticket_Issuer en escenarios conectados, que sirve como mecanismo complementario a la autorización sin conexión
- **Terminal_Resource**: Recurso del terminal, dispositivos de hardware o software cliente en el terminal que son accesibles y operables
- **Descriptor_Issuer**: Emisor de la descripción de autorización, una entidad de confianza responsable de generar y emitir Authorization_Descriptors tras ser autorizada por una Natural_Person o un Official_Post
- **Descriptor_Validator**: Validador de descriptores, un componente del lado del terminal responsable de verificar la legitimidad y validez de los Authorization_Descriptors
- **Registration_Authority**: Autoridad de registro, una entidad de confianza responsable de gestionar el registro de hardware, software y sistemas operativos de terminales, y de distribuir Verification_Keys
- **Verification_Key**: Clave de verificación, una clave obtenida por el terminal mediante el registro, utilizada para verificar la firma digital de los Authorization_Descriptors
- **Protocol_Engine**: Motor de protocolo, el componente del sistema que ejecuta la lógica central del Control Authority Protocol
- **Session**: Sesión de control, el ciclo de vida completo desde la aprobación de la verificación de autorización hasta el fin del acceso
- **Handover_Policy**: Estrategia de transferencia de control, que define las reglas de decisión para la transferencia de control entre múltiples Fays o entre Fays y humanos
- **Resource_Access_Mode**: Modo de acceso a recursos, un mecanismo de gestión escalonada del acceso a recursos por tipo de operación (modelo de bloqueo de lectura-escritura)
- **Liveness_Detection**: Detección de actividad, la detección de si una sesión Fay sigue activa mediante la combinación de conexiones persistentes y latidos a nivel de aplicación
- **Capability_Matrix**: Matriz de capacidades, la descripción estructurada de las capacidades centrales del protocolo CAP en el plan arquitectónico
- **Audit_Logger**: Registrador de auditoría, el componente responsable de registrar todas las operaciones de verificación de autorización y acceso a recursos
