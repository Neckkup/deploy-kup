---
name: deploy-kup
description: วางโครงหรือปรับ CI/CD ของ GitLab, docker compose, Dockerfile และ .dockerignore ของ repo ที่แยก backend/frontend ให้ build image ขึ้น registry แล้ว deploy ไปเครื่องเดียว (compose ผ่าน SSH) หรือ Docker Swarm (Swarmpit) หลัง Traefik ใช้เมื่อผู้ใช้ขอให้ทำ .gitlab-ci.yml, ตั้ง pipeline, เขียน docker-compose สำหรับ server หรือเครื่องพัฒนา, เขียน Dockerfile, ย้ายการ deploy หรือตรวจของเดิมว่าตรงตามแบบไหม
---

# deploy-kup

วางไฟล์ build และ deploy ของ repo ให้เป็นแบบเดียวกันทุกโปรเจกต์: GitLab CI build image แยกฝั่ง push ขึ้น registry แล้ว deploy stack ที่อยู่หลัง Traefik
แบบนี้มาจาก repo ที่ใช้งานจริง (`wrf/water-disasters`, `wrf/next.js-example` และ steamkid) แม่แบบทุกไฟล์อยู่ใน [references/templates.md](references/templates.md)

## กฎสำคัญ

- **สงสัยเมื่อไหร่ ห้ามเดา ให้ถาม** ด้วยเครื่องมือ **AskUserQuestion** (หน้าต่างเด้ง) ห้ามถามเป็นข้อความหรือตารางในแชท
  - คำถามต้องอ่านเข้าใจได้เอง ให้ตัวเลือก 2–4 ข้อ ข้อที่แนะนำไว้ข้อแรกต่อท้าย "(Recommended)" รวมไม่เกิน 4 คำถามต่อครั้ง
  - ค่าที่อ่านจาก repo ได้ (ชื่อ group/project, พอร์ต, คำสั่ง build) อ่านเอง ไม่ต้องถาม
  - ถ้าผู้ใช้ปฏิเสธหน้าต่างคำถาม ให้หยุดและรอคำสั่ง
- **ผู้ใช้วางตัวอย่างมา ให้ทำตามรูปแบบนั้น** ถามเฉพาะค่าที่ไม่รู้ ถ้าตัวอย่างมีจุดที่ขัดกับ "ห้ามทำ" ข้างล่าง ให้บอกแล้วถามว่าจะตามตัวอย่างหรือแก้
- **สรุปแผนแล้วรอคำสั่งก่อนแก้ไฟล์** ตอบคำถามใน popup อย่างเดียวยังไม่นับเป็นคำสั่ง
- **ห้ามใส่ค่าลับจริง** ลงไฟล์ใด ๆ ค่าลับอยู่ใน CI/CD Variables เท่านั้น

## โครงเป้าหมาย

```
.gitlab-ci.yml              ← build-backend, build-frontend, deploy
docker-compose.yml          ← server: ดึง image จาก registry, label ของ Traefik, network ภายนอก
docker-compose.local.yml    ← ทั้ง stack บนเครื่องพัฒนา build จาก source, ค่าจาก backend/.env
backend/
├── Dockerfile              ← multi-stage, ล็อกเวอร์ชัน, user ไม่ใช่ root, HEALTHCHECK
├── .dockerignore
├── .env.example            ← ทุกตัวแปรที่ backend อ่าน ค่าสำหรับเครื่องพัฒนาเท่านั้น
└── docker-compose.yml      ← (ถ้า dev รัน app บนเครื่อง) เฉพาะ DB, S3, mail ฯลฯ
frontend/
├── Dockerfile
└── .dockerignore
docs/ci-cd.md               ← เขียนด้วยสกิล docs-kup (หัวข้อ ci-cd.md)
```

ถ้า repo ไม่ได้แยก backend/frontend หรือมี service มากกว่านี้ ใช้ชื่อโฟลเดอร์ตามที่มีจริง หนึ่งโฟลเดอร์ต่อหนึ่ง image ต่อหนึ่ง job build

## ปลายทาง deploy (ถามผู้ใช้ถ้ายังไม่รู้)

