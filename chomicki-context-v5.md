# Chomicki Constructora — Context Document v10

## Resumen del proyecto

Sistema de registro y seguimiento de gastos para un proyecto de loteo y construcción en Cipolletti, Río Negro, Argentina. El proyecto se llama **"Estado de Israel 317"** (parcela en calle Estado de Israel 317). Los socios son **Fabián Chomicki** y **Julian Chomicki** (padre e hijo).

> **Cambios v9 → v10:** refactor de robustez de la Edge Function del bot de WhatsApp. Se resolvió el doble mensaje (y un race condition latente que podía perder comprobantes) al recibir imagen + texto casi simultáneos (típico al reenviar desde otro chat). Cambios: **claims atómicos de sesión**, **nudge diferido**, **respuesta 200 inmediata + procesamiento en background (`EdgeRuntime.waitUntil`)**, e **idempotencia por `msg.id`**. Nueva columna `prompt_sent` en `constructora_bot_sessions` y nueva tabla `constructora_bot_processed`.

---

## Stack técnico completo

### Supabase — proyecto `fin-ops`
- **Proyecto:** `fin-ops` (antes llamado `ava-caja`)
- **URL:** `https://vccnehyvpextfodtvvxg.supabase.co`
- **Contexto:** Este proyecto de Supabase también contiene tablas de otros proyectos de Julian (caja de Advaitavidya: `cc`, `cp`, `txs`, `usd_txs`, `users`, `fx_history`, `pfs`). Se usa prefijo `constructora_` para separar.

---

## Tablas en Supabase

### `constructora_gastos` — tabla principal
```sql
id uuid (PK, auto)
fecha date NOT NULL
monto numeric(12,2) NOT NULL
moneda text DEFAULT 'ARS'        -- 'ARS' o 'USD'
categoria text
centro_costo centro_costo_enum
descripcion text
pagado_por text
proveedor text
fuente text                       -- 'whatsapp-bot', 'webapp', 'historial-whatsapp'
comprobante text                  -- nombre original del archivo (ej: IMG-20251117-WA0010)
imagen_url text                   -- URL pública en Supabase Storage
mensaje_original text             -- texto original del mensaje de WhatsApp (solo bot)
presupuesto_id uuid               -- FK opcional → constructora_presupuestos (ON DELETE SET NULL)
created_at timestamptz DEFAULT now()
```
- **Constraint:** `unique_comprobante` sobre campo `comprobante` (evita duplicados al reimportar historial)
- **RLS:** public read, insert, update, delete

### `constructora_presupuestos`
```sql
id uuid (PK, auto)
nombre text NOT NULL
monto_total numeric(12,2) NOT NULL
moneda text DEFAULT 'ARS'        -- 'ARS' o 'USD'
centro_costo text
proveedor text
descripcion text
created_at timestamptz DEFAULT now()
```
- **RLS:** public read, insert, update, delete

### `constructora_presupuesto_archivos`
```sql
id uuid (PK, auto)
presupuesto_id uuid NOT NULL      -- FK → constructora_presupuestos (ON DELETE CASCADE)
nombre_original text NOT NULL
archivo_url text NOT NULL
tipo text                         -- MIME type o extensión
tamanio bigint                    -- bytes
created_at timestamptz DEFAULT now()
```
- **RLS:** public read, insert, delete

### `constructora_proveedores`
```sql
id uuid (PK, auto)
nombre text NOT NULL UNIQUE
created_at timestamptz DEFAULT now()
```
- **RLS:** public read, insert, delete
- **Fuente de verdad** para la lista de proveedores. Independiente de los gastos — un proveedor persiste aunque no tenga gastos asignados.
- Poblada inicialmente con `INSERT ... SELECT DISTINCT proveedor FROM constructora_gastos`

### `constructora_bot_sessions`
```sql
phone text PRIMARY KEY
estado text NOT NULL   -- 'esperando_texto' | 'esperando_comprobante' | 'esperando_imagen'
media_id text          -- image media_id de Meta, si llegó primero
gasto_id uuid          -- FK a constructora_gastos, para poder hacer UPDATE con imagen tardía
texto text             -- ⚠️ VESTIGIAL en v10: ningún path lo escribe (ver nota abajo)
prompt_sent boolean NOT NULL DEFAULT false   -- ⭐ NUEVO v10: guard del nudge diferido
updated_at timestamptz DEFAULT now()
```
- **RLS:** `CREATE POLICY "service_all" ON constructora_bot_sessions USING (true) WITH CHECK (true)`
- Usada por el bot para correlacionar mensajes de texto e imagen que llegan por separado o en paralelo
- TTL: 2 horas — el cron `bot-session-ttl` borra sesiones vencidas y notifica al usuario
- **`prompt_sent` (v10):** controla el nudge diferido. Cuando una imagen sin caption se estaciona (`esperando_texto`), el background task espera `NUDGE_MS` y solo manda "mandame el detalle" si gana un claim atómico sobre `prompt_sent=false`. Si un texto consumió la sesión antes, el nudge no se manda.
- **`texto` es vestigial:** se mantiene en el esquema por compatibilidad pero ningún flujo lo escribe. El caso "texto primero, imagen después" se maneja con un settle + re-claim en el handler de texto, no guardando el texto. No usar este campo sin re-evaluar la lógica.

