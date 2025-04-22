### GitLab CI/CD pipeline

```
stages:
  - build
  - run

variables:
  COMPOSE_PROJECT_NAME: aressfdp
  l: ""  

build:
  stage: build
  script:
    - docker compose build

run:
  stage: run
  script:
    - docker compose up -d
    - sleep 10
    - docker compose logs
    - docker compose down
```
### **What This Pipeline Does**

- **Builds** all services using Docker Compose
    
- **Temporarily runs** the services
    
- **Captures logs** for inspection (great for debugging or CI validation)
    
- **Cleans up** everything afterwards to avoid leftover containers
- 
----

## **Pipeline Overview**

```
stages:
  - build
  - push
```


This pipeline has **two stages**:

1. `build`: Responsible for building Docker images.
2. `push`: Pushes the built images to a Docker registry (in your case, a private Nexus registry).
### Global Variables
```
variables:
  COMPOSE_PROJECT_NAME: aressfdp
  IMAGE_TAG: latest
  REGISTRY_URL: 192.168.100.**:32500
  DOCKER_TLS_CERTDIR: ""

```

- `COMPOSE_PROJECT_NAME`: Sets a project name for Docker Compose to avoid naming collisions.
    
- `IMAGE_TAG`: Default tag for the built Docker images.
    
- `REGISTRY_URL`: URL of your private Docker registry (Nexus in this case).
    
- `DOCKER_TLS_CERTDIR: ""`: Disables Docker's TLS certificate generation. This is required for Docker-in-Docker to work without TLS errors.
 ##  Build Stage
 
 
```
build:
  stage: build
  image: docker:24.0.2
  services:
    - docker:24.0.2-dind

```
Uses docker:24.0.2 as the main image (provides Docker CLI).

Runs docker:24.0.2-dind as a service (Docker-in-Docker) so that Docker commands can run inside the CI job
```
before_script:
    - echo "Logging into Nexus Registry..."
    - echo "$CI_REGISTRY_PASSWORD" | docker login $REGISTRY_URL -u "$CI_REGISTRY_USER" --password-stdin
```
Logs into the Docker registry using credentials stored in GitLab CI/CD environment variables (`CI_REGISTRY_USER` and `CI_REGISTRY_PASSWORD`). 
```
  script:
    - echo "Building all images via Docker Compose..."
    - docker compose build

```

Builds all the Docker images defined in your docker-compose.yml using docker compose build.

 ## Push Stage
 ```
 push:
  stage: push
  image: docker:24.0.2
  services:
    - docker:24.0.2-dind

```
Same setup as the build stage (Docker CLI + Docker-in-Docker service).
```
  before_script:
    - echo "Logging into Nexus for image push..."
    - echo "$CI_REGISTRY_PASSWORD" | docker login $REGISTRY_URL -u "$CI_REGISTRY_USER" --password-stdin

```
Logs into the Docker registry again before pushing images.
```
  script:
    - echo "Pushing all images to Nexus..."
    - docker compose push
```

Pushes all built images to your Nexus registry using docker compose push.
```
stages:
  - build
  - push

variables:
  COMPOSE_PROJECT_NAME: aressfdp
  IMAGE_TAG: latest
  REGISTRY_URL: 192.168.100.**:32500
  DOCKER_TLS_CERTDIR: ""

build:
  stage: build
  image: docker:24.0.2
  services:
    - docker:24.0.2-dind
  before_script:
    - echo "Logging into Nexus Registry..."
    - echo "$CI_REGISTRY_PASSWORD" | docker login $REGISTRY_URL -u "$CI_REGISTRY_USER" --password-stdin
  script:
    - echo "Building all images via Docker Compose..."
    - docker compose build

push:
  stage: push
  image: docker:24.0.2
  services:
    - docker:24.0.2-dind
  before_script:
    - echo "Logging into Nexus for image push..."
    - echo "$CI_REGISTRY_PASSWORD" | docker login $REGISTRY_URL -u "$CI_REGISTRY_USER" --password-stdin
  script:
    - echo "Pushing all images to Nexus..."
    - docker compose push
```

## Summary

This CI/CD pipeline:

- Builds Docker images using Docker Compose.
    
- Logs into a private Nexus registry.
    
- Pushes the built images to that registry.
    
- Uses Docker-in-Docker to make Docker available inside GitLab's containerized CI runner.

----

