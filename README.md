# ReviewPilot AI

ReviewPilot AI es una plataforma SaaS para ayudar a negocios locales a obtener y gestionar reseñas de Google de forma automática usando WhatsApp, Google Business Profile y OpenAI. Este repositorio contiene la base del proyecto (frontend Astro, backend Supabase y funciones edge) junto con lineamientos de arquitectura y convenciones de código.

## Objetivos principales
- Automatizar la solicitud de reseñas a clientes detectados desde WhatsApp Business.
- Importar nuevas reseñas desde Google Business Profile y generar respuestas con OpenAI.
- Publicar respuestas de reseñas en Google bajo aprobación del negocio.
- Ofrecer un dashboard claro, seguro y fácil de usar desplegable en Vercel + Supabase.

## Tech stack
- **Frontend**: Astro + TypeScript + TailwindCSS (landing pública y dashboard privado).
- **Backend**: Supabase (PostgreSQL + Auth), Edge Functions para lógica de negocio y cron jobs.
- **Integraciones**: Stripe (Checkout + Webhooks), WhatsApp Business Cloud API, Google Business Profile API, OpenAI GPT.

## Estructura propuesta del repositorio
```
/
├─ apps/
│  ├─ web/                 # Frontend Astro (landing + app privada)
│  └─ edge-functions/      # Supabase Edge Functions (TypeScript)
├─ supabase/               # Migraciones SQL, seeds, policies
├─ docs/                   # Documentación técnica y flows
└─ .env.example            # Variables de entorno para frontend y edge functions
```

## Convenciones generales
- TypeScript siempre que sea posible (incluyendo Edge Functions).
- Variables y funciones con nombres descriptivos; comentarios en puntos críticos de seguridad/errores.
- Secrets se leen desde `process.env` y nunca se exponen al cliente.
- No añadir `try/catch` alrededor de imports. Manejar errores dentro de las funciones.

## Cómo empezar (resumen)
1. **Instalar dependencias** (se añadirá `package.json` en siguientes iteraciones):
   ```bash
   pnpm install
   ```
2. **Configurar variables de entorno** en `.env.local` (frontend) y `.env` (edge functions). Ver `docs/architecture.md` para el listado completo.
3. **Ejecutar frontend**:
   ```bash
   pnpm dev --filter web
   ```
4. **Ejecutar Edge Functions** (Supabase CLI):
   ```bash
   supabase functions serve --env-file ./apps/edge-functions/.env
   ```
5. **Correr migraciones**:
   ```bash
   supabase db push
   ```

Consulta `docs/architecture.md` para detalles de módulos, data model y flujos críticos.