| | A. เครื่องเดียว | B. Docker Swarm |
|---|---|---|
| ใช้เมื่อ | EC2/VM เครื่องเดียว ไม่มี Swarm | cluster ที่มี Swarmpit (เช่น coe-swarm) |
| job deploy | `docker:<ver>` ตั้ง `DOCKER_HOST=ssh://$DEPLOY_HOST` แล้ว `docker compose pull` → `up -d --wait` | `alpine` + `curl jq gettext` ส่ง compose ที่ `envsubst` แล้วไป Swarmpit API แล้วสั่ง redeploy |
| compose | `restart: unless-stopped`, `depends_on` + healthcheck, migrate เป็น service ที่รันจบ | `deploy:` (replicas, placement, `update_config` + `failure_action: rollback`, `restart_policy`), label อยู่ใต้ `deploy.labels` |
| ไฟล์ถาวร | volume ของเครื่อง หรือ S3 | volume NFS (`driver_opts`) |
| CI variables | `DEPLOY_HOST`, `SSH_PRIVATE_KEY` (File), `SSH_KNOWN_HOSTS` | `SWARMPIT_URL`, `SWARMPIT_TOKEN` |

## ขั้นตอน

### 1. สำรวจ (อ่านอย่างเดียว)

- `git status`, `git log -5`, `git remote -v` (ได้ host ของ GitLab และ path ของ registry)
- ไฟล์เดิม: `.gitlab-ci.yml`, compose ทุกตัว, Dockerfile, `.dockerignore`, `.env.example`, `.gitignore`
- แต่ละฝั่ง: ภาษา/framework, คำสั่ง build และ start, พอร์ต, endpoint สุขภาพ (`/healthz`, `/readyz`, หน้า login), process แยก (worker, migrate, cron)
- ตัวแปรที่โค้ดอ่านจริง: `grep -rhoE 'Getenv\("[A-Z_]+"|process\.env\.[A-Z_]+|import\.meta\.env\.[A-Z_]+'` เทียบกับ `.env.example`
- ค่าที่ฝัง ณ ตอน build (`VITE_*`, `NEXT_PUBLIC_*`) ต้องส่งเป็น `--build-arg` ไม่ใช่ env ของ container

### 2. ถามค่าที่ไม่รู้

ด้วย AskUserQuestion ในครั้งเดียว เฉพาะที่อ่านจาก repo ไม่ได้:
- ปลายทาง deploy (A หรือ B) และชื่อ stack
- domain ของแต่ละ service และแบบเส้นทาง (แยก subdomain หรือ path เดียวกัน เช่น `/api/v1` → backend)
- network ภายนอก (`public`, network ของ DB), entrypoint และ certresolver ของ Traefik
- deploy อัตโนมัติหรือกดเอง (`when: manual`)

### 3. สรุปแผนแล้วรอคำสั่ง

ในข้อความเดียว:
- tree ของไฟล์ที่จะสร้าง/แก้
- ตาราง job: ชื่อ, stage, รันเมื่อไฟล์ไหนเปลี่ยน, ทำอะไร
- ตาราง service ใน compose: image, network, domain/path, healthcheck
- ตาราง CI/CD variables ที่ผู้ใช้ต้องตั้ง (ชื่อ, ชนิด, Protected/Masked)
- จุดที่ของเดิมต่างจากแบบนี้ และจะแก้หรือคงไว้

แล้ว **หยุด** จนกว่าผู้ใช้จะสั่งให้ลงมือ

### 4. เขียนไฟล์

ตามแม่แบบใน [references/templates.md](references/templates.md) และข้อตกลงข้างล่าง คอมเมนต์ในไฟล์ใช้ภาษาเดียวกับคอมเมนต์ที่มีอยู่ใน repo

### 5. ตรวจ

1. `docker compose -f docker-compose.yml config` ผ่าน ทั้งตอนไม่ตั้งตัวแปร (ได้ path image ค่าสำรอง) และตอนตั้ง `BACKEND_IMAGE`/`FRONTEND_IMAGE`
2. `docker compose -f docker-compose.local.yml config` ผ่าน
3. `docker build backend` และ `docker build frontend` ผ่าน ถ้าไม่ได้รันจริงให้บอกผู้ใช้
4. บรรทัด `script:` ที่มี `: ` (เช่น `echo "URL: ..."`) ต้องครอบทั้งบรรทัดด้วย `'...'` ไม่งั้น YAML อ่านเป็น map แล้ว pipeline ไม่ขึ้น
5. ทุก `${VAR}` ใน compose มีในตาราง CI/CD variables และใน `backend/.env.example`
6. ทุก path ใน `rules.changes` มีอยู่จริง

### 6. เอกสารและรายงาน

