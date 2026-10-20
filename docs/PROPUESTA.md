# Viba RE · Plataforma de gestión documental y control — Propuesta técnica

> Estado: **borrador para validación**. No se programa nada hasta que Dirección lo confirme.
> Fecha: 20/10/2026

## 0. Revisión del entorno

- El repositorio `Gesti-n-de-Proyectos` está **vacío** (sin commits ni ramas). No hay código heredado de la Plataforma de Gestión de Proyectos, de los agentes de IA del grupo ni de Almirante.
- La única pista de stack del grupo es Microsoft 365 / Teams. Por eso la propuesta usa piezas que se integran bien con ese ecosistema (Entra ID, Microsoft Graph, Azure) sin atarse a él en la lógica de negocio.
- Si existe código de la Plataforma o de los agentes en otro repositorio, conviene revisarlo antes de cerrar el stack (ver preguntas).

## 1. Stack propuesto

| Capa | Elección | Por qué |
|---|---|---|
| Lenguaje | **TypeScript** en todo (front, back, PWA) | Un solo lenguaje, tipos compartidos entre la app principal y la técnica, esquemas de extracción tipados. |
| App web + API | **Next.js 15 (App Router)** con una **capa de servicios propia** (`src/server/services`) y API REST `/api/v1` | Una sola app desplegable. La API REST la consumen la app principal y la PWA técnica (backend compartido). La lógica de negocio no depende de Next: se puede extraer a un servicio aparte si crece. |
| Base de datos | **PostgreSQL 16** + **Prisma** (migraciones) | Relacional (el dominio lo es), JSONB para metadatos variables, búsqueda de texto completo en español (`tsvector 'spanish'` + `pg_trgm` para direcciones aproximadas). Sin buscador aparte en la v1. |
| Cola de trabajos | **pg-boss** (sobre el mismo Postgres) | Procesado de documentos, OCR, IA, generación de alertas y correos sin añadir Redis. |
| Archivos | Almacenamiento de objetos (**Azure Blob**, en local **MinIO**) con **cifrado en reposo** (SSE + cifrado por sobre AES‑256‑GCM a nivel de aplicación para documentos personales) | Requisito RGPD; clave maestra en Key Vault. |
| Autenticación | **Microsoft Entra ID** (OIDC, Auth.js) para el equipo interno; **enlace mágico por correo + PIN** para proveedores externos | El equipo ya tiene cuenta M365; los proveedores no. |
| UI | React + **Tailwind + shadcn/ui**, tablas con exportación CSV/XLSX, calendario FullCalendar (fase 2) | Responsive, en español, rápido de construir. |
| PWA técnica | Misma app, rutas `/tecnico`, **service worker + IndexedDB (Dexie)** con cola de cambios offline y sincronización | Instalable sin tiendas, funciona sin cobertura. |
| IA | **Claude** (Anthropic) vía API con condiciones comerciales (sin entrenamiento) o vía Microsoft Foundry / AWS Bedrock en región UE | Lectura de PDF e imágenes nativa, salida estructurada con esquema. Detrás de una interfaz `ExtractionProvider` intercambiable. |
| OCR | Capa de texto del PDF si existe; si no, **Azure AI Document Intelligence (Read)** en región UE, con **Tesseract (spa)** como alternativa local | Texto con coordenadas para indexar, buscar y localizar el fragmento de cada dato. |
| Antivirus | **ClamAV** (contenedor) | Análisis de cada archivo subido o recibido por correo. |
| Tests | **Vitest** (unitarios e integración contra Postgres de test) + Playwright (flujos clave) | |
| Despliegue | Docker; destino recomendado **Azure Container Apps + Azure Database for PostgreSQL + Blob + Key Vault** en región UE (Spain Central / West Europe) | Encaja con M365 y RGPD. En local: `docker compose up`. |

## 2. Arquitectura

```
                 ┌───────────────────────────── Next.js (una app) ─────────────────────────────┐
 Navegador ───►  │  App principal (/)     PWA técnica (/tecnico)      API REST /api/v1          │
 Móvil     ───►  │        │                       │                         │                   │
                 │        └───────────────┬───────┴─────────────────────────┘                   │
                 │                 Capa de servicios (dominio)                                   │
                 │   inmuebles · documentos · extracción · seguros · vencimientos · permisos     │
                 │   incoherencias · auditoría · (fase 2+) incidencias · agenda · correo ...    │
                 └────────┬─────────────┬───────────────┬───────────────┬──────────────────────┘
                          │             │               │               │
                    PostgreSQL     Blob cifrado     Cola pg-boss    Adaptadores externos
                    (+ FTS)        (MinIO/Azure)        │          (puertos desacoplados)
                                                       ▼           ├─ IA (Claude)
                                                 Worker(s):        ├─ OCR (Azure DI / Tesseract)
                                                 antivirus → OCR   ├─ Antivirus (ClamAV)
                                                 → clasificar      ├─ Correo (Microsoft Graph)  [F2]
                                                 → extraer         ├─ Calendario (iCal/Graph)   [F2/F4]
                                                 → vincular        ├─ Contabilidad (Almirante)  [stub]
                                                 → reglas          └─ Talento (plataforma grupo) [stub]
```

