# Hostinger KVM 8 VPS — Complete Production Setup Guide (2026)

**Use case:** 4–5 Node.js / Next.js applications + PostgreSQL + Redis + RabbitMQ + WebSockets, enterprise-grade single VPS hosting.

**Base OS assumed:** Ubuntu 24.04 LTS (Noble Numbat) — Hostinger KVM VPS ka default aur recommended choice hai. Security updates April 2029 tak (Ubuntu Pro ke saath aur bhi lambe).

**Latest versions used in this guide (as of Sept 2026):**
| Tool | Version | Source |
|---|---|---|
| Ubuntu | 24.04 LTS | ubuntu.com |
| Node.js | 22.x LTS (safe) / 24.x Active LTS | nodejs.org |
| PostgreSQL | 18.x (PGDG official repo) | postgresql.org |
| Redis | 8.x (Open Source, `packages.redis.io`) | redis.io |
| RabbitMQ | 4.3.x | rabbitmq.com |
| Nginx | Ubuntu repo (stable) | nginx.org |
| PM2 | Latest via npm | pm2.keymetrics.io |
| Certbot | via snap (recommended) | certbot.eff.org |
| Docker | Latest CE (official repo) | docs.docker.com |

**Important note on Node.js version choice:**
- **Node 22 LTS** → Maintenance LTS, safest for enterprise (production-tested).
- **Node 24 LTS** → Active LTS, latest features. **Use this** for new projects.
- **Node 26** → October 2026 ke baad LTS banega. Abhi Current (avoid for production).

---

## Table of Contents

