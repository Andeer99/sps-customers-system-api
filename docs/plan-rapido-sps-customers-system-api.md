# Plan Rápido: SPS Customers System API en MuleSoft

## Resumen

Construir una **System API** en MuleSoft que exponga `GET /api/v1/sps/customers`, consulte MySQL `training.customers`, proteja credenciales con Secure Properties, despliegue en CloudHub y complete los extras: API Manager, Autodiscovery y Client-ID Enforcement. Gestión: checklist aquí, sin Jira/Confluence.

## Decisiones Técnicas

- Usar **Java 17 + Mule Runtime 4.9 LTS** si CloudHub/Studio lo ofrece; Anypoint Studio 7.22+ ya trabaja con Java 17.
- Repo/app: `sps-customers-system-api`; CloudHub app: `sps-customers-system-api-<iniciales>`.
- RAML 1.0:
  - Base path: `/api/v1/sps`
  - Recurso: `/customers`
  - Método: `GET`
  - Response: `200 application/json` con arreglo de `Customer`.
- Modelo de salida con DataWeave:
  - `firstName`, `lastName`, `email`, `company`
  - Fuente esperada del training DB: `FirstName`, `LastName`, `Email`, `Company`.
- Properties:
  - `local.yaml` y `dev.yaml`
  - `mule.env=local|dev`
  - `mule.key` solo por runtime, nunca en repo
  - `api.id` para Autodiscovery
- XML:
  - `global.xml`: HTTP listener, DB config, Secure Properties, APIkit config, Autodiscovery.
  - `implementation.xml`: flow principal, DB select, DataWeave transform, error handler simple.

## Ruta De Trabajo

1. Setup desde cero: instalar JDK 17, Anypoint Studio, Postman, Git/GitHub, crear cuenta/trial Anypoint.
2. Crear API Specification en API Designer, publicarla en Exchange e importarla en Studio.
3. Crear proyecto Mule desde la spec; configurar DB connector MySQL con `mudb.learn.mulesoft.com:3306`, DB `training`, query `SELECT * FROM customers;`.
4. Separar archivos: globales en `global.xml`, lógica en `implementation.xml`, properties en `local.yaml` y `dev.yaml`.
5. Cifrar `db.user` y `db.password` con Mule Secure Properties; referenciar como `secure::db.user` y `secure::db.password`.
6. Probar local con `-Dmule.env=local -Dmule.key=<clave_privada>`.
7. Desplegar a CloudHub con properties `mule.env=dev`, `mule.key`, `anypoint.platform.client_id`, `anypoint.platform.client_secret`.
8. Crear API en API Manager desde Exchange, copiar `apiId`, ponerlo como `api.id`, configurar Autodiscovery y redesplegar.
9. Aplicar Client-ID Enforcement, crear client application/contract y probar con y sin headers.
10. Entregar repo público + documentación reproducible con URL CloudHub, screenshots, instrucciones y pruebas Postman.

## Pruebas De Aceptación

- Local: `GET http://localhost:8081/api/v1/sps/customers` devuelve `200` y JSON array.
- CloudHub: `GET https://<app>.cloudhub.io/api/v1/sps/customers` responde correctamente antes de activar política.
- Seguridad: no existe `mule.key`, usuario plano ni password plano en el repo.
- API Manager: Autodiscovery aparece activo y el `api.id` corresponde al API creado.
- Client-ID:
  - Sin credenciales: `401`
  - Con `client_id` y `client_secret` válidos: `200`
- Entrega: README permite reproducir setup, deploy y pruebas sin adivinar pasos.

## Supuestos Y Fuentes

- La entrega del HTML dice **lunes 18 de mayo de 2026, 11:00am**; se prioriza funcionalidad y extras oficiales antes de mejoras decorativas.
- Si queda tiempo después de validar Client-ID, agregar solo un extra pequeño: error handler JSON o colección Postman.
- Fuentes oficiales revisadas en el plan original:
  - [Java Support](https://docs.mulesoft.com/general/java-support)
  - [Studio runtime versions](https://docs.mulesoft.com/studio/install-mule-runtime-versions)
  - [Secure Configuration Properties](https://docs.mulesoft.com/mule-runtime/4.9/secure-configuration-properties)
  - [API Autodiscovery Mule 4](https://docs.mulesoft.com/mule-gateway/mule-gateway-config-autodiscovery-mule4)
  - [Deploy to CloudHub](https://docs.mulesoft.com/runtime-manager/deploying-to-cloudhub)
  - [Client ID Enforcement](https://docs.mulesoft.com/mule-gateway/policies-included-client-id-enforcement)
