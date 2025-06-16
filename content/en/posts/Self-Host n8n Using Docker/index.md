---
author: Habib Mustofa
title: Self-Host n8n Using Docker
date: 2025-06-16
description: Self-Host n8n Using Docker and Expose It via Cloudflare Tunnel
---

> **n8n** is a powerful, open-source workflow automation tool. In this guide, you’ll learn how to self-host n8n using Docker and securely expose it to the internet using **Cloudflare Tunnel** — without opening any ports or needing a reverse proxy.

---

## 🧰 Prerequisites

Make sure you have the following ready:

- A VPS/server (Ubuntu/Debian preferred)
- A domain connected to Cloudflare
- Docker & Docker Compose installed
- A free Cloudflare account
- `cloudflared` CLI installed

---

## 1. 📁 Setup Project Directory

Create a directory for your n8n:

```bash
mkdir -p n8n-docker
cd n8n-docker
mkdir local-files
```


Inside the n8n-docker directory, create an .env file to customize your n8n details
```bash
vi .env
```
````dotenv
# DOMAIN_NAME and SUBDOMAIN together determine where n8n will be reachable

# The top level domain to serve from
DOMAIN_NAME=example.dev

# The subdomain to serve from
SUBDOMAIN=n8n

# The above example serve n8n at: https://n8n.example.dev

# Optional timezone to set which gets used by Cron and other scheduling nodes
# New York is the default value if not set
GENERIC_TIMEZONE=Asia/Jakarta

# The email address to use for the TLS/SSL certificate creation
SSL_EMAIL=test@example.dev
````

## 2. 📝 Create `docker-compose.yml`

Create a `docker-compose.yml` file with the following content:

```yaml
services:
  traefik:
    image: "traefik"
    restart: always
    command:
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.web.http.redirections.entryPoint.to=websecure"
      - "--entrypoints.web.http.redirections.entrypoint.scheme=https"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.mytlschallenge.acme.tlschallenge=true"
      - "--certificatesresolvers.mytlschallenge.acme.email=${SSL_EMAIL}"
      - "--certificatesresolvers.mytlschallenge.acme.storage=/letsencrypt/acme.json"
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - traefik_data:/letsencrypt
      - /var/run/docker.sock:/var/run/docker.sock:ro

  n8n:
    image: docker.n8n.io/n8nio/n8n
    restart: always
    ports:
      - "127.0.0.1:5678:5678"
    labels:
      - traefik.enable=true
      - traefik.http.routers.n8n.rule=Host(`${SUBDOMAIN}.${DOMAIN_NAME}`)
      - traefik.http.routers.n8n.tls=true
      - traefik.http.routers.n8n.entrypoints=web,websecure
      - traefik.http.routers.n8n.tls.certresolver=mytlschallenge
      - traefik.http.middlewares.n8n.headers.SSLRedirect=true
      - traefik.http.middlewares.n8n.headers.STSSeconds=315360000
      - traefik.http.middlewares.n8n.headers.browserXSSFilter=true
      - traefik.http.middlewares.n8n.headers.contentTypeNosniff=true
      - traefik.http.middlewares.n8n.headers.forceSTSHeader=true
      - traefik.http.middlewares.n8n.headers.SSLHost=${DOMAIN_NAME}
      - traefik.http.middlewares.n8n.headers.STSIncludeSubdomains=true
      - traefik.http.middlewares.n8n.headers.STSPreload=true
      - traefik.http.routers.n8n.middlewares=n8n@docker
    environment:
      - N8N_HOST=${SUBDOMAIN}.${DOMAIN_NAME}
      - N8N_PORT=5678
      - N8N_PROTOCOL=https
      - NODE_ENV=production
      - WEBHOOK_URL=https://${SUBDOMAIN}.${DOMAIN_NAME}/
      - GENERIC_TIMEZONE=${GENERIC_TIMEZONE}
    volumes:
      - n8n_data:/home/node/.n8n
      - ./local-files:/files

volumes:
  n8n_data:
  traefik_data:
```

## 3. 🚀 Start n8n container
```bash
docker compose up -d
```

Verify it's running:

```bash
docker ps
```

![Gambar docker ps n8n](../img/n8n%20docker%20ps.png)

## 4. 🌐 Configure Cloudflare Tunnel

- login to your `cloudflare` dashboard
- navigate to __Zero Trust__ > __Networks__ > __Tunnel__
- create a `tunnel`
- enter and type a name for the tunnel
- choose the environment you were using, then install and run the `cloudflared connector` on your VPS/server
- setup sub domain for the n8n and point it to the backend service which running the n8n

![Gambar cloudflare tunnel n8n](../img/cloudflare%20tunnel%20n8n.png)

Verify the tunnel status it's __Healthy__

## 5. ✅ Access n8n dashboard

Now open your browser and navigate to:

https://n8n.yourdomain.com

![Gambar dashboard n8n](../img/dashboard%20n8n.png)

---

## 🛡️ Security Tips

- Always enable Basic Auth for n8n
- Use HTTPS (secured by Cloudflare Tunnel)

---

## 🧩 Wrapping Up

With this guide, you've deployed n8n securely and efficiently using Docker and Cloudflare Tunnel. Whether you're building automations for yourself, your team, or a client — you now have a scalable and secure foundation.

This setup is great for personal automation, private workflows, or even small production use cases. And because you’re using Docker, keeping things up to date is easy and clean.

---

## 🤝 Let’s Connect

If you have suggestions, questions, or just want to say hi, I’d love to hear from you.

Feel free to reach out — I’m always happy to help fellow builders:

- 📧 Email: [habibmustofa962@gmail.com](mailto:habibmustofa962@gmail.com)
- 💼 LinkedIn: [linkedin.com/in/habibmustofa-attamimi/](https://www.linkedin.com/in/habibmustofa-attamimi/)

If you found this helpful, consider sharing it or giving it a star wherever you found it — it really helps!

Thanks for stopping by. See you in the next post! ✌️
