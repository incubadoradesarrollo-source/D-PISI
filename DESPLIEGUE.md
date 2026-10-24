# Preparación del despliegue

La aplicación se ejecuta con Docker Compose: PostgreSQL, API y web. En el repositorio D-PISI, `backend` y `frontend` son submódulos de B-PISI y F-PISI. Actualizar siempre ambos a los commits indicados por D-PISI.

## Configuración privada

El `.env` local contiene las claves de base de datos, JWT, cifrado TOTP y correo. No se publica en Git ni se incluye en las imágenes Docker. Conservarlo en un canal privado y restringir su acceso en el servidor. No generar otra clave TOTP al trasladar una base con 2FA activo.

El correo usa `siciunivalle@gmail.com` como remitente técnico. Esto es independiente de la política de registro: las cuentas nuevas del portal deben usar `univalle.edu` o sus subdominios institucionales (por ejemplo, `est.univalle.edu`) y verificar su correo. Las cuentas anteriores conservan sus direcciones; la recuperación llega al correo registrado de cada usuario. Una dirección ficticia del seed no puede recibir mensajes reales.

La URL pública confirmada es **https://www.isilp.com**. Usar ese origen en `PUBLIC_WEB_URL` y `WEB_ORIGIN` de producción. El `.env` local conserva localhost para las pruebas en este equipo.

Desde la raíz y con Node 24:

```bash
node scripts/preparar-despliegue.cjs https://www.isilp.com
```

El comando genera `.env.production` conservando los secretos existentes y configurando la URL pública, CORS, correo real, Swagger desactivado y seed desactivado. No modifica el `.env` local. Está pensado para Nginx con `/api/` y `/uploads/` dirigidos directamente al backend (`TRUST_PROXY_HOPS=1`). Si se añade otra capa de proxy, revisar esa cantidad y las cabeceras confiables. Detrás de Cloudflare (producción actual en Coolify) configurar `CLIENT_IP_HEADER=cf-connecting-ip` en la API; sin ella la API ve la IP del proxy interno y todos los usuarios comparten el límite de peticiones (error «Too Many Requests» al registrarse).

## Servidor

1. Clonar D-PISI con sus submódulos:

   ```bash
   git clone --recurse-submodules https://github.com/incubadoradesarrollo-source/D-PISI.git
   cd D-PISI
   ```

2. En un servidor nuevo, transferir `.env.production` de forma privada y guardarlo como `.env` en esa carpeta. En Linux: `chmod 600 .env`. En el servidor ya publicado, respaldar su configuración y conservar sus claves de base de datos, JWT y TOTP; incorporar únicamente la URL, CORS, SMTP y opciones necesarias. El archivo generado parte de las claves locales, que no necesariamente coinciden con las del servidor.
3. Respaldar/restaurar la base y los archivos de `uploads` si se trasladan los datos actuales. El seed queda desactivado y una base nueva no tendrá las cuentas anteriores. No ejecutar el seed de demostración sobre datos que deban conservarse.
   Si se actualiza un servidor existente, conservar su nombre de proyecto Compose mediante `COMPOSE_PROJECT_NAME` o `docker compose -p NOMBRE_EXISTENTE`; cambiar de carpeta puede seleccionar volúmenes vacíos. Consultar el nombre actual con `docker compose ls`.
4. Configurar HTTPS en Nginx. Usar el ejemplo `nginx/socesi.conf.example` de D-PISI dentro del servidor HTTPS correspondiente al dominio real. Los puertos de Compose se publican solo en loopback.
5. Iniciar:

   ```bash
   docker compose up -d --build --wait --wait-timeout 240
   docker compose exec api npm run mail:verify
   docker compose ps
   ```

6. Comprobar `https://www.isilp.com/api/health`, el login, un registro institucional y la recuperación de una cuenta con correo real. Los enlaces de verificación y recuperación solo se utilizan una vez. Revisar también spam.

Las migraciones se aplican al iniciar la API. La actualización de integridad de sesiones cierra sesiones antiguas y exige volver a iniciar sesión; no cambia contraseñas ni secretos TOTP.

