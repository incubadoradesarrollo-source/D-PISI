# Despliegue del portal de Univalle

Configuración de despliegue de la Carrera de Ingeniería de Sistemas de la Universidad Privada del Valle, Bolivia.

Consulta la [guía de despliegue y correo](DESPLIEGUE.md) y el [informe de revisión final](REVISION-FINAL.md). El dominio público definitivo está pendiente de confirmación. El registro admite solo `@univalle.edu`, con activación por correo; SMTP/Gmail está soportado directamente.

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

Copiar `.env.example` a `.env` y completar `POSTGRES_PASSWORD`, `JWT_ACCESS_SECRET`, `JWT_REFRESH_SECRET` y `TOTP_ENCRYPTION_KEY` con valores propios y distintos. La clave TOTP debe generarse con `openssl rand -hex 32` y conservarse protegida para futuros reinicios/restauraciones. Después:

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
| `PUBLIC_WEB_URL` | `http://localhost:3000` | Origen HTTPS definitivo |
| `WEB_ORIGIN` | localhost y 127.0.0.1 | El mismo origen HTTPS |
| `TRUST_PROXY_HOPS` | 1 con Next | 1 con Nginx directo a API; 2 pasando por Next |
| `AUTH_DEV_LINKS` | true solo en demo local | false |
| `SEED_ON_FIRST_RUN` | true para inicializar una demo vacía | false |
| `ALLOW_DEMO_SEED` | false | false |

Compose deriva `PUBLIC_API_URL` y `NEXT_PUBLIC_SITE_URL` de `PUBLIC_WEB_URL`; no configurar URLs privadas como origen público. `NEXT_PUBLIC_SITE_URL` se incorpora al build. Si cambias `WEB_PORT`, ajustar también `PUBLIC_WEB_URL` y `WEB_ORIGIN`.

La administración `/admin/institucion` permite cambiar logos claros/oscuros, favicon, tamaños y paletas. La identidad utiliza guindo `#7B1113`, borgoña `#6B1D2F`, blanco y grises, con una variante oscura de contraste accesible. La migración de paleta actualiza la configuración guardada conservando logos y colores de estados.

Las cuentas predeterminadas conservan sus contraseñas iniciales y deben cambiarlas antes de acceder al sistema. TOTP se configura en **Mi cuenta → Seguridad**. Las migraciones se aplican automáticamente al iniciar la API. Consultar [seguridad y autenticación](backend/SEGURIDAD.md) para los flujos, códigos de recuperación y respaldo de la clave de cifrado.

## Actualizar el servidor existente

1. Respaldar PostgreSQL y uploads antes de actualizar.
2. Preservar el **nombre del proyecto Compose anterior**. Ejecutar desde otra carpeta cambia el nombre predeterminado y puede crear volúmenes vacíos diferentes. Establecer `COMPOSE_PROJECT_NAME` en `.env` con el nombre que ya utiliza el servidor, o usar `docker compose -p NOMBRE_EXISTENTE` en todos los comandos. Consultar los proyectos con `docker compose ls`.
3. Configurar `PUBLIC_WEB_URL` y `WEB_ORIGIN` con el origen HTTPS definitivo, desactivar demo/seed y conservar los secretos. Mantener `TOTP_ENCRYPTION_KEY`; no cambiarla sin recifrar los registros existentes. Configurar SMTP y su contraseña de aplicación, o el webhook de correo como alternativa. Completar el cambio inicial de las cuentas asignadas antes de abrir el portal al público. El generador `node scripts/preparar-despliegue.cjs https://tu-dominio-real` prepara `.env.production` a partir de las claves del `.env` local.
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
npm run test:auth:integration
cd ../frontend
npm ci
npm run check
npm run test:proxy
```

Las pruebas de contratos del backend necesitan el frontend hermano; esta estructura de submódulos lo proporciona. CI descarga ambos recursivamente. Si los repositorios son privados, configurar en D-PISI el secreto `SUBMODULES_TOKEN` con permiso de lectura de B-PISI y F-PISI: el token predeterminado de Actions normalmente solo permite leer el repositorio actual.

Los contenedores y CI utilizan Node 24 LTS. La prueba de autenticación inicia un PostgreSQL temporal mediante Docker, prueba las migraciones y el flujo real y elimina su contenedor al terminar.

La publicación de los repositorios no ejecuta el despliegue en producción. El motor Docker y las cuentas reales del servidor deben estar disponibles para completar la validación operativa.
