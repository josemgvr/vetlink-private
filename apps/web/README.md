# `apps/web` — Frontend VetLink

SPA de la demo de **VetLink** (equipo **Joniha**, Reto 3 · VIII Hackathon Boehringer Ingelheim España).

> **Prototipo de hackathon.** No es un producto sanitario ni una implementación lista para producción. Los datos de la demo son ficticios.

## Stack

- React 18 + TypeScript 5.6
- Vite 5
- Tailwind CSS 3.4 + DaisyUI 4.12 (tema `vetlink`)
- React Router 6
- TanStack Query 5
- `lucide-react` + `react-dropzone`
- **Instrument Sans** local, bajo SIL OFL 1.1

## Arranque

### Opción rápida: todo el proyecto

Desde la raíz:

```bash
python start.py
```

El script prepara el entorno, compila la SPA y levanta FastAPI sirviendo `apps/web/dist` y `/api` en el mismo origen.

### Solo frontend

Requisitos: **Node.js 20+**.

```bash
cd apps/web
npm install
npm run dev
```

La SPA queda en `http://127.0.0.1:5173` y Vite reenvía `/api` a `http://localhost:8000`.

En otra terminal, arranca la API:

```bash
cd services/api
python -m uvicorn app.main:app --reload --port 8000
```

Si ya tienes el entorno de Python del proyecto, puedes arrancar la API desde la raíz con `python start.py`; para desarrollar la SPA con recarga en caliente, usa `npm run dev` en otra terminal.

### Build de producción

```bash
cd apps/web
npm run build
```

El build ejecuta primero `tsc --noEmit` y genera `dist/`. FastAPI sirve ese directorio cuando existe.

| Script | Qué hace |
|---|---|
| `npm run dev` | Servidor de desarrollo con proxy `/api` |
| `npm run build` | Typecheck + build de producción |
| `npm run preview` | Sirve `dist/` en el puerto 5173 |
| `npm run typecheck` | Solo comprobación de TypeScript |

## Variable de entorno

`.env.example`:

```text
VITE_API_URL=/api
```

Con el proxy de Vite y el modo “un proceso” no hace falta cambiarla.

## Pantallas y rutas

| Ruta | Pantalla | Rol |
|---|---|---|
| `/login` | Selección de rol demo: Laura (clínica) / Marc (especialista) | público |
| `/patients` | Pacientes, importación multiformato y revisión | CLINIC |
| `/patients/:id` | Ficha canónica, alertas, procedencia, revisión y nota de cierre | CLINIC |
| `/specialists` | Directorio de especialistas | CLINIC |
| `/specialists/:id` | Perfil, disponibilidad y solicitud de derivación | CLINIC |
| `/requests` | Solicitudes visibles según rol | ambos |
| `/requests/new` | Nueva derivación | CLINIC |
| `/requests/:id` | Detalle, resumen, briefing, aceptación, cierre y auditoría | ambos |
| `/calendar` | Citas de la clínica o disponibilidad del especialista | ambos |
| `/` | Redirección según rol | autenticado |
| `*` | Página no encontrada | público |

Las rutas de clínica están protegidas con `ClinicOnly`. El especialista no ve el marketplace en la navegación actual: su flujo es **Solicitudes + Calendario**.

### Importación

La zona de importación acepta:

- `.docx`
- `.xlsx`
- `.json`
- `.csv`

Muestra el resultado por archivo, el estado `READY` / `NEEDS_REVIEW`, el botón de revisión de mapeo y el aviso de campos pendientes. El editor de mapeo protege las columnas de identidad: no se pueden ignorar ni reasignar.

### Derivaciones y agenda

- La clínica crea la solicitud, elige urgencia y nivel de compartición, y hace seguimiento.
- El especialista acepta una franja, desbloquea el briefing, puede cerrar la visita y dejar una nota en la ficha.
- El calendario muestra citas para la clínica y permite al especialista editar sus franjas.
- El resumen previo a la aceptación no incluye datos identificativos del tutor.

## Diseño y responsive