Para una base de producción nueva, el backend incluye `npm run seed:deploy`: crea únicamente roles, reglas, insignias y la cuenta administrativa. No carga proyectos ni noticias de demostración y conserva las reglas existentes al repetirse. Proporcionar `ADMIN_SEED_EMAIL`, `ADMIN_SEED_USERNAME` y `ADMIN_SEED_PASSWORD` mediante variables privadas del proceso que ejecuta el comando. La cuenta inicial exige cambiar su contraseña. `ADMIN_SEED_RESET_PASSWORD=true` solicita expresamente restablecerla y revoca las sesiones anteriores; no usar esa opción para un reinicio ordinario.

## Variante con Coolify

Los repositorios B-PISI y F-PISI conservan sus workflows de Coolify. Cada push ejecuta comprobaciones; publicar imágenes y llamar al webhook requiere ejecutar manualmente **Actions → Run workflow**. Esto permite completar las variables y la URL antes de activar el servidor.

Configurar las variables privadas de la API en Coolify desde el `.env`, incluidas `PUBLIC_WEB_URL` y las variables SMTP. La variable de repositorio `PUBLIC_WEB_URL` de F-PISI quedó configurada como `https://www.isilp.com`. Esto se incorpora al construir la siguiente imagen; no actualiza por sí solo el sitio publicado. Los secretos de Actions `COOLIFY_WEBHOOK` y `COOLIFY_TOKEN` permanecen en GitHub; no se copian al código ni al `.env` del frontend. Revisar la ruta interna `API_INTERNAL_URL` del frontend de acuerdo con la red del servidor.

## Acceso de CI a los submódulos privados

B-PISI es privado. El CI de D-PISI necesita el secreto de Actions `SUBMODULES_TOKEN`; el `GITHUB_TOKEN` automático solo tiene acceso al repositorio del workflow. Un responsable autorizado debe crear un token de acceso detallado con **Contents: Read-only**, seleccionar D-PISI, B-PISI y F-PISI y, si corresponde, obtener aprobación de la organización. Guardarlo en **D-PISI → Settings → Secrets and variables → Actions → New repository secret** con el nombre `SUBMODULES_TOKEN`.

No pegar el token en el código, en `.env` ni en el chat. Después de configurarlo, ejecutar **Actions → CI → Run workflow**. La sesión utilizada para esta revisión dispone de escritura de código en B-PISI, pero no de administración de sus claves; no se cambió la visibilidad del repositorio ni se copió una credencial personal amplia a Actions.

## Operación local

```bash
docker compose up -d --build --wait
docker compose logs --tail=100 api web
docker compose stop
```

Portal: `http://localhost:3000`. El correo real también funciona localmente, pero sus enlaces abren localhost y deben probarse en el mismo equipo.

`docker compose stop` conserva los datos. Para actualizaciones: respaldar primero, actualizar el repositorio y sus submódulos, y reconstruir. No usar `down -v` para una actualización: elimina los volúmenes de base de datos y archivos.

## Diagnóstico de correo tras actualizar Coolify

Actualizar primero la imagen del backend. Si Coolify utiliza GHCR, ejecutar manualmente el workflow de B-PISI y F-PISI para publicar las imágenes nuevas; un push por sí solo ejecuta las comprobaciones. Desde la terminal del contenedor de la API, ejecutar `npm run mail:verify`. No ejecutar Docker dentro del contenedor.

El diagnóstico comprueba TLS y autenticación sin enviar mensajes. `SMTP_AUTH` indica credenciales rechazadas; `SMTP_CONNECTION` indica un problema de DNS, red, puerto o TLS; `SMTP_CONFIG` indica variables ausentes o inválidas. `MAIL_DELIVERY` requiere revisar el proveedor. Si aparece «Missing script», la imagen no contiene la versión actual. Compartir solo la categoría del error, nunca secretos ni enlaces de autenticación.

Una verificación correcta no confirma recepción: completar después un registro con un buzón institucional propio, abrir el enlace y comprobar el perfil. Conservar `PUBLIC_WEB_URL=https://www.isilp.com` y los secretos existentes del servidor.

## Actualización: cursos de formación continua, Sociedad Científica y tema del sistema (2026-10)

### Alcance

1. Repositorios y ramas:
   - B-PISI: rama `feature/cursos-sociedad-cientifica`.
   - F-PISI: rama `feature/actualizacion-cursos-sociedad-tema`.
