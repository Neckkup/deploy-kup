# แม่แบบ

แทน `<...>` ด้วยค่าของโปรเจกต์ ตัดส่วนที่โปรเจกต์ไม่มี (worker, migrate, S3) และเพิ่ม service ตามที่มีจริง
เวอร์ชันของ image ในแม่แบบเป็นตัวอย่าง ให้ใช้เวอร์ชันล่าสุดที่ repo ใช้อยู่หรือที่ผู้ใช้เลือก

## .gitlab-ci.yml: ส่วน build (ใช้ทุกแบบ)

```yaml
stages: [build, deploy]

variables:
  DOCKER_TLS_CERTDIR: ""
  DOCKER_DRIVER: overlay2
  CI_REGISTRY: <registry เช่น registry.rmutsv.app>
  BACKEND_IMAGE: <registry>/<group>/<project>/backend
  FRONTEND_IMAGE: <registry>/<group>/<project>/frontend

# ===========================================
# Build Backend: build เฉพาะเมื่อ backend/ เปลี่ยน tag เป็น commit และ latest
# ===========================================
build-backend:
  stage: build
  image: docker:24-dind
  services: [docker:24-dind]
  rules:
    - if: '$CI_COMMIT_BRANCH == "main" && $CI_PIPELINE_SOURCE =~ /^(push|web)$/'
      changes:
        - backend/**
        - .gitlab-ci.yml
  before_script:
    - echo "$CI_REGISTRY_PASSWORD" | docker login -u "$CI_REGISTRY_USER" --password-stdin "$CI_REGISTRY"
  script:
    - echo "Building backend image..."
    - docker build -t ${BACKEND_IMAGE}:${CI_COMMIT_SHA} ./backend
    - docker push ${BACKEND_IMAGE}:${CI_COMMIT_SHA}
    - docker tag ${BACKEND_IMAGE}:${CI_COMMIT_SHA} ${BACKEND_IMAGE}:latest
    - docker push ${BACKEND_IMAGE}:latest
    - echo "Backend image pushed successfully"

# build-frontend: เหมือน build-backend เปลี่ยน backend → frontend
# ถ้า frontend ฝังค่าตอน build (VITE_*, NEXT_PUBLIC_*) ใส่ --build-arg NAME=${NAME} ทีละตัว
# และ Dockerfile ต้องมี ARG NAME + ENV NAME=$NAME ก่อนคำสั่ง build
```

## .gitlab-ci.yml: deploy แบบ A (Linux server/VM ผ่าน SSH) และ C

```yaml
deploy:
  stage: deploy
  image: docker:24
  resource_group: deploy
  rules:
    - if: '$CI_COMMIT_BRANCH == "main" && $CI_PIPELINE_SOURCE =~ /^(push|web)$/'
      changes:
        - backend/**
        - frontend/**
        - docker-compose.yml
        - .gitlab-ci.yml
      # when: manual   ← ถ้าผู้ใช้ต้องการกดเอง
  needs:
    - { job: build-backend, optional: true }
    - { job: build-frontend, optional: true }
  environment: { name: production, url: "https://<domain>" }
  variables:
    COMPOSE_FILE: docker-compose.yml
  before_script:
    - apk add --no-cache openssh-client
    - install -d -m 700 ~/.ssh
    - install -m 600 "$SSH_PRIVATE_KEY" ~/.ssh/id_deploy
    - echo "$SSH_KNOWN_HOSTS" > ~/.ssh/known_hosts
    - printf 'Host *\n  IdentityFile ~/.ssh/id_deploy\n' > ~/.ssh/config
    - export DOCKER_HOST="ssh://${DEPLOY_HOST:?ตั้งใน CI/CD variables}"
  script:
    - 'echo "Step 1: Pulling images on ${DEPLOY_HOST}..."'
    - echo "$CI_REGISTRY_PASSWORD" | docker login -u "$CI_REGISTRY_USER" --password-stdin "$CI_REGISTRY"
    - docker compose pull
    - 'echo "Step 2: Starting services..."'
    - docker compose up -d --no-build --wait --remove-orphans
    - docker compose ps
    - 'echo "Step 3: Removing unused images..."'
    - docker image prune -f
    - 'echo "URLs:"'
    - 'echo "  - Frontend: https://<domain>"'
    - 'echo "  - Backend:  https://<domain>/api"'
```

