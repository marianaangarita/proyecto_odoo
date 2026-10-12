# Odoo 19 con Docker Compose (dev / prod)

## Puesta en marcha
1. Copiar `.env.example` → `.env` y rellenar `DEV_DB_USER`, `DEV_DB_PASSWORD`, `PROD_DB_USER`, `PROD_DB_PASSWORD` (contraseñas distintas e independientes).
2. Copiar `config/odoo.conf.example` → `config/odoo.conf` y `config/odoo.prod.conf.example` → `config/odoo.prod.conf`, y poner `admin_passwd` (distinta en cada uno).
3. **Desarrollo:** `docker compose --profile dev up -d` → `localhost:8069` y crear la base de datos desde el gestor (con datos demo).
4. **Producción** (solo la primera vez, porque el gestor de BD está desactivado):
   ```
   docker compose --profile prod run --rm odoo-prod -c /etc/odoo/odoo.prod.conf -d odoo_prod -i base --stop-after-init
   ```
   Después: `docker compose --profile prod up -d`, entrar con `admin` / `admin` y **cambiar la contraseña inmediatamente**.

> Los dos perfiles publican el puerto 8069: se levanta uno u otro, no ambos a la vez. Para cambiar: `docker compose --profile dev down` y luego `up` del otro.

---

## Conceptos de Docker

### Comandos usados

| Comando | Qué hace |
|---|---|
| `docker compose --profile dev up -d` | Crea y arranca los servicios del perfil `dev` (más los que no tienen perfil, como `db`). `-d` = en segundo plano. |
| `docker compose --profile dev down` | Para y elimina los contenedores y la red del proyecto. Los volúmenes se conservan. |
| `docker compose down --remove-orphans` | Además elimina contenedores huérfanos (de servicios que ya no están en el compose). |
| `docker compose down -v` | Elimina también los **volúmenes** → borra los datos. Usar con cuidado. |
| `docker compose up -d --force-recreate` | Recrea los contenedores aunque no haya cambios (útil tras modificar `odoo.conf`). |
| `docker compose logs odoo-dev --tail 50` | Muestra las últimas 50 líneas de log de un servicio. Primer paso para depurar. |
| `docker ps -a` | Lista todos los contenedores, incluidos los parados. |
| `docker pull odoo:19` | Descarga una imagen del registro (Docker Hub). |
| `docker images` | Lista las imágenes descargadas. |
| `docker exec -it <contenedor> bash` | Abre una terminal dentro de un contenedor en ejecución. |
| `docker volume ls` | Lista los volúmenes. |

### Bloques del `compose.yml`

- **`services`**: cada servicio es un contenedor. Aquí hay cuatro: `db-dev` y `odoo-dev` (desarrollo), `db-prod` y `odoo-prod` (producción).
  - `image`: imagen de la que se crea el contenedor (`postgres:18`, `odoo:19`).
  - `environment`: variables de entorno. Los valores `${DB_USER}` se leen del archivo `.env`.
  - `volumes`: qué carpetas o volúmenes se montan dentro del contenedor.
  - `ports`: `"host:contenedor"`. `127.0.0.1:8069:8069` solo escucha en local.
  - `depends_on` con `condition: service_healthy`: Odoo no arranca hasta que el `healthcheck` de su Postgres (`pg_isready`) confirma que acepta conexiones.
  - `healthcheck`: comando que Docker ejecuta periódicamente para saber si el servicio está sano.
  - `command`: argumentos que se pasan a Odoo (`--dev=all`, `-c /etc/odoo/odoo.prod.conf`).
  - `profiles`: el servicio solo arranca si se activa ese perfil (`--profile dev`).
  - `restart: unless-stopped`: Docker lo reinicia si se cae (producción).
  - `networks`: red a la que se conecta.
- **`volumes`** (nivel raíz): declara los volúmenes con nombre gestionados por Docker: `odoo-db-data` y `odoo-web-data` (dev), `prod-db-data` y `prod-web-data` (prod).
- **`networks`** (nivel raíz): declara dos redes internas, `net-dev` y `net-prod`. Los servicios se encuentran por nombre (`HOST=db-dev`), y un servicio de una red no puede ver los de la otra.

> Ninguna base de datos tiene `ports`: PostgreSQL no es accesible desde fuera de Docker, solo desde su Odoo.

### ¿Qué es una imagen?
Una **plantilla de solo lectura** que contiene el sistema de archivos y todo lo necesario para ejecutar una aplicación (sistema base, dependencias, código). Se construye por capas y se descarga de un registro. `odoo:19` es la imagen oficial de Odoo 19.

### ¿Qué es un contenedor?
Una **instancia en ejecución de una imagen**: un proceso aislado con su propia red y sistema de archivos. Es **desechable**: al recrearlo (`--force-recreate`, actualizar la imagen) todo lo escrito dentro se pierde. Por eso los datos no deben vivir en el contenedor.

