# Afiliados PRO (Backend) — Project Context

## Objetivo de negocio
Backend que soporta Afiliados PRO: afiliación, referidos (hasta 4 niveles), suscripciones/pagos, webhooks, cálculo de comisiones y endpoints para dashboard/superadmin.

## Tech Stack
- Runtime: Node.js (>= 20) en Cloud Run
- Framework: Express
- Validación: zod
- Auth/DB: Firebase Admin SDK (Auth + Firestore)
- Observabilidad: morgan
- Deploy: Google Cloud Run

## Arquitectura (decisiones clave)
- Servicio stateless:
  - Toda persistencia va en Firestore.
  - La identidad se valida con Firebase ID token (Authorization bearer).
- CORS explícito:
  - `CORS_ORIGIN` controla orígenes permitidos (CSV); no usar `*` en producción salvo necesidad.
- Roles/admin:
  - `ADMIN_EMAILS` define correos con permisos de superadmin.
  - El backend también protege cuentas owner (no modificar/borrar).
- Webhooks:
  - PayPal/Culqi entran por endpoints de webhook y actualizan estado/plan de forma idempotente.
- Seguridad:
  - Secrets y credenciales viven solo en Cloud Run (env vars).
  - Frontend no debe tener claves privadas.

## Modelo de datos (alto nivel)
- `users/{uid}`: perfil, plan (basic/pro/elite), status (TRIAL/ACTIVE/SUSPENDED), referralCode, referredBy, disabled, timestamps.
- `stats/{uid}`: métricas (earnings/balances/plan).
- `users/{uid}/network/*` y/o consultas por `referredBy` para construir downline.
- `users/{uid}/activity/*`: actividad reciente.

## Reglas UI/UX (impacto en backend)
- Respuestas consistentes y rápidas (evitar payloads gigantes).
- Errores: no filtrar secretos, usar HTTP status correctos.

## Convenciones de código
- ESM (`type: module`).
- Validar `req.body/query/params` con zod en el borde.
- Fallar rápido si faltan variables críticas (ej: `FIREBASE_SERVICE_ACCOUNT`).

## Variables de entorno (resumen)
- Firebase: `FIREBASE_SERVICE_ACCOUNT`
- CORS: `CORS_ORIGIN`
- Admin: `ADMIN_EMAILS`
- PayPal/Culqi y settings de negocio (ver `README.md`)