1. Pre-flight: VPS Purchase ke Baad Kya Chahiye
2. Root Login & Pehla Update
3. Hostname, Timezone, Swap Setup
4. Non-root Sudo User Banao
5. SSH Key Authentication (Password Login Band)
6. UFW Firewall Configuration
7. Fail2Ban (Brute-force Protection)
8. Unattended Security Updates
9. Node.js Installation (via NVM)
10. PM2 (Process Manager)
11. Nginx (Reverse Proxy) Install & Basic Config
12. SSL Certificates with Certbot (Let's Encrypt)
13. PostgreSQL 18 Install & Hardening
14. Redis 8 Install & Config
15. RabbitMQ 4.x Install & Config
16. Docker & Docker Compose (Optional but Powerful)
17. Multi-App Hosting Strategy (Ports + Subdomains)
18. Node.js Express App Deploy Karna
19. Next.js App Deploy Karna
20. WebSocket / Long-polling Nginx Config
21. Environment Variables Management
22. Git-based Deployment Workflow
23. Log Rotation & Monitoring
24. Backups (DB + Files)
25. Common Troubleshooting

---

## 1. Pre-flight: VPS Purchase ke Baad Kya Chahiye

Hostinger hPanel me jaake:

1. **Operating System:** Ubuntu 24.04 LTS (Clean install, no panel like CyberPanel — hum manually setup karenge for full control).
2. **Root password** set kar le (Hostinger tujhe UI se karne dega).
3. **VPS ka IP address** note kar le (hPanel me VPS overview me milega).
4. **SSH port:** Default 22 (baad me change karenge for security).
5. **Firewall:** Hostinger ka apna firewall panel bhi hai — hum server-side UFW use karenge, dono independent chalte hain. Dono me port khulna chahiye.

Local Windows machine par tu jo terminal use karega — **Windows Terminal + OpenSSH** (built-in Windows 10/11 me), ya **WSL2 Ubuntu**, ya **PuTTY** — koi bhi chalega. Main assume kar raha hoon tu Windows Terminal use kar raha hai.

---

## 2. Root Login & Pehla Update

Apne local Windows terminal me:

```bash
ssh root@YOUR_VPS_IP
```

Password puchega (jo Hostinger panel me set kiya tha). Enter kar. Pehli baar "yes" bolna hoga fingerprint accept karne ke liye.

**Turant sab kuch update kar:**

```bash
apt update && apt upgrade -y
apt autoremove -y
```

Kernel update hua ho to reboot maar:

```bash
reboot
```

30 seconds baad wapas login kar:

```bash
ssh root@YOUR_VPS_IP
```

---

## 3. Hostname, Timezone, Swap Setup

### 3.1 Hostname set karo

```bash
hostnamectl set-hostname my-prod-server
```

`/etc/hosts` me apna hostname add kar:

```bash
nano /etc/hosts
```

Ye line add/edit kar (top ke paas):
```
127.0.1.1   my-prod-server
```

Save karo: `Ctrl+O`, Enter, `Ctrl+X`.

### 3.2 Timezone set karo (IST)

```bash
timedatectl set-timezone Asia/Kolkata
timedatectl
```

*(Servers pe UTC bhi common practice hai, but IST me logs padhna easier hoga tere liye.)*

### 3.3 Swap file banao (RAM extend karne ke liye)

KVM 8 pe RAM kaafi hai, but swap safety net hai OOM-kill se bachne ke liye. **Swap = RAM ke barabar ya adha** (max 8 GB tak enough).

Assume 4 GB swap:

```bash
fallocate -l 4G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo '/swapfile none swap sw 0 0' | tee -a /etc/fstab

# Swappiness kam rakho (10 = agar bahut zaroori ho tabhi swap use ho)
sysctl vm.swappiness=10
echo 'vm.swappiness=10' | tee -a /etc/sysctl.d/99-swappiness.conf

# Verify
free -h
```

---

## 4. Non-root Sudo User Banao

**Root se kabhi bhi day-to-day kaam mat karo.** Ek normal user banao.

```bash
adduser deploy
```

Password puchega (strong password rakho — password manager use karo). Baaki fields blank chhod sakte ho (Enter dabate raho).

Isko sudo group me daalo:

```bash
usermod -aG sudo deploy
```

Ab test karo — **naya terminal window** kholo (existing wala band mat karo — agar kuch tuta to backup access hai), aur:

```bash
ssh deploy@YOUR_VPS_IP
sudo whoami   # "root" aana chahiye
```

Kaam kar gaya to purani root wali window me `exit` maar de. Ab sirf `deploy` user se kaam karo, `sudo` se elevate karo.

---

## 5. SSH Key Authentication (Password Login Band)

Password brute-force sabse common attack vector hai. SSH keys hi solution hai.

### 5.1 Local Windows machine par key generate karo (agar pehle nahi ki)

Windows Terminal me (local machine, VPS pe nahi):

```powershell
ssh-keygen -t ed25519 -C "your_email@example.com"
```

Enter dabate raho (default path `C:\Users\YourName\.ssh\id_ed25519`). Passphrase optional but recommended.

### 5.2 Public key VPS pe copy karo

Local Windows me:

```powershell
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh deploy@YOUR_VPS_IP "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

Ab test karo:

```powershell
ssh deploy@YOUR_VPS_IP
```

Password nahi puchna chahiye (ya sirf key passphrase puchega).

### 5.3 SSH server harden karo

VPS pe:

```bash
sudo nano /etc/ssh/sshd_config
```

Ye changes karo (search karke uncomment/edit):

```
Port 22                       # Baad me change karenge (optional)
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
ChallengeResponseAuthentication no
UsePAM yes
X11Forwarding no
ClientAliveInterval 300
ClientAliveCountMax 2
MaxAuthTries 3
AllowUsers deploy
```

Save, phir SSH service reload:

```bash
sudo systemctl reload ssh
```

**Doosri terminal window** me test karo — naya SSH connection aa raha hai key se? Root login block hai? Agar sab OK, tabhi purani session close karo.

### 5.4 (Optional) SSH port change karo

`sshd_config` me `Port 2222` (ya koi bhi non-standard, 1024–65535 me) set karo. UFW me bhi allow karna hoga (aage aayega). Fir login:

```bash
ssh -p 2222 deploy@YOUR_VPS_IP
```

---

## 6. UFW Firewall Configuration

Ubuntu me UFW built-in aata hai. **Pehle SSH allow karo, phir enable karo — warna server se lockout ho jaoge.**

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing

# SSH (agar port change ki hai to woh number daal, warna 22)
sudo ufw allow 22/tcp
# sudo ufw allow 2222/tcp   # agar custom port

# Web
sudo ufw allow 80/tcp     # HTTP
sudo ufw allow 443/tcp    # HTTPS

# Enable
sudo ufw enable
sudo ufw status verbose
```

**Redis/PostgreSQL/RabbitMQ ports (5432, 6379, 5672, 15672) UFW me MAT KHOL** — ye services localhost pe hi chalengi, apps unse local socket / 127.0.0.1 pe connect karengi. Isse security bahut mazboot hoti hai.

Agar future me kisi ko remote se DB access dena ho, tab specific IP se hi allow karna:

```bash
sudo ufw allow from 203.0.113.5 to any port 5432
```

---

## 7. Fail2Ban (Brute-force Protection)

```bash
sudo apt install fail2ban -y
```

Local config file banao (default `jail.conf` edit mat karo, wo update pe overwrite ho jaati hai):

```bash
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo nano /etc/fail2ban/jail.local
```

`[sshd]` section dhundo (Ctrl+W me `[sshd]` type karo), uske neeche ye rakho:

```ini
[sshd]
enabled = true
port    = 22
# port  = 2222   # agar custom SSH port
filter  = sshd
logpath = %(sshd_log)s
maxretry = 3
bantime  = 3600
findtime = 600
```

Save karo. Fir:

```bash
sudo systemctl enable --now fail2ban
sudo fail2ban-client status
sudo fail2ban-client status sshd
```

---

## 8. Unattended Security Updates

Automatic security patches ke liye:

```bash
sudo apt install unattended-upgrades -y
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

Yes select karo. Fir config verify:

```bash
sudo nano /etc/apt/apt.conf.d/50unattended-upgrades
```

Ye lines uncomment/enable karo (recommended):

```
Unattended-Upgrade::Remove-Unused-Dependencies "true";
Unattended-Upgrade::Automatic-Reboot "false";     // Reboot manually kar (production safety)
Unattended-Upgrade::Automatic-Reboot-Time "04:00";
```

---

## 9. Node.js Installation (via NVM)

**Direct apt se Node install mat karo** — versions purane hote hain aur switch karna mushkil. NVM (Node Version Manager) best hai — multiple versions coexist kar sakte hain.

### 9.1 NVM install

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
```

*(Latest NVM version check kar https://github.com/nvm-sh/nvm — script version time-to-time update hoti hai.)*

Terminal reload karo:

```bash
source ~/.bashrc
# ya
exec bash
```

Verify:

```bash
command -v nvm    # "nvm" print hona chahiye
```

### 9.2 Node.js LTS install

```bash
# Node 22 (Maintenance LTS — safest)
nvm install 22
nvm alias default 22

# Ya Node 24 (Active LTS — recommended for new projects)
nvm install 24
nvm alias default 24

# Verify
node -v     # v22.x.x ya v24.x.x
npm -v
```

### 9.3 Global tools

```bash
npm install -g pm2 yarn pnpm
```

- **PM2** — Node process manager
- **Yarn / pnpm** — faster package installs (choice tera)

---

## 10. PM2 (Process Manager)

PM2 tera "systemctl for Node apps" hai. Auto-restart on crash, log management, cluster mode, startup on reboot.

### 10.1 PM2 systemd integration

```bash
pm2 startup systemd
```

Ye ek command print karega — usko copy karke `sudo` se run karo (PM2 ko system boot pe start hone ke liye register karta hai):

```bash
sudo env PATH=$PATH:/home/deploy/.nvm/versions/node/v24.x.x/bin \
    /home/deploy/.nvm/versions/node/v24.x.x/lib/node_modules/pm2/bin/pm2 \
    startup systemd -u deploy --hp /home/deploy
```

### 10.2 PM2 ecosystem file (best practice)

Har app ke liye ecosystem file rakho — ek jagah se sab manage hoga. `~/apps/ecosystem.config.js`:

```javascript
module.exports = {
  apps: [
    {
      name: 'api-1',
      cwd: '/home/deploy/apps/api-1',
      script: 'dist/server.js',      // ya npm start
      instances: 2,                   // cluster mode (CPU cores ke basis pe)
      exec_mode: 'cluster',
      env: {
        NODE_ENV: 'production',
        PORT: 3001
      },
      max_memory_restart: '500M',
      error_file: '/home/deploy/logs/api-1-error.log',
      out_file: '/home/deploy/logs/api-1-out.log',
      time: true
    },
    {
      name: 'api-2',
      cwd: '/home/deploy/apps/api-2',
      script: 'dist/server.js',
      instances: 1,
      exec_mode: 'fork',
      env: {
        NODE_ENV: 'production',
        PORT: 3002
      }
    },
    {
      name: 'web-nextjs',
      cwd: '/home/deploy/apps/web-nextjs',
      script: 'node_modules/next/dist/bin/next',
      args: 'start -p 3003',
      instances: 1,
      exec_mode: 'fork',
      env: { NODE_ENV: 'production' }
    }
  ]
};
```

Run karne ke liye:

```bash
mkdir -p ~/apps ~/logs
pm2 start ~/apps/ecosystem.config.js
pm2 save         # current state save (reboot pe restore hoga)
pm2 status
pm2 logs api-1
pm2 monit        # real-time dashboard
```

### 10.3 Useful PM2 commands

```bash
pm2 restart api-1
pm2 reload api-1              # zero-downtime reload (cluster mode)
pm2 stop api-1
pm2 delete api-1
pm2 flush                     # logs clear
pm2 logs --lines 200
pm2 describe api-1
```

---

## 11. Nginx (Reverse Proxy) Install & Basic Config

Nginx tera front door hai — port 80/443 pe sunta hai, aur request ko backend Node apps (localhost:3001, 3002, etc.) pe forward karta hai.

### 11.1 Install

```bash
sudo apt install nginx -y
sudo systemctl enable --now nginx
sudo nginx -v
```

Browser me `http://YOUR_VPS_IP` khol — "Welcome to nginx!" page dikhna chahiye.

### 11.2 Directory structure

```
/etc/nginx/
├── nginx.conf              # main config
├── sites-available/        # yahan naye site configs banao
└── sites-enabled/          # yahan symlink karo enable karne ke liye
```

Default site hata do:

```bash
sudo rm /etc/nginx/sites-enabled/default
```

### 11.3 Ek reverse proxy site (example: api1.example.com → localhost:3001)

```bash
sudo nano /etc/nginx/sites-available/api1.example.com
```

Content:

```nginx
server {
    listen 80;
    server_name api1.example.com;

    # Certbot ke liye jagah
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        proxy_pass http://127.0.0.1:3001;
        proxy_http_version 1.1;

        # Standard headers
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebSocket support (ye important hai — Socket.io, ws, etc. ke liye)
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        # Timeouts (long-polling / SSE ke liye zaroori)
        proxy_read_timeout 86400;
        proxy_send_timeout 86400;
    }
}
```

Enable karo:

```bash
sudo ln -s /etc/nginx/sites-available/api1.example.com /etc/nginx/sites-enabled/
sudo nginx -t          # syntax test — "successful" aana chahiye
sudo systemctl reload nginx
```

### 11.4 Basic global hardening

```bash
sudo nano /etc/nginx/nginx.conf
```

`http { }` block ke andar ye add/verify karo:

```nginx
server_tokens off;              # Nginx version chhupao
client_max_body_size 20M;       # File upload limit
gzip on;
gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;
```

Reload:

```bash
sudo nginx -t && sudo systemctl reload nginx
```

---

## 12. SSL Certificates with Certbot (Let's Encrypt)

**Snap version recommended hai** — auto-updates milte hain, plugin issues nahi hote.

```bash
# Snap install (Ubuntu 24 me pre-installed hoti hai)
sudo snap install core; sudo snap refresh core
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot
```

### 12.1 DNS pehle set karo

Har subdomain (api1.example.com, api2.example.com, etc.) ka **A record** apne VPS IP pe point kar. Domain registrar ke DNS panel me. Propagate hone me 5–30 min lagte hain.

Check:
```bash
dig +short api1.example.com
```
VPS IP dikhna chahiye.

### 12.2 Certificate issue karo

```bash
sudo certbot --nginx -d api1.example.com
```

- Email daalega (renewal notifications ke liye)
- Terms accept karo
- HTTPS redirect enable karo (option 2)

Certbot khud tera Nginx config edit karega — 443 listener, cert paths, HTTP→HTTPS redirect add karega.

### 12.3 Auto-renewal test

```bash
sudo certbot renew --dry-run
```

Snap version me automatic timer chalu rehta hai (`systemctl list-timers | grep certbot` se verify kar sakte ho).

---

## 13. PostgreSQL 18 Install & Hardening

Ubuntu 24.04 me default repo se PostgreSQL 16 milta hai. Latest 18 ke liye official PGDG repo use karo.

### 13.1 Official PGDG repo add karo

```bash
sudo apt install curl ca-certificates -y
sudo install -d /usr/share/postgresql-common/pgdg
sudo curl -o /usr/share/postgresql-common/pgdg/apt.postgresql.org.asc \
  --fail https://www.postgresql.org/media/keys/ACCC4CF8.asc

sudo sh -c 'echo "deb [signed-by=/usr/share/postgresql-common/pgdg/apt.postgresql.org.asc] \
  https://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" \
  > /etc/apt/sources.list.d/pgdg.list'

sudo apt update
sudo apt install postgresql-18 postgresql-contrib-18 -y
```

### 13.2 Service check

```bash
sudo systemctl enable --now postgresql
sudo systemctl status postgresql
psql --version
```

### 13.3 Initial user & database setup

PostgreSQL install pe `postgres` naam ka Linux user aur `postgres` DB superuser bana deta hai.

```bash
sudo -u postgres psql
```

`psql` prompt me:

```sql
-- Apna app user banao (strong password!)
CREATE USER myapp_user WITH PASSWORD 'REPLACE_WITH_STRONG_PASSWORD';

-- Database banao
CREATE DATABASE myapp_db OWNER myapp_user;

-- Zaroori privileges
GRANT ALL PRIVILEGES ON DATABASE myapp_db TO myapp_user;

-- Quit
\q
```

Har app ke liye alag DB aur alag user rakhna best practice hai.

### 13.4 Local-only listening (default hi hai, verify karo)

```bash
sudo nano /etc/postgresql/18/main/postgresql.conf
```

`listen_addresses` line dhundo:
```
listen_addresses = 'localhost'
```
Ye hi rakho. External access chahiye tabhi change karna.

### 13.5 Authentication

```bash
sudo nano /etc/postgresql/18/main/pg_hba.conf
```

Local connections ke liye `scram-sha-256` (default hai Ubuntu 24 me — verify karo, `md5` ho to badal do):

```
local   all             all                                     peer
host    all             all             127.0.0.1/32            scram-sha-256
host    all             all             ::1/128                 scram-sha-256
```

Reload:
```bash
sudo systemctl reload postgresql
```

### 13.6 Test connection from your app-user

```bash
psql -h 127.0.0.1 -U myapp_user -d myapp_db
# Password prompt aayega
```

### 13.7 Node.js connection string

```
postgresql://myapp_user:PASSWORD@127.0.0.1:5432/myapp_db
```

Use `pg` (node-postgres) ya `Prisma`/`Drizzle` ORM.

### 13.8 Performance tuning (KVM 8 = ~8 GB RAM assumed)

`/etc/postgresql/18/main/postgresql.conf` me:

```
shared_buffers = 2GB               # RAM ka ~25%
effective_cache_size = 6GB         # RAM ka ~75%
work_mem = 16MB
maintenance_work_mem = 512MB
max_connections = 100
wal_buffers = 16MB
random_page_cost = 1.1             # SSD ke liye
effective_io_concurrency = 200     # SSD
```

Reload:
```bash
sudo systemctl restart postgresql
```

*(Values apne actual RAM ke hisaab se adjust kar — [pgtune.leopard.in.ua](https://pgtune.leopard.in.ua/) online calculator hai.)*

---

## 14. Redis 8 Install & Config

Ubuntu 24 ke default repo me Redis 7.x hai. Redis 8 (latest) ke liye official repo use karo.

### 14.1 Official Redis repo

```bash
sudo apt install lsb-release curl gpg -y

curl -fsSL https://packages.redis.io/gpg | sudo gpg --dearmor -o /usr/share/keyrings/redis-archive-keyring.gpg
sudo chmod 644 /usr/share/keyrings/redis-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/redis-archive-keyring.gpg] https://packages.redis.io/deb $(lsb_release -cs) main" | \
  sudo tee /etc/apt/sources.list.d/redis.list

sudo apt update
sudo apt install redis -y

sudo systemctl enable --now redis-server
redis-server --version
```

### 14.2 Config harden

```bash
sudo nano /etc/redis/redis.conf
```

Ye check/change karo:

```
bind 127.0.0.1 ::1
protected-mode yes
port 6379
requirepass REPLACE_WITH_STRONG_PASSWORD

# Persistence — dono chalao production me
appendonly yes
appendfsync everysec

# Memory limit (RAM ka ~15-20% suggest)
maxmemory 1gb
maxmemory-policy allkeys-lru

# Slow log
slowlog-log-slower-than 10000
slowlog-max-len 128
```

Restart:
```bash
sudo systemctl restart redis-server
sudo systemctl status redis-server
```

### 14.3 Test

```bash
redis-cli
127.0.0.1:6379> AUTH REPLACE_WITH_STRONG_PASSWORD
OK
127.0.0.1:6379> PING
PONG
127.0.0.1:6379> exit
```

### 14.4 Node.js connection

Use `ioredis` (better cluster support) ya `redis` (official):

```
redis://:PASSWORD@127.0.0.1:6379
```

---

## 15. RabbitMQ 4.x Install & Config

RabbitMQ needs Erlang. Ubuntu default repo me purani Erlang hoti hai — official Team RabbitMQ repo use karo latest ke liye.

### 15.1 Signing keys + repos

Ek script me sab (official docs ke steps):

```bash
sudo apt install curl gnupg apt-transport-https -y

# Team RabbitMQ main signing key
curl -1sLf "https://keys.openpgp.org/vks/v1/by-fingerprint/0A9AF2115F4687BD29803A206B73A36E6026DFCA" | \
    sudo gpg --dearmor -o /usr/share/keyrings/com.rabbitmq.team.gpg

# Community mirror (Erlang)
curl -1sLf https://github.com/rabbitmq/signing-keys/releases/download/3.0/cloudsmith.rabbitmq-erlang.E495BB49CC4BBE5B.key | \
    sudo gpg --dearmor -o /usr/share/keyrings/rabbitmq.E495BB49CC4BBE5B.gpg

# Community mirror (RabbitMQ server)
curl -1sLf https://github.com/rabbitmq/signing-keys/releases/download/3.0/cloudsmith.rabbitmq-server.9F4587F226208342.key | \
    sudo gpg --dearmor -o /usr/share/keyrings/rabbitmq.9F4587F226208342.gpg
```

Repo list add karo (`noble` = Ubuntu 24.04):

```bash
sudo tee /etc/apt/sources.list.d/rabbitmq.list <<EOF
## Modern Erlang from Cloudsmith
deb [arch=amd64 signed-by=/usr/share/keyrings/rabbitmq.E495BB49CC4BBE5B.gpg] https://ppa1.novemberain.com/rabbitmq/rabbitmq-erlang/deb/ubuntu noble main
deb-src [signed-by=/usr/share/keyrings/rabbitmq.E495BB49CC4BBE5B.gpg] https://ppa1.novemberain.com/rabbitmq/rabbitmq-erlang/deb/ubuntu noble main

## RabbitMQ server from Cloudsmith
deb [arch=amd64 signed-by=/usr/share/keyrings/rabbitmq.9F4587F226208342.gpg] https://ppa1.novemberain.com/rabbitmq/rabbitmq-server/deb/ubuntu noble main
deb-src [signed-by=/usr/share/keyrings/rabbitmq.9F4587F226208342.gpg] https://ppa1.novemberain.com/rabbitmq/rabbitmq-server/deb/ubuntu noble main
EOF
```

Install:

```bash
sudo apt update

sudo apt install -y erlang-base \
    erlang-asn1 erlang-crypto erlang-eldap erlang-ftp erlang-inets \
    erlang-mnesia erlang-os-mon erlang-parsetools erlang-public-key \
    erlang-runtime-tools erlang-snmp erlang-ssl \
    erlang-syntax-tools erlang-tftp erlang-tools erlang-xmerl

sudo apt install rabbitmq-server -y --fix-missing
```

### 15.2 Service

```bash
sudo systemctl enable --now rabbitmq-server
sudo systemctl status rabbitmq-server
sudo rabbitmqctl status
```

### 15.3 Management UI + Admin user

```bash
sudo rabbitmq-plugins enable rabbitmq_management
```

Default `guest`/`guest` user sirf localhost pe kaam karta hai — production me disable karo aur naya admin banao:

```bash
# Naya admin user
sudo rabbitmqctl add_user admin REPLACE_WITH_STRONG_PASSWORD
sudo rabbitmqctl set_user_tags admin administrator
sudo rabbitmqctl set_permissions -p / admin ".*" ".*" ".*"

# Default guest hatao
sudo rabbitmqctl delete_user guest
```

### 15.4 App-specific user + vhost (best practice)

```bash
sudo rabbitmqctl add_vhost myapp_vhost
sudo rabbitmqctl add_user myapp_mq_user REPLACE_WITH_STRONG_PASSWORD
sudo rabbitmqctl set_permissions -p myapp_vhost myapp_mq_user ".*" ".*" ".*"
```

Node connection URL:
```
amqp://myapp_mq_user:PASSWORD@127.0.0.1:5672/myapp_vhost
```

### 15.5 Management UI Nginx pe expose karna (optional)

Management port 15672 direct expose mat karo. Nginx ke through subdomain pe daalo with SSL + basic auth:

```bash
# Basic auth setup
sudo apt install apache2-utils -y
sudo htpasswd -c /etc/nginx/.htpasswd_mq admin
```

Nginx config `mq.example.com`:

```nginx
server {
    listen 80;
    server_name mq.example.com;

    location / {
        auth_basic "Restricted";
        auth_basic_user_file /etc/nginx/.htpasswd_mq;

        proxy_pass http://127.0.0.1:15672;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Enable + Certbot:
```bash
sudo ln -s /etc/nginx/sites-available/mq.example.com /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
sudo certbot --nginx -d mq.example.com
```

---

## 16. Docker & Docker Compose (Optional but Powerful)

Docker se tu Redis/RabbitMQ/Postgres ko containers me chala sakta hai — versioning easier, isolation better. Alternative approach hai native install ka. **Beginners ke liye:** Native install + Docker sirf special apps ke liye (e.g., ek isolated Postgres for staging).

### 16.1 Install (official Docker CE repo)

```bash
sudo apt install ca-certificates curl -y
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
  https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
```

### 16.2 User ko docker group me daalo (sudo ke bina docker chalane ke liye)

```bash
sudo usermod -aG docker deploy
newgrp docker
docker --version
docker compose version
docker run hello-world
```

### 16.3 Example: Redis + Postgres Docker Compose (agar native install nahi karna)

`~/infra/docker-compose.yml`:

```yaml
services:
  postgres:
    image: postgres:18
    restart: always
    environment:
      POSTGRES_USER: myapp_user
      POSTGRES_PASSWORD: STRONG_PASSWORD
      POSTGRES_DB: myapp_db
    ports:
      - "127.0.0.1:5432:5432"
    volumes:
      - ./pg_data:/var/lib/postgresql/data

  redis:
    image: redis:8-alpine
    restart: always
    command: redis-server --requirepass STRONG_PASSWORD --appendonly yes
    ports:
      - "127.0.0.1:6379:6379"
    volumes:
      - ./redis_data:/data

  rabbitmq:
    image: rabbitmq:4-management
    restart: always
    environment:
      RABBITMQ_DEFAULT_USER: admin
      RABBITMQ_DEFAULT_PASS: STRONG_PASSWORD
    ports:
      - "127.0.0.1:5672:5672"
      - "127.0.0.1:15672:15672"
    volumes:
      - ./rabbitmq_data:/var/lib/rabbitmq
```

Run:
```bash
cd ~/infra
docker compose up -d
docker compose ps
docker compose logs -f
```

**Ports `127.0.0.1:` prefix se bind hote hain — public me expose nahi hote. UFW ka koi jhanjhat nahi.**

---

## 17. Multi-App Hosting Strategy (Ports + Subdomains)

### Port allocation plan (example)

| App | Type | Internal Port | Subdomain |
|---|---|---|---|
| api-1 | Node/Express | 3001 | api1.example.com |
| api-2 | Node/Express | 3002 | api2.example.com |
| api-3 | Node/Fastify | 3003 | api3.example.com |
| web-1 | Next.js | 3010 | www.example.com |
| web-2 | Next.js | 3011 | app.example.com |
| socket | Socket.io | 3020 | ws.example.com |
| mq-ui | RabbitMQ mgmt | 15672 | mq.example.com |

Har app ek Nginx server block me, port 80/443 pe listen karta Nginx sabko route karta hai.

### Folder structure

```
/home/deploy/
├── apps/
│   ├── api-1/          # Git clone yahan
│   ├── api-2/
│   ├── web-nextjs/
│   └── ecosystem.config.js
├── logs/
├── backups/
└── infra/              # docker-compose.yml (if using)
```

---

## 18. Node.js Express App Deploy Karna

### 18.1 Code laao (Git)

```bash
cd ~/apps
git clone https://github.com/yourname/api-1.git
cd api-1

# Node version ensure (agar .nvmrc file ho)
nvm use

# Dependencies
npm ci --production=false     # devDeps chahiye build ke liye

# Build (agar TypeScript)
npm run build

# Production install
npm prune --production
```

### 18.2 Env file

```bash
nano .env
```

```
NODE_ENV=production
PORT=3001
DATABASE_URL=postgresql://myapp_user:PASS@127.0.0.1:5432/myapp_db
REDIS_URL=redis://:PASS@127.0.0.1:6379
RABBITMQ_URL=amqp://myapp_mq_user:PASS@127.0.0.1:5672/myapp_vhost
JWT_SECRET=very-long-random-string
```

**Permission secure karo:**
```bash
chmod 600 .env
```

### 18.3 PM2 se start karo

`ecosystem.config.js` me app entry pehle se hai (Section 10 dekh). Fir:

```bash
pm2 start ~/apps/ecosystem.config.js --only api-1
pm2 save
pm2 logs api-1
```

### 18.4 Nginx site file (Section 11.3 dekh) + Certbot (Section 12)

Ho gaya — `https://api1.example.com` live.

---

## 19. Next.js App Deploy Karna

Next.js 14+ (App Router) production build:

```bash
cd ~/apps
git clone https://github.com/yourname/web-nextjs.git
cd web-nextjs

nvm use
npm ci

# Build
npm run build

# Optional: standalone output (chhota bundle)
# next.config.js me: output: 'standalone'
```

### 19.1 PM2 se start

Ecosystem me:

```javascript
{
  name: 'web-nextjs',
  cwd: '/home/deploy/apps/web-nextjs',
  script: 'npm',
  args: 'start',
  env: {
    NODE_ENV: 'production',
    PORT: 3010
  }
}
```

Ya standalone build ke liye:
```javascript
{
  name: 'web-nextjs',
  cwd: '/home/deploy/apps/web-nextjs',
  script: '.next/standalone/server.js',
  env: { NODE_ENV: 'production', PORT: 3010, HOSTNAME: '127.0.0.1' }
}
```

**Important:** Next.js standalone mode me static assets (`public/` aur `.next/static/`) ko `.next/standalone/` me copy karna hoga:

```bash
cp -r public .next/standalone/
cp -r .next/static .next/standalone/.next/
```

### 19.2 Nginx (static caching ke saath)

```nginx
server {
    listen 80;
    server_name www.example.com;

    # Static assets — cache long
    location /_next/static/ {
        proxy_pass http://127.0.0.1:3010;
        proxy_cache_valid 200 60m;
        add_header Cache-Control "public, max-age=31536000, immutable";
    }

    location /static/ {
        proxy_pass http://127.0.0.1:3010;
        add_header Cache-Control "public, max-age=3600";
    }

    location / {
        proxy_pass http://127.0.0.1:3010;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

Certbot chala:
```bash
sudo certbot --nginx -d www.example.com -d example.com
```

---

## 20. WebSocket / Long-polling Nginx Config

Socket.io, ws, SSE, long-polling — **teen cheezein pakki chahiye**:

```nginx
location /socket.io/ {
    proxy_pass http://127.0.0.1:3020;
    proxy_http_version 1.1;

    # WebSocket upgrade
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";

    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    # Long timeouts (24 ghante)
    proxy_read_timeout 86400;
    proxy_send_timeout 86400;

    # Buffering off — SSE/streaming ke liye critical
    proxy_buffering off;
    proxy_cache off;
}
```

**Socket.io me sticky sessions:** Agar PM2 cluster mode me chala rahe ho (multiple instances), to Socket.io ko `@socket.io/redis-adapter` chahiye — sab instances Redis ke through communicate karenge. Warna connections random instance pe distribute honge aur break honge.

```javascript
// Node app me
import { createAdapter } from '@socket.io/redis-adapter';
import { createClient } from 'redis';

const pubClient = createClient({ url: 'redis://:PASS@127.0.0.1:6379' });
const subClient = pubClient.duplicate();
await Promise.all([pubClient.connect(), subClient.connect()]);
io.adapter(createAdapter(pubClient, subClient));
```

---

## 21. Environment Variables Management

- Har app ke root me `.env` file, `chmod 600`
- **`.env` ko `.gitignore` me daalo** — kabhi commit mat karo
- Repo me `.env.example` rakho (structure ke liye, blank values ke saath)
- Secrets management (production-serious ke liye): **HashiCorp Vault**, **AWS Secrets Manager**, ya **doppler.com** — but starting me `.env` chalega
- PM2 me `env:` block use kar sakte ho, but bade env vars ke liye `dotenv` npm package + `.env` file better hai

---

## 22. Git-based Deployment Workflow

Simple deploy script `~/apps/api-1/deploy.sh`:

```bash
#!/bin/bash
set -e

cd /home/deploy/apps/api-1

echo "→ Pulling latest code..."
git pull origin main

echo "→ Installing dependencies..."
npm ci

echo "→ Building..."
npm run build

echo "→ Reloading PM2 (zero-downtime)..."
pm2 reload api-1

echo "✓ Deployed successfully."
```

```bash
chmod +x ~/apps/api-1/deploy.sh
```

Chalao:
```bash
~/apps/api-1/deploy.sh
```

**Next step (advanced):** GitHub Actions me SSH-based deploy pipeline — push pe automatic deploy. Iske liye VPS pe ek `deployer` SSH key pair banao, private key GitHub Secrets me daalo, action `appleboy/ssh-action` use karo.

---

## 23. Log Rotation & Monitoring

### 23.1 PM2 logrotate module

```bash
pm2 install pm2-logrotate
pm2 set pm2-logrotate:max_size 50M
pm2 set pm2-logrotate:retain 14         # 14 files retain
pm2 set pm2-logrotate:compress true
pm2 set pm2-logrotate:rotateInterval '0 0 * * *'   # daily
```

### 23.2 Nginx logs — Ubuntu me default logrotate configured hota hai (`/etc/logrotate.d/nginx`).

### 23.3 Monitoring — options

**Lightweight (built-in):**
```bash
htop            # CPU/RAM (install: sudo apt install htop)
pm2 monit       # PM2 dashboard
```

**Full dashboard (recommended for production):**
- **Netdata** — one-command install, web UI, real-time:
  ```bash
  wget -O /tmp/netdata-kickstart.sh https://get.netdata.cloud/kickstart.sh
  sh /tmp/netdata-kickstart.sh
  ```
  Nginx ke through auth-protected subdomain pe expose kar (jaise mq.example.com kiya tha).

- **Uptime monitoring (external):** UptimeRobot, BetterStack — free tiers milte hain.

---

## 24. Backups (Databases + Files)

**Backup nahi hai to production nahi hai.** Kam se kam:

### 24.1 PostgreSQL daily backup script

`~/scripts/pg_backup.sh`:

```bash
#!/bin/bash
set -e

BACKUP_DIR=/home/deploy/backups/postgres
mkdir -p "$BACKUP_DIR"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)

PGPASSWORD='YOUR_DB_PASS' pg_dump -h 127.0.0.1 -U myapp_user -d myapp_db \
  -F c -f "$BACKUP_DIR/myapp_db_$TIMESTAMP.dump"

# 14 din se purane delete
find "$BACKUP_DIR" -name "*.dump" -mtime +14 -delete
```

```bash
chmod +x ~/scripts/pg_backup.sh
mkdir -p ~/backups/postgres
```

Cron me daily 3 AM:
```bash
crontab -e
```
```
0 3 * * * /home/deploy/scripts/pg_backup.sh >> /home/deploy/logs/pg_backup.log 2>&1
```

### 24.2 Offsite backup (critical!)

Backups sirf VPS pe rakhna galat hai — VPS crash to sab gaya. Options:

- **rclone** + Google Drive / Backblaze B2 / AWS S3 / Wasabi
- **restic** — encrypted, deduplicated backup tool

Basic B2 setup with rclone:

```bash
sudo apt install rclone -y
rclone config              # interactive — B2 account setup kar

# Daily upload
rclone sync /home/deploy/backups b2:my-vps-backups
```

Cron me daily 4 AM:
```
0 4 * * * /usr/bin/rclone sync /home/deploy/backups b2:my-vps-backups
```

### 24.3 Redis persistence

Section 14 me `appendonly yes` set kiya tha — RDB + AOF dono chal rahe hain. `/var/lib/redis/` folder backup me include kar.

### 24.4 App code backup

Git me code hai to backup already hai. Bas `.env` files aur uploaded user files (`public/uploads/` etc.) alag se backup me daalo.

---

## 25. Common Troubleshooting

### 25.1 "SSH me lock out ho gaya"

Hostinger hPanel me **VNC / recovery console** hota hai. Waha se root login karke `/etc/ssh/sshd_config` fix kar, ya `ufw disable` kar temporarily.

### 25.2 "Port 80/443 pe kuch aa nahi raha"

```bash
sudo ss -tlnp | grep -E '80|443'      # Nginx listen kar raha hai?
sudo ufw status                        # firewall me allow hai?
curl -I http://YOUR_IP                 # local se test
```

### 25.3 "Nginx 502 Bad Gateway"

Matlab backend Node app down hai:
```bash
pm2 status
pm2 logs api-1 --lines 100
curl -I http://127.0.0.1:3001         # backend direct test
```

### 25.4 "Certbot fail — DNS not pointing"

```bash
dig +short api1.example.com           # VPS IP aana chahiye
```
DNS propagation wait karo (~30 min).

### 25.5 "Postgres/Redis connection refused"

```bash
sudo systemctl status postgresql
sudo systemctl status redis-server
sudo ss -tlnp | grep -E '5432|6379'
```

Check karo listen address `127.0.0.1` hai (agar app same VPS pe).

### 25.6 "Disk full"

```bash
df -h
du -h --max-depth=1 /var/log
du -h --max-depth=1 /home/deploy
pm2 flush                              # PM2 logs clear
sudo journalctl --vacuum-time=7d       # 7 din se purane journal logs delete
```

### 25.7 "Server slow / OOM"

```bash
free -h
htop
pm2 status                             # memory usage per app
# max_memory_restart set karo ecosystem.config me
```

### 25.8 "SSL renewal fail"

```bash
sudo certbot renew --dry-run
sudo systemctl status snap.certbot.renew.timer
```

---

## Bonus: Quick Reference Command Cheat Sheet

```bash
# System
sudo systemctl status <service>
sudo journalctl -u <service> -f
sudo ufw status
htop

# PM2
pm2 status
pm2 restart <name>
pm2 reload <name>          # zero-downtime
pm2 logs <name>
pm2 monit
pm2 save

# Nginx
sudo nginx -t
sudo systemctl reload nginx
sudo tail -f /var/log/nginx/error.log
sudo tail -f /var/log/nginx/access.log

# Postgres
sudo -u postgres psql
sudo systemctl restart postgresql
psql -h 127.0.0.1 -U user -d dbname

# Redis
redis-cli -a PASSWORD
sudo systemctl restart redis-server

# RabbitMQ
sudo rabbitmqctl status
sudo rabbitmqctl list_queues
sudo rabbitmqctl list_users

# Certbot
sudo certbot certificates
sudo certbot renew --dry-run

# Docker
docker ps
docker compose up -d
docker compose logs -f
docker system prune -a     # cleanup unused images

# Node/NVM
nvm ls
nvm use <version>
node -v
```

---

## Final Production Checklist ✅

- [ ] Root login disabled, SSH key auth only
- [ ] UFW enabled, sirf 22/80/443 open
- [ ] Fail2Ban chalu
- [ ] Unattended security updates on
- [ ] Non-root sudo user use ho raha
- [ ] Swap file configured
- [ ] Node.js LTS installed via NVM
- [ ] PM2 startup registered, `pm2 save` kiya
- [ ] Nginx running, `server_tokens off`
- [ ] Har domain pe SSL (Certbot), auto-renewal working
- [ ] PostgreSQL 18 `scram-sha-256`, localhost-only, strong password
- [ ] Redis 8 `requirepass`, localhost-only, persistence on
- [ ] RabbitMQ 4.x default `guest` user deleted, per-app vhost
- [ ] Database + files daily backup with offsite upload
- [ ] PM2 log rotate installed
- [ ] Monitoring (Netdata / external uptime) chalu
- [ ] `.env` files `chmod 600`, git me ignored
- [ ] Deploy script + rollback plan ready

---

**Yaad rakh:**
- Kisi bhi change ke pehle backup, especially DB.
- Nginx / SSHD ke config edits ke baad **always** `sudo nginx -t` / `sudo sshd -t` chalao **before reload/restart**.
- Har `sudo` command dhyan se — production server pe rm -rf accidentally sab kha jaata hai.
- Passwords: har service ka **alag**, kam se kam 24 characters, password manager me store.
- Firewall me DB/cache ports **kabhi public** mat karo — always through app.

Happy shipping! 🚀
