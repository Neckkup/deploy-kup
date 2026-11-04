# เตรียม VM ตาม OS (ปลายทางแบบ A)

ใช้หลังผู้ใช้ตอบว่า VM เป็น OS อะไร ส่วนนี้เป็นงานที่ผู้ใช้ทำบนเครื่องเองครั้งเดียว skill ไม่ได้ SSH เข้าไปทำให้
ให้ใส่ขั้นตอนของ OS ที่ผู้ใช้เลือกลงหัวข้อ "ต้องเตรียมครั้งเดียว" ของ `docs/ci-cd.md` และบอกในรายงาน

## ทุก OS

1. ติดตั้ง Docker Engine และ compose plugin (`docker compose version` ต้องได้ v2) ด้วยวิธีของ OS ข้างล่าง ห้ามใช้ `curl ... | sh`
2. `sudo systemctl enable --now docker`
3. user ที่ CI ใช้ SSH เข้า: `sudo usermod -aG docker <user>` แล้ว logout/login ใหม่
4. ใส่ public key ของ deploy key ใน `~<user>/.ssh/authorized_keys` และปิด login ด้วยรหัสผ่าน (`PasswordAuthentication no`)
5. จำกัดขนาด log ของ container ใน `/etc/docker/daemon.json` แล้ว `sudo systemctl restart docker`:
   ```json
   { "log-driver": "local", "log-opts": { "max-size": "10m", "max-file": "3" } }
   ```
6. สร้าง network ภายนอกที่ compose ใช้: `docker network create public` (และ network ของ DB)
7. เปิด port 22 ให้ runner, 80/443 ให้ Traefik ที่ firewall ของ cloud หรือของเครื่อง
8. VM ที่ RAM ไม่ถึง 2 GB ให้เพิ่ม swap เพราะ container ของ Next.js และ Postgres อาจโดน OOM

`DEPLOY_HOST` = `<user>@<host>` โดย user เริ่มต้นต่างกันตาม OS (ตารางข้างล่าง) ถ้าสร้าง user แยกสำหรับ deploy ให้ใช้ user นั้น

## ตาราง OS

| OS | user เริ่มต้นบน cloud image | ติดตั้ง Docker | firewall | ระวัง |
|---|---|---|---|---|
| Ubuntu 22.04 / 24.04 | `ubuntu` | apt repo ของ Docker | `ufw` | port ที่ publish ด้วย `ports:` ข้าม ufw ไปเลย อย่าพึ่ง ufw กัน port ของ container |
| Debian 12 / 13 | `admin` (AWS), `debian` (GCP และอื่น ๆ) | apt repo ของ Docker | `nftables` หรือ `ufw` | เหมือน Ubuntu |
| Amazon Linux 2023 | `ec2-user` | `dnf install docker` + ติดตั้ง compose plugin เอง | security group ของ AWS | repo ของ AL2023 ไม่มี compose plugin และ buildx |
| RHEL / Rocky / AlmaLinux 9 | `ec2-user` (RHEL บน AWS), `rocky`, `almalinux`, `cloud-user` | dnf repo ของ Docker | `firewalld` | มี podman ติดมา ต้องลบก่อน, SELinux ทำให้ bind mount อ่านไม่ได้ |
| Amazon Linux 2 | `ec2-user` | – | – | หมดอายุการดูแลแล้ว (2026-06-30) แนะนำให้ย้ายเป็น AL2023 ถ้าผู้ใช้ยืนยันจะใช้ ให้ถามก่อนทำต่อ |
| อื่น ๆ | ถามผู้ใช้ | ดูเอกสารของ Docker สำหรับ OS นั้น | ถามผู้ใช้ | ถ้า Docker ไม่รองรับ OS นั้นอย่างเป็นทางการ ให้บอกผู้ใช้ |

## Ubuntu / Debian

```
sudo apt-get update
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/<ubuntu|debian>/gpg -o /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/<ubuntu|debian> $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

ใส่ `ubuntu` หรือ `debian` ให้ตรงกับ OS ห้ามใช้ `docker.io` จาก repo ของ OS เพราะเวอร์ชันเก่าและไม่มี compose v2 บางรุ่น

## Amazon Linux 2023

```
sudo dnf install -y docker
sudo systemctl enable --now docker
```

compose plugin ต้องติดตั้งเอง ล็อกเวอร์ชันและตรวจ sha256 จากหน้า release ของ `docker/compose` บน GitHub:

```
VER=<เช่น v2.40.0>
ARCH=$(uname -m)            # x86_64 หรือ aarch64
sudo mkdir -p /usr/local/lib/docker/cli-plugins
sudo curl -fsSLo /usr/local/lib/docker/cli-plugins/docker-compose \
  https://github.com/docker/compose/releases/download/$VER/docker-compose-linux-$ARCH
echo "<sha256 จากไฟล์ checksums.txt ของ release>  /usr/local/lib/docker/cli-plugins/docker-compose" | sha256sum -c -
sudo chmod +x /usr/local/lib/docker/cli-plugins/docker-compose
docker compose version
```

อัปเกรด compose ต้องทำเองแบบเดียวกัน `dnf update` ไม่อัปเกรดให้

## RHEL / Rocky / AlmaLinux

```
sudo dnf remove -y podman buildah runc
sudo dnf -y install dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
sudo firewall-cmd --permanent --add-service=http --add-service=https && sudo firewall-cmd --reload
```

- **SELinux:** named volume ใช้ได้ปกติ แต่ bind mount (`./file:/path`) ต้องต่อท้าย `:z` (ใช้ร่วมหลาย container) หรือ `:Z` (container เดียว) ไม่งั้น container อ่านไฟล์ไม่ได้ ห้ามแนะนำให้ปิด SELinux
- Rocky และ Alma ใช้ repo `rhel` ของ Docker ได้

## เช็กหลังเตรียมเสร็จ (ให้ผู้ใช้รันจากเครื่องตัวเอง)

```
ssh <user>@<host> docker compose version
ssh <user>@<host> docker network ls
DOCKER_HOST=ssh://<user>@<host> docker ps
```

คำสั่งสุดท้ายผ่าน แปลว่า job deploy ของ CI ต่อเข้าเครื่องได้
