# Arquitectura de ReviewPilot AI

## Visión general
ReviewPilot AI automatiza la obtención y gestión de reseñas de Google para negocios locales. El sistema se despliega sobre Vercel (frontend) y Supabase (backend, auth, edge functions, cron) y se integra con Stripe, WhatsApp Business Cloud API, Google Business Profile API y OpenAI.

```
Usuarios (Dueños) → Frontend Astro → Edge Functions (Supabase) → DB (PostgreSQL)
                                                ↘ Stripe Webhooks
                                                 ↘ WhatsApp Webhooks
                                                 ↘ Google Business Profile
                                                 ↘ OpenAI
```

## Componentes principales
- **Frontend (apps/web)**: Astro + TypeScript + TailwindCSS. Landing pública, onboarding con Stripe Checkout y dashboard privado. Usa Supabase Auth client-side para sesión.
- **Edge Functions (apps/edge-functions)**: Handlers REST/cron en TypeScript. Expone endpoints firmados para webhooks de Stripe/WhatsApp/Google, y lógica interna (e.g., disparar mensajes de reseña, sincronizar reseñas nuevas, publicar respuestas).
- **Base de datos (supabase/)**: PostgreSQL con schemas versionados por migraciones. RLS activado. Supabase Auth para usuarios de negocio.
- **Cron jobs**: Tareas programadas en Supabase (e.g., sincronizar reseñas y enviar recordatorios pendientes).

## Data model inicial (PostgreSQL)
- `public.profiles`
  - `id` (uuid, PK, referencia a Auth user)
  - `business_name` (text)
  - `gmb_location_id` (text, referencia al location de Google)
  - `wa_phone_number_id` (text, WhatsApp sender)
  - `plan_tier` (enum: starter | pro | business)
  - `stripe_customer_id` (text)
  - `stripe_subscription_id` (text)
  - `created_at`, `updated_at`

- `public.customers`
  - `id` (uuid, PK)
  - `profile_id` (uuid, FK → profiles)
  - `display_name` (text)
  - `phone_e164` (text, unique per profile)
  - `last_interaction_at` (timestamptz)
  - `metadata` (jsonb)
  - RLS: owner can read/write; edge functions with service role can insert/update.

- `public.review_requests`
  - `id` (uuid, PK)
  - `profile_id` (uuid, FK → profiles)
  - `customer_id` (uuid, FK → customers)
  - `channel` (enum: whatsapp | qr)
  - `status` (enum: pending | sent | delivered | link_opened | review_left)
  - `gmb_review_id` (text, nullable)
  - `last_message_id` (text, WhatsApp message id)
  - `created_at`, `updated_at`

- `public.reviews`
  - `id` (uuid, PK)
  - `profile_id` (uuid, FK → profiles)
  - `customer_id` (uuid, nullable)
  - `gmb_review_id` (text, unique)
  - `rating` (int)
  - `comment` (text)
  - `response_text` (text)
  - `response_status` (enum: draft | queued_publish | published | failed)
  - `published_at` (timestamptz)
  - `created_at`, `updated_at`

- `public.audit_logs`
  - `id` (uuid, PK)
  - `profile_id` (uuid)
  - `event` (text)
  - `payload` (jsonb)
  - `created_at`

## Flujos clave
- **Onboarding y suscripción**
  1. Usuario crea cuenta (Supabase Auth email/password) desde frontend.
  2. Edge Function crea customer en Stripe y genera Checkout Session con price según tier.
  3. Webhook de Stripe (`/stripe/webhook`) confirma `checkout.session.completed` → guarda `stripe_customer_id`, `subscription_id` y `plan_tier` en `profiles`.
  4. Se muestra dashboard habilitado según plan.

- **Detección de clientes vía WhatsApp**
  1. Webhook de WhatsApp (`/whatsapp/webhook`) recibe mensajes entrantes.
  2. Edge Function identifica `phone_e164`; si no existe, crea `customers` y actualiza `last_interaction_at`.
  3. Aplica reglas (p.ej., si cliente interactuó después de visita marcada) para crear `review_requests` en estado `pending`.

- **Solicitud de reseñas (WhatsApp + QR)**
  - Edge Function `review-requests-send` toma solicitudes `pending`, genera mensaje con OpenAI (prompt con tono amable) y envía por WhatsApp API usando `wa_phone_number_id` y plantilla aprobada.
  - Para QR, el frontend genera un link firmado (`/r/:token`) que redirige al enlace directo de reseña de Google.

- **Sincronización de reseñas de Google**
  - Cron job `sync-gmb-reviews` consulta Google Business Profile para cada `profile.gmb_location_id` y upserta en `reviews`.
  - Si encuentra reseñas sin respuesta, crea respuesta con OpenAI y marca `response_status = draft`.

- **Publicación automática de respuestas**
  - Edge Function `publish-responses` toma `reviews` con `response_status = queued_publish`, llama a Google API para publicar la respuesta y actualiza `published_at` o marca `failed` con detalle del error.

## Endpoints y handlers (Edge Functions)
- `POST /stripe/webhook` — Verifica firma `STRIPE_WEBHOOK_SECRET`, procesa eventos de suscripción.
- `POST /whatsapp/webhook` — Verifica token `WA_VERIFY_TOKEN`, persiste clientes y reintentos; responde 200 rápidamente.
- `POST /gmb/webhook` (opcional) — Para notificaciones push de Google si estuvieran disponibles.
- `POST /review-requests/send` — Protected por `Authorization: Bearer <SERVICE_ROLE_KEY>`; envía solicitudes pendientes.
- `POST /reviews/sync` — Protected; ejecuta sync on-demand.
- `POST /reviews/publish` — Protected; publica respuestas en Google.

## Variables de entorno clave
- Supabase: `SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY`, `SUPABASE_ANON_KEY` (frontend usa solo `ANON`).
- Stripe: `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, `STRIPE_PRICE_STARTER`, `STRIPE_PRICE_PRO`, `STRIPE_PRICE_BUSINESS`.
- WhatsApp: `WA_GRAPH_API_TOKEN`, `WA_PHONE_NUMBER_ID`, `WA_VERIFY_TOKEN`.
- Google: `GCP_SERVICE_ACCOUNT` (JSON), `GMB_LOCATION_ID` (por negocio), `GMB_DELEGATED_USER` si aplica.
- OpenAI: `OPENAI_API_KEY`, `OPENAI_MODEL`.

## Seguridad y buenas prácticas
- RLS habilitado para tablas de negocio; edge functions usan `service_role` solo en backend.
- Webhooks verifican firma/token antes de procesar. Responder 200 rápido y delegar trabajo pesado a colas/cron si es necesario.
- Evitar almacenar PII innecesaria; cifrar o enmascarar teléfonos cuando sea posible en la UI.
- Logs en `audit_logs` para acciones sensibles (envío de solicitudes, publicaciones automáticas).
- Manejo de errores estructurado: registrar `code`, `message`, `context` y no exponer detalles internos al cliente.

## Próximos pasos recomendados
- Añadir `package.json` con workspaces (`apps/web`, `apps/edge-functions`).
- Configurar Supabase CLI y migraciones iniciales para tablas anteriores.
- Implementar autenticación en frontend con `@supabase/auth-helpers-astro`.
- Prototipo de dashboard: métricas de reseñas, estado de envíos y configuración de integraciones.
- Tests unitarios para prompts y helpers de APIs externas.
