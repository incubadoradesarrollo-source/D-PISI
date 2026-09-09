# Despliegue del portal de Univalle

Configuración de despliegue de la Carrera de Ingeniería de Sistemas de la Universidad Privada del Valle, Bolivia.

El código se incorpora mediante submódulos Git:

| Carpeta | Repositorio |
|---|---|
| `backend` | [B-PISI](https://github.com/incubadoradesarrollo-source/B-PISI) |
| `frontend` | [F-PISI](https://github.com/incubadoradesarrollo-source/F-PISI) |

Cada commit de este repositorio fija las versiones exactas de ambos componentes. Los directorios `backend` y `frontend` son referencias Git, no copias del código.

## Clonar

```bash
git clone --recurse-submodules https://github.com/incubadoradesarrollo-source/D-PISI.git
cd D-PISI
```

Si ya se clonó sin submódulos:

```bash
git submodule update --init --recursive
```

## Docker local

Copiar `.env.example` a `.env` y completar `POSTGRES_PASSWORD`, `JWT_ACCESS_SECRET` y `JWT_REFRESH_SECRET` con valores propios y distintos. Después:

```bash
docker compose config --quiet
docker compose up -d --build
docker compose ps
```

Abrir `http://localhost:3000`. La salud de la API se consulta por `http://localhost:3000/api/health`.

PostgreSQL no publica su puerto. Web y API publican sus puertos únicamente en loopback del host. Los datos se conservan en `db_data` y `api_uploads`; no usar `docker compose down -v` para actualizar.

## API, identidad y variables

El navegador utiliza `/api/...` y `/uploads/...`. Next reenvía esas rutas a `http://api:4000` dentro de Docker, utilizando `API_INTERNAL_URL` durante la ejecución. `NEXT_PUBLIC_API_URL` ya no se utiliza.

| Variable | Docker local | Producción |
|---|---|---|
| `PUBLIC_WEB_URL` | `http://localhost:3000` | `https://socesi.com` |
| `WEB_ORIGIN` | localhost y 127.0.0.1 | `https://socesi.com` |
| `TRUST_PROXY_HOPS` | 1 con Next | 1 con Nginx directo a API; 2 pasando por Next |
| `AUTH_DEV_LINKS` | true solo en demo local | false |
| `SEED_ON_FIRST_RUN` | true para inicializar una demo vacía | false |
| `ALLOW_DEMO_SEED` | false | false |

Compose deriva `PUBLIC_API_URL` y `NEXT_PUBLIC_SITE_URL` de `PUBLIC_WEB_URL`; no configurar URLs privadas como origen público. `NEXT_PUBLIC_SITE_URL` se incorpora al build. Si cambias `WEB_PORT`, ajustar también `PUBLIC_WEB_URL` y `WEB_ORIGIN`.

La administración `/admin/institucion` permite cambiar logos claros/oscuros, favicon, tamaños y paletas. La migración aditiva `20260909180000_parametric_identity` se aplica antes de iniciar la API. Se conserva la paleta azul predeterminada; el morado no está declarado como color oficial.

## Actualizar el servidor existente

1. Respaldar PostgreSQL y uploads antes de actualizar.
2. Preservar el **nombre del proyecto Compose anterior**. Ejecutar desde otra carpeta cambia el nombre predeterminado y puede crear volúmenes vacíos diferentes. Establecer `COMPOSE_PROJECT_NAME` en `.env` con el nombre que ya utiliza el servidor, o usar `docker compose -p NOMBRE_EXISTENTE` en todos los comandos. Consultar los proyectos con `docker compose ls`.
3. Configurar `PUBLIC_WEB_URL=https://socesi.com`, `WEB_ORIGIN=https://socesi.com`, desactivar demo/seed y conservar los secretos. El backend exige `AUTH_EMAIL_WEBHOOK_URL` y su secreto para el correo transaccional fuera de la demo local.
4. Adaptar `nginx/socesi.conf.example` dentro del bloque HTTPS existente. El ejemplo envía `/api/` y `/uploads/` directamente a la API y utiliza un salto confiable. Conservar los certificados TLS. Ajustar la cadena de confianza si existe CDN u otro proxy anterior.
5. Obtener las versiones fijadas y reconstruir:

```bash
git pull --ff-only
git submodule update --init --recursive
docker compose config --quiet
docker compose up -d --build
docker compose ps
docker compose logs --tail=100 api web
```

6. Validar y recargar Nginx mediante el mecanismo del servidor. Comprobar `/ranking`, login, refresh, reglas de puntos, aplicaciones, carga del logo, modo oscuro y móviles. Sin sesión, refresh puede devolver 401 legítimamente. Limpiar la caché antigua de HTML/JS si existe CDN; no cachear respuestas privadas.
7. Revisar URLs antiguas de uploads que pudieran estar guardadas con localhost y reemplazarlas desde los editores o mediante una migración de datos específica. No se reescriben automáticamente datos históricos.

## Publicar nuevas versiones de los componentes

Primero publicar los commits en B-PISI y F-PISI. Después actualizar las referencias de despliegue:

```bash
git -C backend fetch origin
git -C backend checkout COMMIT_BACKEND
git -C frontend fetch origin
git -C frontend checkout COMMIT_FRONTEND
git add backend frontend
git commit -m "Actualiza las versiones del backend y frontend"
git push origin main
```

Usar hashes concretos permite reproducir el despliegue. Para revertir, recuperar las referencias anteriores y revisar por separado la compatibilidad de migraciones/datos.

## Validación y CI

```bash
cd backend
npm ci
npm run prisma:generate
npm run check
cd ../frontend
npm ci
npm run check
npm run test:proxy
```

Las pruebas de contratos del backend necesitan el frontend hermano; esta estructura de submódulos lo proporciona. CI descarga ambos recursivamente. Si los repositorios son privados, configurar en D-PISI el secreto `SUBMODULES_TOKEN` con permiso de lectura de B-PISI y F-PISI: el token predeterminado de Actions normalmente solo permite leer el repositorio actual.

La publicación de los repositorios no ejecuta el despliegue en producción. El motor Docker y las cuentas reales del servidor deben estar disponibles para completar la validación operativa.
