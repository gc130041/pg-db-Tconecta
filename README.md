# pg-db-Tconecta — Base de Datos PostgreSQL

Contenedor Docker con **PostgreSQL 16 (Alpine)** que sirve como base de datos principal para el módulo de autenticación de **Transmetro Conecta**.

---

## Tabla de Contenidos

1. [Stack Tecnológico](#stack-tecnológico)
2. [Prerrequisitos](#prerrequisitos)
3. [Variables de Entorno](#variables-de-entorno)
4. [Pasos para Levantar el Contenedor](#pasos-para-levantar-el-contenedor)
5. [Operaciones Comunes](#operaciones-comunes)
6. [Errores Frecuentes y Soluciones](#errores-frecuentes-y-soluciones)
7. [Limpieza y Respaldo](#limpieza-y-respaldo)
8. [Notas](#notas)

---

## Stack Tecnológico

| Componente          | Tecnología / Versión        |
| ------------------- | --------------------------- |
| Motor de BD         | PostgreSQL **16**           |
| Imagen base         | `postgres:16-alpine`        |
| Orquestador         | Docker Compose **v3.8**     |
| Sistema operativo   | Alpine Linux (dentro del contenedor) |
| Red Docker          | `transmetro_network` (bridge) |
| Volumen persistente | `pgdata`                    |

---

## Prerrequisitos

Antes de levantar el contenedor, asegúrate de contar con lo siguiente:

1. **Docker Desktop** (Windows / macOS) o **Docker Engine** (Linux)
   - Versión mínima recomendada: **20.10+**
   - Descarga: <https://docs.docker.com/get-docker/>

2. **Docker Compose**
   - Si usas Docker Desktop, ya viene incluido.
   - Para instalaciones independientes, verifica con:

     ```bash
     docker compose version
     # o, para la versión legacy:
     docker-compose --version
     ```

   - Versión mínima recomendada: **2.0+** (plugin) o **1.29+** (standalone).

3. **Puerto 5432 disponible**
   - El contenedor expone PostgreSQL en el puerto `5432` del host.
   - Verifica que no haya otra instancia de PostgreSQL u otro servicio ocupando ese puerto.

4. **Espacio en disco**
   - La imagen `postgres:16-alpine` pesa aproximadamente **80 MB**.
   - Reservar espacio adicional para el volumen de datos (`pgdata`) según el tamaño esperado de la base de datos.

---

## Variables de Entorno

Las siguientes variables se configuran dentro del archivo `docker-compose.yml` en la sección `environment` del servicio `postgres-db`:

| Variable              | Valor por defecto      | Descripción                                                                 |
| --------------------- | ---------------------- | --------------------------------------------------------------------------- |
| `POSTGRES_USER`       | `postgres`             | Usuario administrador de la base de datos.                                  |
| `POSTGRES_PASSWORD`   | `postgres`             | Contraseña del usuario administrador.                                       |
| `POSTGRES_DB`         | `TransmetroAuthDb`     | Nombre de la base de datos que se crea automáticamente al iniciar el contenedor. |

> ⚠️ **Importante:** Para entornos de **producción**, cambia las credenciales por defecto y utiliza mecanismos seguros para gestionar secretos (por ejemplo, Docker Secrets o un archivo `.env` excluido del repositorio).

### Ejemplo con archivo `.env`

Puedes sobreescribir las variables creando un archivo `.env` en el mismo directorio:

```env
POSTGRES_USER=mi_usuario
POSTGRES_PASSWORD=mi_contraseña_segura
POSTGRES_DB=TransmetroAuthDb
```

Y luego referenciarlo en el `docker-compose.yml`:

```yaml
environment:
  POSTGRES_USER: ${POSTGRES_USER}
  POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
  POSTGRES_DB: ${POSTGRES_DB}
```

---

## Pasos para Levantar el Contenedor

### 1. Navegar al directorio del proyecto

```bash
cd pg-db-Tconecta
```

### 2. Levantar el contenedor

```bash
# Modo foreground (ver logs en tiempo real):
docker compose up

# Modo background (detached):
docker compose up -d
```

### 3. Verificar que el contenedor está corriendo

```bash
docker compose ps
```

Deberías ver algo como:

```
NAME                  STATUS                   PORTS
transmetro_postgres   Up (healthy)             0.0.0.0:5432->5432/tcp
```

El estado `healthy` indica que el healthcheck (`pg_isready`) pasó correctamente.

### 4. Verificar conexión a la base de datos

```bash
docker exec -it transmetro_postgres psql -U postgres -d TransmetroAuthDb -c "SELECT 1;"
```

Si la conexión es exitosa, verás:

```
 ?column?
----------
        1
(1 row)
```

---

## Operaciones Comunes

### Conectarse a la base de datos con `psql`

```bash
docker exec -it transmetro_postgres psql -U postgres -d TransmetroAuthDb
```

### Listar bases de datos

```bash
docker exec -it transmetro_postgres psql -U postgres -c "\l"
```

### Listar tablas de la base de datos

```bash
docker exec -it transmetro_postgres psql -U postgres -d TransmetroAuthDb -c "\dt"
```

### Ver logs del contenedor

```bash
docker compose logs -f postgres-db
```

### Detener el contenedor

```bash
docker compose down
```

> **Nota:** `docker compose down` detiene y elimina el contenedor, pero **preserva el volumen** `pgdata` con los datos.

### Reiniciar el contenedor

```bash
docker compose restart
```

### Conectarse desde una aplicación externa (host)

Utiliza los siguientes parámetros de conexión:

| Parámetro | Valor                |
| --------- | -------------------- |
| Host      | `localhost`          |
| Puerto    | `5432`               |
| Usuario   | `postgres`           |
| Contraseña| `postgres`           |
| Base de datos | `TransmetroAuthDb` |

**Connection string** (formato .NET):

```
Host=localhost;Database=TransmetroAuthDb;Username=postgres;Password=postgres
```

---

## Errores Frecuentes y Soluciones

### 1. `Error: port is already allocated` (Puerto 5432 ya en uso)

**Causa:** Otro proceso está usando el puerto 5432 (posiblemente una instalación local de PostgreSQL).

**Soluciones:**

- Detener el servicio local de PostgreSQL:

  ```bash
  # Windows (PowerShell como administrador):
  Stop-Service -Name postgresql-x64-16

  # Linux:
  sudo systemctl stop postgresql
  ```

- O cambiar el puerto mapeado en `docker-compose.yml`:

  ```yaml
  ports:
    - "5433:5432"  # Usar puerto 5433 en el host
  ```

---

### 2. `Error: No such container: transmetro_postgres`

**Causa:** El contenedor no se ha creado o se eliminó.

**Solución:** Levantar el contenedor nuevamente:

```bash
docker compose up -d
```

---

### 3. `FATAL: password authentication failed for user "postgres"`

**Causa:** La contraseña configurada no coincide con la almacenada en el volumen de datos (esto sucede si cambias la contraseña en el `docker-compose.yml` después de haber creado el volumen).

**Solución:** Eliminar el volumen y recrear el contenedor:

```bash
docker compose down -v
docker compose up -d
```

> ⚠️ **Advertencia:** Esto eliminará **todos los datos** de la base de datos.

---

### 4. `docker: Error response from daemon: driver failed programming external connectivity`

**Causa:** Conflicto de red o Docker necesita reiniciarse.

**Solución:**

```bash
# Reiniciar Docker Desktop (Windows/macOS)
# O en Linux:
sudo systemctl restart docker
```

---

### 5. El contenedor se reinicia constantemente (estado `Restarting`)

**Causa:** El healthcheck falla repetidamente.

**Solución:**

1. Revisar los logs:

   ```bash
   docker compose logs postgres-db
   ```

2. Verificar que el volumen no tenga datos corruptos:

   ```bash
   docker compose down -v
   docker compose up -d
   ```

---

### 6. `ERROR: could not find an available, non-overlapping IPv4 address pool`

**Causa:** Docker se quedó sin rangos de red disponibles.

**Solución:**

```bash
docker network prune
```

---

## Limpieza y Respaldo

### Crear un respaldo de la base de datos

```bash
# Respaldo completo en formato SQL:
docker exec -t transmetro_postgres pg_dump -U postgres TransmetroAuthDb > backup_transmetro_auth.sql

# Respaldo en formato comprimido (custom):
docker exec -t transmetro_postgres pg_dump -U postgres -Fc TransmetroAuthDb > backup_transmetro_auth.dump
```

### Restaurar un respaldo

```bash
# Desde un archivo SQL:
cat backup_transmetro_auth.sql | docker exec -i transmetro_postgres psql -U postgres -d TransmetroAuthDb

# Desde un archivo comprimido (custom):
docker exec -i transmetro_postgres pg_restore -U postgres -d TransmetroAuthDb < backup_transmetro_auth.dump
```

> En **Windows (PowerShell)**, reemplaza `cat` por `Get-Content` o usa `type`:
>
> ```powershell
> Get-Content backup_transmetro_auth.sql | docker exec -i transmetro_postgres psql -U postgres -d TransmetroAuthDb
> ```

### Eliminar contenedor y volumen (limpieza total)

```bash
docker compose down -v
```

Esto elimina:
- El contenedor `transmetro_postgres`.
- El volumen `pgdata` con todos los datos.
- La red `transmetro_network`.

### Eliminar solo el contenedor (preservar datos)

```bash
docker compose down
```

### Eliminar imágenes Docker no utilizadas

```bash
docker image prune -a
```

---

## Notas

- **Persistencia de datos:** Los datos de PostgreSQL se almacenan en el volumen Docker llamado `pgdata`. Mientras este volumen exista, los datos sobrevivirán a reinicios y recreaciones del contenedor.

- **Healthcheck:** El contenedor incluye un healthcheck que ejecuta `pg_isready -U postgres` cada 5 segundos, con un timeout de 5 segundos y hasta 5 reintentos. Los microservicios que dependen de esta base de datos pueden usar la condición `service_healthy` para esperar a que PostgreSQL esté completamente listo antes de arrancar.

- **Red Docker:** El contenedor se conecta a la red `transmetro_network` (driver: bridge). Si otros servicios de Transmetro Conecta necesitan comunicarse con esta base de datos a través de Docker, deben conectarse a la misma red y usar `postgres-db` como hostname (nombre del servicio en compose).

- **Imagen Alpine:** Se utiliza `postgres:16-alpine` en lugar de la imagen estándar para reducir significativamente el tamaño de la imagen (~80 MB vs ~400 MB), manteniendo la misma funcionalidad.

- **Entorno de desarrollo:** La configuración actual está orientada a un entorno de desarrollo local. Para producción, se recomienda:
  - Usar credenciales seguras (no valores por defecto).
  - Configurar SSL/TLS para conexiones cifradas.
  - Implementar respaldos automáticos.
  - Limitar el acceso a la red solo a los servicios necesarios.
  - No exponer el puerto 5432 al host si no es necesario.

- **Versión de Compose:** El archivo utiliza la especificación `version: '3.8'`. En versiones recientes de Docker Compose (v2+), el campo `version` es informativo y puede omitirse.