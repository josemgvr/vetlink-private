# API VetLink

API de VetLink para normalizar historias clínicas veterinarias y conectar clínica generalista y especialista: importación multiformato, ficha canónica, alertas, derivaciones, disponibilidad, agenda, briefing pre-visita y cierre.

> **Prototipo de hackathon.** No es un producto sanitario ni una implementación lista para producción. Usa datos ficticios y autenticación de demostración.

- **Stack:** Python 3.11+ · FastAPI · SQLModel · SQLite · cookies firmadas con `itsdangerous`.
- **Contrato:** [`contracts/openapi-v1.json`](../../contracts/openapi-v1.json) — **27 rutas / 30 operaciones**.
- **Swagger:** `http://127.0.0.1:8000/docs` · **OpenAPI:** `http://127.0.0.1:8000/openapi.json`.
- **Sin LLM:** el briefing y las alertas son deterministas.
- **Licencia:** MIT en el [`LICENSE`](../../LICENSE) raíz; terceros en [`THIRD_PARTY_NOTICES.md`](../../THIRD_PARTY_NOTICES.md).

## Arranque rápido

Desde la raíz del repositorio:

```bash
python start.py
```

El script crea el entorno, instala dependencias, compila la SPA si hace falta y arranca el servidor. La API queda en:

```text
http://127.0.0.1:8000/api/health
http://127.0.0.1:8000/docs
```

### Solo la API

**Windows**

```powershell
cd services/api
python -m venv .venv
.venv\Scripts\python.exe -m pip install -r requirements.txt
.venv\Scripts\python.exe -m uvicorn app.main:app --reload --port 8000
```

**macOS / Linux**

```bash
cd services/api
python3 -m venv .venv
.venv/bin/python -m pip install -r requirements.txt
.venv/bin/python -m uvicorn app.main:app --reload --port 8000
```

La base de datos y los uploads se crean solos en `data/vetlink.db` y `data/uploads/`. Si la base está vacía, se siembra al arrancar. `POST /api/demo/reset` borra y vuelve a sembrar los datos de demostración.

### Variables de entorno

| Variable | Uso | Valor por defecto |
|---|---|---|
| `VETLINK_SECRET_KEY` | Firma de la cookie de sesión | Clave de demo |

En producción hay que definir una clave propia y no reutilizar la de demo. La ruta de SQLite y los orígenes CORS se configuran en `app/config.py`.

## Autenticación demo

`POST /api/auth/demo` con `{ "user": "laura" }` o `{ "user": "marc" }` crea una cookie `session` HttpOnly de 12 horas.

| Usuario | Rol | Uso |
|---|---|---|
| `laura` | `CLINIC` | Pacientes, importación, revisión, derivaciones y agenda de la clínica |
| `marc` | `SPECIALIST` | Solicitudes asignadas, aceptación, briefing, disponibilidad y cierre |

No hay registro, contraseña, JWT, SSO ni MFA. Es una autenticación deliberadamente limitada para la demo.

## Endpoints

Prefijo: `/api`. Las rutas marcadas como clínica o especialista aplican RBAC y pueden devolver `403`.

### Sistema y sesión

| Método | Ruta | Notas |
|---|---|---|
| GET | `/health` | Estado, base, seed y contadores |
| GET | `/me` | Usuario de la sesión |
| POST | `/auth/demo` | Login de demostración |
| GET | `/auth/me` | Alias del usuario actual |
| POST | `/auth/logout` | Elimina la cookie |
| POST | `/demo/reset` | Borra y resiembra la demo |
| GET | `/demo/state` | Estado del dataset de demostración |

### Pacientes y ficha clínica

| Método | Ruta | Notas |
|---|---|---|
| GET | `/patients` | Pacientes visibles para la clínica, con alertas |
| GET | `/patients/{patient_id}/record` | Registro canónico, procedencia y alertas; audita la consulta |
| PATCH | `/patients/{patient_id}/record/review` | Confirma la revisión humana de campos dudosos |

### Importación y mapeo

| Método | Ruta | Notas |
|---|---|---|
| POST | `/imports` | Multipart `files`: `.docx`, `.xlsx`, `.json` o `.csv` |
| GET | `/imports` | Historial de importaciones |
| GET | `/imports/{job_id}` | Estado de una importación |
| GET | `/imports/{job_id}/mapping` | Propuesta de mapeo de columnas |
| POST | `/imports/{job_id}/mapping` | Confirma o corrige el mapeo |

Los ficheros exactos de ejemplo se reconocen por SHA-256 y se resuelven al instante. Las variantes y los CSV genéricos pasan por extracción y quedan en `NEEDS_REVIEW` si necesitan revisión.

