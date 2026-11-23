# Despliegue con Docker — Inventario FabLab UFPS

Tres contenedores: **MySQL 8.4** (datos), **API** (Node 20) y **web** (nginx: sirve
la app y reenvía `/api` a la API). Solo se publica **un puerto** (web). MySQL y la
API no son alcanzables desde fuera de la red de Docker.

```
navegador ──► web (nginx, puerto WEB_PORT) ──/api──► api (Node) ──► mysql
```

## Requisitos del servidor

- Linux con Docker Engine y el plugin `docker compose` (v2).
- ~1 GB de RAM en funcionamiento.
- Acceso a `ghcr.io` para descargar las imágenes ya compiladas (públicas, sin
  credenciales). Solo si se compila en el propio servidor (`--build`) hacen falta
  ~4 GB de RAM libres y salida a npm y al CDN de SheetJS durante la construcción.
- Un dominio y HTTPS: esta pila sirve HTTP; el certificado va en el proxy inverso de
  la universidad, delante del puerto `WEB_PORT`. Para el público, el proxy debe también
  aplicar HTTPS y filtrado de origen.
- La API publica un **healthcheck** en su Dockerfile (responde `/health`). El compose
  espera a que la API esté viva antes de arrancar nginx y ambos contenedores usan
  `no-new-privileges`.

## Instalación

Desde la carpeta `fablab-inventario/`, **antes del primer arranque**, con los dos
archivos de la carpeta `fablab-inventario` en Drive:

1. `fablab-docker.env` → copiarlo como `docker/.env` (ya trae todos los valores).
2. `fablab-aiven-2026-11-22.sql` (copia completa de producción: inventario, fotos,
   usuarios y roles) → reemplazar con él `docker/init/01-inventario.sql`.

```bash
docker compose -f docker/compose.yml --env-file docker/.env up -d
```

Descarga las imágenes publicadas (`ghcr.io/alvaro-portillo/fablab-api` y
`fablab-web`, etiqueta `estable`). Para compilarlas aquí desde el código, agregar `--build`.
La primera carga de la base tarda unos minutos y solo ocurre con el volumen vacío. Si
ya se arrancó con el archivo viejo: `docker compose -f docker/compose.yml --env-file docker/.env down -v`
(borra la base local) y volver a subir.

La clave de la cuenta principal (`laboratorio`) la entrega el equipo aparte; en
`docker/.env` solo está su hash. Para instalar sin los archivos de Drive, partir de
`docker/.env.example`, que explica cómo generar cada valor.

## Comprobación

```bash
docker compose -f docker/compose.yml --env-file docker/.env ps
```

```bash
curl -s -o /dev/null -w 'health %{http_code}\n' http://localhost:8080/health
curl -s -o /dev/null -w 'salas sin sesion %{http_code}\n' http://localhost:8080/api/salas
curl -s -X POST http://localhost:8080/api/login -H 'Content-Type: application/json' \
  -d '{"usuario":"laboratorio","clave":"<la-clave-que-elegiste>"}'
```

Deben verse los tres servicios en marcha, `/health` responder `200` y `/api/salas`
responder **`401`** (todo exige sesión). El último comando debe devolver `200` con
un `token`; con él, `curl -H "Authorization: Bearer <token>" http://localhost:8080/api/salas`
lista las 12 salas. Abrir `http://<servidor>:8080` muestra la pantalla de entrada;
al entrar con la cuenta, la app.

## Operación

| Tarea | Comando |
|---|---|
| Ver estado | `docker compose -f docker/compose.yml --env-file docker/.env ps` |
| Ver registros de la API | `docker compose -f docker/compose.yml --env-file docker/.env logs -f api` |
| Detener (conserva datos) | `docker compose -f docker/compose.yml --env-file docker/.env down` |
| Actualizar a mano | `… pull` y luego `… up -d` (o `git pull` + `up -d --build` si se compila aquí) |
| Respaldo | `sh docker/respaldo.sh` (deja `docker/respaldos/*.sql.gz`, conserva 30 días) |

