# Contrato de integracion actual - afiliados-pro-hub

Documento descriptivo (no prescriptivo) del comportamiento observado en codigo.

## Proposito

Describir el contrato real entre:

- Frontend `afiliados-pro-hub`
- Backend `afiliados-pro-hub-backend`
- Firestore (colecciones y subcolecciones usadas)
- Integraciones externas (PayPal, Culqi)

## Modelos de datos compartidos

Colecciones observadas en backend:

### `users/{uid}`

Campos observados:

- `uid`
- `email`
- `fullName`
- `plan` (`basic` | `pro` | `elite`)
- `status` (`TRIAL` | `ACTIVE` | `SUSPENDED`)
- `referralCode` (formato esperado `AF-...`)
- `referredBy` (`uid` del sponsor o `null`)
- `disabled` (sincronizado con Firebase Auth al administrar usuarios)
- `paypalSubscriptionId`
- `paypalPlanId`
- `pendingPlan`
- `planSource` (ej. `culqi`)
- `paymentMeta`
- `createdAt`, `updatedAt`

### `stats/{uid}`

Campos observados:

- `totalEarningsUsd`
- `availableBalanceUsd`
- `pendingBalanceUsd`
- `plan`
- `networkTotal`
- `updatedAt`

### `users/{uid}/network/{memberUid}`

Campos observados:

- `uid`
- `name`
- `plan`
- `level`
- `earnings`
- `createdAt`

### `users/{uid}/activity/{activityId}`

Campos observados:

- `name`
- `action`
- `level`
- `time`
- `createdAt`

### `commissions/{commissionId}`

Campos observados:

- `transactionId`
- `beneficiaryId`
- `level`
- `percent`
- `amountUsd`
- `status` (`pending` / `approved`)
- `holdUntil`
- `releasedAt`
- `createdAt`, `updatedAt`

### `bundleSales/{saleId}`

Campos observados:

- `buyerEmail`
- `amountPen`
- `amountUsd`
- `referralCode`
- `referrerId`
- `status`
- `source`
- `holdUntil`
- `createdAt`

### `culqiOrders/{orderId}`

Campos observados:

- `uid`
- `plan`
- `paymentMethod`
- `amountPen`
- `amount`
- `status`
- `rawEventType`
- `createdAt`, `updatedAt`

## Endpoints del backend

Servicio: `afiliados-pro-hub-backend/src/index.js`.

### Auth y bootstrap

#### `GET /health`

- `200`: `{ ok: true }`

#### `GET /referrals/validate?code=...`

- Si codigo vacio/no valido/no existe: `200` `{ valid: false }`
- Si existe: `200` `{ valid: true, referrerId, referrerName }`

#### `POST /users/bootstrap` (Bearer Firebase requerido)

Body:

- `fullName?`
- `referrerCode?`

Respuestas:

- `200`: `{ user: SerializedUser }`
- `400`: `{ error: "Invalid payload" }`
- `401`: `{ error: "Missing token" | "Invalid token" }`

#### `GET /me` (Bearer Firebase requerido)

- `200`: `{ user: SerializedUser }`
- `404`: `{ error: "User not found" }`
- `401`: auth error

#### `PATCH /me` (Bearer Firebase requerido)

Body:

- `fullName?`
- `email?` (email valido)

Respuestas:

- `200`: `{ user: SerializedUser }`
- `400`: `{ error: "Invalid payload" }`
- `401`: auth error

`SerializedUser` observado:

- `uid`, `email`, `fullName`, `plan`, `status`, `referralCode`, `referredBy`, `disabled`, `createdAt`

### Dashboard y red

#### `GET /dashboard` (Bearer Firebase requerido)

- `200`:
  - `stats`: arreglo de 4 cards (`Ganancias Totales`, `Saldo Disponible`, `Plan Actual`, `Red Total`) con `{ title, value, change, variant }`
  - `recentActivity`: eventos recientes
- `500`: `{ error: "Failed to load dashboard" }`

#### `GET /tools` (Bearer Firebase requerido)

- `200`: `{ tools: [{ id, name, description, color, minPlan, status }] }`

#### `GET /network` (Bearer Firebase requerido)

- `200`:
  - `levels`: comisiones por nivel + miembros por nivel
  - `members`: downline plano `{ id, name, plan, level, earnings }`
  - `totalPotential`, `currentPotential`
  - `upline` (o `null`)

