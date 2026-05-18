## Capítulo 5: Relación entre Schema y Entregables

Este capítulo describe la estructura del sistema de entregables del proyecto CAP, establece la posición central de los archivos de definición del Schema del protocolo dentro de los entregables, y la relación de correspondencia entre la versión del Schema y la versión del protocolo. Comprender la relación jerárquica y las dependencias entre los entregables es la base para que los implementadores del protocolo y los contribuyentes de la comunidad participen en la colaboración del proyecto.

### 5.1 Tres Categorías de Entregables Centrales

El sistema de entregables del proyecto CAP se compone de tres categorías de entregables centrales, formando una cadena de entrega completa desde la documentación hasta la definición y la implementación:

| Categoría de Entregable | Descripción | Ejemplo de Ruta en el Repositorio |
|------------------------|-------------|----------------------------------|
| **Documentación multilingüe del protocolo** | Incluye el plan arquitectónico (Blueprint) y la especificación técnica del protocolo (Specification), publicados de forma equivalente en 9 idiomas, proporcionando descripciones legibles por humanos de la intención de diseño, el alcance de capacidades y los detalles técnicos del protocolo | `docs/{lang}/blueprint/`, `docs/{lang}/specification/` |
| **Archivos de definición del Schema del protocolo** | La definición autoritativa del formato de mensajes del protocolo, proporcionada en múltiples formatos legibles por máquinas y humanos, siendo la referencia para la implementación de SDKs y las pruebas de interoperabilidad | `schema/{version}/` |
| **Implementaciones de SDK en varios lenguajes** | Las implementaciones concretas del protocolo en lenguajes de programación específicos, desarrolladas basándose en los archivos de definición del Schema, proporcionando capacidades de integración del protocolo listas para usar para desarrolladores de diferentes stacks tecnológicos | `sdk/{language}/` |

Relación jerárquica entre las tres categorías de entregables:

- **La documentación multilingüe del protocolo** se sitúa en la capa estratégica y normativa, definiendo el diseño de alto nivel de "qué hace" y "cómo lo hace" el protocolo
- **Los archivos de definición del Schema del protocolo** se sitúan en la capa de definición, transformando los formatos de mensaje de la especificación del protocolo en definiciones formales precisas y procesables por máquinas
- **Las implementaciones de SDK en varios lenguajes** se sitúan en la capa de implementación, transformando las capacidades del protocolo basándose en los archivos de definición del Schema en interfaces de programación directamente invocables

Las actualizaciones de las tres categorías de entregables siguen una ruta de propagación de arriba hacia abajo: los cambios en la documentación del protocolo impulsan las actualizaciones de la definición del Schema, y las actualizaciones de la definición del Schema impulsan la adaptación de las implementaciones de SDK.

### 5.2 Función de los Archivos de Definición del Schema

Los archivos de definición del Schema son la **definición autoritativa (Authoritative Definition)** del formato de mensajes del protocolo CAP, y desempeñan las siguientes funciones centrales en el sistema de entregables del proyecto:

- **Fuente única de verdad (Single Source of Truth) del formato de mensajes**: Los archivos de definición del Schema describen con precisión los nombres de campo, tipos de datos, restricciones de obligatorio/opcional y estructuras anidadas de todos los tipos de mensaje del protocolo CAP. Cuando existe ambigüedad entre la descripción textual de la documentación del protocolo y la definición del Schema, prevalece la definición del Schema
- **Referencia de desarrollo para la implementación de SDKs**: La lógica de serialización/deserialización de mensajes de los SDKs en cada lenguaje debe seguir estrictamente la definición del Schema. Los desarrolladores de SDKs conocen la estructura precisa de cada tipo de mensaje a través de los archivos de definición del Schema, asegurando que las implementaciones de SDKs en diferentes lenguajes sean completamente consistentes en el formato de mensajes
- **Referencia de verificación para pruebas de interoperabilidad**: Las pruebas de interoperabilidad entre SDKs de diferentes lenguajes utilizan la definición del Schema como estándar de verificación. Un mensaje serializado por un SDK debe poder ser correctamente deserializado por otro SDK; los archivos de definición del Schema son la única base para determinar la "corrección"
- **Base para la verificación automatizada**: Los archivos de definición del Schema (especialmente schema.json) pueden ser consumidos directamente por herramientas automatizadas para la verificación automática del formato de mensajes, la generación de código y la generación de documentación, reduciendo los errores humanos

