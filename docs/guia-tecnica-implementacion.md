# Guía técnica de implementación

Esta guía documenta el código agregado para que puedas explicarlo y mantenerlo sin depender de memoria.

## Arquitectura

La aplicación es una System API sencilla del patrón API-led. Recibe llamadas HTTP en `/api/v1/sps/customers`, enruta el request con APIkit, consulta la base MySQL de entrenamiento y transforma el resultado a JSON con DataWeave.

Flujo general:

```text
Cliente/Postman
  -> HTTP Listener
  -> APIkit Router
  -> DB Select: SELECT * FROM customers;
  -> DataWeave
  -> JSON response
```

## Archivos principales

- `src/main/resources/api/sps-customers-api.raml`: contrato RAML 1.0 que define `GET /customers`, el tipo `Customer` y el error estándar.
- `src/main/mule/global.xml`: configuración reusable: Secure Properties, HTTP Listener, conexión MySQL, APIkit y Autodiscovery.
- `src/main/mule/implementation.xml`: lógica de ejecución: flow principal, consulta a base de datos, transformación DataWeave y manejo de errores.
- `src/main/resources/local.yaml` y `src/main/resources/dev.yaml`: propiedades por ambiente. El usuario y contraseña de la base están cifrados.
- `postman/SPS Customers System API.postman_collection.json`: colección para probar local, CloudHub y CloudHub con Client-ID.
- `mule-artifact.json`: descriptor Mule con propiedades sensibles para ocultar en CloudHub.

## Seguridad de propiedades

La app carga el archivo `${mule.env}.yaml` y lo descifra con `${mule.key}`:

```xml
<secure-properties:config file="${mule.env}.yaml" key="${mule.key}">
    <secure-properties:encrypt algorithm="Blowfish" mode="CBC"/>
</secure-properties:config>
```

Los valores se consumen con el prefijo `secure::`, por ejemplo:

```xml
user="${secure::db.user}"
password="${secure::db.password}"
```

La llave no debe subirse al repositorio. Para esta copia local está en `LOCAL_SECRETS_DO_NOT_COMMIT.md`, que ya está ignorado por `.gitignore`.

## Endpoint implementado

El HTTP Listener escucha el prefijo:

```text
/api/v1/sps/*
```

APIkit usa el RAML para resolver el recurso:

```text
GET /customers
```

Por eso el endpoint completo queda:

```http
GET http://localhost:8081/api/v1/sps/customers
```

## Transformación DataWeave

La consulta devuelve filas de MySQL. El transform mapea los nombres esperados del training DB a una respuesta más limpia para consumidores:

```json
{
  "firstName": "...",
  "lastName": "...",
  "email": "...",
  "company": "..."
}
```

El mapper acepta variantes de mayúsculas/minúsculas para evitar fallos si el driver devuelve columnas con otro casing.

## Manejo de errores

El flow principal devuelve errores JSON para:

- Solicitud inválida: `400`
- Recurso no encontrado: `404`
- Método no permitido: `405`
- Error inesperado o de base de datos: `500`

Cada error incluye `code`, `timestamp` y `correlationId` para facilitar trazabilidad.

## Autodiscovery y API Manager

`global.xml` incluye Autodiscovery apuntando a `api.id`:

```xml
<api-gateway:autodiscovery apiId="${secure::api.id}" flowRef="sps-customers-main"/>
```

Durante desarrollo local, `api.id` queda en `0` hasta que crees la instancia real en API Manager. Después de crear el API desde Exchange, copia el API Instance ID en `dev.yaml` y redespliega en CloudHub.

## Uso con IntelliJ

IntelliJ te sirve para editar y revisar:

- `pom.xml`
- RAML
- XML de Mule
- Markdown
- JSON de Postman

Para ejecutar el runtime Mule, desplegar a CloudHub o gestionar API Manager, usa Anypoint Studio/Platform.