### ¿Qué es un volumen?
**Almacenamiento persistente fuera del contenedor** que Docker monta al arrancarlo. Sobrevive a que el contenedor se destruya. Dos tipos usados aquí:
- **Volumen con nombre** (`odoo-db-data`, `odoo-web-data`): gestionado por Docker. Para datos (base de datos, filestore, sesiones).
- **Bind mount** (`./addons`, `./config`): una carpeta del ordenador montada en el contenedor. Para código y configuración que edito.

Comprobado en la práctica: tras `--force-recreate` la base de datos seguía existiendo porque estaba en el volumen.

**Resumen:** imagen = plantilla · contenedor = instancia desechable · volumen = datos que sobreviven al contenedor.

### Eficiencia de recursos

- **Imágenes ligeras**: `postgres:18-alpine` ocupa bastante menos que `postgres:18`.
- **Limitar recursos** por servicio para que uno no acapare la máquina:
  ```yaml
  deploy:
    resources:
      limits:
        cpus: "1.0"
        memory: 2g
  ```
- **Workers de Odoo**: en desarrollo `workers = 0` (un proceso, más fácil de depurar). En producción, varios workers (regla orientativa: `(núm. CPU × 2) + 1`) más `limit_memory_soft`/`limit_memory_hard` para reciclar procesos que consumen demasiado.
- **Healthcheck** en cada base de datos + `depends_on: condition: service_healthy` (implementado), para que Odoo no arranque antes de que Postgres acepte conexiones.
- **Volúmenes con nombre en vez de bind mounts para datos**: en Windows (WSL2) los bind mounts a carpetas de Windows son lentos; los volúmenes de Docker van mucho más rápido.
- **Perfiles**: solo se levantan los servicios necesarios.
- **Rotación de logs** (`logging: options: max-size`) para que los logs no llenen el disco.

### Variables de entorno de forma segura (`.env`, secrets)

- **`.env`**: Compose lo lee automáticamente para sustituir `${VAR}`. Separa la configuración del compose. **Nunca se sube al repo** (`.gitignore`); se sube un `.env.example` sin valores reales.
- Limitación del `.env`: las variables acaban en el entorno del contenedor y se ven con `docker inspect`.
- **Docker secrets**: el secreto se monta como archivo en `/run/secrets/<nombre>` en lugar de variable de entorno. La imagen de Postgres admite `POSTGRES_PASSWORD_FILE`:
  ```yaml
  services:
    db:
      environment:
        POSTGRES_PASSWORD_FILE: /run/secrets/db_password
      secrets:
        - db_password
  secrets:
    db_password:
      file: ./secrets/db_password.txt   # también en .gitignore
  ```
- No confundir **configuración** con **secreto**: el nombre de usuario o de la base de datos son configuración; las contraseñas son secretos.
- Cada secreto en **un solo sitio**: la contraseña de Postgres llega por entorno y **no** se repite en `odoo.conf`.

---

## Configuración de Odoo

### Variables de entorno vs `odoo.conf`

| | Variables de entorno (compose) | `odoo.conf` |
|---|---|---|
| Qué cubre | Solo lo que la imagen soporta: `HOST`, `PORT`, `USER`, `PASSWORD` (conexión a BD) | **Todas** las opciones de Odoo: `admin_passwd`, `workers`, `list_db`, `dbfilter`, `addons_path`, límites… |
| Visibilidad | Visibles con `docker inspect` y en el entorno del proceso | Archivo con permisos propios, montado en el contenedor |
| Cambio | Recrear el contenedor | Recrear/reiniciar el contenedor |

**¿Qué es más seguro?** `odoo.conf` montado como archivo (y fuera de git) es más seguro que las variables de entorno, porque estas se exponen en `docker inspect`. Lo más seguro es usar secrets/archivos para las contraseñas.

**¿Qué es más práctico para desarrollo local?** Combinar ambos: el entorno para la conexión a la base de datos (cambia según la máquina) y `odoo.conf` montado desde `./config` para el resto, editable sin tocar el compose. Además, `--dev=all` en `command` activa la recarga automática del código.

**Motivo práctico observado:** la master password que Odoo genera la primera vez se guarda *dentro* del contenedor y se pierde al recrearlo. Con `odoo.conf` montado, `admin_passwd` persiste.

### Desarrollo vs producción

