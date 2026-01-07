# Prompt de handoff — Integración Meta (WhatsApp/Instagram) en nuevo repo

**Rol:** Actúa como un ingeniero senior. Revisa el nuevo repositorio y agrega SOLO lo necesario para portar la integración Meta (WhatsApp/Instagram) que ya existe en este proyecto, sin romper código existente.

## Objetivo
Portar a un nuevo repo (Next.js + Supabase) un MVP que permita:
1) Guardar conexiones por organización para canales Meta (WhatsApp/Instagram).
2) Recibir webhooks de Meta y convertir eventos en leads internos (crear/actualizar contacto, thread, mensaje, deal, insights).
3) Mostrar una UI simple en Settings para crear/eliminar conexiones.

## Restricciones
- No cambies diseño/UX más de lo necesario.
- No agregues features “nice to have” (OAuth completo, dashboards extra, etc.).
- Mantén la estructura y convenciones del repo destino.
- Evita refactors grandes; prioriza cambios pequeños, aislados, revisables.
- Asegura que `pnpm build` (o el build equivalente) quede en verde.

## Qué integrar (artefactos mínimos)
### 1) Migración SQL (Supabase)
Crear una migración equivalente a:
- Tabla: `channel_connections`
- Campos clave:
  - `org_id` (uuid, FK a `organizations`), `provider` (texto), `channel` (texto), `external_id` (texto)
  - `display_name` (texto), `access_token` (texto), `is_enabled` (bool)
  - `created_at`, `updated_at`
  - unique: `(org_id, provider, channel, external_id)`
- Índices:
  - `idx_channel_connections_org (org_id)`
  - `idx_channel_connections_lookup (provider, channel, external_id)`
- RLS habilitado.
- Policies: SOLO `owner/admin` de la org pueden SELECT/INSERT/UPDATE/DELETE.
  - El check debe basarse en `org_members` (o el equivalente del repo destino) y `auth.uid()`.

**Nota:** respeta el modelo multi-tenant del repo destino. Si no existe `organizations/org_members`, adapta a su esquema pero mantén el concepto “por org/tenant”.

### 2) Endpoint webhook Meta
Implementar ruta equivalente:
- `GET/POST /api/webhooks/meta`

**GET (verificación):**
- Leer query params: `hub.mode`, `hub.verify_token`, `hub.challenge`
- Si `mode === "subscribe"` y el token coincide con `process.env.META_WEBHOOK_VERIFY_TOKEN`, responder con el `challenge` (status 200)
- Si no, responder 403.

**POST (ingesta):**
- Parsear JSON; si inválido devolver 200 `{status:"ignored"}`.
- Extraer mensajes (best-effort) para:
  - WhatsApp Cloud API: `value.metadata.phone_number_id` + `value.messages[]` tipo `text`.
  - Instagram messaging: best-effort leyendo `value.messaging[]` o `entry.messaging[]`.
- Mapear a un `LeadPayload` interno:
  - `org_id` se obtiene buscando `channel_connections` por `(provider='meta', channel, external_id=<connection_external_id>)`.
  - Si no hay conexión o está deshabilitada → “skip” (NO error).
  - `meta.external_thread_id` debe ser estable (ej: `wa:<phone_number_id>:<from>` / `ig:<entryId>:<senderId>`).
- Llamar a un helper compartido de ingesta (ver siguiente punto).
- Siempre responder 200 con contadores `{received, ingested, skipped, failed}` para evitar retries infinitos.

### 3) Helper compartido de ingesta de lead
Crear una función (en `lib/` o donde corresponda en el repo destino) que reciba:
- `supabase` (service role) + `LeadPayload` (org_id, channel, from, message.text, meta.external_thread_id)

Y haga:
- Upsert/insert de `contacts` (conflict por `(org_id,email)` o `(org_id,phone)` si existe).
- Crear o reutilizar `conversation_thread` por `external_id` o por (contact_id+channel) fallback.
- Insertar `conversation_message` inbound.
- Crear o reutilizar `deal` abierto asociado al contacto.
- Upsert de `deal_insights` usando proveedor AI existente (o dummy provider si aplica).
- Insertar en `event_log` tipo `lead_captured`.

**Importante:** NO asumas nombres exactos de tablas si el repo destino difiere; adapta pero conserva el comportamiento.

### 4) UI Settings — Conexiones
Agregar una página simple (server component + client widget) tipo:
- `/settings/channels`

Funcionalidad MVP:
- Listar conexiones actuales `provider='meta'`.
- Form para crear/upsert conexión:
  - `channel`: `whatsapp | instagram`
  - `external_id`: string
  - `display_name` opcional
  - `access_token` opcional (por ahora no usado por la ingesta)
- Acción para eliminar una conexión.

API interna sugerida (si el repo destino ya usa app router):
- `GET /api/integrations/meta/connections?org_id=...`
- `POST /api/integrations/meta/connections` (upsert)
- `DELETE /api/integrations/meta/connections/:id`

### 5) Variables de entorno / documentación
Agregar a `.env.example` / docs del repo destino:
- `META_WEBHOOK_VERIFY_TOKEN`

## Checklist de validación (no romper código)
1) `pnpm build` (o equivalente) OK.
2) Rutas nuevas no interfieren con rutas existentes.
3) RLS/policies no causan recursion ni bloquean lecturas necesarias.
4) Webhook Meta responde verificación GET correctamente.
5) Conexión guardada → webhook POST crea contacto/deal/thread/mensaje.

## Notas prácticas
- Dominio para Meta webhook: `https://<tu-dominio>/api/webhooks/meta` (en Vercel, es el dominio del proyecto).
- Para WhatsApp: el `external_id` que debes guardar es el `phone_number_id` que llega en el payload del webhook.
- Para Instagram: el `external_id` suele ser `entry.id` (confirmar con el primer payload recibido).

## Entregable
Un PR o set de commits con:
- Migración
- Rutas API
- Helper de ingesta
- Página de settings
- Docs/env

No incluyas cambios no relacionados.