### `constructora_bot_processed` — ⭐ NUEVA en v10 (idempotencia)
```sql
msg_id text PRIMARY KEY
created_at timestamptz NOT NULL DEFAULT now()
```
- **RLS:** `CREATE POLICY "service_all" ON constructora_bot_processed USING (true) WITH CHECK (true)`
- Evita que un **reintento de Meta** del mismo webhook duplique el gasto. Al entrar un mensaje, la función inserta `msg.id` con `Prefer: resolution=ignore-duplicates, return=representation`: si devuelve fila → es nuevo (procesar); si vacío → duplicado (ignorar).
- **Fail-open:** si la tabla no existe o falla la inserción, la función procesa igual (no bloquea). Por eso es "recomendada" pero no estrictamente bloqueante.
- **Cleanup:** el cron `bot-session-ttl` borra registros con `created_at` > 24h.

### Enum `centro_costo_enum`
- Infraestructura agua y cloacas
- Infraestructura gas
- Infraestructura electricidad
- Albañilería / obra civil
- Honorarios profesionales
- Trámites / legal
- Otros

### Categorías (texto libre, validado en app)
Materiales, Mano de obra, Honorarios, Servicios, Transporte, Impuestos/Tasas, Administrativo, Otro

---

## Supabase Storage

### Bucket `constructora-comprobantes`
- **Tipo:** Public
- **Policies:** public upload (INSERT), public read (SELECT), public delete (DELETE)
- **Upsert:** `x-upsert: true` en todos los uploads
- **Formato historial:** `historial-{nombre_original}.{ext}`
- **Formato webapp:** `{timestamp}.{ext}`
- **Formato bot:** `bot-{timestamp}.{ext}`

### Bucket `constructora-presupuestos`
- **Tipo:** Public
- **Policies:** public upload (INSERT), public read (SELECT), public delete (DELETE)
- **Formato:** `{presupuesto_id}/{timestamp}-{nombre_sanitizado}.{ext}`
- Almacena los archivos adjuntos a presupuestos (PDF, Excel, Word, imágenes, etc.)

---

## API Keys Supabase
- **Legacy anon key** (para uso en browser/webapp): empieza con `eyJhbGci...` — hardcodeada en el HTML
- **Secret key** (para Edge Function): guardada como secret `CONSTRUCTORA_DB_KEY`
- **⚠️ Nota importante:** La nueva "publishable key" (`sb_publishable_...`) NO funciona con las RLS policies actuales. Siempre usar la **Legacy anon key** para el frontend.

---

## Supabase Edge Function: `quick-processor`
- **URL:** `https://vccnehyvpextfodtvvxg.supabase.co/functions/v1/quick-processor`
- **JWT verification:** OFF
- **Secrets:**
  - `ANTHROPIC_API_KEY` — API key de Anthropic (cuenta chomicki.julian@gmail.com)
  - `WHATSAPP_TOKEN` — Token permanente del bot de WhatsApp
  - `CONSTRUCTORA_DB_URL` — `https://vccnehyvpextfodtvvxg.supabase.co`
  - `CONSTRUCTORA_DB_KEY` — Secret key de Supabase
- **Modelo Claude:** `claude-sonnet-4-5` (no cambiar — puede no estar disponible con otro nombre en esta cuenta)
- **Cuatro modos de operación:**
  1. **GET** → verificación webhook Meta (hub.challenge)
  2. **POST `_mode: "cron"`** → invocado por pg_cron cada 15 min; sesiones vencidas + cleanup de `constructora_bot_processed`
  3. **POST `whatsapp_business_account`** → recibe mensaje WhatsApp. **Devuelve 200 inmediato y procesa en background.**
  4. **POST webapp proxy** → recibe `{messages, system, _mode}`. Si `_mode === 'historial'` devuelve JSON sin insertar. Si no, inserta el gasto extraído. **Síncrono** (devuelve datos reales).

### Arquitectura de robustez del bot (v10)

El problema de fondo: WhatsApp puede entregar mensajes enviados casi simultáneamente (típico al **reenviar** imagen + texto desde otro chat) **en cualquier orden** y en **invocaciones paralelas** de la Edge Function, que es stateless. La versión anterior usaba un delay fijo de 2.5s antes de `getSession()`, pero como ambas invocaciones esperaban lo mismo, despertaban juntas y producían:
- **Doble mensaje** ("Imagen recibida, mandame el detalle" + "Gasto registrado"), o
- **Pérdida de comprobante** (ambas leían `null`: el texto registraba un gasto sin imagen y pisaba la sesión).

