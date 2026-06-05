# Chomicki Constructora — Context Document v5

## Resumen del proyecto

Sistema de registro y seguimiento de gastos para un proyecto de loteo y construcción en Cipolletti, Río Negro, Argentina. El proyecto se llama **"Estado de Israel 317"** (parcela en calle Estado de Israel 317). Los socios son **Fabián Chomicki** y **Julian Chomicki** (padre e hijo).

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
- **Tres modos de operación:**
  1. **GET** → verificación webhook Meta (hub.challenge)
  2. **POST whatsapp_business_account** → recibe mensaje WhatsApp, extrae gasto con IA, inserta en Supabase, responde confirmación. Guarda `mensaje_original`
  3. **POST webapp proxy** → recibe `{messages, system, _mode}`. Si `_mode === 'historial'` devuelve JSON sin insertar. Si no, inserta el gasto extraído.

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
- **Archivo:** `chomicki-gastos.html` — archivo único (~1900 líneas), todo en un solo HTML (CSS + HTML + JS)
- **⚠️ La SUPABASE_KEY está hardcodeada en el HTML** — al subir un archivo nuevo verificar que la legacy anon key esté correcta

---

## Dashboard — estructura y funcionalidades

### Tab: Dashboard
- **Tabs de centro de costo** en la parte superior: "Resumen" + uno por cada centro de costo
  - **Resumen:** métricas globales, barras por centro de costo (clickeables → navegan al centro), donut por categoría, últimos 8 gastos
  - **Centro específico:** métricas filtradas, barras por categoría (en azul), donut y últimos gastos del centro
  - Click en barra de centro → activa ese tab
- **Métricas:** total invertido (ARS + equivalente USD), cantidad de gastos, promedio por gasto, último registro
- **Tipo de cambio blue** en tiempo real (API: bluelytics.com.ar) — usado para conversiones en todo el dashboard

### Tab: Gastos
- Tabla completa con todos los gastos, paginación (20 por página)
- **Filtros:** Año, Mes, Día, Centro de costo, Categoría, Proveedor (texto libre)
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

### Tab: Presupuestos *(nuevo)*
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

### Tab: ⚙ Config
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
- El tipo de cambio blue (`tc`) se obtiene de bluelytics.com.ar al cargar
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

## Pendientes

### Bloqueado
- **Verificación del número de WhatsApp en Meta** — bloqueado por demasiados intentos de SMS. Sin esto el bot no puede recibir mensajes reales.

### Ideas conversadas, no implementadas
- **Bot en grupo de WhatsApp** — Meta API no soporta grupos nativamente. Alternativas: `!gasto` en mensajes individuales, o Baileys (WhatsApp Web unofficial API)
- **Guardar comprobantes del bot** — cuando llega imagen por WhatsApp, descargarla de Meta API y subirla a Storage. Requiere modificar Edge Function para `msg.type === 'image'`
- **Exportar gastos a Excel/CSV**
- **Gráfico de evolución mensual** (línea de gastos en el tiempo)
- **Agregar campo "proyecto"** para cuando haya más de un loteo
- **Notificaciones por email** cuando se carga un gasto nuevo

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

9. **Proveedores:** La fuente de verdad es `constructora_proveedores`, no los gastos. Al hacer merge de duplicados, se actualiza la tabla de gastos Y se borra el nombre viejo de `constructora_proveedores`. Al guardar un gasto nuevo con proveedor desconocido, se registra automáticamente.

10. **Storage bucket presupuestos:** Las RLS policies del bucket `constructora-presupuestos` hay que crearlas manualmente desde el SQL Editor (no desde la UI de Storage), igual que se hizo con `constructora-comprobantes`.

11. **Archivos en presupuestos:** Se organizan por carpeta `{presupuesto_id}/` dentro del bucket. Google Docs Viewer a veces tarda con archivos pesados — el botón "↗ Abrir" es el fallback confiable.
