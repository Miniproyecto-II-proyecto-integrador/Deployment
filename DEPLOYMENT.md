# Despliegue y ejecución con Docker

El proyecto contiene tres servicios independientes. El archivo
`Deployment/docker-compose.yml` permite ejecutarlos juntos localmente; cada
backend conserva su propio Dockerfile y sus propias variables de entorno.

## Requisitos

- Docker Engine y Docker Compose v2.
- Cuenta de servicio de Firebase con acceso al mismo proyecto de Firestore para
  ambos backends.
- Configuración Firebase Web para el frontend.

## Ejecutar los tres servicios localmente

Desde la raíz del proyecto, prepara los archivos de entorno:

```bash
cp backend-api/.env.example backend-api/.env
cp backend-realtime/.env.example backend-realtime/.env
cp frontend/.env.example frontend/.env
```

Completa los archivos localmente y no los subas al repositorio:

- `backend-api/.env`: configura `FIREBASE_SERVICE_ACCOUNT_JSON` y
  `CORS_ORIGINS=http://localhost:5173`. Mantén `PORT=3000`.
- `backend-realtime/.env`: configura el mismo `FIREBASE_SERVICE_ACCOUNT_JSON`,
  `CORS_ORIGINS=http://localhost:5173` y `PORT=3001`.
- `frontend/.env`: configura la Firebase Web App y
  `VITE_API_URL=http://localhost:3000`. El `.env.example` actual muestra
  `localhost:8000`; cámbialo a `3000`, que es el puerto publicado por Compose.
  `VITE_REALTIME_URL=http://localhost:3001` puede añadirse para la futura
  integración, pero el código actual del frontend todavía no consume esa
  variable ni implementa el cliente Socket.IO.

`FIREBASE_SERVICE_ACCOUNT_JSON` es el JSON completo de la cuenta de servicio,
en una sola línea. Nunca lo pongas en `frontend/.env`: las variables `VITE_*`
se incorporan al JavaScript que recibe el navegador. El frontend solo necesita
la configuración pública de Firebase Web; los dos backends usan la cuenta de
servicio.

Valida la configuración y levanta los servicios:

```bash
docker compose -f Deployment/docker-compose.yml config -q
docker compose -f Deployment/docker-compose.yml up --build
```

Abre <http://localhost:5173>. La API está en <http://localhost:3000>, su
Swagger en <http://localhost:3000/docs> y su health check en
<http://localhost:3000/health>. El health check de realtime está en
<http://localhost:3001/health>.

El navegador usa los puertos publicados en `localhost`, no los nombres DNS
internos de Compose. Para detener los servicios:

```bash
docker compose -f Deployment/docker-compose.yml down
```

## Desplegar `backend-api` en Render

1. Crea un **Web Service** con runtime **Docker** conectado al repositorio del
   backend. Si Render usa el repositorio que contiene los tres servicios,
   configura **Root Directory** como `backend-api`; si el repositorio ya es el
   del backend, deja la raíz predeterminada.
2. Usa el Dockerfile existente del servicio y configura `/health` como health
   check path.
3. En **Environment**, configura:
   - `FIREBASE_SERVICE_ACCOUNT_JSON`: JSON completo y secreto de la cuenta de
     servicio.
   - `CORS_ORIGINS`: origen exacto del frontend, por ejemplo
     `https://frontend-liart-eight-39.vercel.app` (sin `/` final).
   - `FIRESTORE_COLLECTION`, `FIRESTORE_DOCUMENT` y `FIRESTORE_TIMEOUT_MS`
     solo si necesitas valores distintos de los defaults.
4. No agregues `PORT` manualmente; Render asigna ese valor al servicio.
5. Después del despliegue, comprueba `https://<api>.onrender.com/health` y abre
   `https://<api>.onrender.com/docs` para Swagger.

## Desplegar `backend-realtime` en Render

1. Crea otro **Web Service** con runtime **Docker**. Si usas el repositorio con
   los tres servicios, configura **Root Directory** como `backend-realtime`; si
   el repositorio es solo de realtime, deja la raíz predeterminada.
2. Configura `/health` como health check path.
3. En **Environment**, configura:
   - `FIREBASE_SERVICE_ACCOUNT_JSON`: la misma cuenta de servicio del proyecto
     Firebase, guardada como secreto.
   - `CORS_ORIGINS`: origen exacto del frontend, por ejemplo
     `https://frontend-liart-eight-39.vercel.app` (sin `/` final).
4. Render proporciona `PORT`; no lo fijes manualmente.
5. Comprueba `https://<realtime>.onrender.com/health`.

No se declara ningún servicio como desplegado hasta que el despliegue se ejecute
y se verifique en Render. La presencia y las salas Socket.IO actuales viven en
memoria; mantener una sola instancia evita que clientes conectados a instancias
distintas queden aislados. Para escalar realtime a varias instancias se
necesitará un adaptador compartido, por ejemplo Redis.

## Configurar el frontend desplegado

El frontend actual se despliega en Vercel. En las variables de entorno de
producción de Vercel configura:

- `VITE_API_URL=https://<api>.onrender.com`
- Las variables públicas `VITE_FIREBASE_API_KEY`,
  `VITE_FIREBASE_AUTH_DOMAIN`, `VITE_FIREBASE_PROJECT_ID` y `VITE_FIREBASE_APP_ID`.
- `VITE_REALTIME_URL=https://<realtime>.onrender.com` cuando el frontend
  integre el cliente Socket.IO. Actualmente esta variable no es consumida por
  el código del frontend.

Vite incorpora `VITE_*` en el bundle durante la compilación, por lo que un
cambio de URL requiere reconstruir y redesplegar el frontend. En Render, ambos
backends deben permitir en `CORS_ORIGINS` el origen exacto del frontend
desplegado, por ejemplo `https://frontend-liart-eight-39.vercel.app`.

## Credenciales y archivos de entorno

Los archivos `.env` de cada servicio son locales y están excluidos por sus
`.gitignore`. No añadirlos al repositorio, a una imagen ni a una variable
`VITE_*`. En Render, configurar los secretos directamente en la sección
**Environment** del servicio correspondiente. Esta guía describe el despliegue;
no implica que se haya realizado un despliegue en Render.
