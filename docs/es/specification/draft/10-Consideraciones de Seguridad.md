# Capítulo 10: Consideraciones de Seguridad

Este capítulo describe el modelo de amenazas, riesgos conocidos y mitigaciones del protocolo CAP. Este capítulo es contenido no normativo (no usa palabras clave como MUST/SHOULD/MAY), pero los riesgos y mitigaciones descritos en este capítulo DEBERÍAN ser tomados en serio por los implementadores.

## 10.1 Modelo de Amenazas

El modelo de amenazas del protocolo CAP se basa en las siguientes suposiciones y consideraciones.

### 10.1.1 Suposiciones de Confianza

El protocolo CAP asume que las siguientes entidades son confiables:

1. **Registration_Authority**: Ancla de confianza; su capacidad de distribución de claves es la base de la seguridad del protocolo
2. **Claves de ancla de confianza pre-instaladas**: Claves públicas de RA pre-instaladas en la fabricación o despliegue del terminal
3. **Almacenamiento seguro del terminal**: Módulos de seguridad de hardware/software usados para almacenar Verification_Keys y claves de firma locales

El protocolo CAP **no** asume que las siguientes entidades son confiables:

1. **Instancias de Fay**: Fay puede ser programado maliciosamente o controlado por un atacante
2. **iFay_Runtime**: Puede contener bugs o ser reemplazado
3. **Canales de red**: Pueden ser interceptados, manipulados o bloqueados
4. **Procesos de aplicación no autorizados**: Otros procesos en el terminal pueden intentar evadir el control de CAP

### 10.1.2 Capacidades del Atacante

Atacantes considerados con las siguientes capacidades:

| Tipo de Atacante | Capacidad |
|-----------|------|
| Atacante de red | Interceptar, manipular, bloquear, repetir mensajes de red |
| Fay malicioso | Posee credenciales legítimas pero exhibe comportamiento anormal |
| Ladrón de credenciales | Obtiene una copia de credencial de otra persona a través de otros canales |
| Amenaza interna | Descriptor_Issuer mismo comprometido |
| Atacante físico | Contacto físico con dispositivo terminal |

El protocolo CAP proporciona protección contra los primeros 4 tipos de atacantes; la defensa contra atacantes físicos depende de los mecanismos de seguridad de hardware del terminal (no abordada a nivel de protocolo).

## 10.2 Riesgos Conocidos y Mitigaciones

### 10.2.1 Riesgo de Filtración de Credenciales

**Riesgo**: Después de que un archivo Authorization_Descriptor sea robado por un atacante, puede ser enviado al terminal por una entidad distinta del Fay que posee la credencial.

**Mitigaciones**:

- El `subject_fay_id` de Authorization_Descriptor está enlazado a un Fay específico; el terminal verifica la coincidencia de sujeto en el Paso 4 de §3.3.2
- El atacante debe controlar simultáneamente el runtime del Fay objetivo (es decir, atacar al Fay mismo); la filtración de credenciales por sí sola no es suficiente para constituir un ataque
- El almacenamiento de credenciales DEBE estar cifrado (§3.2.3) para prevenir extracción sin conexión
- Trusted_Ticket limita la ventana de daño a través de períodos de validez cortos (máx. 7 días) y consultas de revocación en tiempo real

**Riesgo residual**: Si el atacante controla completamente el proceso Fay (incluyendo sus claves), la credencial todavía puede ser abusada. Este protocolo no puede resolver el problema de un "Fay completamente comprometido" — este es un problema de protección de integridad de Fay de nivel superior.

### 10.2.2 Riesgo de Retraso de Revocación

**Riesgo**: Hay una ventana de retraso entre la revocación de credencial y que la declaración de revocación llegue al terminal, durante la cual la credencial revocada todavía puede pasar la validación.

**Mitigaciones**:

- Cuando el terminal está continuamente en línea, las declaraciones de revocación se sincronizan dentro de 1 hora por predeterminado (§3.4.5)
- Al usar Trusted_Ticket, el terminal PUEDE eliminar el retraso a través de consultas de revocación en tiempo real (§4.3.2 Paso 5)
- El techo de validez de credencial (90 días sin conexión / 7 días en línea) limita el retraso máximo
- Los emisores DEBERÍAN usar períodos de validez más cortos (por ejemplo, 1 día) en escenarios de alta sensibilidad