- **Permisos en la capa de servicios**, no en la UI: cada servicio recibe el `ctx` del usuario y aplica la política (rol + ámbito). La PWA técnica usa los mismos servicios con un rol restringido, así que es imposible que un técnico reciba datos de inquilinos aunque llame a la API a mano.
- **Integraciones como puertos** (`ports/accounting.ts`, `ports/talent.ts`…) con una implementación «nula» en v1. Las entidades llevan `codigoExterno` / `almiranteId` para el enlace futuro.

## 3. Modelo de datos

### Fase 1

```
Sociedad 1───* Inmueble 1───* Unidad
   │              │  │           │
   │              │  └──* Foto   └──* Foto
   │              │
   │              ├──* Adquisicion (escritura)          Tercero (compañía aseguradora, notario,
   │              ├──* ReciboIBI ──? Unidad              proveedor, persona física/jurídica)
   │              └──* Poliza ──* PolizaAmbito(Unidad)        ▲
   │                     │  ├──* Cobertura                   │ tomador / asegurado / compañía
   │                     │  └──* Siniestro [F2: ↔ Incidencia]
   │
Documento 1───* DocumentoVersion          Documento *───* (Inmueble|Unidad|Poliza|Tercero|Contrato…)
   │                                          vía DocumentoVinculo (polimórfico)
   └──* Extraccion 1───* CampoExtraido  (valor, confianza, fragmento, página, bbox, estado de validación)

Vencimiento  (origen polimórfico: póliza, documento, IBI…; fecha, preaviso, estado)
   └──* Alerta (usuario, canal, fecha programada, enviada)
Incoherencia (regla, severidad, entidad, mensaje, estado)
RequisitoDocumental (tipo de documento obligatorio por tipo de inmueble/unidad/estado)
Usuario (rol, entraId | proveedor externo)    Auditoria (quién, acción, entidad, cuándo, IP)
```

Campos principales:

- **Sociedad**: razón social, CIF, domicilio, `almiranteId`. Semilla: *Viba Real Estate 1976, S.L.U.* y *Viba Properties 1976, S.L.*
- **Inmueble**: nombre, tipo (edificio, vivienda, local…), dirección normalizada (vía, nº, CP, municipio, provincia), referencia catastral (parcela, 14 caracteres), finca registral y registro, superficies (construida, útil, parcela), sociedad titular, fecha y precio de compra, valor de referencia, **valor de reconstrucción** (para comparar con capitales asegurados), estado, renta objetivo, fotos.
- **Unidad**: inmueble, denominación (p. ej. «3ºB»), planta, puerta, referencia catastral completa (20 caracteres), finca registral, superficies, dormitorios, baños, terraza, estado (*en reforma, disponible, reservado, alquilado, en venta*), renta objetivo.
- **Documento / DocumentoVersion**: tipo, título, estado (*recibido → analizando → pendiente de revisión → validado / rechazado*), fechas de emisión y caducidad, origen (manual, correo), hash SHA‑256 (duplicados), archivo cifrado, MIME, páginas, **texto extraído** (indexado `tsvector`), versión vigente.
- **Extraccion / CampoExtraido**: tipo detectado y confianza, inmueble/unidad propuestos y por qué, modelo y versión de prompt usados; por campo: entidad y campo destino, valor propuesto, valor normalizado, confianza, **cita literal**, página y coordenadas, estado (*pendiente, aceptado, corregido, rechazado*), valor final, quién y cuándo validó.
- **Adquisicion** (escritura): fecha, notario, protocolo, vendedores, sociedad compradora, precio, ITP, AJD, IVA, superficies, referencia catastral, cargas, retenciones.
- **ReciboIBI**: ejercicio, referencia catastral, valor catastral (suelo, construcción, total), cuota, fechas de pago.
- **Poliza**: compañía, número, ramo (*continente, contenido, RC, impago, comunidad, multirriesgo hogar*), tomador, asegurado, ámbito (inmueble y/o unidades), capitales de continente y contenido, coberturas (tabla) y franquicias, prima y forma de pago, fechas de efecto y vencimiento, renovación tácita, estado.
- Documentos sin ficha propia (certificado energético, cédula, licencia, nota simple, certificado catastral, factura…) guardan sus datos en `Documento.metadatos` (JSONB validado por esquema) y alimentan campos del inmueble/unidad y vencimientos.

