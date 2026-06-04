# Capítulo 8: Criptografía y Firmas

Este capítulo define los algoritmos de firma, formatos de claves, gestión del ciclo de vida de claves y requisitos de distribución utilizados por el protocolo CAP. Los requisitos criptográficos en este capítulo son normativos — cualquier implementación que se ajuste al protocolo CAP DEBE implementar según las definiciones en este capítulo.

## 8.1 Conjunto de Algoritmos

CAP v1 define dos conjuntos de algoritmos de firma obligatorios y formatos de claves:

| ID de Algoritmo | Algoritmo | Longitud de Clave Pública | Longitud de Firma | Estado |
|---------|------|---------|---------|------|
| `ed25519` | Ed25519 (RFC 8032) | 32 bytes | 64 bytes | Obligatorio |
| `ecdsa-p256-sha256` | ECDSA P-256 + SHA-256 (RFC 6979) | 65 bytes (sin comprimir) / 33 bytes (comprimido) | 64 bytes (codificado en DER o raw) | Obligatorio |

Todas las implementaciones de CAP DEBEN soportar simultáneamente ambos conjuntos de algoritmos. Cuando Trusted_Ticket usa JWS, los nombres de algoritmo correspondientes son:

| ID de Algoritmo CAP | Campo `alg` JWS |
|------------|---------------|
| `ed25519` | `EdDSA` |
| `ecdsa-p256-sha256` | `ES256` |

### 8.1.1 Recomendaciones de Selección de Algoritmo

- **Preferir `ed25519`**: Mejor rendimiento, firmas más cortas, gestión de claves más simple
- **`ecdsa-p256-sha256`**: Para uso bajo requisitos regulatorios como FIPS 140-3

Los emisores DEBERÍAN usar `ed25519` por predeterminado en todas las credenciales recién emitidas, excepto cuando haya requisitos explícitos de cumplimiento.

### 8.1.2 Algoritmos No Permitidos

Las implementaciones de CAP v1 NO DEBEN usar los siguientes algoritmos para emitir o aceptar nuevas credenciales:

- Cualquier variante de firma RSA (rendimiento pobre, claves más grandes)
- ECDSA secp256k1 (curva no estándar FIPS)
- ECDSA P-384 / P-521 (no obligatorio en v1, puede agregarse en versiones futuras)
- Cualquier algoritmo de firma derivado de SHA-1

## 8.2 Formatos de Claves

### 8.2.1 Ed25519

Las claves públicas se codifican según RFC 8032 §5.1.5:

- Punto de curva Edwards comprimido little-endian de 32 bytes

En DescriptorPayload, `signature.signature_value` es una firma raw de 64 bytes (sin codificación ASN.1).

En JWS, las firmas se codifican según RFC 8037, con la firma raw de 64 bytes codificada en base64url.

### 8.2.2 ECDSA P-256

Las claves públicas se codifican según RFC 5480:

- Formato sin comprimir: 65 bytes (0x04 || X || Y)
- Formato comprimido: 33 bytes (0x02/0x03 || X)

Las implementaciones DEBEN aceptar simultáneamente claves públicas tanto sin comprimir como comprimidas.

Formato de firma:

- En DescriptorSignature: firma raw de 64 bytes (R || S, cada uno 32 bytes big-endian), DER no usado
- En JWS: codificada según RFC 7518 §3.4 (64 bytes R || S)

### 8.2.3 Representación Binaria de Claves

El campo `VerificationKey.key_material` almacena directamente la secuencia de bytes del formato anterior:

- ed25519 → 32 bytes
- ecdsa-p256-sha256 (sin comprimir) → 65 bytes
- ecdsa-p256-sha256 (comprimido) → 33 bytes

## 8.3 Construcción de Entrada de Firma

### 8.3.1 Entrada de Firma de Authorization_Descriptor

Construir la entrada de firma siguiendo estos pasos:

1. Tomar `AuthorizationDescriptor.payload` (estructura DescriptorPayload)
2. Serializar en una secuencia de bytes usando RFC 8949 CBOR **Codificación Determinista**
3. La secuencia de bytes se convierte en la entrada al algoritmo de firma

Restricciones clave de la Codificación Determinista CBOR (de RFC 8949 §4.2):

- Las claves de map se ordenan en orden lexicográfico
- Los valores numéricos usan la codificación más corta
- Las cadenas y cadenas de bytes usan la codificación de longitud más corta
- No se usa codificación de longitud indefinida

Las implementaciones DEBEN adherirse estrictamente a estas restricciones para asegurar consistencia de verificación de firma entre implementaciones.

### 8.3.2 Entrada de Firma de Trusted_Ticket

Construir la entrada de firma JWS según RFC 7515 §5.1:

```
Signing Input = base64url(UTF8(Header)) + "." + base64url(UTF8(Payload))
```

Donde:

- Header y Payload se serializan en JSON UTF-8 según RFC 7159
- Orden de campos JSON: Las implementaciones DEBERÍAN ordenar las claves en orden lexicográfico, pero el requisito estricto de orden de campo es determinado por los bytes codificados en base64url

### 8.3.3 Entrada de Firma de RevocationStatement

La construcción de entrada de firma para declaraciones de revocación es la misma que para Authorization_Descriptor:

1. Eliminar el campo `signature`, reteniendo el resto de los campos de RevocationStatement
2. Serialización Codificación Determinista CBOR
3. Usar la secuencia de bytes como la entrada de firma

## 8.4 Distribución de Claves

### 8.4.1 Modos de Distribución

El terminal DEBE soportar los siguientes dos modos de distribución de Verification_Key:

#### Pre-instalación Sin Conexión

El Verification_Key inicial se pre-instala en la fábrica o despliegue del terminal. Las claves pre-instaladas:

- Marcadas `source = "pre-installed"` en la estructura `VerificationKey`
- DEBEN escribirse a través de un canal seguro físico o controlado durante la fabricación o despliegue
- Típicamente corresponden a Descriptor_Issuers de nivel raíz (por ejemplo, Issuers de autenticación del fabricante del dispositivo)

#### Distribución En Línea

Registration_Authority distribuye nuevos Verification_Keys a través de interfaces en línea:

- Marcados `source = "ra-distributed"` en la estructura `VerificationKey`
- DEBEN ser transmitidos sobre canales TLS/mTLS (ver §1.3.4)
- El terminal DEBE verificar que el transmisor sea una Registration_Authority de confianza

### 8.4.2 Configuración de Ancla de Confianza

La clave pública de Registration_Authority del terminal (utilizada para verificar firmas de mensajes push de RA) sirve como ancla de confianza:

- DEBE ser pre-instalada durante la fabricación o despliegue inicial del terminal
- Solo puede ser reemplazada a través de canales físicos o controlados
- No actualizada a través de mecanismos de tiempo de ejecución del protocolo CAP

### 8.4.3 Mensajes de Distribución

Mensajes de distribución de claves empujados por Registration_Authority:

```
VerificationKeyDistribution {
  required version       : uint32 (= 1)
  required key           : VerificationKey
  required ra_signature  : DescriptorSignature       // Firmado por Registration_Authority
}
```

Flujo de procesamiento del terminal:

1. Verificar que `ra_signature` fue emitida por una Registration_Authority de confianza
2. Verificar que `key.algorithm` esté en el conjunto de algoritmos permitido por §8.1
3. Verificar que `key.valid_from` sea razonable relativo al tiempo actual del terminal (por ejemplo, no excede tiempo actual + 24 horas)
4. Escribir al almacén de claves del terminal
5. Devolver una confirmación a Registration_Authority (el mecanismo de confirmación queda fuera del alcance de esta especificación)

## 8.5 Almacenamiento de Claves