### 5.3 Relación de Correspondencia entre la Versión del Schema y la Versión del Protocolo

Cada versión oficialmente publicada del protocolo CAP corresponde a una versión del Schema, y los archivos del Schema se almacenan en directorios nombrados con la fecha de publicación de la versión del protocolo, asegurando que la relación de correspondencia de versiones sea clara y rastreable.

**Reglas de nomenclatura de directorios:**

| Estado del Protocolo | Directorio del Schema | Descripción |
|---------------------|----------------------|-------------|
| Borrador (Draft) | `schema/draft/` | Definición del Schema en desarrollo, puede cambiar en cualquier momento, sin compromiso de compatibilidad |
| Publicación oficial | `schema/{YYYY-MM-DD}/` | Nombrado con la fecha de publicación, como `schema/2025-10-25/`, inmutable tras la publicación |

**Reglas de correspondencia de versiones:**

- **Correspondencia uno a uno**: Cada versión del protocolo oficialmente publicada corresponde exactamente a un directorio de versión del Schema, cuyo nombre de directorio utiliza la fecha de publicación de esa versión del protocolo
- **Borrador compartido**: Todos los cambios del Schema en fase de borrador comparten el directorio `schema/draft/`; la fase de borrador no distingue números de versión
- **Inmutabilidad**: Los archivos en los directorios del Schema oficialmente publicados no se modifican tras la publicación. Cualquier cambio en el Schema (incluyendo correcciones de errores) se realiza publicando una nueva versión
- **Conservación histórica**: Los directorios del Schema de versiones anteriores se conservan permanentemente en el repositorio para referencia histórica y verificación de compatibilidad retroactiva

**Relación de correspondencia de versiones actual:**

| Versión del Protocolo | Directorio del Schema | Estado |
|----------------------|----------------------|--------|
| draft | `schema/draft/` | En desarrollo |
| 2025-10-25 | `schema/2025-10-25/` | Publicado |

### 5.4 Descripción de los Formatos de Archivos del Schema

Cada directorio de versión del Schema contiene 3 formatos de archivos de definición del Schema, cada uno orientado a diferentes escenarios de uso y consumidores:

| Formato | Nombre de Archivo | Uso | Consumidores Principales |
|---------|-------------------|-----|-------------------------|
| **JSON Schema** | `schema.json` | Definición de formato de mensajes legible por máquinas, utilizada para verificación automatizada, generación de código y verificación de interoperabilidad entre lenguajes | Herramientas automatizadas, pipelines CI/CD, generadores de código de SDK |
| **TypeScript** | `schema.ts` | Definición de tipos TypeScript, proporcionando soporte de tipos nativos para el ecosistema TypeScript/JavaScript, utilizada para el desarrollo de SDKs TS/JS y la verificación de tipos en el IDE | Desarrolladores de SDKs TypeScript/JavaScript, desarrolladores de integración frontend |
| **MDX** | `schema.mdx` | Formato de documentación renderizable, presentando la definición del Schema de forma amigable para humanos, utilizada para la visualización en sitios de documentación y el aprendizaje del protocolo | Estudiantes del protocolo, sitios de documentación, revisores técnicos |

**Relación entre los tres formatos:**

- **schema.json es el formato autoritativo**: Cuando existen diferencias entre los tres formatos, prevalece schema.json. schema.ts y schema.mdx deben mantener consistencia semántica con schema.json
- **schema.ts es el formato de conveniencia para desarrollo**: schema.ts transforma las definiciones de tipos de schema.json en tipos nativos de TypeScript, permitiendo que los desarrolladores TS/JS obtengan directamente sugerencias de tipos y verificación en tiempo de compilación en el IDE
- **schema.mdx es el formato de presentación**: schema.mdx transforma las definiciones estructuradas de schema.json en contenido de documentación renderizable, soportando la navegación interactiva de la definición del Schema en sitios de documentación

**Ejemplo de estructura de directorios de archivos:**

```
schema/
├── draft/
│   ├── schema.json
│   ├── schema.ts
│   └── schema.mdx
└── 2025-10-25/
    ├── schema.json
    ├── schema.ts
    └── schema.mdx
```