#### `GET /subscription` (Bearer Firebase requerido)

- `200`: `{ plans, currentPlan }`

#### `POST /subscription/upgrade` (Bearer Firebase requerido)

Body:

- `plan` en `basic|pro|elite`

Respuestas:

- `200`: `{ ok: true, plan }`
- `400`: `{ error: "Invalid plan" }`

### PayPal

#### `POST /paypal/create-subscription` (Bearer Firebase requerido)

Body:

- `planCode` (`basic|pro|elite` esperado por backend)

Respuestas:

- `200`: `{ approvalUrl, subscriptionId }`
- `400`: `{ error: "Invalid plan" }`
- `500`: `{ error: "No approval link" | "Server error" }`
- Error de PayPal: mismo status de PayPal con `{ error }`

#### `POST /paypal/webhook`

Respuestas:

- `400`: `{ error: "Webhook not verified" }`
- `200`: `{ ok: true, ignored: true }` cuando no encuentra usuario
- `200`: `{ ok: true }` cuando aplica update
- `500`: `{ error: "Webhook error" | detalle }`

### Culqi

#### `POST /culqi/orders` (Bearer Firebase requerido)

Body:

- `plan`: `basic|pro|elite`
- `phone`
- `paymentMethod?`: `yape|plin`

Respuestas:

- `200`: `{ orderId, publicKey, amount, currencyCode: "PEN" }`
- `400`: `{ error: "Invalid payload" | "Invalid plan" }`
- `500`: `{ error: "Culqi not configured" | "Culqi error" }`

#### `POST /culqi/webhook`

Respuestas:

- `400`: `{ error: "Missing event type" }`
- `200`: `{ ok: true, ignored: true }` si no hay `orderId`
- `200`: `{ ok: true }`
- `500`: `{ error: "Webhook error" | detalle }`

### Ventas bundle y comisiones

#### `POST /bundle/sales`

Header requerido:

- `x-sales-key` == `SALES_API_KEY`

Body:

- `externalId?`
- `buyerEmail`
- `amountPen`
- `referralCode?`
- `source?`

Respuestas:

- `500`: `{ error: "Sales key not configured" }`
- `401`: `{ error: "Unauthorized" }`
- `400`: `{ error: "Invalid payload" }`
- `200`: `{ ok: true, saleId, status: "exists" | "no-referrer" | "recorded" }`

### Admin

#### `GET /admin/users` (Bearer Firebase + admin)

Query:

- `limit?` (max 200)
- `cursor?`

Respuestas:

- `200`: `{ users, nextCursor }`
- `403`: `{ error: "Admin access not configured" | "Forbidden" }`
- `401`: auth error

#### `PATCH /admin/users/:uid` (Bearer Firebase + admin)

Body permitido:

- `plan?`: `basic|pro|elite`
- `status?`: `TRIAL|ACTIVE|SUSPENDED`
- `disabled?`: boolean
- `fullName?`

Respuestas:

- `200`: `{ ok: true }`
- `400`: `{ error: "Invalid payload" }`
- `403`: `{ error: "Cannot modify owner account" }`

#### `DELETE /admin/users/:uid` (Bearer Firebase + admin)

Respuestas:

- `200`: `{ ok: true }`
- `403`: `{ error: "Cannot delete owner account" }`
- `500`: `{ error: "Failed to delete auth user" }`

## Formato de errores

Formatos observados:

- JSON simple: `{ error: "..." }`
- Casos de exito usualmente con `ok: true` o payload de negocio
- Webhooks devuelven `200` con `ignored: true` cuando aplica
- Endpoints autenticados devuelven `401` para token faltante/invalido
- Endpoints admin devuelven `403` cuando no cumple rol/config

## Reglas de compatibilidad hacia atras

Comportamientos actuales que el frontend consume hoy:

- `apiFetch` exige respuesta JSON y trata `!ok` como error de texto
- Flujo de registro/login depende de `POST /users/bootstrap`
- Pantallas consumen rutas estables: `/dashboard`, `/me`, `/tools`, `/network`, `/subscription`, `/paypal/create-subscription`, `/admin/users`
- `GET /referrals/validate` siempre responde JSON con `valid`
- `POST /bundle/sales` permite idempotencia por `externalId`
- Webhooks PayPal/Culqi deben seguir actualizando `users` sin romper campos existentes (`plan`, `status`, `pendingPlan`, `paypalSubscriptionId`, etc.)