**Solución v10 — cuatro piezas:**

1. **Respuesta 200 inmediata + procesamiento en `EdgeRuntime.waitUntil`.** El webhook responde a Meta al instante; todo el trabajo (incluida la espera del nudge) corre en background. Baja reintentos de Meta y evita que un error devuelva 5xx. Helper: `runInBackground()`. Fallback: si `EdgeRuntime` no existe (local), no se bloquea el 200.

2. **Claims atómicos de sesión.** En vez de `getSession` + `borrarSession` en dos pasos, se usan operaciones condicionales con `Prefer: return=representation`. Postgres bloquea la fila: solo una invocación recibe la fila (gana), el resto ve vacío (no-op). Helpers:
   - `claimSessionByEstado(phone, estado)` → `DELETE ?phone=eq.X&estado=eq.Y` con `return=representation`. Devuelve la fila si ganó.
   - `claimNudge(phone, mediaId)` → `PATCH ?phone=eq.X&estado=eq.esperando_texto&media_id=eq.M&prompt_sent=eq.false` SET `prompt_sent=true`. Devuelve true si ganó el derecho a nudgear.
   - `claimComprobanteAImagen(phone)` → `PATCH ?estado=eq.esperando_comprobante` SET `estado=esperando_imagen`. Transición atómica para el "sí".

3. **Nudge diferido.** La imagen sin caption estaciona la sesión (`esperando_texto`) y **no responde**. El background task espera `NUDGE_MS` (8s) y recién ahí intenta `claimNudge`. Si un texto consumió la sesión antes → no hay nada → no se manda nudge. Elimina el doble mensaje en reenvíos.

4. **Idempotencia por `msg.id`.** `marcarProcesado(msg.id)` inserta el id en `constructora_bot_processed` con `ignore-duplicates`. Si es duplicado, `handleWhatsApp` retorna sin procesar. Fail-open.

**Constantes tuneables (top del archivo):**
- `SETTLE_MS = 2500` — cuánto espera el handler de texto, si no encontró imagen estacionada, antes de reintentar el claim (cubre "texto primero, imagen un instante después").
- `NUDGE_MS = 8000` — cuánto espera la imagen antes de mandar "mandame el detalle". Debe ser mayor que el gap de entrega de un reenvío y que el tiempo razonable de tipeo.

**Estados de sesión:**
- `esperando_texto` — llegó imagen sin caption (con `media_id`), esperando el detalle del gasto
- `esperando_comprobante` — gasto registrado (con `gasto_id`), se preguntó al usuario si tiene comprobante
- `esperando_imagen` — usuario respondió "sí" al comprobante, esperando la foto

**Flujos (v10):**

| Entrada | Sesión / contexto | Acción |
|---|---|---|
| Imagen con caption | cualquiera | `borrarSession` + procesar directo (insert con imagen) + confirmar |
| Imagen sin caption | claim `esperando_imagen` gana | Subir imagen, PATCH gasto (`gasto_id`), confirmar "Comprobante guardado" |
| Imagen sin caption | sin claim | Estacionar `esperando_texto`+`media_id`, esperar `NUDGE_MS`, `claimNudge`; si gana → "mandame el detalle" |
| Texto | `esperando_comprobante` + "sí" | `claimComprobanteAImagen`; si gana → "mandame la foto" |
| Texto | `esperando_comprobante` + "no" | claim `esperando_comprobante`; si gana → "guardado sin comprobante" |
| Texto | claim `esperando_texto` gana (tiene `media_id`) | Procesar texto+imagen (insert con imagen), confirmar |
| Texto | sin claim → settle `SETTLE_MS` → re-claim gana | Procesar texto+imagen, confirmar |
| Texto | sin claim tras settle | Registrar gasto, guardar `gasto_id`, `esperando_comprobante`, preguntar por comprobante |

**Cómo se resuelve cada orden de reenvío:**
- *Imagen primero, texto después:* imagen estaciona y espera; texto gana el claim de `esperando_texto`, procesa ambos, confirma una vez; el nudge a los 8s no encuentra sesión → no-op. **Un mensaje.**
- *Texto primero, imagen después:* texto no encuentra nada, espera `SETTLE_MS`, reintenta; para entonces la imagen ya estacionó → claim gana, procesa ambos. **Un mensaje.**
- *Carrera exacta:* el claim atómico garantiza un solo ganador; sin gasto-sin-comprobante.