El terminal DEBE almacenar de forma segura todos los Verification_Keys y claves de firma locales (ver §4.4.3).

### 8.5.1 Requisitos de Almacenamiento

El terminal DEBE:

1. Almacenar la porción de clave privada de todas las claves cifradas (las claves públicas pueden almacenarse en texto plano)
2. El acceso a clave privada está protegido por permisos de proceso del SO; solo Protocol_Engine puede leer
3. DEBERÍA almacenar claves privadas en elementos seguros de hardware (por ejemplo, TPM, Secure Enclave, TEE)

El terminal NO DEBE:

1. Escribir claves privadas en texto plano a medios persistentes
2. Transmitir claves privadas vía mensajes del protocolo CAP
3. Exponer contenido de claves privadas en logs de depuración o salida de error

### 8.5.2 Manejo de Filtración de Claves

Si el terminal detecta que una clave puede haber sido filtrada (por ejemplo, almacenamiento seguro atacado):

1. Detener inmediatamente el uso de esa clave
2. Reportar a Registration_Authority a través del flujo §8.6
3. Antes de que la nueva clave sea completamente distribuida, rechazar todas las validaciones de credenciales relacionadas (política conservadora)

## 8.6 Actualizaciones y Rotación de Claves

### 8.6.1 Transición Suave

Al rotar Verification_Keys, Registration_Authority DEBE proporcionar una transición suave:

1. Distribuir la nueva clave a todos los terminales
2. Durante el período de transición (predeterminado 30 días), tanto la clave nueva como la antigua son válidas
3. Después de que termine el período de transición, el `valid_until` de la clave antigua se alcanza, invalidándola automáticamente

Durante el período de transición, el terminal DEBE mantener simultáneamente la clave nueva y antigua, seleccionando la clave correspondiente para validación según `key_id`.

### 8.6.2 Revocación de Emergencia

```
VerificationKeyRevocation {
  required version          : uint32 (= 1)
  required key_id           : string
  required revocation_time  : timestamp
  required reason           : enum["compromised", "superseded", "ra_decision"]
  required ra_signature     : DescriptorSignature
}
```

Después de recibir el mensaje de revocación, el terminal DEBE:

1. Verificar que la firma sea de una Registration_Authority de confianza
2. Marcar inmediatamente la key_id como revocada
3. Rechazar todas las credenciales que usan esta key_id en solicitudes de validación subsiguientes (devolver `E_VERIFICATION_KEY_INVALID`)
4. Verificar proactivamente todas las Sesiones activas: terminar forzosamente las Sesiones que usan credenciales emitidas con esta key_id (ver §5.5)

### 8.6.3 Rotación de Claves de Firma Locales del Terminal

Las claves de firma locales mantenidas por el terminal para soportar el flujo de conversión §4.4:

- DEBERÍAN ser rotadas automáticamente cada 90 días
- Durante la rotación, el terminal genera un nuevo par de claves; la clave antigua continúa siendo usada para la validación de credenciales emitidas hasta que todas expiren
- El terminal PUEDE poseer simultáneamente hasta 4 claves de firma locales (cubriendo el período máximo de validez + transición suave)

## 8.7 Resumen de Requisitos Criptográficos

| Requisito | Alcance |
|------|------|
| El algoritmo de firma DEBE ser elegido de ed25519 / ecdsa-p256-sha256 | Todos los escenarios de firma |
| Las claves privadas DEBEN ser almacenadas de forma segura | Todos los emisores y terminales |
| La distribución de claves públicas DEBE ser vía canales cifrados | Registration_Authority → terminal |
| Las anclas de confianza DEBEN ser pre-instaladas vía canales físicos o controlados | Todos los terminales |
| Codificación Determinista CBOR para entrada de firma de credenciales sin conexión | Authorization_Descriptor, RevocationStatement |
| JWS Compact Serialization para tickets en línea | Trusted_Ticket |