compose ถูกแทนค่าตัวแปรที่ฝั่ง CI แล้วส่งให้ Docker บนเครื่องปลายทาง จึงไม่ต้องวางไฟล์หรือ `.env` บนเครื่อง

**แบบ C (runner บนเครื่องปลายทาง):** job เดียวกันแต่
```yaml
deploy:
  tags: [<tag ของ runner บนเครื่องปลายทาง>]
  # ลบ before_script ส่วน SSH และ DOCKER_HOST ทิ้ง docker compose รันบนเครื่องนี้ตรง ๆ
```
runner ต้องเป็น executor แบบ shell (หรือ docker ที่ mount `/var/run/docker.sock`) และ user ของ runner อยู่ในกลุ่ม `docker`

## .gitlab-ci.yml: deploy แบบ B (Swarm ผ่าน Swarmpit)

```yaml
variables:
  SWARMPIT_STACK_NAME: <stack เช่น uat-wrf-project>

deploy:
  stage: deploy
  image: alpine:3.22
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      changes:
        - backend/**
        - frontend/**
        - docker-compose.yml
        - .gitlab-ci.yml
  needs:
    - { job: build-backend, optional: true }
    - { job: build-frontend, optional: true }
  before_script:
    - apk add --no-cache curl jq gettext
  script:
    - |
      echo "Step 1: Updating stack definition in Swarmpit..."
      COMPOSE_CONTENT=$(envsubst < docker-compose.yml | jq -Rs .)
      curl -X POST "${SWARMPIT_URL}/api/stacks/${SWARMPIT_STACK_NAME}" \
        -H "Authorization: Bearer ${SWARMPIT_TOKEN}" \
        -H "Content-Type: application/json" \
        -d "{\"name\":\"${SWARMPIT_STACK_NAME}\",\"spec\":{\"compose\":${COMPOSE_CONTENT}}}" \
        --fail --silent --show-error

      echo "Step 2: Redeploying stack..."
      curl -X POST "${SWARMPIT_URL}/api/stacks/${SWARMPIT_STACK_NAME}/redeploy" \
        -H "Authorization: Bearer ${SWARMPIT_TOKEN}" \
        -H "Content-Type: application/json" \
        --fail --silent --show-error

      echo "Stack ${SWARMPIT_STACK_NAME} updated and redeployed successfully!"
      echo "URLs:"
      echo "  - Frontend: https://<frontend-domain>"
      echo "  - Backend:  https://<backend-domain>"
```

`envsubst` แทนทุก `$VAR` ในไฟล์ ถ้า compose มี `$` ที่ไม่ใช่ตัวแปร (เช่นใน label) ให้จำกัดชื่อ: `envsubst '$BACKEND_IMAGE $DATABASE_URL ...'`

## docker-compose.yml: แบบ A และ C