**Tradeoff conocido:** si alguien manda una imagen sola y **tarda > `NUDGE_MS` (8s)** en escribir el detalle, verá el nudge y después la confirmación (dos mensajes). Solo aplica cuando hay una pausa real. Subir `NUDGE_MS` a 12000 da más margen de tipeo; bajarlo a 5000 hace que el nudge salga más rápido.

**Helpers de procesamiento compartidos:**
- `gastoBase(exp, extra)` — arma el objeto de insert con defaults (`fuente: "whatsapp-bot"`).
- `registrarConImagen(phone, texto, mediaId)` — extrae IA + descarga + sube imagen + insert con comprobante + confirma una vez. Usado por todos los caminos que combinan texto e imagen.

> **Estado de deploy:** el refactor v10 fue impactado y **probado OK** por Julian (los cuatro caminos: reenvío imagen+texto, imagen sola, texto solo, texto + "sí" + foto). Funcionó bien.

---

## pg_cron — job `bot-session-ttl`

- **Extensiones habilitadas:** `pg_cron`, `pg_net`
- **Frecuencia:** cada 15 minutos (`*/15 * * * *`)
- **Job ID:** 2
- **Comando:**
```sql
SELECT net.http_post(
  url := 'https://vccnehyvpextfodtvvxg.supabase.co/functions/v1/quick-processor',
  headers := '{"Content-Type":"application/json"}'::jsonb,
  body := '{"_mode":"cron"}'::jsonb
);
```
- En `_mode: "cron"`, la función: (1) notifica y borra sesiones vencidas (TTL 2h), (2) borra `constructora_bot_processed` con > 24h.
- Verificar con: `SELECT jobid, jobname, schedule, command FROM cron.job;`
- Log de ejecuciones: `SELECT * FROM cron.job_run_details ORDER BY start_time DESC LIMIT 10;`

---

## WhatsApp Bot
- **App Meta:** `expense-bot-cipo` (App ID: `2158666968009501`)
- **Portfolio:** `Chomi Group`
- **WABA ID:** `887509451031165`
- **Número del bot:** `+54 9 11 7674-8914`
- **Phone Number ID:** `1109693305568652` (⚠️ hay dos IDs en la historia: `1082456601625630` original y `1109693305568652` actual)
- **Webhook URL:** `https://vccnehyvpextfodtvvxg.supabase.co/functions/v1/quick-processor`
- **Verify token:** `chomicki2026`
- **Estado:** Número pendiente de verificación en Meta (bloqueado por demasiados intentos de SMS)

---

## GitHub Pages — Dashboard principal
- **Repo:** `https://github.com/julianfromarg/Whatsapp-expense-tracker`
- **URL:** `https://julianfromarg.github.io/Whatsapp-expense-tracker/chomicki-gastos.html`
- **Archivo:** `chomicki-gastos.html` — archivo único (~2400 líneas), todo en un solo HTML (CSS + HTML + JS)
- **⚠️ La SUPABASE_KEY está hardcodeada en el HTML** — al subir un archivo nuevo verificar que la legacy anon key esté correcta

---

## Dashboard — estructura y funcionalidades

### Tab: Dashboard
- **Tabs de centro de costo** en la parte superior: "Resumen" + uno por cada centro de costo
  - **Resumen:** métricas globales, barras por centro de costo (clickeables → navegan al centro), donut por categoría, últimos 8 gastos
  - **Centro específico:** métricas filtradas, barras por categoría (en azul, **clickeables** → abre modal con lista de gastos de esa categoría), donut, últimos gastos del centro, y **sección de presupuestos del centro** (barra de progreso por presupuesto con presupuestado/ejecutado/saldo — click navega al detalle del presupuesto)
  - Click en barra de centro → activa ese tab
- **Métricas:** total invertido (ARS + equivalente USD), cantidad de gastos, promedio por gasto, último registro
- **Tipo de cambio blue** en tiempo real (API primaria: **dolarapi.com** `GET /v1/dolares/blue`, fallback: bluelytics.com.ar) — usado para conversiones en todo el dashboard. Bluelytics bloqueaba CORS desde GitHub Pages, por eso se migró a dolarapi.com.
- **Tabla Budget vs Actual (BVA)** — aparece en todos los tabs del dashboard (resumen y cada centro), debajo de los presupuestos del centro y arriba de los últimos gastos. Función: `renderBvaTable()`, div: `#dashBvaTable`.
  - **Columnas (izquierda a derecha):** Centro/Categoría · Presup. Ejecutado · Por Ejecutar · Presup. Total · No Presup. · **Total**
  - **Definiciones:**
    - *Presup. Ejecutado:* suma de gastos con `presupuesto_id` asignado (normalizados a ARS via `gastoEnARS`)
    - *Por Ejecutar:* `max(0, Presup. Total - Presup. Ejecutado)` — saldo futuro, columna informativa, **no suma al Total**
    - *Presup. Total:* suma de `monto_total` de los presupuestos del centro (via `presupuestoEnARS`)
    - *No Presup.:* suma de gastos sin `presupuesto_id`
    - *Total:* `Presup. Total + No Presup.` en filas de centro/TOTAL · `Presup. Ejecutado + No Presup.` en filas de categoría
  - **Vista Resumen:** una fila por centro de costo (solo los que tienen datos) + fila TOTAL global. Sin desglose de categorías.
  - **Vista centro específico:** una fila por categoría (solo con gastos) + fila TOTAL del centro. Las filas de categoría muestran `—` en Presup. Total y Por Ejecutar porque el presupuesto está asignado al centro, no a la categoría.
  - **Formato montos:** millones con 2 decimales (`$38,45M`). Cero → `—` en gris.
  - **Colores de columnas:** Presup. Total en dorado (accent) · Presup. Ejecutado en blanco · Por Ejecutar en gris muted · No Presup. en azul (blue) · Total en blanco bold

