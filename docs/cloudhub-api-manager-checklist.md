# Checklist de CloudHub y API Manager

Usa esta lista después de que el endpoint local funcione.

## 1. Publicar la especificación del API

1. Abre Anypoint Platform.
2. Entra a Design Center o API Designer.
3. Crea o importa el RAML desde `src/main/resources/api/sps-customers-api.raml`.
4. Publícalo en Exchange como `sps-customers-system-api`.

## 2. Desplegar en CloudHub

1. En Anypoint Studio, da clic derecho sobre el proyecto Mule.
2. Selecciona `Anypoint Platform > Deploy to CloudHub`.
3. Usa un nombre de aplicación como `sps-customers-system-api-<iniciales>`.
4. Runtime: Mule 4.9.x LTS, Java 17.
5. Worker: el más pequeño disponible para cuentas trial.
6. Agrega las propiedades de aplicación:
   - `mule.env=dev`
   - `mule.key=<llave local secreta>`
7. Despliega y espera a que la aplicación quede en estado `Started`.
8. Valida:
   - `GET https://<app>.cloudhub.io/api/v1/sps/customers`

## 3. Configurar API Manager y Autodiscovery

1. Entra a API Manager.
2. Agrega el API desde Exchange.
3. Tipo de administración: Basic Endpoint.
4. Implementation URI:
   - `https://<app>.cloudhub.io/api/v1/sps`
5. Marca la opción para Mule 4 o superior.
6. Copia el API Instance ID generado.
7. Reemplaza `api.id` en `src/main/resources/dev.yaml`.
8. Redespliega la aplicación.
9. Confirma que la instancia del API aparece activa en API Manager.

## 4. Aplicar Client-ID Enforcement

1. En API Manager, abre la instancia del API.
2. Entra a Policies.
3. Aplica `Client ID Enforcement`.
4. Crea una client application y solicita acceso al API.
5. Copia el `client_id` y el `client_secret` generados.
6. Prueba sin headers y espera `401`.
7. Prueba con headers válidos y espera `200`.