- Tema visual cálido: crema, arcilla y tinta, con superficies glass.
- **Instrument Sans** como tipografía principal.
- Navegación adaptada por rol: la clínica ve Pacientes, Especialistas, Calendario y Solicitudes; el especialista ve Solicitudes y Calendario.
- Revisado a 1440, 390 y 320 px, sin overflow horizontal en las rutas principales.
- Estados de carga, vacío y error en las pantallas de datos.

## Estructura

```text
apps/web
├── index.html
├── vite.config.ts             # proxy /api → localhost:8000
├── tailwind.config.js         # tema, tipografía y tokens visuales
├── postcss.config.js
├── tsconfig.json
├── tsconfig.node.json
└── src
    ├── main.tsx               # QueryClientProvider + App
    ├── App.tsx                # rutas y protección por rol
    ├── index.css              # Tailwind, @font-face y clases base
    ├── api
    │   ├── client.ts          # fetch con cookies y errores normalizados
    │   ├── types.ts           # contrato de API tipado
    │   └── hooks.ts           # hooks React Query
    ├── components
    │   ├── Layout.tsx
    │   ├── Nav.tsx
    │   ├── AlertCard.tsx
    │   ├── AvailabilityEditor.tsx
    │   ├── login/             # marca, intro, roles y mascota CSS
    │   ├── patients/          # importación, mapeo, tarjetas y revisión
    │   ├── request/           # detalle, briefing, auditoría y cierre
    │   └── request-new/       # formulario y resumen de derivación
    ├── lib
    │   ├── format.ts          # fechas, edad, severidades y roles
    │   └── routes.ts          # aterrizaje por rol
    └── pages
        ├── Login.tsx
        ├── Patients.tsx
        ├── PatientRecord.tsx
        ├── Specialists.tsx
        ├── SpecialistDetail.tsx
        ├── Requests.tsx
        ├── RequestNew.tsx
        ├── RequestDetail.tsx
        └── Calendar.tsx
```

## Contrato con la API

- Sesión por cookie: `POST /api/auth/demo` con `{"user":"laura"}` o `{"user":"marc"}`; `GET /api/me` devuelve el usuario y su rol.
- Importación: `POST /api/imports` con multipart `files`; mapeo en `GET/POST /api/imports/{job_id}/mapping`.
- Revisión humana: `PATCH /api/patients/{patient_id}/record/review`.
- Especialistas: `GET /api/specialists`, `GET /api/specialists/{id}` y `PATCH /api/specialists/{id}/availability`.
- Derivaciones: `POST /api/requests`, aceptar con `{slot_iso}`, rechazar, cancelar y cerrar.
- Briefing: `GET /api/requests/{id}/briefing` responde `403` hasta que la solicitud está aceptada. La UI muestra el estado bloqueado sin reintentar.
- Cierre: `POST /api/requests/{id}/close` solo lo puede hacer el especialista asignado; la nota aparece en la ficha de la clínica.
- Resumen: `GET /api/requests/{id}/summary` no incluye tutor, microchip, documentos ni facturas.
- Errores: se muestra el `detail` de FastAPI; si el backend no responde, la UI explica cómo arrancarlo.

El contrato completo está en [`contracts/openapi-v1.json`](../../contracts/openapi-v1.json) (**27 rutas / 30 operaciones**) y en `http://127.0.0.1:8000/docs`.

## Calidad

- Build de producción y typecheck correctos.
- **70 tests de API** pasando.
- **34 comprobaciones E2E** del flujo clínica → especialista.
- Sin overflow horizontal en las rutas principales a 1440, 390 y 320 px.
- Importación de los cuatro formatos y corrección de mapeo verificadas en navegador.

## Licencia y atribución

El código original del frontend se distribuye bajo **MIT**. La fuente **Instrument Sans** conserva su licencia **SIL OFL 1.1** y su aviso está en `src/assets/fonts/OFL-InstrumentSans.txt`.

Consulta [`LICENSE`](../../LICENSE), [`THIRD_PARTY_NOTICES.md`](../../THIRD_PARTY_NOTICES.md), [`LICENSES/`](../../LICENSES/) y [`CONTRIBUTORS.md`](../../CONTRIBUTORS.md).