**Riesgo residual**: Los terminales sin conexión a largo plazo pueden aceptar credenciales revocadas durante días o más. Este es el compromiso inherente del mecanismo de autorización sin conexión — eligiendo disponibilidad sobre puntualidad de revocación.

### 10.2.3 Riesgo de Ataque de Repetición

**Riesgo**: Un atacante captura mensajes legítimos del protocolo y los repite al terminal.

**Mitigaciones**:

- AuthRequest contiene `message_id` (UUID v7, relacionado con tiempo); el terminal PUEDE mantener un caché de IDs vistos a corto plazo para rechazar repeticiones
- El `sequence_number` de Heartbeat es monotónicamente creciente; las repeticiones son detectadas (§5.4.2)
- Los canales TLS proporcionan protección básica contra repetición (cada sesión TLS es independiente)
- El `revocation_id` de la declaración de revocación es único; las repeticiones son inválidas

**Riesgo residual**: Dentro de una sesión TLS, las repeticiones de mensajes son posibles si un atacante ya ha roto el cifrado TLS. Esta es una amenaza a nivel TLS, no dentro del alcance del protocolo CAP.

### 10.2.4 Riesgo de Ataque de Reloj

**Riesgo**: Un atacante evade verificaciones de período de validez modificando el reloj del terminal o expone vulnerabilidades relacionadas con el reloj.

**Mitigaciones**:

- La fuente del reloj del terminal debería ser confiable (reloj del sistema, sincronización NTP)
- §3.6 define detección de deriva del reloj: la validación es rechazada al detectar saltos significativos del reloj
- Los terminales sin conexión por demasiado tiempo DEBERÍAN solicitar al usuario re-sincronizar el reloj

**Riesgo residual**: Un atacante que controle físicamente el terminal puede ser capaz de manipular el reloj. La seguridad física es responsabilidad del hardware del terminal.

### 10.2.5 Riesgo de Compromiso de Descriptor_Issuer

**Riesgo**: Descriptor_Issuer es completamente controlado por un atacante, quien puede emitir credenciales arbitrarias.

**Mitigaciones**:

- §8.6.2 define un mecanismo de revocación de claves de emergencia
- Registration_Authority puede invalidar todas las credenciales emitidas por Descriptor_Issuer revocando su Verification_Key
- El terminal termina inmediatamente las Sesiones relacionadas al recibir la revocación de Verification_Key (§5.5)
- La seguridad de la raíz de confianza (Registration_Authority) es crítica — su compromiso colapsará toda la cadena de confianza

**Riesgo residual**: Las credenciales emitidas antes de la revocación de Verification_Key todavía pueden ser abusadas. La ventana de revocación es inevitable.

### 10.2.6 Riesgo de Ataque de Agotamiento de Recursos

**Riesgo**: Un Fay malicioso agota recursos del terminal (conexiones, Sesiones, almacenamiento) a través de solicitudes masivas.

**Mitigaciones**:

- §5.8 define límites de cantidad de Session
- §3.2.3 define el techo de almacenamiento de credenciales
- El timeout de latido recupera Sesiones inválidas y libera recursos
- El terminal DEBERÍA implementar limitación de tasa basada en fay_id (además de las restricciones de frecuencia §7.4.2)

**Riesgo residual**: Múltiples credenciales emitidas legítimamente pueden conspirar para agotar recursos. La política del terminal debería limitar la cuota total de un solo Fay.

### 10.2.7 Riesgo de Condición de Carrera de Transferencia

**Riesgo**: Un atacante explota estados intermedios en el flujo de transferencia (por ejemplo, el estado "ninguna Session activa" en §6.5.1 [T2]) para apoderarse del recurso.

**Mitigaciones**:

- Durante la transferencia, el recurso está en estado de pre-ocupación (`handover_pending`) y no acepta otras solicitudes de Session (§6.5.3)
- Incluso si ocurre fallo entre [T2] y [T3], el recurso entra claramente en estado "inactivo", con solicitudes subsiguientes compitiendo justamente (§6.5.2)
- El timeout de transferencia previene que el recurso permanezca en estado pendiente por mucho tiempo

**Riesgo residual**: En el caso extremo de fallo [T3]–[T4], la sesión del Fay original es no recuperable — AuthRequest debe ser re-iniciado. Este es un costo necesario, evitando el análisis de seguridad más complejo traído por "recuperación automática".

### 10.2.8 Riesgo de Ataque de Degradación

