# Innovatech Chile – Backend Ventas (Spring Boot + MySQL)

API backend contenedorizada de Innovatech Chile para la gestión de ventas. Desplegada en una instancia EC2 en **subred privada** de AWS. Solo accesible desde la EC2 del frontend (controlado por Security Groups).

---

## Tecnologías

| Componente | Versión |
|---|---|
| Java | 17 (JRE Alpine) |
| Spring Boot | 3.4.4 |
| Maven (build) | 3.9 Alpine |
| MySQL | 8.0 |

---

## Estructura relevante

```
innovatech-ventas-backend/
├── Dockerfile                    # Multi-stage: build (maven) + run (jre)
├── docker-compose.yml            # Stack completo: backend + MySQL
├── .env.example                  # Variables de entorno requeridas
└── .github/
    └── workflows/
        └── deploy.yml            # Pipeline CI/CD GitHub Actions
```

---

## Ejecutar localmente con Docker

### 1. Clonar y configurar variables

```bash
git clone https://github.com/TU_USUARIO/innovatech-ventas-backend.git
cd innovatech-ventas-backend
cp .env.example .env
# Editar .env: al menos definir DB_PASSWORD
```

### 2. Levantar el stack completo (backend + MySQL)

```bash
# Construir y levantar todos los servicios
docker compose up -d --build

# Verificar contenedores
docker ps

# Ver logs del backend
docker compose logs -f backend

# Ver logs de la base de datos
docker compose logs -f db
```

### 3. Verificar que la API responde

```bash
curl http://localhost:8082/api/v1/ventas
# Esperado: [] (lista vacía si no hay datos)
```

### 4. Consultar documentación Swagger

```
http://localhost:8082/swagger-ui.html
```

### 5. Detener sin borrar datos

```bash
docker compose down
# El volumen citt_mysql_ventas_data se conserva
```

### 6. Detener y borrar todo (incluyendo datos)

```bash
docker compose down -v
```

---

## Endpoints disponibles

| Método | Endpoint | Descripción |
|---|---|---|
| `GET` | `/api/v1/ventas` | Listar todas las ventas |
| `GET` | `/api/v1/ventas/{id}` | Obtener venta por ID |
| `POST` | `/api/v1/ventas` | Crear nueva venta |
| `PUT` | `/api/v1/ventas/{id}` | Actualizar venta existente |
| `DELETE` | `/api/v1/ventas/{id}` | Eliminar venta |

---

## Persistencia de datos

Se utiliza un **named volume** llamado `citt_mysql_ventas_data` para los datos de MySQL.

### ¿Por qué named volume en lugar de bind mount?

| Criterio | Named Volume | Bind Mount |
|---|---|---|
| Portabilidad | ✅ Gestionado por Docker | ❌ Depende de rutas del host |
| Seguridad | ✅ No expone directorios del host | ⚠️ Expone sistema de archivos |
| Facilidad | ✅ Docker gestiona la ubicación | ❌ Requiere crear directorios |
| Backup | ✅ `docker volume` comandos | Manual |

Los datos sobreviven: `docker compose down` y `docker compose up` sin perder información.

---

## Pipeline CI/CD (GitHub Actions)

El pipeline se activa al hacer **push a la rama `deploy`**.

### Flujo

```
push a rama deploy
       │
       ▼
┌─────────────────────────────┐
│  Job 1: build-and-push      │
│  • Checkout código          │
│  • Docker Buildx            │
│  • Login Docker Hub         │
│  • mvn package (en imagen)  │
│  • docker push (sha+latest) │
└────────────┬────────────────┘
             │ éxito
             ▼
┌──────────────────────────────────┐
│  Job 2: deploy                   │
│  • SSH a EC2 backend ventas      │
│    VÍA EC2 frontend (bastion)    │
│  • docker pull                   │
│  • Verificar MySQL running       │
│  • docker stop/rm backend        │
│  • docker run nuevo backend      │
│  • docker image prune            │
└──────────────────────────────────┘
```

> **Nota sobre el bastion**: la EC2 del backend ventas está en subred privada (sin IP pública). El workflow usa la EC2 del frontend como jump host (`proxy_host`) para llegar a ella por SSH.

### GitHub Secrets requeridos

Configurar en: `Settings → Secrets and variables → Actions`

| Secret | Descripción |
|---|---|
| `DOCKERHUB_USERNAME` | Usuario de Docker Hub |
| `DOCKERHUB_TOKEN` | Access token de Docker Hub |
| `EC2_FRONTEND_HOST` | IP pública EC2 frontend (actúa como bastion) |
| `EC2_VENTAS_HOST` | IP privada EC2 backend ventas |
| `EC2_USER` | Usuario SSH (ej: `ec2-user`) |
| `EC2_SSH_KEY` | Contenido de la clave privada `.pem` |
| `DB_NAME` | Nombre de la base de datos (`citt_ventas`) |
| `DB_USERNAME` | Usuario de MySQL |
| `DB_PASSWORD` | Contraseña de MySQL |

### Activar despliegue

```bash
git checkout deploy
git push origin deploy
```

---

## Despliegue manual en EC2 backend ventas

```bash
# En la EC2 backend ventas (acceder vía bastion frontend)
docker network create backend-net 2>/dev/null || true

# Levantar MySQL si no está corriendo
docker run -d \
  --name citt-ventas-db \
  --restart unless-stopped \
  --network backend-net \
  -v citt_mysql_ventas_data:/var/lib/mysql \
  -e MYSQL_DATABASE=citt_ventas \
  -e MYSQL_ROOT_PASSWORD=<password> \
  -e MYSQL_USER=innovauser \
  -e MYSQL_PASSWORD=<password> \
  mysql:8.0

# Levantar backend ventas
docker pull TUUSUARIO/citt-ventas-backend:latest
docker run -d \
  --name citt-ventas-backend \
  --restart unless-stopped \
  --network backend-net \
  -p 8082:8082 \
  -e DB_ENDPOINT=citt-ventas-db \
  -e DB_PORT=3306 \
  -e DB_NAME=citt_ventas \
  -e DB_USERNAME=innovauser \
  -e DB_PASSWORD=<password> \
  TUUSUARIO/citt-ventas-backend:latest
```

---

## Arquitectura AWS

```
Internet
    │
    ▼ (solo puerto 80)
[EC2 Frontend – Subred Pública]
    │
    │ Puerto 8082 (regla SG interna)
    ▼
[EC2 Backend Ventas – Subred Privada] ←── Sin acceso desde internet
    │
    │ Red Docker interna (backend-net)
    ▼
[Contenedor MySQL]
    │
    └── Named Volume: citt_mysql_ventas_data
```

---

## Buenas prácticas aplicadas (DevOps)

- **Multi-stage build**: imagen final ~200MB (solo JRE + JAR, sin Maven ni fuentes)
- **Usuario no root**: el proceso Java corre como `appuser`
- **Dependency caching**: `pom.xml` se copia antes que `src/` para cachear `mvn dependency:go-offline`
- **Health check**: endpoint `/api/v1/ventas` verificado en Docker y en Compose
- **`depends_on` con `condition: service_healthy`**: el backend espera a que MySQL esté listo
- **Named volume justificado**: datos de BD persisten entre reinicios de contenedor
- **Secrets en GitHub Actions**: credenciales de BD y AWS nunca en el código
- **Bastion host**: acceso SSH a subred privada respetando arquitectura de seguridad
