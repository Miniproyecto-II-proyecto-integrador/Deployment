# Despliegue del Sprint 3

## Arquitectura

- `frontend` es una SPA de Vite/React. Usa Firebase Auth para obtener el ID token
  y consume la API REST. El cliente Socket.IO usará el mismo token cuando FE Lead
  complete esa integración. Los datos de la aplicación pasan por los backends.
- `backend-api` gestiona perfiles, salas e historial paginado. Publica Swagger en
  `/docs/`. Es un servicio Docker en Render.
- `backend-realtime` gestiona Socket.IO, presencia y escritura de mensajes. Usa
  Node.js 22, se ejecuta como usuario `node` y se despliega en Render. Los
  dos backends acceden al mismo proyecto Firebase mediante Firebase Admin.

Compose se usa para ejecutar los tres repositorios hermanos en local. El sitio
de producción se compila en Vercel y se comunica por HTTPS con los backends de
Render. Docker Compose no determina la configuración de esos despliegues.

## Variables de entorno

| Servicio | Variable | Uso y alcance |
| --- | --- | --- |
| API y realtime | `FIREBASE_SERVICE_ACCOUNT_JSON` | Secreto de ejecución. JSON completo de una cuenta de servicio del proyecto destino. Realtime lo necesita al iniciar. La API lo necesita al acceder a Firebase. |
| API con US-11 | `CURSOR_SECRET` | Secreto de ejecución. Clave dedicada de 32 bytes en hexadecimal (64 caracteres) para firmar cursores. Historial responde `503 HISTORIAL_NO_DISPONIBLE` si falta o es inválida. |
| API y realtime | `CORS_ORIGINS` | Lista de orígenes permitidos, separados por comas y sin barra final. Se configura en ejecución. |
| API y realtime | `PORT` | Puerto HTTP. Compose fija `3000` y `3001` respectivamente. En Render se usa el puerto de la plataforma. |
| API | `FIRESTORE_COLLECTION`, `FIRESTORE_DOCUMENT`, `FIRESTORE_TIMEOUT_MS` | Opcionales en ejecución. Defaults `health-data`, `main` y `5000` ms para el diagnóstico de Firestore. |
| API y realtime | `NODE_ENV` | Las imágenes finales fijan `production`. |
| Pruebas locales | `FIRESTORE_EMULATOR_HOST` | Opcional. Host y puerto del emulador. Mantener fuera de producción. |
| Frontend | `VITE_API_URL` | URL pública base de la API, sin `/api` al final. Vite la incorpora al compilar. |
| Frontend | `VITE_REALTIME_URL` | URL pública base de realtime. Incorporarla cuando el cliente Socket.IO la consuma. |
| Frontend | `VITE_FIREBASE_API_KEY`, `VITE_FIREBASE_AUTH_DOMAIN`, `VITE_FIREBASE_PROJECT_ID`, `VITE_FIREBASE_APP_ID` | Configuración pública de Firebase Web. El cliente actual necesita las cuatro para inicializar Auth. Vite las incorpora al compilar. |

Guarda los secretos de desarrollo en el `.env` ignorado de cada backend. Usa
solo credenciales de un proyecto aislado para pruebas integradas contra Firebase.
En Render configura los secretos directamente en **Environment** del servicio
que los necesita. Evita ponerlos en argumentos de build, en variables `VITE_*`,
en el repositorio o en capturas de logs. La configuración web de Firebase es
pública y debe corresponder al proyecto de los backends.

Para US-11, genera una clave independiente en un entorno autorizado.

```bash
openssl rand -hex 32
```

