# Proyecto Tareas — Entrega 2

Aplicación web para gestionar tareas mediante una API desarrollada con Python y Flask. La aplicación utiliza PostgreSQL como segundo servicio y se ejecuta con Docker Compose. En esta entrega se integra Jenkins como gestor del pipeline de integración continua.

## Repositorio

Repositorio oficial de esta entrega:

<https://github.com/catalinaseverino/proyecto-tareas>

## Arquitectura

La solución está compuesta por:

- `app`: API Flask que recibe las solicitudes HTTP y administra las tareas.
- `db`: PostgreSQL 16.6, encargado de almacenar las tareas.
- `jenkins`: servidor Jenkins utilizado para ejecutar el pipeline de integración continua.

La aplicación se conecta a PostgreSQL mediante el nombre interno del servicio `db`, no mediante `localhost` ni una dirección IP fija. Ambos servicios de la aplicación se comunican dentro de la red Docker `tareas-net`.

## Estructura del repositorio

```text
proyecto-tareas-entrega2/
├── app/
│   ├── Dockerfile
│   ├── __init__.py
│   ├── app.py
│   └── database.py
├── tests/
│   └── test_app.py
├── jenkins/
│   └── Dockerfile
├── .github/workflows/ci.yml
├── .dockerignore
├── .env.example
├── .gitignore
├── docker-compose.yml
├── docker-compose.jenkins.yml
├── Jenkinsfile
├── docs/
│   ├── arquitectura-entrega2.svg
│   ├── estructura-jenkins.svg
│   └── estructura-repositorio.svg
├── requirements.txt
└── README.md
```

PostgreSQL utiliza directamente la imagen oficial `postgres:16.6-alpine`, por lo que no necesita una carpeta propia dentro del repositorio. Jenkins se construye a partir de `jenkins/Dockerfile` y utiliza un volumen para conservar su configuración.

## Requisitos

- Docker Desktop con Docker Compose v2.
- Git.
- Para ejecutar pruebas locales fuera de Docker: Python 3.12 o compatible.
- Para administrar Jenkins: un navegador web.

## Ejecutar la aplicación con Docker

Desde la carpeta raíz del repositorio:

```bash
cp .env.example .env
```

Edite `.env` y defina una contraseña local. Este archivo no se sube al repositorio porque está incluido en `.gitignore`.

Construir las imágenes:

```bash
docker compose build
```

Levantar los servicios y esperar a que estén saludables:

```bash
docker compose up -d --wait
```

Consultar el estado:

```bash
docker compose ps
```

La API estará disponible en `http://localhost:5000`. Si el puerto 5000 está ocupado, configure otro puerto externo en `.env`, por ejemplo:

```env
APP_PORT=5001
```

El puerto interno de la aplicación continúa siendo el 5000. Para acceder con el puerto alternativo se usaría `http://localhost:5001`.

## Endpoints de la API

| Método | Endpoint | Descripción |
|---|---|---|
| GET | `/` | Información general de la API |
| GET | `/health` | Verifica que la aplicación esté activa |
| POST | `/tareas` | Crea una tarea |
| GET | `/tareas` | Lista las tareas guardadas |
| PATCH | `/tareas/<id>/completar` | Marca una tarea como completada |

## Prueba de comunicación entre contenedores

La siguiente prueba demuestra una operación real de ida y vuelta: el `POST` guarda una tarea en PostgreSQL y el `GET` la recupera. La aplicación utiliza `db` como host de PostgreSQL dentro de Docker.

Con el puerto externo predeterminado:

```bash
curl -X POST http://localhost:5000/tareas \
  -H "Content-Type: application/json" \
  -d '{"titulo":"Prueba de comunicación"}'

curl http://localhost:5000/tareas
```

Si configuró `APP_PORT=5001`, reemplace `5000` por `5001` en los comandos.

También puede comprobar la red creada por Compose:

```bash
docker network inspect tareas-net
```

En la salida deben aparecer los contenedores `proyecto-tareas-app` y `proyecto-tareas-db`.

## Persistencia y limpieza

PostgreSQL utiliza el volumen `tareas-data` para conservar las tareas aunque se eliminen los contenedores.