### Tab: Gastos
- Tabla completa con todos los gastos, paginación (20 por página)
- **Filtros:** Año, Mes, Día, Centro de costo, Categoría, campo de búsqueda libre (busca simultáneamente en proveedor, descripción y categoría)
- **Ordenable** por cualquier columna
- Fila de totales filtrados (ARS + USD) al pie
- **Edición inline por fila:**
  - Centro de costo (dropdown)
  - Categoría (dropdown)
  - Proveedor (dropdown alimentado desde `constructora_proveedores` + fuzzy suggestions)
  - Monto (input numérico)
  - Descripción (textarea multilínea)
  - **Presupuesto** (dropdown con todos los presupuestos — muestra borde dorado si tiene uno asignado)
- **Icono 📎** → abre comprobante en nueva pestaña
- **Icono 🗑** → elimina gasto con confirmación
- **Click en fila** → modal de detalle completo (fuente, comprobante inline, mensaje original WA)

### Tab: + Cargar
- Formulario directo a Supabase
- Campos: fecha, moneda (ARS/USD), monto, categoría, centro de costo, descripción, proveedor, pagado por
- Campo proveedor con **fuzzy suggestions** al tipear
- Upload de comprobante (imagen o PDF) → sube al bucket y guarda URL
- Al guardar, si el proveedor no existe en `constructora_proveedores`, se registra automáticamente

### Tab: Historial WA
- **Paso 1:** Subir .txt exportado de WhatsApp
- **Paso 2:** Subir imágenes opcionales (selección múltiple)
- Claude analiza el historial vía Edge Function (modo historial, max_tokens: 8000)
- Tabla editable con thumbnails de comprobantes
- Campo proveedor con **fuzzy suggestions** al editar
- ✕ en fila → marca para omitir
- **Cargar:** sube imágenes al bucket (upsert) e inserta gastos en Supabase
- Counter de progreso y resultado final
- **Anti-duplicados:** constraint `unique_comprobante` + `Prefer: resolution=ignore-duplicates`

### Tab: Presupuestos
- Lista de presupuestos con barra de progreso, % ejecutado, presupuestado/ejecutado/saldo
- **Manejo de monedas:** todo se normaliza a ARS usando `tc` (tipo de cambio blue) para calcular % y barra. Los montos se muestran en la moneda del presupuesto (ARS o USD).
- Stats globales arriba: cantidad, total en ARS, % ejecutado general
- **Click en presupuesto → modal de detalle:**
  - Barra de progreso con presupuestado / ejecutado / saldo (en moneda del presupuesto)
  - Si hay gastos en distinta moneda, muestra columna de conversión
  - **Sección Archivos adjuntos:**
    - Dropzone con drag & drop (PDF, Excel, Word, imágenes, texto, etc.)
    - Lista de archivos con ícono por tipo, tamaño, botón "↗ Abrir" y "👁 Preview"
    - **Preview modal:** imágenes inline, PDF con `<iframe>`, Excel/Word/txt via Google Docs Viewer
    - Eliminación individual de archivos (borra de Storage y de la tabla)
  - **Sección Gastos asignados:** tabla con fecha, descripción, proveedor, monto original + conversión si aplica, botón ✕ para desasignar
  - **Asignador de gastos:** panel expandible con todos los gastos sin presupuesto, ordenados por relevancia (matchea proveedor/centro de costo del presupuesto → marcados "sugerido"), buscador
  - Botones editar / eliminar presupuesto
- **"+ Nuevo presupuesto":** modal con nombre, monto, moneda, centro de costo, proveedor, descripción
- **Asignación desde tab Gastos:** columna "Presupuesto" en la tabla permite asignar/cambiar directamente

