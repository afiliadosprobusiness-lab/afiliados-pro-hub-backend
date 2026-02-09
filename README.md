# Afiliados Pro Hub Backend

Backend en Node.js + Express para Afiliados Pro Hub.
Usa Firebase Admin SDK para autenticar y leer/escribir en Firestore.

## Requisitos
- Node 20+
- Firebase Project con Auth + Firestore habilitados
- Service Account JSON

## Variables de entorno
- `FIREBASE_SERVICE_ACCOUNT`: JSON del service account en una sola linea.
- `CORS_ORIGIN`: origen permitido (ej: `https://tu-dominio.vercel.app`).
- `PORT`: puerto (por defecto 8080).
- `ADMIN_EMAILS`: lista separada por comas de correos con acceso al panel admin.

## Comandos
```bash
npm install
npm run dev
```

## Deploy en Cloud Run
```bash
gcloud run deploy afiliados-pro-hub-backend --source . --region us-central1 --allow-unauthenticated
```

## Endpoints
- `GET /health`
- `POST /users/bootstrap`
- `GET /me`
- `GET /dashboard`
- `GET /tools`
- `GET /network`
- `GET /subscription`
- `POST /subscription/upgrade`
- `GET /admin/users`
- `PATCH /admin/users/:uid`
- `DELETE /admin/users/:uid`