2. Migración nueva `20261024120000_cursos_sociedad_cientifica`, estrictamente aditiva:
   - crea los enums `ScientificSocietyMembershipStatus` y `ScientificSocietyRole`;
   - crea la tabla `ScientificSocietyMember`, vacía;
   - añade tres columnas nullable a `Mentorship`: `assistantId`, `introVideoId` y `whatsappUrl`, con índice y clave foránea `ON DELETE SET NULL`.
3. La migración no borra, no renombra ni actualiza filas. Las API anteriores siguen funcionando porque los campos solo se añaden.
4. Compatibilidad:
   - el backend nuevo funciona con el frontend anterior;
   - el frontend nuevo **no** funciona bien con el backend anterior, porque usa `/api/scientific-society`, `/api/communities/memberships/mine` y el parámetro `roles=` del directorio.

### Advertencia: despliegue automático

- Un push a `main` de F-PISI ejecuta el workflow que publica la imagen y llama al webhook de Coolify. El despliegue es **inmediato**.
- B-PISI solo publica al ejecutar **Actions → Run workflow**.

Por eso, **no fusionar F-PISI a `main` antes de que el backend nuevo esté en producción.**

### Orden de despliegue

Seguir estos pasos en el orden indicado.

1. Respaldo previo (obligatorio) de la base de datos y de `uploads`.

   Con Docker Compose, consultar antes el nombre del proyecto con `docker compose ls` y añadir `-p NOMBRE` a los comandos si hace falta:

   ```bash
   docker compose exec -T db sh -c 'pg_dump -U "$POSTGRES_USER" -d "$POSTGRES_DB" -Fc' > respaldo-AAAAMMDD-HHMM.dump
   ls -lh respaldo-AAAAMMDD-HHMM.dump
   pg_restore --list respaldo-AAAAMMDD-HHMM.dump | head
   docker compose cp api:/app/uploads ./uploads-respaldo-AAAAMMDD
   ```

   Verificar que el archivo no tiene tamaño 0 y que `pg_restore --list` muestra tablas como `"User"` y `"CommunityMember"`.

   En Coolify, usar el respaldo programado o manual de la base de datos, o ejecutar el mismo `pg_dump` desde la terminal del contenedor de PostgreSQL, y descargar el archivo antes de seguir.

   **No continuar si el respaldo no se completó y verificó.**

2. Ensayo opcional recomendado: restaurar el respaldo en un PostgreSQL desechable y ejecutar allí la verificación del punto 4 antes de tocar producción.

3. Capturar huellas antes de arrancar la versión nueva. Solo realiza lecturas dentro de una transacción `READ ONLY`. Con la imagen nueva apuntando a la base, ejecutar una vez, antes de arrancar la API:

   ```bash
   docker compose run --rm --entrypoint node -v "$PWD:/host" api scripts/verificar-migracion.cjs capturar /host/antes.json
   ```

   El archivo `antes.json` queda en la carpeta actual del servidor. Conservarlo hasta completar el punto 4. Con Compose, la imagen nueva debe construirse antes (`docker compose build api` con el submódulo `backend` en la rama nueva) sin arrancarla.

   En Coolify, ejecutar el script desde un checkout de B-PISI en la rama nueva con Node 24 (`npm ci`, `npx prisma generate`) y `DATABASE_URL` apuntando a la base de producción o a la copia restaurada del punto 2: `node scripts/verificar-migracion.cjs capturar antes.json`.

4. Fusionar B-PISI a `main`, ejecutar **Actions → Run workflow** y desplegar la API. Las migraciones se aplican al arrancar (`prisma migrate deploy`). Después:
   - comprobar `https://www.isilp.com/api/health`;
   - revisar los logs de arranque y confirmar que aparece la migración aplicada;
   - ejecutar la comparación con las huellas capturadas:

   ```bash
   docker compose run --rm --entrypoint node -v "$PWD:/host" api scripts/verificar-migracion.cjs comparar /host/antes.json
   ```

   La salida debe indicar **«Sin diferencias»**. Si indica otra cosa, detenerse y revisar antes de continuar.

5. Comprobar con el frontend **anterior** todavía en línea: inicio de sesión, perfil, roles y comunidades.

6. Fusionar F-PISI (`feature/actualizacion-cursos-sociedad-tema`) a `main`. Esto despliega el frontend mediante su workflow.