```yaml
name: <project>

x-backend: &backend
  image: ${BACKEND_IMAGE:-<registry>/<group>/<project>/backend}:latest
  environment:
    - TZ=Asia/Bangkok
    - BASE_DOMAIN=<domain>
    - DATABASE_URL=${DATABASE_URL}
    - <SECRET>=${<SECRET>}

services:
  frontend:
    image: ${FRONTEND_IMAGE:-<registry>/<group>/<project>/frontend}:latest
    restart: unless-stopped
    environment:
      - TZ=Asia/Bangkok
      - API_INTERNAL_URL=http://<project>-backend:8080
    depends_on:
      backend: { condition: service_healthy }
    networks:
      default: { aliases: [<project>-frontend] }
      public: {}
    labels:
      traefik.enable: 'true'
      traefik.docker.network: public
      traefik.http.routers.<project>-frontend.entrypoints: websecure
      traefik.http.routers.<project>-frontend.rule: Host(`<domain>`)
      traefik.http.routers.<project>-frontend.tls: 'true'
      traefik.http.routers.<project>-frontend.tls.certresolver: <certresolver>
      traefik.http.services.<project>-frontend.loadbalancer.server.port: '3000'

  backend:
    <<: *backend
    restart: unless-stopped
    command: ["api"]
    depends_on:
      migrate: { condition: service_completed_successfully }
    networks:
      default: { aliases: [<project>-backend] }
      <db-network>: {}
      public: {}
    labels:
      traefik.enable: 'true'
      traefik.docker.network: public
      traefik.http.routers.<project>-backend.entrypoints: websecure
      traefik.http.routers.<project>-backend.rule: Host(`<domain>`) && PathPrefix(`/api`)
      traefik.http.routers.<project>-backend.tls: 'true'
      traefik.http.routers.<project>-backend.tls.certresolver: <certresolver>
      traefik.http.services.<project>-backend.loadbalancer.server.port: '8080'

  # รันจบแล้วออก ก่อน backend เริ่ม
  migrate:
    <<: *backend
    command: ["admin", "migrate"]
    networks:
      - <db-network>

networks:
  public:
    external: true
  <db-network>:
    external: true
```

## docker-compose.yml: แบบ B (ส่วนที่ต่างจาก A)

```yaml
services:
  backend:
    image: ${BACKEND_IMAGE:-<registry>/<group>/<project>/backend}:latest
    environment:
      - TZ=Asia/Bangkok
      - DATABASE_URL=${DATABASE_URL}
    networks:
      - public
      - <db-network>
    volumes:
      - uploads:/uploads
    tmpfs:
      - /tmp:size=100m,noexec,nosuid
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints: [node.labels.node-cpu == true]
      labels:
        traefik.enable: 'true'
        traefik.http.routers.<project>-backend-<env>.entrypoints: websecure
        traefik.http.routers.<project>-backend-<env>.rule: Host(`<backend-domain>`)
        traefik.http.routers.<project>-backend-<env>.tls: 'true'
        traefik.http.routers.<project>-backend-<env>.tls.certresolver: <certresolver>
        traefik.http.services.<project>-backend-<env>.loadbalancer.server.port: '3000'
      update_config:
        parallelism: 1
        delay: 30s
        failure_action: rollback
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3

volumes:
  uploads:
    driver: local
    driver_opts:
      type: nfs
      o: "addr=<nfs-host>,rw,nolock,soft,timeo=900,retrans=3,rsize=1048576,wsize=1048576,tcp,noatime,nfsvers=4.1"
      device: ':/NFS/<path>/<project>/uploads'
```

- Swarm ไม่รองรับ `depends_on` แบบรอ healthy และไม่มี service ที่รันจบ migration จึงต้องรันตอน backend เริ่ม หรือแยกเป็น job ใน CI
- label ของ Traefik ต้องอยู่ใต้ `deploy.labels` ไม่ใช่ `labels` ของ service

## docker-compose.local.yml

ใช้เมื่อผู้ใช้เลือกเอา compose ของเครื่องพัฒนาในขั้น 2 เท่านั้น

```yaml
# เครื่องพัฒนา: ทั้ง stack ใน docker ค่าตั้งของ backend อยู่ที่ backend/.env (คัดลอกจาก backend/.env.example)
#   docker compose -f docker-compose.local.yml up -d --build
name: <project>

x-backend: &backend
  image: <project>-backend:local
  env_file:
    - ./backend/.env
  environment:
    - DATABASE_URL=postgres://<user>:<pass>@postgres:5432/<db>?sslmode=disable

services:
  frontend:
    build: { context: ./frontend }
    image: <project>-frontend:local
    ports: ["3000:3000"]
    environment:
      - API_INTERNAL_URL=http://backend:8080
    depends_on:
      backend: { condition: service_healthy }

  backend:
    <<: *backend
    build: { context: ./backend }
    command: ["api"]
    ports: ["8080:8080"]
    depends_on:
      migrate: { condition: service_completed_successfully }

  migrate:
    <<: *backend
    command: ["admin", "migrate"]
    depends_on:
      postgres: { condition: service_healthy }

  postgres:
    image: postgres:16
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
      - POSTGRES_DB=<db>
    ports: ["5432:5432"]
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 2s
      retries: 30

volumes:
  pgdata: {}
```

