# Odoo 19 con Docker Compose (dev / prod)

## Puesta en marcha
1. Copiar `.env.example` → `.env` y poner contraseña.
2. Copiar `config/*.example` → quitar `.example` y poner `admin_passwd`.
3. Desarrollo: `docker compose --profile dev up -d`
4. Producción: `docker compose --profile prod up -d`

> Los dos perfiles usan el puerto 8069 y el mismo volumen: se levanta uno u otro, nunca ambos a la vez.

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

- **`services`**: cada servicio es un contenedor. Aquí hay tres: `db` (PostgreSQL), `odoo-dev` y `odoo-prod`.
  - `image`: imagen de la que se crea el contenedor (`postgres:18`, `odoo:19`).
  - `environment`: variables de entorno. Los valores `${DB_USER}` se leen del archivo `.env`.
  - `volumes`: qué carpetas o volúmenes se montan dentro del contenedor.
  - `ports`: `"host:contenedor"`. `127.0.0.1:8069:8069` solo escucha en local.
  - `depends_on`: orden de arranque (Odoo después de `db`). No espera a que Postgres esté *listo*, solo arrancado.
  - `command`: argumentos que se pasan a Odoo (`--dev=all`, `-c /etc/odoo/odoo.prod.conf`).
  - `profiles`: el servicio solo arranca si se activa ese perfil (`--profile dev`).
  - `restart: unless-stopped`: Docker lo reinicia si se cae (producción).
  - `networks`: red a la que se conecta.
- **`volumes`** (nivel raíz): declara los volúmenes con nombre gestionados por Docker (`odoo-db-data`, `odoo-web-data`).
- **`networks`** (nivel raíz): declara la red interna `odoo-net` (tipo `bridge`). Los servicios se encuentran por nombre: Odoo conecta a `db` usando `HOST=db`.

> `db` no tiene `ports`: PostgreSQL no es accesible desde fuera de Docker, solo desde Odoo a través de `odoo-net`.

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
- **Healthcheck** en `db` + `depends_on: condition: service_healthy`, para que Odoo no arranque antes de que Postgres acepte conexiones.
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
| Archivo de configuración | `config/odoo.conf` | `config/odoo.prod.conf` |
| `--dev=all` (recarga, depuración) | Sí | No |
| Gestor de BD (`list_db`) | `True` | `False` (comprobado: *"database manager has been disabled"*) |
| Addons | Lectura/escritura | `:ro` (solo lectura) |
| Puerto | `8069` abierto | `127.0.0.1:8069` (detrás de un proxy) |
| `proxy_mode` | No | Sí |
| `workers` | 0 | 2 |
| `log_level` | `info` | `warn` |
| `restart` | No | `unless-stopped` |
| Datos demo | Sí (datos de prueba para desarrollar) | Nunca (no se pueden quitar limpiamente y contaminan informes y contabilidad) |

**Cómo conviven en un único `compose.yml`:** con **perfiles** (`profiles: [dev]` / `profiles: [prod]`). Cada entorno tiene su propia base de
datos, sus propios volúmenes, su propia red y sus propias credenciales: desarrollo no puede acceder a producción ni al revés.

### Qué directorios van al repositorio

| Directorio / archivo | ¿En el repo? | Motivo |
|---|---|---|
| `addons/` | **Sí** | Es nuestro código (módulos propios). |
| `compose.yml`, `config/*.example`, `.env.example` | **Sí** | Configuración sin secretos, permite reproducir el entorno. |
| `filestore/` | **No** | Adjuntos e imágenes subidos a Odoo: datos de usuario, pesados, pueden contener información personal (RGPD). |
| `sessions/` | **No** | Sesiones de usuarios: datos temporales y sensibles (permitirían suplantar sesiones). |
| Datos de PostgreSQL | **No** | Es la base de datos. Se gestiona con backups, no con git. |
| `.env`, `config/odoo.conf`, `config/odoo.prod.conf` | **No** | Contienen contraseñas. |

En esta configuración `filestore` y `sessions` están en el volumen `odoo-web-data` (`/var/lib/odoo`) y la base de datos en `odoo-db-data`, así que ni siquiera existen en la carpeta del proyecto.

---

## Problemas encontrados y soluciones

**1. `failed to set up container networking: network ... not found`**
- *Causa:* quedaba un contenedor de una configuración anterior que apuntaba a una red ya eliminada; Docker intentaba reutilizarlo.
- *Solución:* `docker compose --profile dev down --remove-orphans` y volver a levantar.

**2. Error 500 en `localhost:8069` (`relation "ir_module_module" does not exist`)**
- *Causa:* `POSTGRES_DB: ${DB_NAME}` hacía que Postgres creara una base de datos vacía llamada `odoo`. Odoo la detectaba, intentaba usarla y fallaba porque no tenía sus tablas.
- *Solución:* `POSTGRES_DB: postgres` (base de datos de sistema por defecto) y `docker compose down -v` para eliminar el volumen que contenía la base de datos vacía. Postgres solo ejecuta su inicialización cuando el volumen está vacío, por eso cambiar la variable no bastaba.
- *Nota de seguridad:* el nombre de la base de datos no es un secreto; la protección real es la contraseña y no exponer el puerto 5432.

