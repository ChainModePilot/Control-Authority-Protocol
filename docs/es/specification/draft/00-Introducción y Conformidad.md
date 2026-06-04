# Capítulo 0: Introducción y Conformidad

## 0.1 Estado del Documento

Este documento es la versión de borrador de la especificación técnica normativa del Control Authority Protocol (CAP). La versión de borrador no garantiza compatibilidad hacia atrás y puede sufrir cambios arbitrarios antes del lanzamiento formal. Una vez que este borrador se estabilice y supere la verificación completa de pruebas, será publicado como la primera versión formal en `specification/2025-10-25/`.

Este documento está destinado a usarse junto con el plan arquitectónico de CAP (`docs/es/blueprint/`):

- El **plan arquitectónico** responde "qué, por qué y para qué" — define la intención de diseño del protocolo, los límites de capacidad y los mecanismos centrales
- La **especificación** responde "cómo y cómo verificar" — define los formatos de mensaje del protocolo, los pasos de flujo, el manejo de errores y los requisitos de conformidad

Cuando las descripciones no normativas en el plan arquitectónico entren en conflicto con disposiciones normativas en esta especificación, esta especificación prevalecerá.

## 0.2 Alcance

Esta especificación define los detalles técnicos del protocolo CAP v1, cubriendo las 6 capacidades centrales listadas en la Sección 3.1 del Capítulo 3 del plan arquitectónico:

1. Emisión, almacenamiento, validación, revocación y renovación de la autorización sin conexión (Authorization_Descriptor)
2. Emisión, consulta y conversión a autorización sin conexión de tickets en línea (Trusted_Ticket)
3. Ciclo de vida completo de la gestión de sesiones (Session)
4. Tres tipos de políticas y garantías de atomicidad de la política de transferencia de control (Handover_Policy)
5. Mecanismo de doble determinación de detección de actividad (Liveness_Detection)
6. Modelo de bloqueo lectura-escritura del modo de acceso a recursos (Resource_Access_Mode)

Esta especificación excluye explícitamente las características listadas en la Sección 3.2 del Capítulo 3 del plan arquitectónico, incluyendo migración de sesión entre terminales, autorización colaborativa multi-terminal, cadenas de delegación de autorización, ABAC, formato estandarizado de registro de auditoría, especificación de transmisión cifrada de mensajes de protocolo, consenso de revocación distribuido y elevación dinámica de permisos dentro de una Session.

## 0.3 Terminología de Conformidad

Esta especificación sigue las convenciones de palabras clave de RFC 2119 y RFC 8174. Las siguientes palabras clave, cuando aparezcan en **mayúsculas**, tienen significado normativo:

- **MUST / debe**: Requisito absoluto. Las implementaciones que no satisfagan este requisito no se ajustan a esta especificación
- **MUST NOT / no debe**: Prohibición absoluta. Las implementaciones que violen esta prohibición no se ajustan a esta especificación
- **SHOULD / debería**: Recomendación fuerte. Puede desviarse por razones válidas dado el pleno entendimiento de las consecuencias
- **SHOULD NOT / no debería**: Disuasión fuerte. Puede desviarse por razones válidas dado el pleno entendimiento de las consecuencias
- **MAY / puede**: Opcional. Las implementaciones pueden decidir por sí mismas si proveerlo

El mismo vocabulario que no aparezca en mayúsculas lleva solo su significado literal y no tiene fuerza normativa.

## 0.4 Alineación de Terminología

La terminología utilizada en esta especificación es completamente consistente con el plan arquitectónico `00-Glosario.md`. Cuando esta especificación referencia un término, utiliza el identificador definido en el plan arquitectónico (por ejemplo, `Authorization_Descriptor`, `Fay`, `Terminal_Resource`).

Para facilidad de referencia, esta especificación marca términos clave en **negrita** en su primera aparición en cada capítulo, con definiciones breves del glosario del plan arquitectónico. Las definiciones completas de los términos se rigen por el plan arquitectónico.

## 0.5 Niveles de Conformidad

Esta especificación define 3 niveles de conformidad de implementación. Una implementación DEBE satisfacer al menos el "Nivel de Conformidad de Terminal" para reclamar conformidad con CAP v1.

### 0.5.1 Conformidad de Terminal (Terminal Conformance)

Aplica a dispositivos terminales que implementan `Descriptor_Validator`, `Protocol_Engine` y la lógica de gestión de sesiones. Las implementaciones de terminal DEBEN:

