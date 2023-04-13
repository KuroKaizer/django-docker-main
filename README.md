# Django dentro de contenedores Docker

Este repositorio organiza una estructura para llevar un proyecto de Django a entorno de producción, en la configuración
agregaremos Nginx y Gunicorn.

### Requerimientos:

- [Docker](https://docs.docker.com/engine/install/ubuntu/)
- [Docker-compose](https://docs.docker.com/compose/install/#install-compose-as-standalone-binary-on-linux-systems)

## Configuración del proyecto

Crea un nuevo proyecto de django desde tu entorno de trabajo. (Recuerda tener un entorno virtual activo).

```shell
$ mkdir django-docker && cd django-docker
$ python3.9 -m venv env
$ source env/bin/activate
(env)$

(env)$ pip install django==3.2.12
(env)$ django-admin.py startproject quickstart .
(env)$ python manage.py migrate
(env)$ python manage.py runserver
```

Cree una estructura para requirements dividido en `base.txt`, `dev.txt`, `production.txt`
Este es el contendido de `requirements/base.txt`

```shell
# Django
# -----------------------------------------------------------------------------
django==3.2.12  # pyup: < 4.0  # https://www.djangoproject.com/
```

## Configuración de docker

Instale [Docker](https://docs.docker.com/engine/install/ubuntu/), si aún no lo tiene, luego cree un Dockerfile en el directorio `resources/django/` con el siguente
contenido:

```shell
ARG PYTHON_VERSION=3.9-slim-buster

# define an alias for the specfic python version used in this file.
FROM python:${PYTHON_VERSION} as python

# Python build stage
FROM python as python-build-stage

ARG BUILD_ENVIRONMENT=production

# Install apt packages
RUN apt-get update && apt-get install --no-install-recommends -y \
  # dependencies for building Python packages
  build-essential \
  # psycopg2 dependencies
  libpq-dev

# Requirements are installed here to ensure they will be cached.
COPY ./requirements .

# Create Python Dependency and Sub-Dependency Wheels.
RUN pip wheel --wheel-dir /usr/src/app/wheels  \
  -r ${BUILD_ENVIRONMENT}.txt


# Python 'run' stage
FROM python as python-run-stage

ARG BUILD_ENVIRONMENT=production
ARG APP_HOME=/app

ENV PYTHONUNBUFFERED 1
ENV PYTHONDONTWRITEBYTECODE 1
ENV BUILD_ENV ${BUILD_ENVIRONMENT}

WORKDIR ${APP_HOME}

# Install required system dependencies
RUN apt-get update && apt-get install --no-install-recommends -y \
  # psycopg2 dependencies
  libpq-dev \
  # Translations dependencies
  gettext \
  # cleaning up unused files
  && apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false \
  && rm -rf /var/lib/apt/lists/*

# All absolute dir copies ignore workdir instruction. All relative dir copies are wrt to the workdir instruction
# copy python dependency wheels from python-build-stage
COPY --from=python-build-stage /usr/src/app/wheels  /wheels/

# use wheels to install python dependencies
RUN pip install --no-cache-dir --no-index --find-links=/wheels/ /wheels/* \
	&& rm -rf /wheels/

COPY ./resources/django/entrypoint /entrypoint
RUN chmod +x /entrypoint

ENTRYPOINT ["/entrypoint"]
```

Utilizaremos `3.9-slim-buster`como imagen base de `python`, establecemos un directorio de trabajo, copiamos el
directorio requirements e instalarmos las dependecias para el entorno de producción.

A continuación crea un archivo `docker-compose.yaml` en la raíz del proyecto:

```shell
version: '3.7'

services:
  postgres:
    image: postgres:alpine
    container_name: postgres-quickstart
    restart: always
    environment:
      POSTGRES_DB: ${POSTGRES_DB:-djang_quickstart}
      POSTGRES_USER: ${POSTGRES_USER:-some_user}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-super_secret}
    volumes:
      - /data/django_quickstart:/var/lib/postgresql/data:rw

  django:
    build:
      context: .
      dockerfile: ./resources/django/Dockerfile
    image: django:production
    container_name: django-quickstart
    restart: always
    environment:
      POSTGRES_DB: ${POSTGRES_DB:-djang_quickstart}
      POSTGRES_USER: ${POSTGRES_USER:-some_user}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-super_secret}
      POSTGRES_HOST: ${POSTGRES_HOST:-postgres}
      POSTGRES_PORT: ${POSTGRES_PORT:-5432}
    command: >
      sh -c "
      python manage.py migrate &&
      python manage.py collectstatic --no-input &&
      gunicorn quickstart.wsgi:application --bind 0.0.0.0:8000 --workers=3
      "
    ports:
      - "8000:8000"
    depends_on:
      - postgres
    volumes:
      - .:/app:z
```

Para configurar Postgres, hemos añadido un nuveo servicio al docker-compose.yaml e actualizamos en
archivo `requirementes/base.txt`

```shell
# Postgres
# -----------------------------------------------------------------------------
psycopg2-binary==2.9.3  # https://github.com/psycopg/psycopg2
```

Actualiza `DATABASE` dictado en settings.py

```shell
# DATABASES
# -----------------------------------------------------------------------------
# https://docs.djangoproject.com/en/2.1/ref/settings/#databases
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        "NAME": os.environ.get("POSTGRES_DB"),
        "USER": os.environ.get("POSTGRES_USER"),
        "PASSWORD": os.environ.get("POSTGRES_PASSWORD"),
        "HOST": os.environ.get("POSTGRES_HOST"),
        "PORT": os.environ.get("POSTGRES_PORT"),
    }
}
DATABASES["default"]["ATOMIC_REQUESTS"] = True  # noqa F405
DATABASES["default"]["CONN_MAX_AGE"] = os.getenv("CONN_MAX_AGE", default=60)  # noqa F405

```

Aquí, la base de datos se configura en función de las variables de entorno, para esta guía hemos añadido los valores en
el mismo `docker-compose.yaml`

## Nginx

A continuación añadiremos un nuevo servicio al `docker-compose.yaml` para que Gunicorn maneje las solicitudes de los
clientes y nginx sirva archivos státicos.

```shell
  nginx:
    image: nginx:alpine
    container_name: nginx-django-quickstart
    restart: always
    volumes:
      - .:/app:z
      - ./resources/nginx/nginx.conf:/etc/nginx/conf.d/default.conf
    ports:
      - "85:80"
    depends_on:
      - django
```
En el directorio `./resources/nginx/` crearemos un archivo `nginx.conf` con el siguente contendido:

```shell
upstream django_quickstart {
    server django:8000;
}

server {
    listen 80;
    location / {
        proxy_pass http://django_quickstart;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        proxy_redirect off;
    }

    location /static/ {
        alias /app/staticfiles/;
    }
}
```


## Comandos de docker

Cree una nueva imagen:

```shell
docker-compose up --build -d
```

Detener servicio

```shell
docker-compose down
```

Puede ver logs de nginx o django con el siguente comando:

```shell
 docker logs django-quickstart
 docker logs nginx-django-quickstart
```


Al finalizar tedrá una estructura similar a esta:

```shell
├── docker-compose.yaml
├── manage.py
├── preview
│   └── docker
│       ├── logs-django.png
│       └── logs-nginx.png
├── quickstart
│   ├── asgi.py
│   ├── __init__.py
│   ├── settings.py
│   ├── urls.py
│   └── wsgi.py
├── README.md
├── requirements
│   ├── base.txt
│   └── production.txt
├── resources
│   ├── django
│   │   ├── Dockerfile
│   │   └── entrypoint
│   ├── docker
│   │   └── docker.md
│   └── nginx
│       └── nginx.conf
└── static

```