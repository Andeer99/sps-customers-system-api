# SPS Customers System API

System API construida para el reto técnico de MuleSoft Trainee. Expone la información de clientes desde la base MySQL de entrenamiento de MuleSoft mediante:

```http
GET /api/v1/sps/customers
```

La aplicación está organizada como un proyecto Maven de Mule 4.9 / Java 17. Puedes abrir la carpeta en IntelliJ para revisar RAML, XML, documentación y Postman; usa Anypoint Studio o Anypoint Platform para ejecución local con runtime Mule, publicación en Exchange, despliegue en CloudHub, API Manager, Autodiscovery y políticas.

## Estado del repositorio

- Rama de trabajo actual: `codex/adoc-documentacion-inicial`.
- Remoto GitHub configurado: `https://github.com/Andeer99/sps-customers-system-api.git`.
- Rama publicada: `origin/codex/adoc-documentacion-inicial`.
- Estrategia de entrega: subir avances pequeños por ramas `codex/adoc-*`, dejando README y `HANDOFF.md` actualizados en cada corte.
- Archivo de continuidad: `HANDOFF.md`.
- Plan de trabajo base: `docs/plan-rapido-sps-customers-system-api.md`.

Pull request sugerido para revisar este primer corte:

https://github.com/Andeer99/sps-customers-system-api/pull/new/codex/adoc-documentacion-inicial

## Qué incluye

- Especificación RAML 1.0 en `src/main/resources/api/sps-customers-api.raml`.
- Flujos Mule separados en `src/main/mule/global.xml` e `src/main/mule/implementation.xml`.
- Propiedades por ambiente en `src/main/resources/local.yaml` y `src/main/resources/dev.yaml`.
- Usuario y contraseña de base de datos cifrados con Mule Secure Properties.
- Transformación DataWeave hacia `firstName`, `lastName`, `email`, `company`.
- Respuestas de error JSON con `error`, `code`, `timestamp` y `correlationId`.
- Colección de Postman en `postman/`.
- Checklist de CloudHub y API Manager en `docs/cloudhub-api-manager-checklist.md`.
- Guía técnica de implementación en `docs/guia-tecnica-implementacion.md`.

## Construcción por etapas

Esta sección documenta el proyecto como si se hubiera construido desde cero, para que cada rama de GitHub pueda contar una parte clara de la historia.

El plan rector está guardado en `docs/plan-rapido-sps-customers-system-api.md`: prioriza endpoint funcional, Secure Properties, CloudHub, API Manager, Autodiscovery y Client-ID Enforcement antes de mejoras decorativas.

### 1. Base Maven y runtime Mule

El primer paso fue crear un proyecto Maven con `packaging` de Mule:

- `groupId`: `com.sps.training`
- `artifactId`: `sps-customers-system-api`
- Runtime objetivo: Mule `4.9.0`
- Plugin Mule Maven: `4.3.0`
- Java objetivo: 17

En `pom.xml` se agregaron los conectores necesarios para HTTP, APIkit, Database, Secure Properties y MySQL.

### 2. Contrato RAML

Después se definió el contrato en `src/main/resources/api/sps-customers-api.raml`:

- Recurso: `/customers`
- Método: `GET`
- Respuesta exitosa: arreglo de `Customer`
- Error estándar: `ErrorResponse`

El contrato es la pieza que APIkit usa para enrutar la llamada entrante hacia el flow de implementación.

### 3. Configuración global

La configuración reusable quedó en `src/main/mule/global.xml`:

- `Secure_Properties_Config` carga `${mule.env}.yaml` y descifra valores con `${mule.key}`.
- `HTTP_Listener_config` centraliza host y puerto.
- `Database_Config` conecta a MySQL con valores protegidos.
- `SPS_Customers_API_Config` registra el RAML en APIkit.
- `api-gateway:autodiscovery` deja preparada la vinculación con API Manager.

Esta separación evita mezclar infraestructura con lógica de negocio.

### 4. Implementación del endpoint

La lógica vive en `src/main/mule/implementation.xml`:

1. `sps-customers-main` recibe solicitudes HTTP bajo `/api/v1/sps/*`.
2. APIkit resuelve el recurso definido en RAML.
3. `get:\customers:SPS_Customers_API_Config` ejecuta `SELECT * FROM customers;`.
4. DataWeave transforma las filas a un JSON más limpio para consumidores.
5. Los logs incluyen `correlationId` para rastrear cada request.

### 5. Manejo de errores

El flow principal responde con JSON consistente para:

- `400` cuando APIkit detecta solicitud inválida.
- `404` cuando el recurso no existe.
- `405` cuando el método no está permitido.
- `500` para errores inesperados o fallas de infraestructura.

Cada respuesta de error incluye `correlationId`, útil para revisar logs en runtime local o CloudHub.

### 6. Validación y entrega

