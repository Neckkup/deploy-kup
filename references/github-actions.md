# GitHub Actions

ใช้เมื่อ repo อยู่บน GitHub หลักการเหมือนแบบ GitLab ทุกข้อ (build แยกฝั่งตามไฟล์ที่เปลี่ยน, tag SHA + `latest`, deploy ด้วย `latest`, compose และ Dockerfile ชุดเดียวกัน) ต่างกันแค่ไวยากรณ์
compose, Dockerfile และ `.dockerignore` ใช้แม่แบบใน [templates.md](templates.md) ตามเดิม

## เทียบ GitLab กับ GitHub

| GitLab | GitHub Actions |
|---|---|
| `.gitlab-ci.yml` | `.github/workflows/deploy.yml` |
| `rules: changes:` ต่อ job | job `changes` ใช้ `dorny/paths-filter` แล้ว job อื่นเช็ก `if:` จาก output |
| `needs: optional: true` | `needs:` + `if: always() && !failure() && !cancelled()` |
| `resource_group: deploy` | `concurrency: { group: deploy, cancel-in-progress: false }` |
| `environment:` | `environment:` (ตั้ง required reviewers ได้ = กดอนุมัติก่อน deploy) |
| `when: manual` | `environment` ที่มี required reviewers หรือ `workflow_dispatch` |
| CI/CD Variables (Protected, Masked) | Settings → Secrets and variables → Actions: **Secrets** สำหรับค่าลับ, **Variables** สำหรับค่าไม่ลับ |
| ตัวแปรชนิด File (`SSH_PRIVATE_KEY`) | Secret ข้อความ แล้วเขียนลงไฟล์ใน step |
| `CI_COMMIT_SHA` | `github.sha` |
| `CI_REGISTRY_USER` / `CI_REGISTRY_PASSWORD` | GHCR: `github.actor` / `secrets.GITHUB_TOKEN` (ต้องมี `permissions: packages: write`) registry อื่น: secret ที่ตั้งเอง |
| `docker:24-dind` | runner `ubuntu-latest` มี Docker ในตัว ใช้ `docker/setup-buildx-action` |
| Run pipeline | `workflow_dispatch` (กดรันเองได้ใน Actions) |

## สิ่งที่ต้องถามเพิ่ม (นอกจากข้อในขั้น 2 ของ SKILL.md)

- registry: GHCR (`ghcr.io/<owner>/<repo>`) หรือ registry ของทีมเอง (เช่น `registry.example.com`)
- deploy อัตโนมัติ หรือให้รออนุมัติผ่าน environment ที่มี required reviewers

## ล็อกเวอร์ชันของ action

ใช้ tag ของ major version ที่ระบุชัด (`actions/checkout@v4`) ห้ามใช้ `@main` หรือ `@master`
ถ้าผู้ใช้ต้องการเข้มขึ้น ล็อกด้วย commit SHA แล้วใส่ tag ไว้ในคอมเมนต์ (`uses: actions/checkout@<sha> # v4`)

## .github/workflows/deploy.yml: ส่วน build (ใช้ทุกแบบ)

```yaml
name: deploy

on:
  push:
    branches: [main]
  workflow_dispatch:

env:
  REGISTRY: <ghcr.io หรือ registry ของทีม>
  BACKEND_IMAGE: <registry>/<owner>/<repo>/backend
  FRONTEND_IMAGE: <registry>/<owner>/<repo>/frontend

permissions:
  contents: read
  packages: write   # ใช้เมื่อ push ไป GHCR

jobs:
  # ===========================================
  # ดูว่าไฟล์ฝั่งไหนเปลี่ยน กดรันเอง (workflow_dispatch) ถือว่าเปลี่ยนทุกฝั่ง
  # ===========================================
  changes:
    runs-on: ubuntu-latest
    outputs:
      backend: ${{ github.event_name == 'workflow_dispatch' || steps.filter.outputs.backend == 'true' }}
      frontend: ${{ github.event_name == 'workflow_dispatch' || steps.filter.outputs.frontend == 'true' }}
      deploy: ${{ github.event_name == 'workflow_dispatch' || steps.filter.outputs.deploy == 'true' }}
    steps:
      - uses: actions/checkout@v4
      - uses: dorny/paths-filter@v3
        id: filter
        with:
          filters: |
            backend:
              - 'backend/**'
              - '.github/workflows/deploy.yml'
            frontend:
              - 'frontend/**'
              - '.github/workflows/deploy.yml'
            deploy:
              - 'backend/**'
              - 'frontend/**'
              - 'docker-compose.yml'
              - '.github/workflows/deploy.yml'

  # ===========================================
  # Build Backend: build เฉพาะเมื่อ backend/ เปลี่ยน tag เป็น commit และ latest
  # ===========================================
  build-backend:
    needs: changes
    if: needs.changes.outputs.backend == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}                 # registry อื่น: ${{ secrets.REGISTRY_USER }}
          password: ${{ secrets.GITHUB_TOKEN }}         # registry อื่น: ${{ secrets.REGISTRY_PASSWORD }}
      - uses: docker/build-push-action@v6
        with:
          context: ./backend
          push: true
          tags: |
            ${{ env.BACKEND_IMAGE }}:${{ github.sha }}
            ${{ env.BACKEND_IMAGE }}:latest
          cache-from: type=gha,scope=backend
          cache-to: type=gha,scope=backend,mode=max

  # build-frontend: เหมือน build-backend เปลี่ยน backend → frontend
  # ค่าที่ฝังตอน build (VITE_*, NEXT_PUBLIC_*) ใส่ใน build-args: |
  #   VITE_API_URL=${{ vars.VITE_API_URL }}
```

