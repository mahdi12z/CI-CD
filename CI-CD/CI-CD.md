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