ถ้า dev รัน backend/frontend บนเครื่องเอง (hot reload) ให้มี `backend/docker-compose.yml` ที่มีเฉพาะ postgres, S3, mail ใช้ชื่อ project และ volume เดียวกับไฟล์นี้ แล้วเขียนในคอมเมนต์ว่าเปิดพร้อมกันไม่ได้

## Dockerfile: Go

```dockerfile
# build จากโฟลเดอร์ backend: docker build backend
FROM golang:<ver>-alpine<ver> AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . ./
RUN CGO_ENABLED=0 go build -trimpath -ldflags "-s -w" -o /out/api ./cmd/api

FROM alpine:<ver>
RUN apk add --no-cache ca-certificates tzdata wget && adduser -D -H -u 10001 app
COPY --from=build /out/ /usr/local/bin/
ENV API_ADDR=:8080 TZ=Asia/Bangkok
USER 10001
EXPOSE 8080
HEALTHCHECK --interval=15s --timeout=3s --start-period=20s --retries=3 CMD wget -qO- http://127.0.0.1:8080/healthz >/dev/null || exit 1
ENTRYPOINT ["/usr/local/bin/api"]
```

## Dockerfile: Next.js (standalone)

`next.config` ต้องมี `output: "standalone"`

```dockerfile
FROM node:<ver>-alpine<ver> AS deps
WORKDIR /app
RUN corepack enable
COPY package.json pnpm-lock.yaml ./
RUN pnpm install --frozen-lockfile

FROM node:<ver>-alpine<ver> AS build
WORKDIR /app
RUN corepack enable
COPY --from=deps /app/node_modules ./node_modules
COPY . ./
ENV NEXT_TELEMETRY_DISABLED=1
RUN pnpm build

FROM node:<ver>-alpine<ver>
WORKDIR /app
ENV NODE_ENV=production NEXT_TELEMETRY_DISABLED=1 PORT=3000 HOSTNAME=0.0.0.0 TZ=Asia/Bangkok
RUN apk add --no-cache tzdata wget && adduser -D -H -u 10001 app
COPY --from=build --chown=10001 /app/public ./public
COPY --from=build --chown=10001 /app/.next/standalone ./
COPY --from=build --chown=10001 /app/.next/static ./.next/static
USER 10001
EXPOSE 3000
HEALTHCHECK --interval=15s --timeout=3s --start-period=20s --retries=3 CMD wget -qO- "http://127.0.0.1:${PORT}/" >/dev/null || exit 1
CMD ["node", "server.js"]
```

ใช้ npm แทน pnpm: `COPY package.json package-lock.json ./` + `npm ci` ตัด `public` ออกถ้าไม่มีโฟลเดอร์นี้

## Dockerfile: Vite + nginx (SPA)

```dockerfile
FROM node:<ver>-alpine<ver> AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
# ค่าที่ฝังตอน build ส่งมาจาก CI ด้วย --build-arg
ARG VITE_API_URL
ENV VITE_API_URL=$VITE_API_URL
RUN npm run build

FROM nginx:<ver>-alpine
COPY --from=build /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
HEALTHCHECK --interval=15s --timeout=3s --retries=3 CMD wget -qO- http://127.0.0.1/ >/dev/null || exit 1
```

`nginx.conf` ต้องมี `try_files $uri $uri/ /index.html;` ให้ route ของ SPA ใช้ได้

## .dockerignore

```
node_modules/
.next/
dist/
build/
coverage/
test-results/
.env*
!.env.example
*.log
.git
Dockerfile
.dockerignore
docker-compose*.yml
```

ฝั่ง Go เพิ่ม `bin/`, `*.test`, `*.out` และโฟลเดอร์ที่ไม่ได้ใช้ตอน build (fixtures, scripts)