7. En D-PISI, actualizar los punteros de los submódulos `backend` y `frontend` a los commits fusionados y hacer commit:

   ```bash
   git -C backend checkout main && git -C backend pull
   git -C frontend checkout main && git -C frontend pull
   git add backend frontend
   git commit -m "Actualiza submódulos con cursos, Sociedad Científica y tema"
   ```

### Verificación posterior

1. Iniciar sesión con cuentas existentes.
2. Revisar perfiles, roles y puntos.
3. Revisar membresías de comunidades y contadores. `/api/communities` debe seguir mostrando 11, 12 y 9 miembros en las tres comunidades actuales, salvo altas nuevas.
4. `/sociedad-cientifica` muestra la Sociedad vacía hasta que ADMIN cargue miembros y directiva desde `/admin/sociedad`.
5. `/mentorias` redirige con 308 a `/cursos`.
6. `/comunidades?vista=sociedad` redirige a `/sociedad-cientifica`.
7. Crear un curso sin campos opcionales y otro con auxiliar, enlace de WhatsApp y video.
8. Tema: en una primera visita (sin `isi-theme` en `localStorage`) se usa el tema del dispositivo.
9. Las páginas con ISR pueden mostrar datos del build durante hasta 60 segundos tras el despliegue. Esperar antes de dar por verificado el contenido.

### Primeros pasos funcionales

1. La Sociedad Científica empieza sin miembros. No se derivó de las comunidades porque son organizaciones distintas.
2. ADMIN debe registrar la directiva y los miembros, o aprobar las solicitudes, en `/admin/sociedad`.

### Recuperación si algo falla

1. Si falla el frontend:
   - volver a desplegar la imagen anterior de F-PISI (tag `ghcr.io/incubadoradesarrollo-source/f-pisi:<sha anterior>` en Coolify), o revertir el merge en `main`, lo que vuelve a desplegar;
   - el backend nuevo es compatible con el frontend anterior, por lo que no es necesario revertirlo.
2. Si falla el backend con la migración ya aplicada:
   - antes de volver atrás el backend, volver atrás también el frontend;
   - desplegar la imagen anterior de B-PISI (`ghcr.io/incubadoradesarrollo-source/b-pisi:<sha anterior>`);
   - no hace falta revertir la migración: es aditiva y el código anterior ignora las tablas y columnas nuevas.
3. Si la migración falla a medias o hay daño de datos:
   1. detener la API;
   2. restaurar el respaldo verificado con `pg_restore --clean --if-exists -U USUARIO -d BASE respaldo.dump` dentro del contenedor de la base, o con la restauración de Coolify;
   3. restaurar `uploads` si se tocó;
   4. desplegar las imágenes anteriores;
   5. comprobar login y conteos.

   Si `prisma migrate deploy` dejó la migración marcada como fallida, tras restaurar el respaldo ya no figura. No usar `prisma migrate resolve` sin revisar antes el estado.
4. Reversión manual de solo la migración. Es el último recurso y solo si se confirma que nada usa las tablas nuevas y tras hacer un respaldo:

   ```sql
   BEGIN;
   ALTER TABLE "Mentorship" DROP CONSTRAINT IF EXISTS "Mentorship_assistantId_fkey";
   DROP INDEX IF EXISTS "Mentorship_assistantId_idx";
   ALTER TABLE "Mentorship" DROP COLUMN IF EXISTS "assistantId", DROP COLUMN IF EXISTS "introVideoId", DROP COLUMN IF EXISTS "whatsappUrl";
   DROP TABLE IF EXISTS "ScientificSocietyMember";
   DROP TYPE IF EXISTS "ScientificSocietyRole";
   DROP TYPE IF EXISTS "ScientificSocietyMembershipStatus";
   DELETE FROM "_prisma_migrations" WHERE migration_name = '20261024120000_cursos_sociedad_cientifica';
   COMMIT;
   ```

   Esta reversión elimina las membresías de la Sociedad, los auxiliares, los enlaces de WhatsApp y los videos cargados después de la actualización.

### Prohibido durante la actualización

- `docker compose down -v`.
- `prisma migrate reset`.
- `prisma db push`.
- Seeds de demostración.
- `ALLOW_DEMO_SEED=true` sobre producción.