**Riesgo**: Un atacante fuerza al terminal al modo de degradación (§4.5) bloqueando la conexión entre el terminal y Ticket_Issuer, evadiendo así verificaciones de revocación en tiempo real.

**Mitigaciones**:

- El terminal rechaza la aceptación de nuevos Trusted_Tickets por predeterminado en modo de degradación
- El período de validez de tickets ya convertidos a Authorization_Descriptors locales es corto (máx. 7 días)
- El terminal DEBERÍA indicar a iFay_Runtime que actualmente está en modo de degradación, permitiendo que las operaciones sensibles sean activamente rechazadas

**Riesgo residual**: Las credenciales convertidas localmente durante el período de degradación todavía pueden ser usadas. Este es el compromiso inherente del diseño orientado a sin conexión.

### 10.2.9 Riesgo de Ataque de Canal Lateral

**Riesgo**: Inferir información sensible (por ejemplo, existencia de credenciales válidas) observando el tiempo de respuesta del protocolo CAP o detalles del código de error.

**Mitigaciones**:

- Los códigos de error no exponen detalles internos (§9.9)
- Las implementaciones DEBERÍAN mantener tiempos de respuesta similares a través de diferentes ramas de fallo de validación
- No filtrar huellas de claves, IDs internos, etc., en respuestas de error

**Riesgo residual**: Eliminar completamente los ataques de canal lateral es extremadamente difícil. Esta capa de protocolo proporciona medidas básicas; la defensa profunda depende de la calidad de implementación.

## 10.3 Recomendaciones de Seguridad de Despliegue

Los desplegadores que implementan el protocolo CAP deberían considerar las siguientes prácticas de seguridad:

### 10.3.1 Gestión de Claves

- Usar módulos de seguridad de hardware (HSM, TPM, Secure Enclave) para almacenamiento de claves privadas
- Período de rotación de claves no más de 90 días (claves locales del terminal) / 365 días (claves de Issuer)
- Establecer procedimientos de respuesta de emergencia para filtración de claves
- Operaciones críticas de firma requieren aprobación de múltiples personas o PIN de hardware

### 10.3.2 Monitoreo y Auditoría

- Registrar todos los eventos de fallo de validación de autorización con alertas
- Monitorear frecuencias anormales de solicitudes de revocación (puede indicar compromiso de Descriptor_Issuer)
- Los logs de auditoría usan almacenamiento persistente independiente con anti-manipulación
- Auditar exhaustivamente todas las operaciones en recursos de alta sensibilidad (modo configure, restricción `require_human_confirmation`)

### 10.3.3 Endurecimiento del Terminal

- Habilitar control de acceso obligatorio del SO (SELinux, AppArmor)
- Usar sandboxes para aislar iFay_Runtime de otros procesos
- Deshabilitar servicios del sistema innecesarios para reducir la superficie de ataque
- Actualizar parches del sistema regularmente

### 10.3.4 Segmentación de Red

- Aislar redes de despliegue de Registration_Authority de redes públicas
- Desplegar Descriptor_Issuer en entornos controlados
- Restringir comunicación de terminal a Issuer a puertos/protocolos necesarios
- Implementar mTLS (autenticación mutua) para proteger comunicaciones críticas

## 10.4 Limitaciones del Protocolo

Esta especificación explícitamente no aborda los siguientes problemas de seguridad:

1. **Integridad de Fay**: El protocolo CAP asume que la integridad del código y comportamiento de la propia instancia de Fay está garantizada por otros mecanismos (por ejemplo, firma de código, medición de integridad en tiempo de ejecución)
2. **Seguridad física**: Cuando el terminal es completamente controlado por un atacante físico, la capa de software no puede proporcionar protección
3. **Formato de log de auditoría**: v1 no estandariza el formato de log de auditoría (ítem de exclusión §3.2 del plan arquitectónico)
4. **Federación de identidad entre terminales**: v1 no soporta gestión de identidad unificada entre terminales; cada terminal confía independientemente en Registration_Authority

## 10.5 Evolución de Seguridad en Versiones Subsiguientes

Mejoras de seguridad planeadas para introducción en versiones futuras (v2+):

1. Consenso de revocación distribuido (ítem de exclusión §3.2 del plan arquitectónico)
2. Formato estandarizado de log de auditoría (ítem de exclusión §3.2 del plan arquitectónico)
3. Algoritmos criptográficos resistentes a cuántica (rastreando estándares NIST PQC)
4. Integración más profunda de principios de Zero Trust
5. Prueba formal de seguridad del protocolo