### Tab: Config
- **Proveedores:** lista desde `constructora_proveedores` (persiste independientemente de los gastos)
  - Botón **🔍 Duplicados** → detección automática por fuzzy matching (umbral 60%)
    - Muestra grupos de posibles duplicados con chips seleccionables
    - Input para nombre final canónico
    - "Unificar →" actualiza todos los gastos en Supabase y borra nombres viejos de la tabla
  - Botón **+ Nuevo** → POST a `constructora_proveedores`, persiste para siempre
  - Botón ✕ → DELETE en `constructora_proveedores`
- **Categorías:** lista fija con conteo de gastos por categoría
- **Centros de costo:** lista fija con conteo y total en millones por centro

### Botón: ⚠ Borrar datos (nav)
- Doble confirmación
- Borra todos los registros de `constructora_gastos`
- Lista y borra todos los archivos del bucket `constructora-comprobantes`

---

## Sistema de fuzzy matching de proveedores

Implementado 100% en el browser, sin APIs externas. Algoritmo: **Levenshtein distance + token overlap**.

- **Normalización:** lowercase, remove accents, split tokens
- **Score:** combinación de token overlap (palabras que matchean con tolerancia de typos) y edit distance global
- **Umbral sugerencias:** 55% (dropdowns al tipear)
- **Umbral detección duplicados:** 60% (botón Config)
- **Aplicado en:** campo proveedor de "+ Cargar", campo proveedor de tabla editable en "Historial WA", botón "🔍 Duplicados" en Config

---

## Manejo de monedas

- Todos los gastos tienen campo `moneda` ('ARS' o 'USD')
- El tipo de cambio blue (`tc`) se obtiene de **dolarapi.com** (`/v1/dolares/blue`, campo `venta`) al cargar. Fallback: bluelytics.com.ar. Bluelytics no tiene CORS abierto para GitHub Pages.
- **En dashboard:** total se muestra en ARS + equivalente USD
- **En tab Gastos:** columna ARS muestra monto, columna USD muestra conversión (o `—` si el gasto ya es USD)
- **En presupuestos:**
  - `presupuestoEnARS(p)`: convierte monto del presupuesto a ARS según su moneda
  - `getEjecutadoARS(presId)`: suma todos los gastos asignados normalizados a ARS
  - `arsAMonedaPres(arsAmount, moneda)`: convierte de vuelta a la moneda del presupuesto para mostrar
  - Los montos siempre se muestran en la moneda del presupuesto; la conversión es solo interna para calcular %

---

## Datos cargados (referencia)
- **52 gastos** del historial de WhatsApp del grupo "Estado de Israel 317 - Gastos Obra"
- **Período:** octubre 2025 → junio 2026
- **Total:** ~$69.137.828 ARS
- **51 comprobantes** subidos al bucket `constructora-comprobantes`
- Los gastos del historial NO tienen `mensaje_original` (ese campo solo se llena desde el bot)

---

## Google Apps Script (legacy — no se usa)
- **URL webhook:** `https://script.google.com/macros/s/AKfycbyXUS2gLk8PzjFCrUvjx1d8dB8w_Kq5qRqN354rWvdzDOJ9PIiTsmXbnYwbKUb6sItCEw/exec`
- **Sheet ID:** `1GANVJEOhWz4vtf9z8WJxUOgZuSCElFWzvlfErEhhY30`
- Reemplazado por Supabase. El Sheet existe pero está desactualizado.

---

## Sistema de diseño — "Mercury" (v8, sin cambios en v9/v10)

En v8 se rediseñó toda la UI siguiendo `Mercury_Design_Guidelines.md` (estética neobanco: light-mode default, azul institucional, grises neutros, tablas limpias, confianza silenciosa). **Decisión de implementación clave:** se mantuvieron todos los nombres de clase CSS existentes y se reescribió solo el sistema de tokens + componentes, de modo que las ~2.000 líneas de JS que generan HTML no se tocaron.

### Tokens (HSL) — en `:root`
Tokens semánticos como canales HSL (se usan vía `hsl(var(--token))`):
`--background, --foreground, --card, --surface, --muted-bg, --muted-fg, --border-c, --border-strong, --primary, --primary-hover, --gain, --loss, --warn-c, --accent-data, --shadow-card, --shadow-modal`.

**Aliases legacy:** los nombres viejos (`--bg, --bg2, --bg3, --border, --border2, --text, --muted, --accent, --accent2, --green, --red, --blue, --warn, --mono, --serif, --mono-num`) ahora **apuntan a los tokens Mercury** vía `var()`. Esto hace que todos los estilos inline del JS resuelvan automáticamente y sean theme-aware. **No eliminar los aliases** o se rompen los estilos inline.

