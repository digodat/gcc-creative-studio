# Creative Studio — Setup y Deploy (bbva-genai-veo)

Registro de todo lo realizado para levantar el entorno local y deployar en GCP el proyecto `bbva-genai-veo`.

---

## 1. Entorno local

### Prerequisitos verificados
- Docker Desktop 29.2.1
- Docker Compose v5.1.0
- Google Cloud SDK 570.0.0
- Node.js v24.14.0

### Configuracion de GCP
```bash
gcloud config set project bbva-genai-veo
gcloud auth application-default set-quota-project bbva-genai-veo
```

### APIs habilitadas
```bash
gcloud services enable \
  firebase.googleapis.com identitytoolkit.googleapis.com \
  storage.googleapis.com aiplatform.googleapis.com \
  texttospeech.googleapis.com workflows.googleapis.com \
  cloudtasks.googleapis.com speech.googleapis.com \
  discoveryengine.googleapis.com iam.googleapis.com iap.googleapis.com
```

### Recursos GCP creados manualmente
- **GCS Bucket**: `bbva-genai-veo-cs-bucket` (us-central1)
- **Service Account**: `cs-signing-sa@bbva-genai-veo.iam.gserviceaccount.com`
  - Rol `roles/storage.objectViewer` en el bucket
  - Rol `roles/iam.serviceAccountTokenCreator` para `gaston.baeza@monks.com`

### Firebase
- Proyecto Firebase vinculado a `bbva-genai-veo`
- Web app registrada: `creative-studio-local`
- Google Sign-In habilitado en Authentication
- Dominio `localhost` en Authorized Domains

### Archivos de configuracion local creados
- `frontend/src/environments/environment.development.ts` — config Firebase + `isLocal: true`
- `frontend/proxy.conf.json` — proxy `/api` → `http://backend:8080`
- `backend/.env` — variables de entorno para Docker Compose local

### Levantar el entorno local
```bash
docker compose up
```

### Seed inicial (primera vez)
```bash
docker exec -w /app creative-studio-backend bash -c "PYTHONPATH=/app /app/.venv/bin/python -m bootstrap.bootstrap"
```

---

## 2. Cambios realizados en el frontend

### Selector de duracion de video
- Archivo: `frontend/src/app/video/video.component.html` — descomentado el selector
- Archivo: `frontend/src/app/video/video.component.ts` — opciones: `[4, 6, 8]` segundos
- Valores validos segun Vertex AI: 4, 6 u 8 segundos

### Selector de resolucion de video
- Modelo actualizado: `frontend/src/app/common/models/search.model.ts` — campo `resolution?: '720p' | '1080p' | '4k'`
- Estado: `frontend/src/app/services/video-state.service.ts` — campo `resolution` agregado
- Componente: `frontend/src/app/video/video.component.ts` — `resolutionOptions`, `selectResolution()`
- HTML: `frontend/src/app/video/video.component.html` — selector con icono `hd`
- Backend DTO: `backend/src/videos/dto/create_veo_dto.py` — campo `resolution` con validacion
- Backend service: `backend/src/videos/veo_service.py` — `resolution` pasado a `GenerateVideosConfig`

### Modelos Veo deprecados eliminados
Archivo: `frontend/src/app/common/config/model-config.ts`

Eliminados:
- `veo-3.0-generate-001`
- `veo-3.0-fast-generate-001`
- `veo-2.0-generate-001`
- `veo-2.0-fast-generate-001`
- `veo-2.0-generate-exp`

Quedan activos (Vertex AI):
- `veo-3.1-generate-001`
- `veo-3.1-lite-generate-001`
- `veo-3.1-fast-generate-001`

---

## 3. Deploy en GCP

### Infraestructura creada via bootstrap.sh
```bash
chmod +x bootstrap.sh && ./bootstrap.sh
```

Terraform crea automaticamente:
- Cloud Run service: `cstudio-be` (us-central1)
- Cloud SQL: `creative-studio-db-b1bd2a7e` (PostgreSQL)
- Artifact Registry para imagenes Docker
- Firebase Hosting site: `bbva-genai-veo`
- Secret Manager con todos los secrets
- Cloud Build triggers para backend y frontend