1. Implementar completamente el flujo de validación de Authorization_Descriptor definido en el Capítulo 3
2. Implementar completamente el ciclo de vida de Session y el mecanismo Liveness_Detection definidos en el Capítulo 5
3. Implementar completamente la semántica de bloqueo lectura-escritura de Resource_Access_Mode definida en el Capítulo 7
4. Rechazar todas las solicitudes que no pasen la validación y devolver códigos de error estandarizados según el Capítulo 9
5. Mantener una lista de revocación local y sincronizarla según la política definida en el Capítulo 3 cuando esté en línea

### 0.5.2 Conformidad de Emisor (Issuer Conformance)

Aplica a entidades de confianza que implementan `Descriptor_Issuer` o `Ticket_Issuer`. Las implementaciones de emisor DEBEN:

1. Generar Authorization_Descriptor y Trusted_Ticket legítimos según las restricciones de campos definidas en el Capítulo 2
2. Aplicar firmas digitales a credenciales según los requisitos criptográficos definidos en el Capítulo 8
3. Mantener registros de estado de credenciales emitidas y soportar operaciones de revocación
4. Implementar la conversión de Trusted_Ticket a Authorization_Descriptor definida en el Capítulo 4

### 0.5.3 Conformidad de Runtime (Runtime Conformance)

Aplica a entornos de ejecución de Fay que implementan `iFay_Runtime`. Las implementaciones de runtime DEBEN:

1. Enviar solicitudes de validación de autorización a Protocol_Engine según el contrato de interfaz definido en el Capítulo 1
2. Mantener conexiones persistentes y enviar latidos de capa de aplicación a la frecuencia definida en el Capítulo 5
3. Recibir y reenviar notificaciones de cambio de estado de sesión empujadas por Protocol_Engine
4. Notificar proactivamente a Protocol_Engine para liberar las Sesiones relacionadas cuando termine una instancia de Fay

Una implementación puede satisfacer múltiples niveles de conformidad simultáneamente. Por ejemplo, un terminal integrado puede servir como implementación de terminal y de runtime al mismo tiempo.

## 0.6 Referencias Normativas

Esta especificación referencia normativamente los siguientes documentos:

- RFC 2119 / RFC 8174: Convenciones de palabras clave usadas en esta especificación
- RFC 8949 (CBOR): Serialización binaria compacta para Authorization_Descriptor (ver Capítulo 2)
- RFC 8032 (EdDSA): Algoritmo de firma digital predeterminado (ver Capítulo 8)
- RFC 7515 (JWS): Serialización y firma JSON para Trusted_Ticket (ver Capítulo 4)
- RFC 5280 (X.509): Formato de certificado opcional (ver Capítulo 8)

El archivo de definición de Esquema CAP (`schema/{version}/schema.json`) sirve como complemento formal al Capítulo 2 de esta especificación, con fuerza normativa equivalente a esta especificación. Cuando schema.json entre en conflicto con la descripción textual en esta especificación, schema.json prevalecerá.

## 0.7 Estructura del Documento

Esta especificación está organizada en el siguiente orden:

| Capítulo | Tema | Contenido Clave |
|------|------|---------|
| Capítulo 1 | Arquitectura y Roles | Roles del protocolo, cadena de confianza, contratos de interfaz externa |
| Capítulo 2 | Modelo de Datos | Definiciones a nivel de campo de las estructuras de datos centrales |
| Capítulo 3 | Protocolo de Autorización Sin Conexión | Flujo completo de Authorization_Descriptor |
| Capítulo 4 | Protocolo de Ticket En Línea | Flujo completo de Trusted_Ticket y degradación |
| Capítulo 5 | Gestión de Sesiones y Detección de Actividad | Máquina de estados de Session y latido |
| Capítulo 6 | Protocolo de Transferencia de Control | Tres tipos de políticas Handover_Policy |
| Capítulo 7 | Modo de Acceso a Recursos | Semántica de read/write/execute/configure |
| Capítulo 8 | Criptografía y Firmas | Conjunto de algoritmos, formatos de claves, distribución |
| Capítulo 9 | Códigos de Error y Niveles de Conformidad | Códigos de error estándar, declaración de nivel |
| Capítulo 10 | Consideraciones de Seguridad | Modelo de amenazas, riesgos conocidos |

Se recomienda a los lectores leer los Capítulos 0–2 en secuencia para establecer una base, luego saltar a los capítulos relevantes según el enfoque de implementación.
