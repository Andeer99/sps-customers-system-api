# SPS Customers System API

Este repositorio implementa una **System API en MuleSoft** para exponer información de clientes de SPS desde la base de datos MySQL de entrenamiento.

```http
GET /api/v1/sps/customers
```

La API consulta la tabla `customers`, transforma la respuesta con DataWeave y entrega un JSON pensado para consumidores que necesitan datos básicos de clientes para campañas personalizadas.

## Alcance

La práctica cubre los puntos obligatorios del ejercicio MuleSoft Trainee:

- Exponer clientes en `/api/v1/sps/customers`.
- Consultar MySQL en `mudb.learn.mulesoft.com:3306`, base `training`.
- Separar configuración global y lógica de implementación.
- Usar dos archivos de propiedades: `local.yaml` y `dev.yaml`.
- Proteger usuario y contraseña con Mule Secure Properties.
- Construir una especificación RAML 1.0.
- Preparar despliegue en CloudHub.
- Dejar lista la integración con API Manager mediante Autodiscovery.
- Documentar el flujo de Client-ID Enforcement como extra de seguridad.

## Arquitectura

```text
Cliente / Postman
  -> HTTP Listener: /api/v1/sps/*
  -> APIkit Router
  -> RAML: api/sps-customers-api.raml
  -> DB Select: SELECT * FROM customers;
  -> DataWeave
  -> JSON response
```

La API sigue el enfoque API-led de MuleSoft como una **System API**: encapsula el acceso a la base de datos y entrega un contrato HTTP estable para futuros consumidores, Process APIs o Experience APIs.

## Estructura Del Proyecto

```text
.
├── docs/
│   ├── cloudhub-api-manager-checklist.md
│   ├── guia-tecnica-implementacion.md
│   ├── plan-rapido-sps-customers-system-api.md
│   └── screenshots/                # evidencia visual reproducible
├── postman/
│   └── SPS Customers System API.postman_collection.json
├── src/main/mule/
│   ├── global.xml
│   └── implementation.xml
├── src/main/resources/
│   ├── api/
│   │   ├── examples/customers-response.json
│   │   └── sps-customers-api.raml
│   ├── dev.yaml
│   ├── local.yaml
│   └── log4j2.xml
├── mule-artifact.json
└── pom.xml
```

## Componentes Principales

| Archivo | Propósito |
| --- | --- |
| `pom.xml` | Define el proyecto Mule Maven, runtime Mule 4.9.0 y dependencias HTTP, APIkit, Database, Secure Properties y MySQL. |
| `src/main/resources/api/sps-customers-api.raml` | Contrato RAML 1.0 del endpoint `GET /customers`. |
| `src/main/mule/global.xml` | Configuración global: Secure Properties, HTTP Listener, Database Config, APIkit y Autodiscovery. |
| `src/main/mule/implementation.xml` | Flow principal, enrutamiento APIkit, consulta SQL, transformación DataWeave y manejo de errores. |
| `src/main/resources/local.yaml` | Propiedades para ejecución local. |
| `src/main/resources/dev.yaml` | Propiedades para despliegue en ambiente dev/CloudHub. |
| `postman/` | Colección para validar local, CloudHub y pruebas con Client-ID. |

## Contrato De Respuesta

La respuesta pública evita exponer la fila cruda de base de datos. El flow mapea columnas del training DB a un modelo simple:

```json
[
  {
    "firstName": "John",
    "lastName": "Doe",
    "email": "john.doe@example.com",
    "company": "Acme"
  }
]
```

Campos esperados desde la base:

- `FirstName`
- `LastName`
- `Email`
- `Company`

El DataWeave también tolera variantes de mayúsculas/minúsculas para reducir fricción con el driver JDBC.

## Seguridad

Las credenciales de base de datos se consumen mediante Mule Secure Properties:

```xml
${secure::db.user}
${secure::db.password}
```

La llave de cifrado se inyecta en runtime con `mule.key` y no debe guardarse en el repositorio. El archivo local `LOCAL_SECRETS_DO_NOT_COMMIT.md` está ignorado por `.gitignore`.

Ejemplo para cifrar un valor con Secure Properties Tool:

```powershell
java -cp secure-properties-tool-j17.jar com.mulesoft.tools.SecurePropertiesTool string encrypt Blowfish CBC "<llave>" "<valor>"
```

Formato esperado en `local.yaml` y `dev.yaml`:

```yaml
db:
  user: "![<valor-cifrado>]"
  password: "![<valor-cifrado>]"
```

## Ejecución Local

Requisitos:

- JDK 17.
- Anypoint Studio 7.22+ o compatible con Mule 4.9 / Java 17.
- Postman o cliente HTTP equivalente.
- Acceso a Anypoint Platform para publicar/desplegar.

Pasos:

1. Abrir el proyecto en Anypoint Studio.
2. Verificar que Maven resuelva el driver JDBC de MySQL.
3. Ejecutar con propiedades de runtime:

```text
-M-Dmule.env=local -M-Dmule.key=<llave-local>
```

4. Probar:

```http
GET http://localhost:8081/api/v1/sps/customers
```

## Despliegue En CloudHub 2.0

La aplicación está desplegada en CloudHub 2.0 (Shared Space, US East Ohio). Endpoint público en producción:

```http
GET https://sps-customers-system-api-ulzm5g.5sc6y6-2.usa-e2.cloudhub.io/api/v1/sps/customers
```