Detener los contenedores y conservar los datos:

```bash
docker compose down
```

Detener los contenedores y eliminar también el volumen y los datos:

```bash
docker compose down -v --remove-orphans
```

## Pruebas locales

Para ejecutar las pruebas unitarias directamente en el entorno Python:

```bash
pip install -r requirements.txt
python -m pytest
```

Las pruebas están en `tests/test_app.py`.

## Jenkins

Jenkins se ejecuta en un contenedor independiente, con Docker CLI y Docker Compose disponibles para construir y probar los servicios del proyecto. El volumen `jenkins_home` conserva la configuración, los plugins, los jobs y las credenciales de Jenkins.

Construir la imagen personalizada de Jenkins:

```bash
docker compose -f docker-compose.jenkins.yml build
```

Levantar Jenkins:

```bash
docker compose -f docker-compose.jenkins.yml up -d
```

Abrir Jenkins en el navegador:

<http://localhost:8080>

Consultar los logs si se necesita obtener la contraseña inicial o revisar el arranque:

```bash
docker compose -f docker-compose.jenkins.yml logs -f jenkins
```

Detener Jenkins sin borrar su configuración:

```bash
docker compose -f docker-compose.jenkins.yml down
```

### Configuración del job

El job de Jenkins se configura como `Pipeline` y utiliza el pipeline definido en el archivo `Jenkinsfile` del repositorio:

- Repositorio: `https://github.com/catalinaseverino/proyecto-tareas.git`.
- Rama: `*/main`.
- Script Path: `Jenkinsfile`.
- Credencial de GitHub: guardada en Jenkins, no escrita en el código.
- Disparador: `Poll SCM` con la expresión `H/2 * * * *`.

El disparador revisa periódicamente el repositorio y ejecuta el pipeline cuando detecta un nuevo commit. También se puede usar un webhook de GitHub si Jenkins está disponible desde Internet.

### Etapas del pipeline

El archivo `Jenkinsfile` contiene las siguientes etapas:

1. **Checkout:** descarga el código desde GitHub.
2. **Build:** construye la imagen de la aplicación y prepara la imagen oficial de PostgreSQL mediante `docker compose build`.
3. **Test:** levanta los servicios, espera sus estados saludables y ejecuta una prueba automática POST/GET.

La prueba de integración se ejecuta dentro del contenedor `app`. Crea una tarea mediante la API y luego la consulta para verificar que la información fue guardada y recuperada desde PostgreSQL. Si la comunicación falla, el build falla.

El bloque `post` ejecuta siempre:

```groovy
docker compose down -v --remove-orphans
```

Así se limpian los contenedores y volúmenes temporales al finalizar cada build.

### Credenciales de Jenkins

Las credenciales se almacenan desde **Manage Jenkins → Credentials**:

- Credencial de GitHub para descargar el repositorio.
- Credencial secreta `postgres-ci-password` para la contraseña usada durante el pipeline.

Ninguna contraseña o token real se incluye en el repositorio.

## GitHub Actions

El workflow `.github/workflows/ci.yml` se conserva como verificación adicional y se ejecuta en cada `push` y `pull_request`. Realiza las siguientes acciones:

1. Instala las dependencias de Python.
2. Ejecuta las pruebas unitarias.
3. Construye las imágenes Docker.
4. Levanta la aplicación y PostgreSQL.
5. Ejecuta la prueba POST/GET de comunicación.
6. Muestra los logs si la integración falla.
7. Detiene y limpia los servicios.

La entrega se centra en Jenkins, pero GitHub Actions permite mantener una segunda validación automática del proyecto.

## Seguridad y buenas prácticas

- Las imágenes utilizan versiones explícitas; no se usa `latest`.
- `.dockerignore` evita incluir archivos innecesarios en la imagen.
- `.env` está excluido del repositorio mediante `.gitignore`.
- `.env.example` documenta las variables necesarias sin exponer secretos reales.
- Las credenciales de GitHub y PostgreSQL se almacenan en Jenkins.
- La comunicación entre la aplicación y la base de datos utiliza el nombre de servicio `db` dentro de la red Docker.