### Fases siguientes (se añaden por migración)

- **F2**: `Incidencia` (+ `IncidenciaEstadoHistorial`, fotos, `Presupuesto`, `ParteTrabajo`, firma), `Proveedor`, `EventoAgenda` (+ participantes y vínculos), `CorreoEntrante`, `AdjuntoCorreo`, `RemitenteAutorizado`, `SuscripcionPush`, `ColaSync` (offline).
- **F3**: `Interesado` (Tercero persona), `Candidatura` (unidad, estado), `ChecklistDocumental`, `Scoring` (ratio, factores, resultado, justificación), `Reserva`, `Garantia` (aval, seguro impago, depósito), `Contrato` (+ partes, prórrogas, `ActualizacionRenta`), `Fianza` (depósito AVRA: fecha, importe, justificante), `Cobro` mensual.
- **F4**: vistas/materializadas para el panel, `SincronizacionCalendario`.

## 4. Lectura automática de documentos

### Flujo (worker en cola, idempotente y reintentable)

1. **Ingesta**: comprobación de tipo real (magic bytes), tamaño, **antivirus**, hash SHA‑256 → si ya existe, se marca como duplicado y se enlaza al original. Cifrado y almacenamiento.
2. **Texto**: capa de texto del PDF (pdf.js) si tiene; si es escaneado o imagen → OCR con coordenadas por palabra. Word → conversión a texto. El texto se indexa para la búsqueda.
3. **Clasificación** (modelo rápido): tipo de documento + confianza + **identificadores** detectados (referencias catastrales, fincas registrales, direcciones, NIF/CIF, números de póliza). Las pistas (asunto del correo, inmueble desde el que se subió) entran como contexto.
4. **Vinculación** a inmueble/unidad, por orden de fiabilidad:
   1. referencia catastral completa (20) → unidad; 14 primeros caracteres → inmueble;
   2. finca registral;
   3. número de póliza / NIF ya conocidos;
   4. dirección aproximada (`pg_trgm`) + pista del usuario o del correo.
   Si no hay coincidencia suficiente → **propone crear** inmueble/unidad con los datos extraídos.
5. **Extracción** con un **esquema por tipo de documento** (Zod → JSON Schema, salida estructurada). Cada campo devuelve `{valor, cita_literal, pagina, confianza_modelo}`. Esquemas iniciales: escritura de compraventa, nota simple, póliza, recibo IBI, certificado catastral, certificado energético, cédula/licencia, factura, contrato de arrendamiento, DNI/NIE, nómina (estos dos ya preparados para F3).
6. **Verificación y confianza final** (no nos fiamos solo de lo que dice el modelo):
   - *Anclaje*: la cita debe aparecer en el texto OCR (coincidencia aproximada); se localiza su página y recuadro para mostrarla resaltada. Sin anclaje → confianza penalizada.
   - *Validadores deterministas*: dígitos de control de **referencia catastral**, **NIF/CIF/NIE**, IBAN; fechas coherentes (efecto < vencimiento); sumas (suelo + construcción = total; capitales); importes con formato español; ITP coherente con el tipo andaluz, etc.
   - Confianza final = combinación de modelo + anclaje + validadores.
7. **Propuesta de cambios**: se calcula un *diff* contra la ficha actual (campo vacío → «rellenar»; distinto → «actualizar», con valor anterior y nuevo). **Nada se escribe en las fichas hasta que un usuario lo confirma.**
8. **Derivados** (al validar): vencimientos y eventos (renovación de póliza con su preaviso, caducidad del certificado energético, pago de IBI…), y ejecución de las **reglas de incoherencia**.

### Umbrales de confianza (configurables)

| Confianza | Tratamiento en la revisión |
|---|---|
| **≥ 0,90** (verde) | Preseleccionado. Se puede confirmar en bloque con un clic. |
| **0,70 – 0,90** (ámbar) | Requiere confirmar el campo individualmente. |
| **< 0,70** (rojo) o validador fallido | No se preselecciona; el documento entra en la **bandeja de revisión** marcado como dudoso. |
| Clasificación < 0,80 o sin inmueble claro | Bandeja de revisión: el usuario elige tipo y/o inmueble antes de ver los campos. |

### Pantalla de validación

Documento a la izquierda (visor PDF con el fragmento resaltado), campos a la derecha agrupados por ficha destino, con valor actual → valor propuesto, semáforo de confianza y acciones aceptar / corregir / rechazar. Al confirmar queda registrado quién validó cada dato.