### Cloud Build triggers
- **Backend** (`cstudio-be-trigger`): dispara con push a `main` en `backend/**`
- **Frontend** (`bbva-genai-veo-trigger`): dispara con push a `main` en `frontend/**`

> La `_BACKEND_URL` del trigger de frontend fue corregida a:
> `https://cstudio-be-921427644872.us-central1.run.app`

### URLs de produccion
- **Frontend**: https://bbva-genai-veo.web.app
- **Backend**: https://cstudio-be-921427644872.us-central1.run.app

---

## 4. Configuracion post-deploy

### Permitir invocaciones no autenticadas en Cloud Run
```bash
gcloud run services add-iam-policy-binding cstudio-be \
  --region=us-central1 --project=bbva-genai-veo \
  --member="allUsers" --role="roles/run.invoker"
```

### OAuth Client ID
- Creado en: https://console.cloud.google.com/apis/credentials?project=bbva-genai-veo
- Nombre: `Creative Studio`
- Authorized JavaScript origins: `http://localhost:4200`, `https://bbva-genai-veo.web.app`, `https://bbva-genai-veo.firebaseapp.com`

### Firebase Authorized Domains
Agregar en Firebase Console → Authentication → Authorized Domains:
- `bbva-genai-veo.web.app`

> **PENDIENTE al hacer deploy en dominio final**: agregar ese dominio tambien en Firebase Authorized Domains y en el OAuth Client ID.

### Restriccion de acceso por dominio
```bash
gcloud run services update cstudio-be \
  --region=us-central1 --project=bbva-genai-veo \
  --update-env-vars="^@^IDENTITY_PLATFORM_ALLOWED_ORGS=monks.com,bbva.com"
```

---

## 5. Seed de datos en produccion

### Cloud SQL Auth Proxy
```bash
./cloud-sql-proxy --port 5433 bbva-genai-veo:us-central1:creative-studio-db-b1bd2a7e &
```

### Ejecutar bootstrap contra Cloud SQL
Modificar temporalmente `backend/.env` con:
```
DB_HOST="localhost"
DB_PORT="5433"
DB_PASS="<ver Secret Manager: creative-studio-db-password>"
USE_CLOUD_SQL_AUTH_PROXY=true
```

```bash
cd backend && uv run python -m bootstrap.bootstrap
```

Restaurar el `.env` original despues.

### Obtener password de DB
```bash
gcloud secrets versions access latest --secret="creative-studio-db-password" --project=bbva-genai-veo
```

> Nota: el `%` al final del output es el indicador de zsh, no parte del password.

---

## 6. Optimizacion de costos

### Downgrade de Cloud SQL
De `db-perf-optimized-N-2` Enterprise Plus (~$350/mes) a `db-g1-small` Enterprise (~$23/mes):

```bash
gcloud sql instances patch creative-studio-db-b1bd2a7e \
  --project=bbva-genai-veo \
  --tier=db-g1-small \
  --edition=ENTERPRISE
```

### Pausar/reactivar Cloud SQL cuando no se usa
```bash
# Pausar
gcloud sql instances patch creative-studio-db-b1bd2a7e \
  --activation-policy=NEVER --project=bbva-genai-veo

# Reactivar
gcloud sql instances patch creative-studio-db-b1bd2a7e \
  --activation-policy=ALWAYS --project=bbva-genai-veo
```

---

## 7. Referencia rapida

| Recurso | Valor |
|---------|-------|
| Proyecto GCP | `bbva-genai-veo` |
| Region | `us-central1` |
| Frontend URL | https://bbva-genai-veo.web.app |
| Backend URL | https://cstudio-be-921427644872.us-central1.run.app |
| Cloud SQL | `bbva-genai-veo:us-central1:creative-studio-db-b1bd2a7e` |
| GCS Bucket (Terraform) | `bbva-genai-veo-cs-development-bucket` |
| Admin email | `gaston.baeza@monks.com` |
| Dominios permitidos | `monks.com`, `bbva.com` |