La colección de Postman cubre pruebas locales y CloudHub. La guía en `docs/` documenta los pasos de Exchange, Runtime Manager, API Manager, Autodiscovery y Client-ID Enforcement.

## Estrategia de ramas

Propuesta de ramas pequeñas para subir el repo poco a poco:

| Orden | Rama | Objetivo |
| --- | --- | --- |
| 1 | `codex/adoc-documentacion-inicial` | README, comentarios de intención y `HANDOFF.md`. |
| 2 | `codex/adoc-contrato-raml` | Ajustes o validación fina del contrato RAML y ejemplos. |
| 3 | `codex/adoc-configuracion-global` | Revisión de properties, Secure Properties y Autodiscovery. |
| 4 | `codex/adoc-implementacion-clientes` | Endurecer flow, DataWeave, logs y manejo de errores. |
| 5 | `codex/adoc-cloudhub-api-manager` | Checklist final de despliegue, Postman y evidencias. |

## Fuentes del alcance

- Enunciado PDF: `C:\Users\alexa\Downloads\Práctica - Mulesoft Trainee.pdf`.
- Análisis HTML: `C:\Users\alexa\Downloads\mulesoft-trainee-analisis.html`.
- Plan rápido del proyecto: `docs/plan-rapido-sps-customers-system-api.md`.

La entrega se enfoca en lo que pide el enunciado: exponer clientes en `/api/v1/sps/customers`, usar MySQL `training.customers`, proteger credenciales, separar archivos globales/implementación, desplegar en CloudHub y completar los extras oficiales de API Manager.

## Prerrequisitos locales

- JDK 17.
- Anypoint Studio 7.22+ o versión compatible con Mule 4.9 / Java 17.
- Cuenta o trial de Anypoint Platform.
- Postman.
- Git para publicar ramas en GitHub.

## Secure Properties

Las credenciales de la base de datos del ejercicio están cifradas en `local.yaml` y `dev.yaml`. La llave de descifrado no debe subirse al repositorio. Usa el archivo local `LOCAL_SECRETS_DO_NOT_COMMIT.md`, que está ignorado por `.gitignore`, o genera tu propia llave y vuelve a cifrar ambos valores.

Comando de ejemplo con Secure Properties Tool para Java 17:

```powershell
java -cp secure-properties-tool-j17.jar com.mulesoft.tools.SecurePropertiesTool string encrypt Blowfish CBC "<tu-llave>" "<valor-a-cifrar>"
```

Después coloca la salida así:

```yaml
db:
  user: "![<valor-cifrado>]"
  password: "![<valor-cifrado>]"
```

## Ejecución local

1. Abre la carpeta en IntelliJ si quieres revisar o editar el proyecto Maven, RAML, XML, docs y archivos de Postman.
2. Importa o abre el proyecto en Anypoint Studio cuando estés listo para ejecutarlo con Mule.
3. Verifica que se resuelva la dependencia del driver JDBC de MySQL.
4. Ejecuta la aplicación con estos argumentos de VM:

```text
-M-Dmule.env=local -M-Dmule.key=<tu-llave-local>
```

5. Prueba el endpoint:

```http
GET http://localhost:8081/api/v1/sps/customers
```

Respuesta esperada:

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

Los valores reales vienen de la base de datos de entrenamiento de MuleSoft.

## Despliegue en CloudHub

Despliega después de validar localmente.

Propiedades que debes agregar en Runtime Manager:

```text
mule.env=dev
mule.key=<tu-llave-local>
```

Si Autodiscovery está activo, configura también las credenciales de Anypoint Platform en Runtime Manager:

```text
anypoint.platform.client_id=<environment-client-id>
anypoint.platform.client_secret=<environment-client-secret>
```

Endpoint objetivo:

```http
GET https://<nombre-app-cloudhub>.cloudhub.io/api/v1/sps/customers
```

## Extras de API Manager

1. Publica el RAML en Exchange.
2. Crea una instancia de API en API Manager desde Exchange.
3. Copia el API Instance ID en `dev.yaml` como `api.id`.
4. Redespliega en CloudHub.
5. Aplica la política Client-ID Enforcement.
6. Prueba:
   - Sin `client_id` / `client_secret`: debe responder `401`.
   - Con headers válidos: debe responder `200`.

## Checklist de aceptación

- `GET /api/v1/sps/customers` funciona localmente.
- El endpoint de CloudHub funciona antes de aplicar Client-ID Enforcement.
- Existen `local.yaml` y `dev.yaml`.
- `global.xml` e `implementation.xml` están separados.
- El usuario y la contraseña de la base de datos están cifrados con Secure Properties.
- La especificación API existe y se puede publicar en Exchange.
- La instancia en API Manager queda vinculada mediante Autodiscovery.
- Client-ID Enforcement bloquea llamadas no autorizadas.
- README, `HANDOFF.md` y colección de Postman permiten que el evaluador reproduzca la prueba.
