# Apache Superset en Debian con Docker

Guía completa para levantar Apache Superset usando Docker Compose en Debian, usando la versión estable `4.1.1`.

---

## Requisitos previos

- Debian (cualquier versión reciente)
- Docker instalado
- Docker Compose plugin instalado
- Git instalado

Verificar que tienes todo:

```bash
docker --version
docker compose version
git --version
```

Si te falta Docker Compose:

```bash
sudo apt update
sudo apt install docker-compose-plugin
```

---

## Instalación

### 1. Clonar el repositorio de Superset

```bash
git clone https://github.com/apache/superset.git
cd superset
```

### 2. Guardar la versión estable como variable de entorno

Esto evita tener que escribir `TAG=4.1.1` cada vez:

```bash
echo "TAG=4.1.1" > docker/.env-local
```

> ⚠️ **Importante:** No uses la imagen `latest-dev` — tiene un bug de dependencias con `sqlglot` que impide que el contenedor `superset-init` complete la inicialización.

### 3. Levantar Superset

```bash
TAG=4.1.1 docker compose -f docker-compose-image-tag.yml up -d
```

El primer arranque tarda entre **2 y 3 minutos** porque inicializa la base de datos. Verifica que todos los contenedores estén arriba:

```bash
docker compose -f docker-compose-image-tag.yml ps
```

Deberías ver estos contenedores corriendo:

| Contenedor | Servicio | Estado |
|---|---|---|
| superset_app | superset | Up |
| superset_db | db (PostgreSQL 17) | Up |
| superset_cache | redis (Redis 7) | Up |
| superset_worker | superset-worker | Up |
| superset_worker_beat | superset-worker-beat | Up |
| superset_init | superset-init | Exited (0) ✔ |

> El contenedor `superset_init` debe terminar con `Exited (0)` — eso es correcto, su trabajo es inicializar y salir.

---

## Acceso

Abre el navegador en:

```
http://localhost:8088
```

Credenciales por defecto:

- **Usuario:** `admin`
- **Contraseña:** `admin`

> 🔒 Cambia la contraseña después del primer acceso en producción.

---

## Cargar un CSV o Excel

1. Ve a **Settings → Database Connections → + Database**
2. Elige **SQLite**
3. Luego ve a **Data → Upload CSV to database**
4. Selecciona tu archivo — Superset crea la tabla automáticamente
5. Ve a **Charts → + Chart** y empieza a visualizar

---

## Comandos del día a día

```bash
# Levantar
TAG=4.1.1 docker compose -f docker-compose-image-tag.yml up -d

# Apagar (conserva los datos)
docker compose -f docker-compose-image-tag.yml down

# Apagar y borrar todos los datos (reset completo)
docker compose -f docker-compose-image-tag.yml down -v

# Ver estado de los contenedores
docker compose -f docker-compose-image-tag.yml ps

# Ver logs en tiempo real
docker compose -f docker-compose-image-tag.yml logs -f

# Ver logs de un servicio específico
docker compose -f docker-compose-image-tag.yml logs superset-init
```

---

## Solución de problemas

### Error: `superset-init` falla con conflicto de `sqlglot`

**Causa:** La imagen `latest-dev` tiene un bug de dependencias incompatibles entre `apache-superset` y `apache-superset-core`.

**Solución:** Usar la versión estable con `TAG=4.1.1`:

```bash
docker compose -f docker-compose-image-tag.yml down -v
TAG=4.1.1 docker compose -f docker-compose-image-tag.yml up -d
```

---

### Error: `port 8088 already in use`

**Causa:** Una instancia anterior de Docker quedó corriendo en el fondo.

**Diagnóstico:**

```bash
sudo ss -tlnp | grep 8088
```

**Solución:** Matar los procesos `docker-proxy` que aparezcan:

```bash
sudo kill -9 <PID>
```

Luego volver a levantar:

```bash
docker compose -f docker-compose-image-tag.yml down -v
TAG=4.1.1 docker compose -f docker-compose-image-tag.yml up -d
```

---

### Los contenedores desaparecen sin error visible

Verificar que estás en la carpeta correcta del proyecto:

```bash
pwd
# Debe mostrar: .../superset
ls docker-compose*.yml
# Debe listar los archivos compose
```

Limpiar redes y volúmenes huérfanos y reintentar:

```bash
docker network prune -f
docker volume prune -f
TAG=4.1.1 docker compose -f docker-compose-image-tag.yml up -d
```

---

## Estructura del proyecto

```
superset/
├── docker/
│   ├── .env               # Variables de entorno (credenciales DB, SECRET_KEY)
│   └── .env-local         # Overrides locales (TAG=4.1.1 va aquí)
├── docker-compose-image-tag.yml   # ← El que usamos
├── docker-compose-non-dev.yml
├── docker-compose-light.yml
└── docker-compose.yml
```

---

## Notas

- Esta configuración **no es para producción** — es para desarrollo y análisis local.
- Los datos persisten en volúmenes Docker (`superset_db_home`, `superset_home`, `redis`) mientras no hagas `down -v`.
- La versión `4.1.1` es la última estable verificada al momento de escribir esta guía.

---

*Probado en Debian con Docker 27.x y Docker Compose v2.x*