cache แบบ `type=gha` เก็บ cache ของทุก stage ไว้ใน GitHub ไม่ต้องตั้ง registry เพิ่ม

## deploy แบบ A (Linux server/VM ผ่าน SSH) และ C

```yaml
  deploy:
    needs: [changes, build-backend, build-frontend]
    if: always() && !failure() && !cancelled() && needs.changes.outputs.deploy == 'true'
    runs-on: ubuntu-latest
    concurrency: { group: deploy, cancel-in-progress: false }
    environment: { name: production, url: "https://<domain>" }
    env:
      DEPLOY_HOST: ${{ vars.DEPLOY_HOST }}
      DATABASE_URL: ${{ secrets.DATABASE_URL }}
      # ทุก ${VAR} ใน docker-compose.yml ต้องส่งมาที่นี่ทีละตัว (ค่าลับจาก secrets, ค่าไม่ลับจาก vars)
    steps:
      - uses: actions/checkout@v4
      - name: Set up SSH
        run: |
          install -d -m 700 ~/.ssh
          printf '%s\n' "${{ secrets.SSH_PRIVATE_KEY }}" > ~/.ssh/id_deploy
          chmod 600 ~/.ssh/id_deploy
          printf '%s\n' "${{ secrets.SSH_KNOWN_HOSTS }}" > ~/.ssh/known_hosts
          printf 'Host *\n  IdentityFile ~/.ssh/id_deploy\n' > ~/.ssh/config
      - uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - name: Deploy
        env:
          DOCKER_HOST: ssh://${{ vars.DEPLOY_HOST }}
        run: |
          echo "Step 1: Pulling images on ${DEPLOY_HOST}..."
          docker compose pull
          echo "Step 2: Starting services..."
          docker compose up -d --no-build --wait --remove-orphans
          docker compose ps
          echo "Step 3: Removing unused images..."
          docker image prune -f
          echo "URLs:"
          echo "  - Frontend: https://<domain>"
          echo "  - Backend:  https://<domain>/api"
```

- `BACKEND_IMAGE`/`FRONTEND_IMAGE` ใน `env:` ระดับ workflow ส่งต่อให้ compose อัตโนมัติ
- ต่างจาก GitLab: ตัวแปรของ GitHub ไม่เข้า environment ของ step เอง ต้องเขียนใน `env:` ของ job ทีละตัว ไม่งั้น compose ได้ค่าว่าง
- GHCR เป็น private โดยค่าเริ่มต้น เครื่องปลายทางดึงได้เพราะ login อยู่ในฝั่ง client (`DOCKER_HOST=ssh://`) แต่ token ของ `GITHUB_TOKEN` ใช้ได้แค่ระหว่างที่ job รัน

**แบบ C (self-hosted runner บนเครื่องปลายทาง):** job เดียวกันแต่ `runs-on: [self-hosted, <label>]` ลบ step `Set up SSH` และ `DOCKER_HOST` ออก
ถ้าเครื่องเป็น Windows ให้ติดตั้ง runner ใน WSL2 จะได้ใช้ script bash ชุดเดิม ถ้าติดตั้งบน Windows ตรง ๆ ต้องใส่ `shell: bash` (Git Bash) ใน step

## deploy แบบ B (Swarm ผ่าน Swarmpit)

```yaml
  deploy:
    needs: [changes, build-backend, build-frontend]
    if: always() && !failure() && !cancelled() && needs.changes.outputs.deploy == 'true'
    runs-on: ubuntu-latest
    concurrency: { group: deploy, cancel-in-progress: false }
    env:
      SWARMPIT_STACK_NAME: <stack>
      SWARMPIT_URL: ${{ vars.SWARMPIT_URL }}
      SWARMPIT_TOKEN: ${{ secrets.SWARMPIT_TOKEN }}
      DATABASE_URL: ${{ secrets.DATABASE_URL }}
    steps:
      - uses: actions/checkout@v4
      - name: Deploy via Swarmpit
        run: |
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
```

runner `ubuntu-latest` มี `jq`, `curl` และ `envsubst` อยู่แล้ว

## ตรวจเพิ่ม (นอกจากขั้น 5 ของ SKILL.md)

1. `if:` ของ deploy มี `always() && !failure() && !cancelled()` ไม่งั้น deploy ถูกข้ามเมื่อ build ฝั่งใดฝั่งหนึ่งไม่ได้รัน
2. ทุก `${VAR}` ใน `docker-compose.yml` มีใน `env:` ของ job deploy
3. path ใน `filters` ของ `paths-filter` มีอยู่จริง และมีชื่อไฟล์ workflow เอง
4. ถ้ามี `actionlint` บนเครื่อง ให้รัน `actionlint .github/workflows/deploy.yml`