- เขียนหรืออัปเดต `docs/ci-cd.md` ตามสกิล **docs-kup** (หัวข้อ ci-cd.md ใน page-templates) และแผนผังโฟลเดอร์ใน `README.md` ที่ root
- รายงานผู้ใช้: ไฟล์ที่สร้าง/แก้, CI/CD variables ที่ต้องตั้งเอง, สิ่งที่ต้องเตรียมบนเครื่องปลายทาง (network, Traefik, DNS, SSH key), สิ่งที่ยังไม่ได้ทดสอบ
- commit เฉพาะเมื่อผู้ใช้สั่ง

## ข้อตกลงที่ใช้เสมอ

**CI**
- job build หนึ่งตัวต่อหนึ่งโฟลเดอร์ `rules` = branch `main` + `changes: [<dir>/**, .gitlab-ci.yml]`
- tag image สองตัว: `${CI_COMMIT_SHA}` และ `latest` deploy ใช้ `latest`
- path ของ image ตั้งเป็นตัวแปรที่ `variables:` ของ CI (`BACKEND_IMAGE`, `FRONTEND_IMAGE`) แล้ว compose อ่านตัวเดียวกัน
- job deploy `rules.changes` = ทุกโฟลเดอร์ที่ build + `docker-compose.yml` + `.gitlab-ci.yml` และ `needs` job build แบบ `optional: true`
- แบบ A ใส่ `resource_group: deploy` และ `environment` (name + url)
- ปิดท้าย deploy ด้วย echo URL ของแต่ละ service

**compose ของ server**
- image: `${BACKEND_IMAGE:-<registry>/<group>/<project>/backend}:latest`
- env ที่ไม่ลับและไม่เปลี่ยนเขียนค่าตรง ๆ (`TZ=Asia/Bangkok`, domain, bucket) ค่าลับหรือค่าต่อเครื่องเขียน `VAR=${VAR}`
- service ที่ใช้ env ชุดเดียวกัน (api, worker, migrate) ใช้ anchor `x-backend: &backend`
- network ภายนอกประกาศ `external: true` service ที่รับ traffic อยู่ใน `public` และตั้ง `traefik.docker.network: public` (แบบ A) เมื่ออยู่หลาย network
- ชื่อ router/service ของ Traefik ขึ้นต้นด้วยชื่อโปรเจกต์ (และ env เช่น `-uat`) กันชนกับ stack อื่น
- service ที่เรียกกันภายในใช้ alias ที่ขึ้นต้นด้วยชื่อโปรเจกต์ (เช่น `steamkid-backend`) เพราะ network `public` มีหลาย stack ที่มี service ชื่อ `backend`

**compose ของเครื่องพัฒนา**
- `build:` จาก source, `env_file: ./backend/.env`, override เฉพาะค่าที่ต่างจากตอนรันนอก docker (host ของ DB เป็นชื่อ service)
- เปิด port ออกเครื่อง, DB/S3/mail ใช้ image ล็อกเวอร์ชัน และ healthcheck

**Dockerfile**
- multi-stage: deps → build → runtime, ล็อกเวอร์ชันของ base image (`node:24-alpine3.23` ไม่ใช่ `node:latest`)
- ติดตั้ง dependency ด้วย lockfile แบบเข้ม (`npm ci`, `pnpm install --frozen-lockfile`, `go mod download`)
- runtime รันด้วย user ที่ไม่ใช่ root และมี `HEALTHCHECK`
- ไฟล์ที่ดาวน์โหลดตอน build ล็อกเวอร์ชันและตรวจ sha256
- `.dockerignore` ตัด `node_modules`, ผล build, `.env*`, log, `.git`

## ห้ามทำ (เจอในตัวอย่างเดิม อย่าคัดลอกต่อ)

- `image: postgres:latest`, `alpine:latest` หรือ base image ที่ไม่ล็อกเวอร์ชัน
- ค่าเริ่มต้นที่เป็นรหัสผ่านจริงใน compose (`${PASSWORD:-รหัสจริง}`)
- `bun install --frozen-lockfile || bun install` หรือ fallback อื่นที่ข้าม lockfile
- `curl ... | bash` เพื่อติดตั้งเครื่องมือโดยไม่ล็อกเวอร์ชัน
- `version: '3.8'` ใน compose (compose เลิกใช้แล้ว)
- ใส่ secret เป็น `--build-arg` (ค้างอยู่ใน layer ของ image)
- `cat docker-compose.yml` ส่งไป Swarmpit ทั้งที่ไฟล์มี `${VAR}` (ต้อง `envsubst` ก่อน)