**`down -v` borra la base de datos** (volumen `fablab-stack_mysql_data`). No usarlo en producción.

Respaldo diario (cron del servidor):

```
0 2 * * * cd /ruta/a/fablab-inventario && sh docker/respaldo.sh
```

La restauración está documentada al inicio de `docker/respaldo.sh`.

### Actualizaciones automáticas

El equipo publica cada versión en `ghcr.io` (workflow `.github/workflows/imagenes.yml`,
al subir un tag `v*`) y mueve la etiqueta `estable` a ella. Con este cron el servidor
la descarga sola; si no hay nada nuevo, `up -d` no reinicia ningún contenedor. La
base no se toca (el volumen persiste y la API ajusta el esquema al arrancar).

```
*/15 * * * * cd /ruta/a/fablab-inventario && docker compose -f docker/compose.yml --env-file docker/.env pull -q api web && docker compose -f docker/compose.yml --env-file docker/.env up -d
```

Para congelar una versión, fijar `APP_VERSION=v1.2.0` en `docker/.env`: el cron
sigue corriendo pero esa etiqueta no cambia. Sin el cron, el servidor se queda en la
versión con la que se instaló.

## Seguridad: lo que hay que saber

- **Sin sesión no se ve nada:** la app abre en una pantalla de entrada y la API rechaza
  (`401`) cualquier petición sin credencial, lecturas incluidas, salvo `/health` y el
  login. La web ya no lleva ningún token.
- **Cuentas:** la principal (`docker/.env`) entra con todos los permisos; las demás
  personas y sus roles se gestionan desde la app (vienen en la copia de la base). Publicar la app solo en la red del
  campus (o VPN) sigue siendo recomendable.
- **Inicio de sesión:** la app pide la cuenta del laboratorio para entrar. Su usuario, el hash de su clave y `SESION_SECRETO` van en `docker/.env` (ver `.env.example`). La web ya no lleva ningún token: el de herramientas (`API_TOKEN`) solo vive en el servidor.
- **Cambiar la clave de la cuenta:** generar otro hash (ver `.env.example`), cambiar `CUENTA_CLAVE_HASH` y reiniciar la API (`up -d api`). Las sesiones abiertas siguen válidas hasta vencer (12 h); para cerrarlas todas, cambiar también `SESION_SECRETO`.
- La API limita las escrituras por IP (120 por minuto, configurable con `LIMITE_ESCRITURAS_MIN` en
  `docker/.env` — por defecto 120) y registra cada cambio en la tabla `auditoria`
  (ver [`../docs/auditoria.md`](../docs/auditoria.md)). Lecturas sin límite: si se publica en internet,
  limitarlas en el proxy.
- **Captcha del login (opcional).** El usuario y la clave del laboratorio bastan para entrar. Si se
  quiere proteger el login con Cloudflare Turnstile, poner en `docker/.env` tanto
  `EXPO_PUBLIC_TURNSTILE_SITEKEY` (pública, de Cloudflare) como `TURNSTILE_SECRET` y
  `TURNSTILE_HOSTNAMES` (la lista de dominios del frontend; **obligatoria** junto al secret: sin ella el
  login queda bloqueado, y si se la deja con los valores por defecto `localhost,127.0.0.1` en producción
  el login se bloquea porque el hostname real no está en la lista). Como el sitekey se hornea en el bundle,
  hay que **reconstruir** la imagen web: en las imágenes publicadas, con la variable
  `TURNSTILE_SITEKEY` del repositorio en GitHub; compilando aquí, con `up -d --build`. Para
  usarlo de verdad hace falta ponerlo a mano en `.env`; por defecto `TURNSTILE_SECRET`
  queda vacío (sin captcha) y `TURNSTILE_HOSTNAMES` arranca como `localhost,127.0.0.1`,
  lo cual debe cambiarse por tu dominio real si activas el secret, porque si no el
  login se bloquea.
- Dependencias: la API no tiene vulnerabilidades conocidas (`npm audit`); la app web
  conserva 46 avisos en la cadena de Expo, pendientes de una actualización de Expo.
