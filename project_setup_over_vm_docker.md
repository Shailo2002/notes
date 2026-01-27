Below is a **clean, step-by-step deployment revision guide** you can keep and reuse for **any full-stack project** (React + Node/Express + Docker + VM + Domain + HTTPS).

No fluff. This is a **practical checklist + notes**.

---

# 🚀 FULL-STACK DEPLOYMENT ON VM USING DOCKER (REVISION GUIDE)

## Tech Stack Assumed

* Frontend: React (Vite)
* Backend: Node.js / Express
* Database: External (MongoDB Atlas / RDS etc.)
* VM: DigitalOcean / AWS EC2 / any Ubuntu VM
* Reverse Proxy: NGINX (host)
* Containers: Docker + Docker Compose
* HTTPS: Certbot (Let’s Encrypt)

---

## 1️⃣ LOCAL PROJECT STRUCTURE (STANDARD)

```text
project-root/
│
├── frontend/
│   ├── Dockerfile
│   ├── nginx.conf
│   ├── .env
│   └── src/
│
├── backend/
│   ├── Dockerfile
│   ├── .env
│   └── index.js
│
├── docker-compose.yml
└── README.md
```

---

## 2️⃣ FRONTEND DOCKERFILE (PRODUCTION SAFE)

**Key points**

* Build React inside container
* Serve using NGINX
* `.env` is needed **at build time** for Vite

```dockerfile
# frontend/Dockerfile
FROM node:20-alpine AS builder
WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

---

## 3️⃣ FRONTEND NGINX (SPA FALLBACK)

```nginx
# frontend/nginx.conf
server {
    listen 80;
    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

---

## 4️⃣ BACKEND DOCKERFILE

```dockerfile
# backend/Dockerfile
FROM node:20-alpine
WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .
EXPOSE 8080

CMD ["node", "index.js"]
```

---

## 5️⃣ DOCKER COMPOSE (CORE FILE)

```yaml
services:
  frontend:
    build: ./frontend
    ports:
      - "127.0.0.1:3000:80"
    restart: always
    depends_on:
      - backend

  backend:
    build: ./backend
    ports:
      - "127.0.0.1:8080:8080"
    env_file:
      - ./backend/.env
    restart: always
```

📌 **Important**

* Bind ports to `127.0.0.1` → only NGINX can access
* Never expose backend publicly

---

## 6️⃣ VM INITIAL SETUP (ONCE)

```bash
ssh root@VM_IP
apt update && apt upgrade -y
```

### Install Docker

```bash
apt install docker.io -y
systemctl enable docker
systemctl start docker
```

### Enable Docker Compose

```bash
docker --version
docker compose version
```

---

## 7️⃣ CLONE PROJECT ON VM

```bash
git clone YOUR_REPO
cd project-root
```

---

## 8️⃣ BUILD & RUN CONTAINERS

```bash
docker compose up --build -d
```

Check:

```bash
docker ps
```

Local test:

```bash
curl http://127.0.0.1:3000
curl http://127.0.0.1:8080/api/health
```

---

## 9️⃣ INSTALL HOST NGINX (REVERSE PROXY)

```bash
apt install nginx -y
systemctl enable nginx
systemctl start nginx
```

---

## 🔟 DOMAIN → VM SETUP

In DNS Manager:

```text
A   foodfetch.site     → VM_IP
A   www.foodfetch.site → VM_IP
```

Wait 1–2 minutes.

---

## 1️⃣1️⃣ HOST NGINX CONFIG (IMPORTANT)

```nginx
# /etc/nginx/sites-available/foodfetch.site
server {
    server_name foodfetch.site www.foodfetch.site;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location /api/ {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location /socket.io/ {
        proxy_pass http://127.0.0.1:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

Enable:

```bash
ln -s /etc/nginx/sites-available/foodfetch.site /etc/nginx/sites-enabled/
nginx -t
systemctl reload nginx
```

---

## 1️⃣2️⃣ HTTPS (CERTBOT)

```bash
apt install certbot python3-certbot-nginx -y
certbot --nginx -d foodfetch.site -d www.foodfetch.site
```

Auto-renew:

```bash
systemctl status certbot.timer
```

---

## 1️⃣3️⃣ BACKEND COOKIE CONFIG (AUTH SAFE)

### Backend must trust proxy

```js
app.set("trust proxy", 1);
```

### Correct cookie settings (HTTPS)

```js
res.cookie("token", token, {
  httpOnly: true,
  secure: true,
  sameSite: "none",
  path: "/",
  maxAge: 10 * 24 * 60 * 60 * 1000,
});
```

🚫 **Do NOT set `domain` unless needed**

---

## 1️⃣4️⃣ FRONTEND AUTH (CRITICAL)

### Axios Instance (MANDATORY)

```js
import axios from "axios";

const api = axios.create({
  baseURL: "/api",
  withCredentials: true,
});

export default api;
```

Every request must use this instance.

---

## 1️⃣5️⃣ COMMON PRODUCTION FIXES

| Problem                | Fix                                         |
| ---------------------- | ------------------------------------------- |
| Blank page on refresh  | NGINX `try_files /index.html`               |
| CORS error             | `credentials:true` + `withCredentials:true` |
| Cookie lost on refresh | HTTPS + proxy headers + axios config        |
| OOM build error        | Add swap / build locally                    |
| Env not working        | Vite envs are **build-time**                |
| Port conflict          | Bind containers to 127.0.0.1                |

---

## 1️⃣6️⃣ DEPLOY UPDATE FLOW (SAFE)

```bash
git pull
docker compose down
docker compose up --build -d
```

---

## 1️⃣7️⃣ SECURITY CHECKLIST

✅ Backend not exposed publicly
✅ HTTPS enabled
✅ HTTP → HTTPS redirect
✅ Cookies: httpOnly + secure
✅ No secrets in GitHub
✅ Firewall allows only 80/443/22

---

## 🧠 MENTAL MODEL (REMEMBER THIS)

```text
Browser
  ↓ HTTPS
Host NGINX
  ↓ proxy
Docker Frontend (3000)
  ↓ /api
Docker Backend (8080)
```

---

## ✅ YOU NOW KNOW

* Docker-based deployment
* Reverse proxy with NGINX
* HTTPS + cookies
* Auth persistence
* Production-safe setup

---

If you want next:

* 📄 Printable one-page checklist
* 🔁 CI/CD with GitHub Actions
* 🧪 Blue-green deployments
* 📦 Separate staging + prod

Just say which one.