-----
----
```bash
stages:
  - build
  - push

variables:
  COMPOSE_PROJECT_NAME: aressfdp
  IMAGE_TAG: latest
  REGISTRY_URL: 192.168.100.94:32500
  DOCKER_TLS_CERTDIR: ""

build:
  stage: build
  image: docker:24.0.2
  services:
    - docker:24.0.2-dind
  before_script:
    - echo "Logging into Nexus Registry..."
    - echo "$CI_REGISTRY_PASSWORD" | docker login $REGISTRY_URL -u "$CI_REGISTRY_USER" --password-stdin
  script:
    - echo "Building all images via Docker Compose (production config)..."
    - docker compose -f docker-compose-production.yml build

push:
  stage: push
  image: docker:24.0.2
  services:
    - docker:24.0.2-dind
  before_script:
    - echo "Logging into Nexus for image push..."
    - echo "$CI_REGISTRY_PASSWORD" | docker login $REGISTRY_URL -u "$CI_REGISTRY_USER" --password-stdin
  script:
    - echo "Pushing all images to Nexus (production config)..."
    - docker compose -f docker-compose-production.yml push

```

```bash
version: "3.8"

services:

  aress_postgres:
    image: postgres:16
    shm_size: 1g
    command: postgres -c 'max_connections=${POSTGRES__MAX_CONNECTIONS}' -c 'shared_buffers=256MB'
    container_name: aress_postgres
    restart: always
    ports:
      - "5432:5432"
    logging:
      options:
        max-size: "50m"
    environment:
      POSTGRES_DB: ${POSTGRES__DB}
      POSTGRES_USER: ${POSTGRES__USER}
      POSTGRES_PASSWORD: ${POSTGRES__PASSWORD}
      TZ: ${TIME_ZONE}
      PGTZ: ${TIME_ZONE}
    healthcheck:
      test: [ "CMD-SHELL", "sh -c 'pg_isready -U ${POSTGRES__USER} -d ${POSTGRES__DB}'" ]
      interval: 10s
      timeout: 3s
      retries: 3
    volumes:
      - postgres_volume:/var/lib/aress_postgresql/data
    networks:
      - aress_net

  aress_timescaledb:
    image: timescale/timescaledb:2.16.1-pg16
    container_name: aress_timescaledb
    restart: always
    ports:
      - "5433:5432"
    environment:
      POSTGRES_DB: ${TIMESCALEDB__DB}
      POSTGRES_USER: ${TIMESCALEDB__USER}
      POSTGRES_PASSWORD: ${TIMESCALEDB__PASSWORD}
      TZ: ${TIME_ZONE}
      PGTZ: ${TIME_ZONE}
    healthcheck:
      test: [ "CMD-SHELL", "sh -c 'pg_isready -U ${TIMESCALEDB__USER} -d ${TIMESCALEDB__DB}'" ]
      interval: 10s
      timeout: 3s
      retries: 3
    volumes:
      - timescaledb_volume:/var/lib/aress_timescaledb/data
    networks:
      - aress_net

  aress_redis:
    image: redis:7.4.0
    container_name: aress_redis
    logging:
      options:
        max-size: "50m"
    deploy:
      restart_policy:
        condition: on-failure
        max_attempts: 3
    ports:
      - "${REDIS__PORT}:6379"
    healthcheck:
      test: [ "CMD", "redis-cli", "--raw", "incr", "ping" ]
      interval: 10s
      timeout: 3s
      retries: 3
    networks:
      - aress_net
    volumes:
      - redis_volume:/data
    command: [ sh, -c, "rm -f /data/dump.rdb && redis-server --save '' --dbfilename '' --appendonly no --appendfsync no" ]

  aress_pgadmin:
    image: dpage/pgadmin4:8.11.0
    container_name: aress_pgadmin
    restart: always
    ports:
      - "${PGADMIN__PORT}:80"
    environment:
      PGADMIN_DEFAULT_EMAIL: ${PGADMIN__DEFAULT_EMAIL}
      PGADMIN_DEFAULT_PASSWORD: ${PGADMIN__DEFAULT_PASSWORD}
    entrypoint: /bin/sh -c "chmod 600 /postgres_pgpass; chmod 600 /timescaledb_pgpass; /entrypoint.sh;"
    user: root
    configs:
      - source: pgadmin4_servers.json
        target: /pgadmin4/servers.json
      - source: pgadmin4_postgres_pgpass
        target: /postgres_pgpass
      - source: pgadmin4_timescaledb_pgpass
        target: /timescaledb_pgpass
    volumes:
      - pgadmin_volume:/var/lib/aress_pgadmin
    networks:
      - aress_net

  aress_admin:
    build: .
    image: 192.168.100.94:32500/aressfdp/aress-admin:latest
    container_name: aress_admin
    pull_policy: missing
    logging:
      options:
        max-size: "100m"
    command: bash -c "python manage.py collectstatic --settings=AressFunds.settings --no-input && python manage.py migrate --settings=AressFunds.settings --database ${DJANGO__POSTGRES_ALIAS} && python manage.py migrate --settings=AressFunds.settings --database ${DJANGO__TIMESCALEDB_ALIAS} && gunicorn --workers=2 --threads=2 AressFunds.wsgi -b 0.0.0.0:8000 --timeout=1800 --max-requests 100 --max-requests-jitter 50"
    environment:
      PREFECT_SERVER_ANALYTICS_ENABLED: false
      PREFECT_RESULTS_PERSIST_BY_DEFAULT: false
    ports:
      - "${DJANGO__ADMIN__PORT}:8000"
    depends_on:
      aress_postgres:
        condition: service_healthy
      aress_timescaledb:
        condition: service_healthy
      aress_redis:
        condition: service_healthy
    volumes:
      - files_static:/src/static
      - files_media:/src/media
    networks:
      - aress_net

  aress_api:
    build: .
    image: 192.168.100.94:32500/aressfdp/aress-api:latest
    container_name: aress_api
    environment:
      - DJANGO_SETTINGS_MODULE=AressFunds.settings
    ports:
      - "${API_PORT}:8000"
    depends_on:
      aress_postgres:
        condition: service_healthy
      aress_timescaledb:
        condition: service_healthy
      aress_redis:
        condition: service_healthy
    command: bash -c "uvicorn --workers 1 --host 0.0.0.0 --port 8000 api.aress_api:app"
    volumes:
      - files_static:/src/static
      - files_media:/src/media
    networks:
      - aress_net

  aress_prefect_server:
    build: .
    image: 192.168.100.94:32500/aressfdp/aress-prefect-server:latest
    container_name: aress_prefect_server
    restart: always
    command: bash -c "prefect server start --port ${PREFECT__PORT}"
    environment:
      PREFECT_UI_URL: "http://${PREFECT__HOST}:${PREFECT__PORT}"
      PREFECT_API_URL: "http://localhost:4200/api"
      # If you want to access Prefect Server UI from anywhere other than the Docker host machine, you will need to change
      # PREFECT_UI_URL and PREFECT_API_URL to match the external hostname/IP of the host machine. For example:
      #- PREFECT_UI_URL=http://external-ip:4200/api
      #- PREFECT_API_URL=http://external-ip:4200/api
      PREFECT_SERVER_API_PORT: ${PREFECT__PORT}
      PREFECT_SERVER_API_HOST: ${PREFECT__SERVER_API_HOST}
      PREFECT_SERVER_ANALYTICS_ENABLED: false
      PREFECT_RESULTS_PERSIST_BY_DEFAULT: false
      PREFECT_UI_ENABLED: true
      PREFECT_API_DATABASE_CONNECTION_URL: postgresql+asyncpg://${POSTGRES__USER}:${POSTGRES__PASSWORD}@${POSTGRES__HOST}:5432/${POSTGRES__DB}
      # Uncomment the following line if you want to use the 'S3 Bucket' storage block instead of the older 'S3' storage
      # - EXTRA_PIP_PACKAGES=prefect-aws
    ports:
      - "${PREFECT__PORT}:4200"
    healthcheck:
      test: ["CMD", "curl", "--request", "POST", "-f", "http://${PREFECT__HOST}:${PREFECT__PORT}/api/flow_runs/count"]
      interval: 10s
      timeout: 3s
      retries: 3
    volumes:
      - prefect_volume:/root/.prefect
    depends_on:
      aress_postgres:
        condition: service_healthy
      aress_timescaledb:
        condition: service_healthy
    networks:
      - aress_net

  aress_prefect_worker_tsetmc:
    build: .
    image: 192.168.100.94:32500/aressfdp/aress-prefect-worker-tsetmc:latest
    container_name: aress_prefect_worker_tsetmc
    restart: always
    command: bash -c "prefect worker start --pool tsetmc"
    environment:
      PREFECT_API_URL: "http://${PREFECT__HOST}:${PREFECT__PORT}/api"
      PREFECT_RESULTS_PERSIST_BY_DEFAULT: false
#       Use PREFECT_API_KEY if connecting the agent to Prefect Cloud
#     - PREFECT_API_KEY=YOUR_API_KEY
    depends_on:
      aress_prefect_server:
        condition: service_healthy
    networks:
      - aress_net

  aress_prefect_worker_funds:
    build: .
    image: 192.168.100.94:32500/aressfdp/aress-prefect-worker-funds:latest
    container_name: aress_prefect_worker_funds
    restart: always
    command: bash -c "prefect worker start --pool funds"
    environment:
      PREFECT_API_URL: "http://${PREFECT__HOST}:${PREFECT__PORT}/api"
      PREFECT_RESULTS_PERSIST_BY_DEFAULT: false
    depends_on:
      aress_prefect_server:
        condition: service_healthy
    networks:
      - aress_net

  aress_prefect_worker_reports:
    build: .
    image: 192.168.100.94:32500/aressfdp/aress-prefect-worker-reports:latest
    container_name: aress_prefect_worker_reports
    restart: always
    command: bash -c "prefect worker start --pool reports"
    environment:
      PREFECT_API_URL: "http://${PREFECT__HOST}:${PREFECT__PORT}/api"
      PREFECT_RESULTS_PERSIST_BY_DEFAULT: false
    depends_on:
      aress_prefect_server:
        condition: service_healthy
    networks:
      - aress_net

  aress_prefect_worker_videos:
    build: .
    image: 192.168.100.94:32500/aressfdp/aress-prefect-worker-videos:latest
    container_name: aress_prefect_worker_videos
    restart: always
    command: bash -c "prefect worker start --pool videos"
    environment:
      PREFECT_API_URL: "http://${PREFECT__HOST}:${PREFECT__PORT}/api"
      PREFECT_RESULTS_PERSIST_BY_DEFAULT: false
    depends_on:
      aress_prefect_server:
        condition: service_healthy
    networks:
      - aress_net

  aress_prefect_deploy:
    build: .
    image: 192.168.100.94:32500/aressfdp/aress-prefect-deploy:latest
    container_name: aress_prefect_deploy
    restart: no
    depends_on:
      aress_prefect_worker_tsetmc:
        condition: service_started
      aress_prefect_worker_funds:
        condition: service_started
      aress_prefect_worker_videos:
        condition: service_started
      aress_prefect_worker_reports:
        condition: service_started
    environment:
      PREFECT_API_URL: "http://${PREFECT__HOST}:${PREFECT__PORT}/api"
    entrypoint: [ "bash", "-c", "prefect deploy --all"]
    networks:
      - aress_net




configs:
  pgadmin4_postgres_pgpass:
    content: ${POSTGRES__HOST}:${POSTGRES__PORT}:${POSTGRES__DB}:${POSTGRES__USER}:${POSTGRES__PASSWORD}
  pgadmin4_timescaledb_pgpass:
    content: ${TIMESCALEDB__HOST}:${TIMESCALEDB__PORT}:${TIMESCALEDB__DB}:${TIMESCALEDB__USER}:${TIMESCALEDB__PASSWORD}
  pgadmin4_servers.json:
    content: |
      {
        "Servers": {
          "1": {
            "Group": "Servers",
            "Name": "Postgres",
            "Host": "${POSTGRES__HOST}",
            "Port": ${POSTGRES__PORT},
            "MaintenanceDB": "${POSTGRES__DB}",
            "Username": "${POSTGRES__USER}",
            "PassFile": "/postgres_pgpass",
            "SSLMode": "prefer"
          },
          "2": {
            "Group": "Servers",
            "Name": "TimeScaleDB",
            "Host": "${TIMESCALEDB__HOST}",
            "Port": ${TIMESCALEDB__PORT},
            "MaintenanceDB": "${TIMESCALEDB__DB}",
            "Username": "${TIMESCALEDB__USER}",
            "PassFile": "/timescaledb_pgpass",
            "SSLMode": "prefer"
          }
        }
      }


volumes:
  postgres_volume:
  timescaledb_volume:
  jupyter_volume:
  redis_volume:
  prefect_volume:
  pgadmin_volume:
  files_static:
  files_media:
  files_notebook:
  zookeeper_data:
  zookeeper_log:
  kafka_broker_data:

networks:
  aress_net:
    external: true
```
