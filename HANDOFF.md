# HANDOFF.md

## Propósito

Este archivo deja contexto suficiente para continuar el trabajo aunque se pierda la ventana de conversación. Nace de la sesión `019e3830-1970-7bc1-9145-a4f719b83b85`, donde se planteó este repositorio como una System API de clientes para MuleSoft.

## Objetivo del usuario

Subir el repositorio a GitHub poco a poco, usando ramas `codex/adoc-*`, documentando el código y actualizando el README como si el proyecto se hubiera construido por partes desde cero.

## Contexto fuente

- Plan rápido confirmado por el usuario: `docs/plan-rapido-sps-customers-system-api.md`.
- Enunciado PDF: `C:\Users\alexa\Downloads\Práctica - Mulesoft Trainee.pdf`.
- Análisis HTML: `C:\Users\alexa\Downloads\mulesoft-trainee-analisis.html`.
- Fecha límite indicada en el HTML: lunes 18 de mayo de 2026, 11:00am.

## Estado actual

- Workspace local: `C:\Users\alexa\OneDrive\Escritorio\sps-customers-system-api`
- Rama local de avance: `codex/adoc-documentacion-inicial`
- Estado Git inicial encontrado: repositorio sin commits en `master`.
- Commit local inicial creado: `ff76115` (`chore: scaffold and document sps customers system api`).
- Remoto GitHub: `https://github.com/Andeer99/sps-customers-system-api.git`.
- Rama publicada: `origin/codex/adoc-documentacion-inicial`.
- PR sugerido: `https://github.com/Andeer99/sps-customers-system-api/pull/new/codex/adoc-documentacion-inicial`.
- GitHub CLI (`gh`) está instalado, pero no tiene sesión iniciada.
- El conector GitHub detecta la cuenta `Andeer99`.
- Archivo local de secretos: `LOCAL_SECRETS_DO_NOT_COMMIT.md`, ignorado por `.gitignore`.

## Qué se documentó en este corte

- README reescrito como bitácora de construcción por etapas.
- Estrategia de ramas `codex/adoc-*` agregada al README.
- Plan rápido guardado en `docs/plan-rapido-sps-customers-system-api.md`.
- Comentarios de intención agregados en:
  - `src/main/resources/api/sps-customers-api.raml`
  - `src/main/mule/global.xml`
  - `src/main/mule/implementation.xml`
- Este archivo `HANDOFF.md` para continuidad.

## Arquitectura rápida

```text
Cliente/Postman
  -> HTTP Listener: /api/v1/sps/*
  -> APIkit Router: RAML api/sps-customers-api.raml
  -> Flow get:\customers:SPS_Customers_API_Config
  -> DB Select: SELECT * FROM customers;
  -> DataWeave: firstName, lastName, email, company
  -> JSON response
```

## Archivos clave

- `pom.xml`: proyecto Maven Mule, runtime 4.9.0, conectores HTTP/APIkit/DB/Secure Properties/MySQL.
- `src/main/resources/api/sps-customers-api.raml`: contrato público de `GET /customers`.
- `src/main/mule/global.xml`: propiedades, Secure Properties, HTTP Listener, DB config, APIkit y Autodiscovery.
- `src/main/mule/implementation.xml`: flow principal, consulta a base de datos, DataWeave y errores.
- `src/main/resources/local.yaml`: configuración local con credenciales cifradas.
- `src/main/resources/dev.yaml`: configuración para CloudHub/dev con credenciales cifradas.
- `docs/guia-tecnica-implementacion.md`: explicación técnica.
- `docs/cloudhub-api-manager-checklist.md`: pasos de despliegue y API Manager.
- `postman/SPS Customers System API.postman_collection.json`: pruebas manuales.

## Decisiones tomadas

- Seguir el plan rápido del usuario como ruta principal.
- Mantener separados `global.xml` e `implementation.xml`.
- Usar Secure Properties con Blowfish/CBC porque las credenciales ya estaban cifradas así.
- No subir la llave local de cifrado.
- Dejar `api.id=0` en desarrollo hasta crear la instancia real en API Manager.
- Documentar por etapas en README para que el historial de ramas tenga sentido.
- Gestionar avance en README/HANDOFF/checklist local, sin Jira ni Confluence.

## Siguiente paso recomendado

1. Abrir PR del primer corte:

```text
https://github.com/Andeer99/sps-customers-system-api/pull/new/codex/adoc-documentacion-inicial
```

2. Crear la siguiente rama desde esta base:

```powershell
git switch -c codex/adoc-contrato-raml
```

3. Validar y documentar el contrato RAML y ejemplos.

## Ramas planeadas

| Orden | Rama | Corte |
| --- | --- | --- |
| 1 | `codex/adoc-documentacion-inicial` | README, comentarios y handoff. |
| 2 | `codex/adoc-contrato-raml` | RAML y ejemplos. |
| 3 | `codex/adoc-configuracion-global` | Secure Properties, listener, DB y Autodiscovery. |
| 4 | `codex/adoc-implementacion-clientes` | Flow, DataWeave, logs y errores. |
| 5 | `codex/adoc-cloudhub-api-manager` | CloudHub, API Manager, Postman y evidencias. |

## Comandos útiles en este entorno

Git no estaba en el `PATH` de la terminal sandbox, pero existe en:

```powershell
C:\Program Files\Git\cmd\git.exe
```

Como el repo pertenece al usuario Windows real y la terminal corre con usuario sandbox, usa:

```powershell
& 'C:\Program Files\Git\cmd\git.exe' -c safe.directory='C:/Users/alexa/OneDrive/Escritorio/sps-customers-system-api' -c core.excludesfile= status --short --branch
```

Para operaciones que escriben en `.git` puede hacer falta aprobación/escalación del sandbox.

## Validaciones pendientes

- Ejecutar build Maven cuando haya acceso a dependencias MuleSoft.
- Abrir en Anypoint Studio y probar con:

```text
-M-Dmule.env=local -M-Dmule.key=<tu-llave-local>
```

- Probar:

```http
GET http://localhost:8081/api/v1/sps/customers
```

- Configurar remoto GitHub y hacer push de las ramas incrementales.