| Atributo | Valor |
| --- | --- |
| Deployment model | CloudHub 2.0 |
| Release channel | Edge |
| Runtime | Mule 4.11.4 |
| Java | 17 |
| Replicas | 1 × 0.1 vCore |
| Region | US East (Ohio) |

Properties inyectadas en Runtime Manager:

```text
mule.env=dev
mule.key=<llave-local>                              # protected
anypoint.platform.client_id=<connected-app-id>      # protected
anypoint.platform.client_secret=<connected-app-secret>  # protected
anypoint.platform.config.analytics.agent.enabled=true
```

Las dos propiedades `anypoint.platform.*` provienen de una **Connected App**
(Access Management → Connected Apps) con scopes:

- `Exchange Viewer` (org SPS)
- `View Environment` (Sandbox)
- `API Manager Environment Administrator` (Sandbox)

En CloudHub 2.0 estas credenciales **no se inyectan automáticamente** (a diferencia
de CH 1.0), por lo que el runtime no puede registrar Autodiscovery sin ellas.
Mientras falten, los logs muestran `Client ID or Client Secret were not provided.
API Platform client is DISABLED.` y la instancia queda como `Unregistered`.

## API Manager Y Client-ID Enforcement

API Instance ID activo: **`20913965`** (sandbox · Mule Gateway).

### Flujo de activación

1. Publicar el RAML `sps-customers-api.raml` en Exchange.
2. Crear una instancia del API en API Manager seleccionando **Mule Gateway** y
   Implementation URI `https://<app>.cloudhub.io/api/v1/sps`.
3. Copiar el API Instance ID en `api.id` dentro de `dev.yaml`.
4. Configurar las propiedades de Connected App en Runtime Manager.
5. Redesplegar la aplicación a CloudHub.
6. Confirmar que la instancia pasa a estado **Active** en API Manager.
7. Aplicar la política **Client-ID Enforcement** con custom expressions:

   ```text
   Client ID Expression:     #[attributes.headers['client_id']]
   Client Secret Expression: #[attributes.headers['client_secret']]
   ```

8. Crear una client application en Exchange (`postman-client`) y solicitar
   contrato; aprobarlo desde API Manager.
9. Probar llamadas sin credenciales y con credenciales válidas.

### Resultados validados en producción

| Caso | Resultado |
| --- | --- |
| Sin `client_id` / `client_secret` | `401 Unauthorized` · `{ "error": "Authentication denied." }` |
| Con credenciales válidas | `200 OK` · arreglo JSON de clientes |

### Lección técnica registrada

`api.id` vive dentro del archivo cargado por `<secure-properties:config>`. Aunque
no esté cifrado, **debe referenciarse con el prefijo `secure::`** porque el módulo
Secure Properties expone todas las claves del archivo bajo ese prefijo:

```xml
<api-gateway:autodiscovery apiId="${secure::api.id}" flowRef="sps-customers-main"/>
```

Sin el prefijo el deploy falla con
`PropertyNotFoundException: Couldn't find configuration property value for key ${api.id}`
durante el arranque de la app.

## Manejo De Errores

El flow principal normaliza errores comunes en JSON:

```json
{
  "error": "Error inesperado al procesar la solicitud",
  "code": "DB:CONNECTIVITY",
  "timestamp": "2026-05-17T18:30:00Z",
  "correlationId": "2f8f3950-1438-11ef-91a9-acde48001122"
}
```

Códigos manejados:

- `400` para solicitudes inválidas.
- `404` para recursos no encontrados.
- `405` para métodos no permitidos.
- `500` para errores inesperados o fallas de infraestructura.

## Pruebas De Aceptación

- `GET /api/v1/sps/customers` responde `200` localmente.
- La respuesta es un arreglo JSON de clientes.
- `local.yaml` y `dev.yaml` existen y no contienen credenciales en texto plano.
- `global.xml` contiene configuraciones globales reutilizables.
- `implementation.xml` contiene la lógica del endpoint.
- El RAML se puede publicar en Exchange.
- CloudHub responde antes de aplicar Client-ID Enforcement.
- API Manager muestra Autodiscovery activo.
- Client-ID Enforcement bloquea llamadas no autorizadas.
- La colección de Postman permite reproducir las pruebas principales.

## Evidencia

Cada paso del flujo está documentado con capturas reproducibles en `docs/screenshots/`.

| # | Capa | Captura |
| --- | --- | --- |
| 03 | Exchange · RAML publicado | [`03-exchange-published.png`](docs/screenshots/03-exchange-published.png) |
| 04 | CloudHub · endpoint público 200 | [`04-cloudhub-200-publico.png`](docs/screenshots/04-cloudhub-200-publico.png) |
| 05 | Postman · CloudHub público 200 | [`05-postman-cloudhub-200-publico.png`](docs/screenshots/05-postman-cloudhub-200-publico.png) |

> Capturas complementarias previstas en futuras iteraciones:
> `02-cloudhub-runtime-running`, `06-api-manager-active`,
> `07-postman-cloudhub-401`, `08-postman-cloudhub-200-clientid`.

## Documentación Complementaria

- `docs/guia-tecnica-implementacion.md`: explicación técnica de la implementación.
- `docs/cloudhub-api-manager-checklist.md`: checklist de CloudHub, API Manager y Client-ID Enforcement.
- `docs/plan-rapido-sps-customers-system-api.md`: plan de trabajo y criterios de aceptación.