El mapeo **no permite ignorar ni reasignar** las columnas de identidad (`patient.name`, `patient.microchip`): la API devuelve `422`. La ficha resultante conserva procedencia y campos pendientes.

### Especialistas y disponibilidad

| Método | Ruta | Notas |
|---|---|---|
| GET | `/specialists` | Búsqueda por especialidad y zona; incluye siguiente hueco |
| GET | `/specialists/{specialist_id}` | Ficha y franjas disponibles |
| PATCH | `/specialists/{specialist_id}/availability` | Solo el propio especialista puede editar su agenda |

### Derivaciones y ciclo de visita

| Método | Ruta | Notas |
|---|---|---|
| POST | `/requests` | Crea una derivación desde la clínica |
| GET | `/requests` | Clínica: propias; especialista: asignadas |
| GET | `/requests/{request_id}` | Detalle de la solicitud |
| GET | `/requests/{request_id}/summary` | Resumen no identificativo antes y después de aceptar |
| POST | `/requests/{request_id}/accept` | Acepta, opcionalmente con `slot_iso`; crea la cita |
| POST | `/requests/{request_id}/decline` | Rechaza la derivación |
| POST | `/requests/{request_id}/cancel` | Cancela desde la clínica |
| POST | `/requests/{request_id}/close` | Cierra la visita y devuelve la nota a la ficha |
| GET | `/requests/{request_id}/briefing` | Solo tras aceptar; `403` si está bloqueado |
| GET | `/requests/{request_id}/audit` | Eventos de auditoría de la solicitud |

Reglas relevantes:

- una franja pasada devuelve `422`;
- una doble reserva devuelve `409`;
- el briefing completo no se sirve antes de aceptar;
- el cierre y la nota clínica solo los puede hacer el especialista asignado;
- el resumen previo filtra tutor, microchip, documentos y facturas.

### Agenda y auditoría global

| Método | Ruta | Notas |
|---|---|---|
| GET | `/appointments?from=&to=` | Citas visibles en un rango |
| GET | `/audit` | Últimos eventos de auditoría visibles según rol |

## Briefing y resumen

`app/briefing.py` construye el briefing desde el registro canónico:

- titular y etiqueta del paciente;
- conteo de alertas por severidad;
- tres acciones prioritarias;
- antigüedad del dato;
- huecos de información;
- alertas renderizadas.

No usa un modelo generativo. El resumen previo a la aceptación aplica un filtro de identidad y no incluye datos del tutor, microchip, documentos ni facturas.

## Estructura

```text
services/api/
├── app/
│   ├── main.py              # app, CORS, /api, SPA opcional
│   ├── config.py            # rutas y configuración
│   ├── context.py           # contexto de petición y usuario
│   ├── db.py                # engine SQLite, sesión y auditoría
│   ├── models.py            # tablas SQLModel
│   ├── schemas.py           # entrada/salida Pydantic
│   ├── auth.py              # cookie firmada y RBAC
│   ├── ingestion.py         # persistencia canónica y matching
│   ├── briefing.py          # briefing y resumen no identificativo
│   ├── seed.py              # seed idempotente y caché SHA-256
│   ├── errors.py            # errores comunes
│   ├── canonical/           # alertas y matching
│   ├── extractors/          # docx, xlsx, json, csv, mapeo y preview
│   └── routers/             # auth, patients, imports, specialists,
│                            # requests, appointments, audit y demo
├── tests/
├── pyproject.toml
└── requirements.txt
```

## Tests

```bash
cd services/api
python -m pytest
```

Estado verificado: **70 tests pasan**. El contrato tiene **27 rutas / 30 operaciones**.

## Alcance y limitaciones

- SQLite y cookie de demo; no hay SSO, MFA, rate limiting ni hardening de producción.
- Sin OCR/PDF, pagos, email real, integraciones PMS ni multi-clínica.
- La caché de importación es una optimización de demo, no un pipeline de producción.
- El briefing es determinista y **no es un dispositivo médico**.
- Antes de usar datos reales hacen falta RGPD/LOPDGDD, consentimiento, retención, DPA y auditoría de producción.
- El historial público del repositorio contiene documentación interna antigua; no debe tomarse como un snapshot limpio.

## Enlaces

- [README del proyecto](../../README.md)
- [Contribuidores](../../CONTRIBUTORS.md)
- [Contrato OpenAPI](../../contracts/openapi-v1.json)
- [Licencia MIT](../../LICENSE)
- [Avisos de terceros](../../THIRD_PARTY_NOTICES.md)