### Pruebas sin depender de la IA

`ExtractionProvider` tiene una implementación **simulada** que lee respuestas grabadas (*fixtures*). Los tests de extracción comprueban esquemas, normalización, validadores, anclaje, vinculación y diff de forma determinista. Con los documentos reales que me pases se crean fixtures anonimizadas y un pequeño banco de evaluación (precisión por campo y tipo).

## 5. Reglas de incoherencia (fase 1)

| Regla | Severidad |
|---|---|
| Tomador o asegurado de la póliza ≠ sociedad titular del inmueble | Alta |
| Referencia catastral del recibo/certificado ≠ la del inmueble/unidad | Alta |
| Póliza vencida sin renovación registrada | Alta |
| Capital de continente < valor de reconstrucción (o < valor de construcción catastral si no hay tasación) | Media |
| Falta un documento obligatorio de la unidad/inmueble (según `RequisitoDocumental`) | Media |
| Titular de la escritura ≠ sociedad titular de la ficha | Alta |
| Documento caducado (certificado energético, licencia…) | Media |

Se recalculan al validar un documento, al editar una ficha y cada noche. Se muestran como aviso en la ficha y en un listado.

## 6. Vencimientos y alertas

Generados a partir de datos validados, con su origen enlazado: renovación de pólizas (con fecha límite de oposición a la prórroga), caducidad de documentos, pagos de IBI y tributos, y un calendario fiscal configurable. Avisos por defecto a **90, 60 y 30 días** (configurables por tipo y por usuario), por correo y dentro de la aplicación. Exportación iCal desde la fase 1 para verlos en Outlook. En F3 se añaden fin de contratos, preavisos (LAU) y actualizaciones de renta.

## 7. Roles y permisos (RGPD)

| Recurso | Dirección | Gestión inmob. | Contabilidad | Técnico interno | Proveedor ext. | Solo lectura |
|---|---|---|---|---|---|---|
| Inmuebles/unidades | CRUD | CRUD | Ver | Dirección y unidad de sus incidencias | Ídem, mínimo | Ver |
| Datos económicos (compra, rentas, primas) | Ver/editar | Ver/editar | Ver | — | — | Ver |
| Documentos de inmueble/pólizas | Todo | Todo | Ver/descargar | — | — | Ver |
| Documentos de interesados/inquilinos (F3) | Todo | Todo | — | — | — | — |
| Validar extracciones | Sí | Sí | Facturas | — | — | — |
| Usuarios y configuración | Sí | — | — | — | — | — |
| Incidencias (F2) | Todo | Todo | Costes | Solo asignadas | Solo asignadas | Ver |

Además: auditoría de cada vista, descarga y modificación de documentos; cifrado en reposo; política de conservación con anonimización programada de candidatos no seleccionados (F3); documento `docs/RGPD-IA.md` con el proveedor de IA, base contractual (no entrenamiento, DPA, región, retención) y registro de tratamiento.

## 8. Plan por fases

**Fase 1 — MVP** (módulos 1, 2 manual, 5, 9 + auth y roles)
1. Esqueleto: Next.js, Prisma, Docker Compose (Postgres, MinIO, ClamAV), lint, Vitest, CI.
2. Modelo de datos fase 1 + migraciones + semilla (sociedades, Blas de Lezo con 9 unidades, Costa Lago en Viba Properties, resto de inmuebles previstos).
3. Autenticación (Entra ID + modo desarrollo con usuarios de prueba) y motor de permisos con tests.
4. Inmuebles y unidades: listados, fichas, fotos, exportación CSV/XLSX.
5. Repositorio documental: subida, versiones, cifrado, vínculos, búsqueda por texto, auditoría.
6. Pipeline de lectura: antivirus, texto/OCR, clasificación, vinculación, extracción por esquemas (escritura, póliza, IBI primero), validadores, anclaje.
7. Pantalla de validación y bandeja de revisión; aplicación del diff a fichas.
8. Seguros: pólizas, coberturas, ámbito, siniestros básicos.
9. Vencimientos y alertas + iCal; reglas de incoherencia.
10. Pruebas con tus documentos reales, ajuste de prompts/esquemas, README.

**Fase 2** — buzón Microsoft Graph, incidencias, agenda, PWA técnica offline.
**Fase 3** — interesados, scoring de solvencia, contratos, fianzas AVRA, cobros.
**Fase 4** — panel de dirección, sincronización Outlook.

Cada fase: migraciones, datos de ejemplo, tests de la lógica clave, README y resumen de entrega. Commits pequeños; no se avanza sin tu validación.
