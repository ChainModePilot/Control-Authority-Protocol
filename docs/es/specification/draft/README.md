# Especificación Técnica del Protocolo CAP (Borrador)

Este directorio contiene la versión de borrador de la especificación técnica del Control Authority Protocol (CAP) v1. La especificación se desarrolla con base en el plan arquitectónico en `docs/es/blueprint/`, cubriendo las 6 capacidades centrales listadas en §3.1 del Capítulo 3 del plan arquitectónico.

## Estructura del Documento

| Capítulo | Archivo | Contenido |
|------|------|------|
| Capítulo 0 | `00-Introducción y Conformidad.md` | Estado del documento, alcance, palabras clave RFC 2119, niveles de conformidad, referencias normativas |
| Capítulo 1 | `01-Arquitectura y Roles.md` | Roles del protocolo, cadena de confianza, contratos de interfaz externa |
| Capítulo 2 | `02-Modelo de Datos.md` | Estructuras de datos centrales (Authorization_Descriptor, Trusted_Ticket, Session, Verification_Key) |
| Capítulo 3 | `03-Protocolo de Autorización Sin Conexión.md` | Flujo completo del protocolo de ciclo de vida de Authorization_Descriptor |
| Capítulo 4 | `04-Protocolo de Ticket En Línea.md` | Flujo completo de Trusted_Ticket y degradación |
| Capítulo 5 | `05-Gestión de Sesiones y Detección de Actividad.md` | Máquina de estados de Session, reglas de enlace, latidos, doble determinación, recuperación por timeout |
| Capítulo 6 | `06-Protocolo de Transferencia de Control.md` | Tres tipos de políticas Handover_Policy, garantías de atomicidad, rollback por timeout |
| Capítulo 7 | `07-Modo de Acceso a Recursos.md` | Semántica de read/write/execute/configure, matriz de bloqueo lectura-escritura |
| Capítulo 8 | `08-Criptografía y Firmas.md` | Conjunto de algoritmos, formatos de claves, distribución y rotación |
| Capítulo 9 | `09-Códigos de Error y Niveles de Conformidad.md` | Tabla de códigos de error estándar, declaración de conformidad |
| Capítulo 10 | `10-Consideraciones de Seguridad.md` | Modelo de amenazas, riesgos conocidos y mitigaciones |

## Orden de Lectura Recomendado

1. Primera lectura: Capítulo 0 → Capítulo 1 → Capítulo 2 → Capítulo 3
2. Implementar terminal: Capítulos 0–3 → Capítulos 5, 7 → Capítulos 8, 9
3. Implementar emisor: Capítulos 0–2 → Capítulos 3, 4 → Capítulo 8
4. Implementar iFay_Runtime: Capítulos 0, 1 → Capítulo 5 → Capítulo 9
5. Revisión de seguridad: Capítulo 10 + lectura cruzada de capítulos relacionados

## Estado del Borrador

Este borrador está en fase de discusión. Antes del lanzamiento formal:

- Los nombres de campos, códigos de error y umbrales de restricciones pueden ajustarse
- La estructura de capítulos puede reorganizarse
- No se garantiza compatibilidad hacia atrás

Después de que se estabilicen las discusiones, los contenidos de este directorio se publicarán como `docs/es/specification/2025-10-25/`, la primera versión formal del protocolo CAP.

## Activos Relacionados

- Plan arquitectónico: `docs/es/blueprint/`
- Definiciones de Schema (borrador): `schema/draft/`
- Otras versiones de idioma: Esta especificación actualmente tiene versiones zh-CN, zh-TW, ja, ko, en y de; los idiomas restantes se traducirán antes del lanzamiento formal