### Dark mode
- Toggle (ícono sol/luna Lucide) en el nav → `toggleTheme()`, persiste en `localStorage` (`mercury-theme`).
- Script en `<head>` setea `data-theme` antes del primer paint (sin flash). Respeta `prefers-color-scheme` en la primera carga.
- Los tokens dark viven en `[data-theme="dark"]`. Como los aliases referencian los tokens semánticos, el dark mode se propaga solo.

### Tipografía
- **Inter** (Google Fonts, pesos 300–700). `font-feature-settings: 'cv02','cv03','cv04','cv11'` + `font-variant-numeric: tabular-nums` en números.
- `--mono-num` = stack Inter con cifras tabulares (reemplazó a IBM Plex Mono en todos los montos).

### Regla aplicada en todo el dashboard
**Nunca colorear valores absolutos** — solo deltas/estados. Todos los montos (KPIs, tabla, BVA, Ejecutado, monto hero de modales, gastos asignados) son `--foreground`. El color queda reservado para: saldo verde/rojo, barras de presupuesto (verde/ámbar/rojo según % ejecutado), % ejecutado, y la tabla BVA es monocromática (jerarquía por peso, no por color).

### Iconos
- Emojis a color reemplazados por **SVGs Lucide inline** (line icons, `stroke="currentColor"`, 14px en botones / 18px en archivos): papelera, clip, lápiz, ojo, lupa, mensaje, cerrar (X), e iconos por tipo de archivo en `archivoIcono()`.
- Emojis decorativos de mensajes/labels (✅❌⚠️🤖💻📋) **eliminados** del dashboard — el estado se comunica por color/borde.
- **Nota:** los mensajes salientes del bot de WhatsApp sí usan emojis (✅ 📎 ⏱), porque van a la interfaz de WhatsApp, no al dashboard. La regla Mercury aplica solo a la UI del dashboard.
- Símbolos monocromáticos que se conservan (encajan en Mercury): flechas de sort `↕`, paginación `← →`, links `↗`, checks de texto.
- **Sin iconos en los tabs de navegación** (Mercury). Se quitaron `⚙` y `+` de los tabs.

### Componentes clave
- **Nav** sticky con `backdrop-blur`; **tabs en píldora** (contenedor `--muted-bg`, tab activo = card + sombra).
- **Cards/KPI:** borde + sombra sutil, sin bordes de color arriba. KPI: número 30px/700 tabular.
- **Tablas:** sin zebra, headers 12px mayúscula `--muted-fg`, hover sutil, cifras tabulares.
- **Botones:** `default` (card+borde), `primary` (azul), `btn-clear`/`btn-danger` (ghost; danger → rojo en hover).
- **Modales:** overlay `rgba(0,0,0,0.55)`, caja con `--shadow-modal` (regla `[id^="modal"] > div`).
- **Skeletons** (no spinner) en la carga inicial de los KPIs (`.skel`, `.skel-metrics`, animación `skelPulse`).
- Scrollbar custom, `scroll-behavior:smooth`, animación `pageIn` al cambiar de tab, `prefers-reduced-motion` respetado.

### Pendientes de diseño (menores, no bloqueantes)
- Spinners en cargas secundarias (historial, presupuestos, tabla) siguen siendo spinner; podrían pasar a skeleton.
- Toasts: las guidelines sugieren toasts bottom-right con auto-dismiss; hoy se usan los bloques `.success-msg`/`.error-msg` inline.

---

## Pendientes

### Bloqueado
- **Verificación del número de WhatsApp en Meta** — bloqueado por demasiados intentos de SMS. Sin esto el bot no puede recibir mensajes reales.

### Ideas conversadas, no implementadas
- **Bot en grupo de WhatsApp** — Meta API no soporta grupos nativamente. Alternativas: `!gasto` en mensajes individuales, o Baileys (WhatsApp Web unofficial API)
- **Exportar gastos a Excel/CSV**
- **Gráfico de evolución mensual** (línea de gastos en el tiempo)
- **Agregar campo "proyecto"** para cuando haya más de un loteo
- **Notificaciones por email** cuando se carga un gasto nuevo
- **Voice note transcription** via Whisper
- **Auto-registro de proveedor desde el bot** — hoy el alta automática en `constructora_proveedores` solo ocurre desde la webapp ("+ Cargar"); el bot inserta el gasto con `proveedor` pero no da de alta el proveedor en la tabla. Posible mejora futura.

---

## Notas importantes / lecciones aprendidas

1. **Supabase API keys:** Usar SIEMPRE la **Legacy anon key** (`eyJhbGci...`) desde Settings → API Keys → Legacy tab. La nueva publishable key (`sb_publishable_...`) no funciona con las RLS policies actuales.

2. **CORS con Anthropic API:** No se puede llamar a la API de Anthropic desde el browser. Siempre usar la Edge Function de Supabase como proxy.

3. **Secrets Supabase:** Los nombres de secrets no pueden empezar con `SUPABASE_`. Usar `CONSTRUCTORA_DB_URL` y `CONSTRUCTORA_DB_KEY`.

