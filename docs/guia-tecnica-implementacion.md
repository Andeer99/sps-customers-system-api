# Guía técnica de implementación

Esta guía explica el porqué de cada archivo y decisión del proyecto. Sirve como complemento a los comentarios incrustados en el código y como referencia rápida para revisores que entran al repo sin contexto previo.

## Arquitectura

La aplicación es una **System API** del patrón API-led: recibe llamadas HTTP, enruta el request con APIkit, consulta la base MySQL y transforma el resultado a JSON con DataWeave antes de devolverlo al consumidor.

```text
Cliente / Postman
  -> HTTP Listener: /api/v1/sps/*
  -> APIkit Router (RAML: api/sps-customers-api.raml)
  -> Flow get:\customers:SPS_Customers_API_Config
  -> DB Select: SELECT * FROM customers;
  -> DataWeave (renombrado de columnas a camelCase)
  -> JSON response (Customer[])
```

En CloudHub la misma cadena pasa además por **Mule Gateway**, que aplica las políticas de API Manager (Client-ID Enforcement en este caso) antes del listener.

## Mapa de archivos

| Archivo | Función | Por qué importa |
| --- | --- | --- |
| `pom.xml` | Build del proyecto. | Fija versiones de runtime y conectores, declara MySQL como shared library para que el driver llegue a CloudHub. |
| `mule-artifact.json` | Descriptor del JAR Mule. | Restringe el runtime a 4.9.0+ y Java 17, y marca `mule.key` y `anypoint.platform.client_secret` como propiedades cifradas en logs y UI. |
| `src/main/mule/global.xml` | Configs reutilizables. | Centraliza Secure Properties, HTTP Listener, DB Config, APIkit Config y Autodiscovery. |
| `src/main/mule/implementation.xml` | Lógica de los flows. | Endpoint público y mapping DB → JSON con error-handler normalizado. |
| `src/main/resources/api/sps-customers-api.raml` | Contrato del API. | Fuente de verdad consumida por APIkit, Exchange y API Manager. |
| `src/main/resources/api/examples/customers-response.json` | Ejemplo del response 200. | Mantiene un único payload usado por RAML, Exchange "Try it" y Postman. |
| `src/main/resources/local.yaml` | Properties para `mule.env=local`. | Mantiene a Autodiscovery desactivado (`api.id=0`) para correr sin Mule Gateway. |
| `src/main/resources/dev.yaml` | Properties para `mule.env=dev`. | Trae el API Instance ID real (20913965) para registrar Autodiscovery en CloudHub. |
| `src/main/resources/log4j2.xml` | Configuración de logging. | Async Root en INFO con `correlationId` en el pattern para trazabilidad punta a punta. |
| `postman/SPS Customers System API.postman_collection.json` | Pruebas manuales. | Cubre local, CloudHub público y CloudHub con Client-ID (401/200). |

## Decisiones clave

### 1. Dos archivos de propiedades, un solo XML

Se cargan con `${mule.env}.yaml`. El mismo `global.xml` funciona en local y en CloudHub porque toda la diferencia entre ambientes vive en los YAML, no en el XML. Ningún flow contiene condicionales por ambiente.

### 2. Secure Properties con Blowfish/CBC

Algoritmo soportado por la **Secure Properties Tool** oficial de MuleSoft. Permite cifrar y rotar valores desde la CLI sin escribir utilidades propias. La llave (`mule.key`) se inyecta como propiedad de runtime y nunca vive en el repositorio.

Todos los valores del YAML — cifrados o no — quedan accesibles bajo el prefijo `secure::`. Por eso `api.id` se referencia con `${secure::api.id}` aunque sea un número plano.

### 3. APIkit + RAML como contrato

APIkit genera los flows por convención (`get:\customers:SPS_Customers_API_Config`) y valida cada request contra el RAML antes de invocar la lógica. Eso permite que el desarrollo del flow no replique reglas que ya están en el contrato (método, content-type, schema).

### 4. DataWeave tolerante a casing

El driver JDBC puede devolver `FirstName`, `firstName` o `FIRSTNAME` según configuración del schema y del driver. El encadenamiento `default` garantiza que el response siempre tenga el campo, en camelCase, sin importar la variante recibida:

