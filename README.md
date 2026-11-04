# deploy-kup

สกิลของ Claude Code สำหรับวางโครงหรือปรับ CI/CD ของ GitLab CI หรือ GitHub Actions, docker compose, Dockerfile และ .dockerignore ของ repo ที่แยก backend/frontend ให้ build image ขึ้น registry แล้ว deploy ไปเครื่องเดียว (compose ผ่าน SSH) หรือ Docker Swarm (Swarmpit) หลัง Traefik

ขั้นตอนและข้อตกลงอยู่ใน [SKILL.md](SKILL.md) แม่แบบอยู่ใน [references/templates.md](references/templates.md) และส่วน CI ของ GitHub อยู่ใน [references/github-actions.md](references/github-actions.md)
เอกสาร `docs/ci-cd.md` ของ repo เขียนด้วยสกิล [docs-kup](https://github.com/Neckkup/docs-kup)

## ติดตั้ง

ใช้กับทุกโปรเจกต์บนเครื่อง:

```
git clone https://github.com/Neckkup/deploy-kup.git ~/.claude/skills/deploy-kup
```

ใช้เฉพาะโปรเจกต์เดียว:

```
git clone https://github.com/Neckkup/deploy-kup.git <project>/.claude/skills/deploy-kup
```

## ใช้งาน

ใน Claude Code พิมพ์ `/deploy-kup` หรือขอแบบปกติ เช่น "ทำ CI/CD กับ docker compose ให้ repo นี้" หรือ "ตรวจ .gitlab-ci.yml ว่าตรงแบบไหม"