4. **Storage upsert:** Usar header `x-upsert: true` en todos los uploads para evitar errores si el archivo ya existe.

5. **Duplicados en historial:** La constraint `unique_comprobante` + `Prefer: resolution=ignore-duplicates` permite reimportar sin duplicar.

6. **Borrar datos:** El botón "Borrar datos" borra registros de `constructora_gastos` Y archivos del bucket `constructora-comprobantes`. No toca presupuestos ni proveedores.

7. **Phone Number ID:** El activo en la Edge Function es `1109693305568652`. Si el bot falla, verificar este ID en el Meta Developer Portal.

8. **Modelo Claude:** En la Edge Function se usa `claude-sonnet-4-5`. No cambiar — puede no estar disponible con otro string en esta cuenta.

9. **Proveedores:** La fuente de verdad es `constructora_proveedores`, no los gastos. Al hacer merge de duplicados, se actualiza la tabla de gastos Y se borra el nombre viejo de `constructora_proveedores`. Al guardar un gasto nuevo con proveedor desconocido desde la webapp, se registra automáticamente (el bot todavía no).

10. **Storage bucket presupuestos:** Las RLS policies del bucket `constructora-presupuestos` hay que crearlas manualmente desde el SQL Editor (no desde la UI de Storage), igual que se hizo con `constructora-comprobantes`.

11. **Archivos en presupuestos:** Se organizan por carpeta `{presupuesto_id}/` dentro del bucket. Google Docs Viewer a veces tarda con archivos pesados — el botón "↗ Abrir" es el fallback confiable.

12. **pg_cron / pg_net:** Ambas extensiones habilitadas en el proyecto `fin-ops`. `ALTER DATABASE ... SET app.edge_function_url` no está permitido desde el SQL Editor de Supabase — hardcodear la URL directamente en el comando del cron job.

13. **Bot — race condition (RESUELTO en v10):** WhatsApp entrega mensajes casi simultáneos (reenvíos) en cualquier orden y en invocaciones paralelas. El delay fijo de 2.5s de versiones previas no alcanzaba (ambas invocaciones esperaban lo mismo y despertaban juntas → doble mensaje, o peor, gasto sin comprobante). **Solución:** claims atómicos de sesión (`DELETE`/`PATCH` condicional con `return=representation` → un solo ganador por fila) + nudge diferido (la imagen espera `NUDGE_MS` y solo nudgea si nadie consumió la sesión) + 200 inmediato con procesamiento en `EdgeRuntime.waitUntil`. Probado OK en los cuatro caminos.

14. **`insertarGasto` debe retornar el ID:** usar `Prefer: return=representation` y retornar `rows[0].id` para poder guardarlo en la sesión y hacer el PATCH posterior con la imagen.

15. **Claims atómicos vía PostgREST (v10):** un `DELETE`/`PATCH` con filtro por `estado` + `Prefer: return=representation` es atómico a nivel de fila en Postgres (READ COMMITTED). La invocación que recibe la fila ganó; el resto recibe array vacío y hace no-op. Patrón lock-free para coordinar invocaciones paralelas de una Edge Function stateless.

16. **Nudge diferido vía `EdgeRuntime.waitUntil` (v10):** permite responder 200 a Meta al instante y seguir trabajando en background (incluyendo `setTimeout`/`sleep`). Hay límite de wall-clock por invocación, pero un task de ~10s está muy por debajo. Fallback si `EdgeRuntime` no existe (local).

17. **Idempotencia del webhook (v10):** Meta puede reintentar el mismo webhook. `constructora_bot_processed` (PK `msg_id`) con `ignore-duplicates` deduplica. Fail-open: si la tabla no existe, no bloquea. El cron limpia registros > 24h.

18. **Campo `texto` de `constructora_bot_sessions` es vestigial (v10):** ningún flujo lo escribe. El orden "texto primero, imagen después" se resuelve con settle + re-claim en el handler de texto, no guardando el texto. No usarlo sin re-evaluar la lógica.

---

## Migración v9 → v10 (SQL ejecutado)

```sql
-- columna para el guard del nudge diferido
ALTER TABLE constructora_bot_sessions
  ADD COLUMN IF NOT EXISTS prompt_sent boolean NOT NULL DEFAULT false;

-- tabla de idempotencia (evita gastos duplicados por reintentos de Meta)
CREATE TABLE IF NOT EXISTS constructora_bot_processed (
  msg_id text PRIMARY KEY,
  created_at timestamptz NOT NULL DEFAULT now()
);
ALTER TABLE constructora_bot_processed ENABLE ROW LEVEL SECURITY;
CREATE POLICY "service_all" ON constructora_bot_processed USING (true) WITH CHECK (true);
```