```dataweave
firstName: customer.FirstName default customer.firstName default customer.FIRSTNAME default null
```

### 5. Error handler unificado

Tipos APIkit (`BAD_REQUEST`, `NOT_FOUND`, `METHOD_NOT_ALLOWED`) y el catch-all (`ANY`) devuelven el mismo schema JSON con `error`, `code`, `timestamp` y `correlationId`. Cualquier consumidor parsea errores con un único contrato.

### 6. Autodiscovery externalizado

`apiId` vive en `dev.yaml` (`api.id`) para que el mismo binario suba a cualquier environment cambiando solo properties. En local se deja `api.id=0` y la app sirve tráfico sin pasar por Mule Gateway.

### 7. Mule Gateway en CloudHub 2.0 requiere Connected App

A diferencia de CloudHub 1.0, las credenciales `anypoint.platform.client_id` y `anypoint.platform.client_secret` no se inyectan automáticamente. Se crean en **Access Management → Connected Apps** con scopes `Exchange Viewer`, `View Environment` y `API Manager Environment Administrator`, y se pasan a Runtime Manager como propiedades protegidas. Sin estas credenciales la Autodiscovery falla y la instancia queda `Unregistered`.

### 8. Driver MySQL como shared library

El conector Database de MuleSoft no incluye driver. Declarar `mysql-connector-j` solo como `dependency` hace que Studio lo resuelva del `.m2` local, pero el JAR final no lo lleve. El `sharedLibraries` del Mule Maven Plugin lo empuja al classpath de la app y elimina el `ClassNotFoundException` típico al desplegar.

### 9. Logger con correlationId

`log4j2.xml` incluye `%X{correlationId}` en el pattern y los flows emiten loggers manuales al inicio y fin de la consulta. Esto permite seguir un request específico desde la entrada hasta la respuesta, incluyendo cualquier error intermedio, en Anypoint Monitoring o Runtime Manager → Logs.

## Notas sobre archivos sin comentarios inline

### `mule-artifact.json`

JSON estricto, no admite comentarios. Sus tres claves significan:

- `minMuleVersion: "4.9.0"` — primera LTS compatible con Java 17 y con todos los conectores usados.
- `javaSpecificationVersions: ["17"]` — bloquea arranque en JDK 8 para evitar sorpresas de módulos sellados.
- `secureProperties` — lista de claves que el runtime enmascara en logs y en la UI de Runtime Manager. Incluye `mule.key` (la llave de cifrado del YAML) y `anypoint.platform.client_secret` (secret del Connected App).

### `customers-response.json`

JSON puro. Contiene un ejemplo del response 200 reutilizado por RAML, Exchange "Try it" y la documentación. Se actualizó con datos reales devueltos por la base de entrenamiento para que el example en Exchange refleje exactamente lo que el endpoint produce.

## Ejecución local

Requisitos:

- JDK 17
- Anypoint Studio 7.22+ o compatible con Mule 4.9 / Java 17
- Postman o cliente HTTP equivalente

Pasos:

1. Importar el proyecto en Studio.
2. Verificar que Maven resuelva todas las dependencias.
3. Ejecutar con argumentos de runtime:
   ```text
   -M-Dmule.env=local -M-Dmule.key=<llave>
   ```
4. Probar:
   ```http
   GET http://localhost:8081/api/v1/sps/customers
   ```

## Despliegue en CloudHub 2.0

1. `mvn clean package -DskipTests`
2. Subir el JAR a Runtime Manager (sandbox, Mule 4.x, Java 17, 0.1 vCore).
3. En **Properties**, declarar:
   - `mule.env=dev`
   - `mule.key=<llave>` (Protect)
   - `anypoint.platform.client_id=<connected-app-client-id>` (Protect)
   - `anypoint.platform.client_secret=<connected-app-client-secret>` (Protect)
   - `anypoint.platform.config.analytics.agent.enabled=true`
4. Confirmar en API Manager que la instancia 20913965 cambia a estado **Active**.
5. (Opcional) aplicar política Client-ID Enforcement y crear contrato desde Exchange para validar 401/200.