Guarda el resultado como `CURSOR_SECRET` en **Environment** de la API y programa
su activación con el despliegue. Si preparas la variable antes de publicar,
usa **Save only** para mantener el orden del despliegue. Las instancias de API
del mismo entorno deben compartir el valor. Genera uno distinto para otros
entornos. Un cambio invalida
los cursores emitidos con la clave anterior. Después de una rotación, el cliente
puede volver a pedir la primera página si recibe un `400` por un cursor anterior.
[Variables y secretos en Render](https://render.com/docs/configure-environment-variables)

## Ejecución local con Compose

Usa Docker Engine con el complemento Docker Compose.

Desde la raíz que contiene los cuatro repositorios, prepara los `.env` locales
a partir de sus ejemplos. Los `env_file` de los backends son obligatorios en
el Compose actual. Configura `CORS_ORIGINS=http://localhost:5173` en ambos. Si
abres el frontend por otro origen, incluye ese origen de forma explícita.

| Servicio | Contexto relativo a `Deployment/` | Puerto publicado | Puerto interno |
| --- | --- | --- | --- |
| API | `../backend-api` | `3000` | `3000` |
| Realtime | `../backend-realtime` | `3001` | `3001` |
| Frontend | `../frontend` | `5173` | `80` |

El nombre de servicio de la red predeterminada de Compose sirve dentro de la red
de contenedores. El navegador usa los puertos publicados en el host. Compose
fija `PORT` para los backends de modo que coincida con el puerto interno, incluso
si un `.env` declara otro valor. Esto solo aplica al Compose de Deployment.

```bash
# Revisar el modelo de Compose sin resolver los env_file de los backends
docker compose --env-file /dev/null -f Deployment/docker-compose.yml config --no-env-resolution --quiet

# Validar los archivos locales y arrancar una vez configurado el entorno aislado
docker compose -f Deployment/docker-compose.yml config --quiet
docker compose -f Deployment/docker-compose.yml up --build
```

En local, el navegador abre <http://localhost:5173>, la API responde en
<http://localhost:3000/health> y Swagger está en <http://localhost:3000/docs/>.
Realtime responde en <http://localhost:3001/health>. Los Compose actuales no
declaran un `healthcheck`. Para parar los servicios usa
`docker compose -f Deployment/docker-compose.yml down`.

Con el Dockerfile actual, Vite puede leer un `frontend/.env` porque `COPY . .`
lo incorpora a la etapa de construcción. Si se utiliza ese flujo antes de la
corrección coordinada, el archivo debe contener únicamente configuración pública
y el contexto debe estar libre de archivos sensibles. En local la URL de API es
`http://localhost:3000`, aunque el `.env.example` del frontend aún indique
`8000`. La compilación con valores Vite ausentes no valida el inicio de sesión,
las llamadas REST ni el chat de navegador.

## Propuesta Docker para FE Lead

Aplicar estos pasos en coordinación con FE Lead. El Compose de este cambio
conserva el build actual del frontend.

1. En `frontend/.dockerignore`, excluir archivos locales y datos ajenos al build.

   ```text
   .git
   node_modules
   dist
   coverage
   .vercel
   .env
   .env.*
   *.local
   *.log
   **/*.pem
   **/*.key
   **/*service-account*.json
   ```

2. Declarar solo argumentos públicos en la etapa Vite del Dockerfile, antes de
   `RUN npm run build`. Cambiar `VITE_API_URL` a `http://localhost:3000` en
   `.env.example`. Añadir `VITE_REALTIME_URL=http://localhost:3001` cuando la
   consuma el cliente Socket.IO.

   ```dockerfile
   ARG VITE_API_URL
   ARG VITE_REALTIME_URL
   ARG VITE_FIREBASE_API_KEY
   ARG VITE_FIREBASE_AUTH_DOMAIN
   ARG VITE_FIREBASE_PROJECT_ID
   ARG VITE_FIREBASE_APP_ID
   ```

3. Usar los argumentos que ya reconozca el Dockerfile en `services.frontend.build.args`
   del Compose. Si falta la integración Socket.IO, la URL queda reservada para
   ese cambio. El `env_file` de un contenedor solo aplica en ejecución.

   ```yaml
   args:
     VITE_API_URL: ${VITE_API_URL:-http://localhost:3000}
     VITE_REALTIME_URL: ${VITE_REALTIME_URL:-http://localhost:3001}
     VITE_FIREBASE_API_KEY: ${VITE_FIREBASE_API_KEY:-}
     VITE_FIREBASE_AUTH_DOMAIN: ${VITE_FIREBASE_AUTH_DOMAIN:-}
     VITE_FIREBASE_PROJECT_ID: ${VITE_FIREBASE_PROJECT_ID:-}
     VITE_FIREBASE_APP_ID: ${VITE_FIREBASE_APP_ID:-}
   ```

4. Validar ambos PRs juntos con configuración pública de un proyecto de prueba.
   Hacer un build limpio y probar Firebase Auth, rutas SPA y la URL REST en
   el navegador. Cuando el cliente esté integrado, comprobar también el destino
   Socket.IO, el handshake, el reingreso y la lectura del historial. Confirmar
   que los archivos de entorno y las credenciales no entran al contexto ni al
   artefacto. Alinear la integración de ambos PRs para que el cambio de
   `.dockerignore` y los argumentos de Compose estén disponibles en conjunto.
   `docker compose --env-file frontend/.env -f Deployment/docker-compose.yml build frontend`
   leerá los valores públicos para interpolar los argumentos. Vercel obtiene
   sus valores del entorno de compilación de su propio proyecto.

## Orden de integración y despliegue

1. US-08 ya está integrada en `main` de realtime y US-11 en `main` de API.
   El PR de US-10 se fusionó en `feature/US-08-unirse-sala` después de integrar
   US-08 a `main`. Abre un PR nuevo desde `feature/US-10-chat-reconexion` hacia
   `main` y verifica que el diff incluya solo US-10. La recuperación completa
   desde el navegador requiere US-10 y US-11 desplegadas. Coordina los
   autodespliegues para respetar el orden indicado a continuación.
2. En Render, verifica que cada backend sea un **Web Service** con runtime
   **Docker** y ruta de salud `/health`. Si cada servicio usa su propio
   repositorio, el Dockerfile está en su raíz. Configura `CORS_ORIGINS` con el
   origen exacto del sitio Vercel, por ejemplo `https://<frontend>.vercel.app`.
   En **Compute** verifica una instancia de realtime y autoscaling desactivado.
   La presencia y la distribución de eventos son locales al proceso.
3. Despliega primero realtime con soporte de `closingAt`, antes de activar el
   borrado de US-07 en API. Esto también cubre las salas que se cierran al
   eliminar una cuenta. Anota los IDs de las instancias antiguas, confirma el
   commit nuevo en **Events** y contrasta el apagado de cada ID en los logs
   tras el plazo de apagado configurado. Usa una conexión de prueba abierta
   antes del cambio para comprobar que se corta y puede reconectar a la
   versión nueva. Un `200` o una conexión nueva solo prueban que alguna
   instancia responde. Si no puedes comprobar el apagado de todas las
   instancias antiguas, conserva la revisión anterior de API.
4. Activa la API cuando se cumpla el paso anterior. Confirma que la
   configuración de `CURSOR_SECRET` esté lista para US-11. Comprueba los
   índices y ejecuta los diagnósticos de la sección siguiente.
5. En Vercel, configura `VITE_API_URL=https://<api>.onrender.com`,
   `VITE_REALTIME_URL=https://<realtime>.onrender.com` cuando se use y los
   cuatro valores públicos de Firebase Web. Confirma que el dominio Vercel
   está autorizado en Firebase Auth. Un cambio de `VITE_*` requiere un build
   nuevo. Verifica en el navegador que las peticiones vayan a los servicios
   HTTPS del entorno correspondiente. [Variables en Vercel](https://vercel.com/docs/environment-variables)

[Despliegues de Render](https://render.com/docs/deploys#zero-downtime-deploys) · [WebSockets en Render](https://render.com/docs/websocket) · [Escalado](https://render.com/docs/scaling)

### Rollback

Conserva una versión de realtime que respete `closingAt` mientras la API pueda
iniciar cierres o queden salas en cierre. Si necesitas volver a una versión
incompatible de API o realtime, desactiva primero el tráfico de borrado de
salas y cuentas, espera las operaciones en curso y comprueba si quedan salas
marcadas. Completa la limpieza con versiones compatibles antes de retirar
ese soporte. Mantén los marcadores de cierres pendientes para poder reintentar.
En los cambios de US-11 conserva la clave del mismo entorno en los rollbacks que
incluyan historial. [Runbook de rollback de API](https://github.com/Miniproyecto-II-proyecto-integrador/backend-api/blob/main/README.md#despliegue-en-render-docker)

## Diagnóstico y aceptación

1. Comprueba `GET /health` de ambos backends. Un `200` confirma que el proceso
   atiende HTTP. Para verificar Auth, permisos y Firestore, usa una sesión
   de prueba y consulta una sala activa. En US-11 lee al menos dos páginas
   con `nextCursor`. Comprueba que el índice descendente de `createdAt` con
   alcance de colección esté habilitado para `messages`. Firestore usa
   `__name__` en la misma dirección como desempate predeterminado.
   [Índices de Firestore](https://firebase.google.com/docs/firestore/query-data/index-overview#default_ordering_and_the_name_field)
2. Abre `/docs/` en API y confirma las operaciones de edición y borrado,
   `GET /api/rooms/closing` e historial `GET /api/rooms/{id}/messages`
   cuando US-11 esté activa.
3. Desde el origen de Vercel, comprueba el preflight REST con `Authorization` y
   `Content-Type`. Comprueba CORS en polling Socket.IO si lo usa el cliente,
   y verifica por separado el handshake autenticado, el upgrade WebSocket
   y `room:joined`. El navegador aplica CORS al polling HTTP. El transporte
   WebSocket requiere su propia comprobación de sesión en Socket.IO.
   [CORS de Socket.IO](https://socket.io/docs/v4/handling-cors)

### Checklist del Sprint 3

- [ ] **BE Lead y QA**. El anfitrión puede editar y eliminar una sala de prueba.
  El invitado recibe `403` y la sala queda intacta tras su intento.
- [ ] **BE Lead, FE Lead y QA**. El borrado de la sala desechable emite
  `room:deleted` solo a sus miembros. Libera su presencia y el reingreso
  devuelve `ROOM_NOT_FOUND`. Una segunda sala sigue operativa.
- [ ] **FE Lead y QA**. Un invitado autenticado entra por ID y ve presencia.
  ID inválido, sala inexistente y sesión inválida producen errores controlados.
- [ ] **BE Lead y QA**. Un mensaje se guarda y llega solo a miembros de su sala.
  El mismo ID aparece en el historial. Un socket sin membresía o con sesión
  inválida no puede escribir.
- [ ] **BE Lead y QA**. El historial devuelve varias páginas, conserva el
  orden y termina con `hasMore: false`. Cursor inválido responde `400`,
  petición sin sesión `401` y sala inexistente o en cierre `404`.
- [ ] **FE Lead y QA**. Tras desconectar un cliente y enviar desde otro, el
  primero reconecta, reingresa y recupera el hueco, combinando historial y
  eventos por `(roomId, id)`. La interfaz detiene el reingreso si la sala se cerró.
- [ ] **Responsable de despliegue y BE Lead**. Registrar commits activos,
  apagado de instancias antiguas, una instancia estable de realtime, URLs,
  permisos e índices verificados y ausencia de errores inesperados en logs.

### Evidencias

- [ ] Guardar SHAs y PRs integrados, versiones de Docker y Compose, salida de
  validación, resultados de build y pruebas, usuario y versión de Node
  verificados en la imagen de realtime.
- [ ] Registrar commit por servicio, comprobación del drenaje, número de
  instancias, `/health`, Swagger, preflight CORS y handshake Socket.IO.
- [ ] Guardar resultados de aceptación de US-07, US-08, US-10 y US-11 con fecha,
  entorno y responsable. Usar salas desechables y cuentas de prueba autorizadas
  para los casos destructivos. Ocultar tokens, cookies, cuentas de servicio
  y `CURSOR_SECRET`.