| | `odoo-dev` | `odoo-prod` |
|---|---|---|
| Base de datos (servicio) | `db-dev` | `db-prod` |
| Volúmenes | `odoo-db-data`, `odoo-web-data` | `prod-db-data`, `prod-web-data` |
| Red | `net-dev` | `net-prod` |
| Credenciales | `DEV_DB_*` | `PROD_DB_*` |
| Archivo de configuración | `config/odoo.conf` | `config/odoo.prod.conf` |
| BD permitida | cualquiera | solo `odoo_prod` (`db_name` + `dbfilter`) |
| `--dev=all` (recarga, depuración) | Sí | No |
| Gestor de BD (`list_db`) | `True` | `False` (comprobado: *"database manager has been disabled"*) |
| Addons | Lectura/escritura | `:ro` (solo lectura) |
| Puerto | `8069` abierto | `127.0.0.1:8069` (detrás de un proxy) |
| `proxy_mode` | No | Sí |
| `http_interface` | por defecto | `0.0.0.0` explícito (Odoo 20 cambiará el defecto a `127.0.0.1`, que dentro de un contenedor lo haría inaccesible) |
| `workers` | 0 | 2 |
| `log_level` | `info` | `warn` |
| `restart` | No | `unless-stopped` |
| Datos demo | Sí (datos de prueba para desarrollar) | Nunca (no se pueden quitar limpiamente y contaminan informes y contabilidad) |

**Cómo conviven en un único `compose.yml`:** con **perfiles**. Todos los servicios tienen perfil (`db-dev` y `odoo-dev` → `dev`; `db-prod` y `odoo-prod` → `prod`), así que solo arranca el entorno que se activa; `docker compose up` sin perfil no arranca nada. Cada entorno tiene su propia base de datos, volúmenes, red y credenciales: desarrollo no puede acceder a producción ni al revés.

**Comprobado:** desarrollo muestra 2 usuarios activos (admin + "Marc Demo", datos demo); producción muestra 1 (solo admin) y una compañía vacía.

### Qué directorios van al repositorio

| Directorio / archivo | ¿En el repo? | Motivo |
|---|---|---|
| `addons/` | **Sí** | Es nuestro código (módulos propios). |
| `compose.yml`, `config/*.example`, `.env.example` | **Sí** | Configuración sin secretos, permite reproducir el entorno. |
| `filestore/` | **No** | Adjuntos e imágenes subidos a Odoo: datos de usuario, pesados, pueden contener información personal (RGPD). |
| `sessions/` | **No** | Sesiones de usuarios: datos temporales y sensibles (permitirían suplantar sesiones). |
| Datos de PostgreSQL | **No** | Es la base de datos. Se gestiona con backups, no con git. |
| `.env`, `config/odoo.conf`, `config/odoo.prod.conf` | **No** | Contienen contraseñas. |

En esta configuración `filestore` y `sessions` están en los volúmenes `odoo-web-data` / `prod-web-data` (`/var/lib/odoo`) y las bases de datos en `odoo-db-data` / `prod-db-data`, así que ni siquiera existen en la carpeta del proyecto.

---

## Problemas encontrados y soluciones

**1. `failed to set up container networking: network ... not found`**
- *Causa:* quedaba un contenedor de una configuración anterior que apuntaba a una red ya eliminada; Docker intentaba reutilizarlo.
- *Solución:* `docker compose --profile dev down --remove-orphans` y volver a levantar.

**2. Error 500 en `localhost:8069` (`relation "ir_module_module" does not exist`)**
- *Causa:* `POSTGRES_DB: ${DB_NAME}` hacía que Postgres creara una base de datos vacía llamada `odoo`. Odoo la detectaba, intentaba usarla y fallaba porque no tenía sus tablas.
- *Solución:* `POSTGRES_DB: postgres` (base de datos de sistema por defecto) y `docker compose down -v` para eliminar el volumen que contenía la base de datos vacía. Postgres solo ejecuta su inicialización cuando el volumen está vacío, por eso cambiar la variable no bastaba.
- *Nota de seguridad:* el nombre de la base de datos no es un secreto; la protección real es la contraseña y no exponer el puerto 5432.

**3. `unrecognized parameters: db-prod` al inicializar producción**
- *Causa:* el comando se partió al pegarlo y `-c` quedó sin ruta. El entrypoint de Odoo añade `--db_host db-prod` al final, así que `--db_host` se tomó como ruta del `-c` y `db-prod` quedó suelto.
- *Solución:* ejecutar el comando completo en una sola línea.

**4. Avisos al arrancar**
- `invalid addons directory '/mnt/extra-addons'`: la carpeta aún no contiene módulos (solo `.gitkeep`). Inofensivo; desaparece al crear el primer módulo.
- `missing http_interface ... will change to 127.0.0.1 in 20.0`: resuelto fijando `http_interface = 0.0.0.0` en `odoo.prod.conf`.

**5. Push rechazado por commits remotos**
- *Causa:* se editó el README desde la web de GitHub.
- *Solución:* `git pull --rebase` antes de `git push`.
