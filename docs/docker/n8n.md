# Guía de Deploy de n8n v1.86.1 Self‑Hosted

Esta guía detalla paso a paso cómo desplegar **n8n** versión **1.86.1** en un entorno Docker Compose con un proxy inverso Nginx.

* **DOMAIN**: tu dominio principal (ej. `example.com`)
* **SUBDOMAIN**: subdominio para n8n (ej. `n8n`)

Por ejemplo, tu URL final será `https://n8n.example.com:8080`.

---

## 🔧 1. Pre-Requisitos

1. **Docker & Docker Compose** instalado en tu servidor.
2. **Certificados TLS/SSL** válidos en `/path/to/certs`:

   * `fullchain.pem`
   * `privkey.pem`
3. **Puerto 8080** libre en el host o redirigido a Nginx.
4. Conexión SSH y permisos `sudo`.

---

## 🛠 2. Comandos Básicos

```bash
# 2.1 Detener y eliminar el contenedor n8n existente
docker stop n8n && docker rm n8n  

# 2.2 Bajar el stack Docker Compose
docker compose down

# 2.3 Pull de nuevas imágenes y levantar servicios
docker compose pull
docker compose up -d

# 2.4 Validar y recargar Nginx en el host
gsudo nginx -t && sudo systemctl reload nginx (opcional si usas Nginx del host)

# 2.5 Logs en tiempo real
docker compose logs -f nginx   # Proxy inverso
docker compose logs -f n8n    # Servicio n8n

# 2.6 Diagnóstico de puertos
sudo lsof -iTCP:8080 -sTCP:LISTEN
```

> Si usas Nginx dentro del contenedor, estos comandos te ayudarán:
>
> ```bash
> docker compose exec nginx nginx -t
> docker compose exec nginx nginx -s reload
> docker compose exec nginx nginx -T
> ```

---

## 📦 3. Configuración Docker Compose

Crea un archivo `docker-compose.yml` en tu proyecto:

```yaml
version: '3.7'

services:
  n8n:
    image: n8nio/n8n:1.86.1
    container_name: n8n
    restart: unless-stopped

    volumes:
      - ./data:/home/node/.n8n
      - ./local-files:/files

    environment:
      - NODE_ENV=production
      - N8N_PROTOCOL=https
      - N8N_HOST=${SUBDOMAIN}.${DOMAIN}
      - N8N_PORT=8080
      - WEBHOOK_URL=https://${SUBDOMAIN}.${DOMAIN}:8080
      - N8N_EDITOR_BASE_URL=https://${SUBDOMAIN}.${DOMAIN}:8080
      - N8N_PUSH_BACKEND=sse
      - N8N_EXPRESS_TRUST_PROXY=true
      - N8N_PROXY_HOPS=1
      - N8N_DISABLE_TELEMETRY=true

    networks:
      - proxy

    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/healthz"]
      interval: 1m
      timeout: 10s
      retries: 3

  nginx:
    image: nginx:stable-alpine
    container_name: nginx_proxy
    restart: unless-stopped
    depends_on:
      - n8n

    ports:
      - "8080:8080"

    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d:ro       # Configuración de Nginx
      - /path/to/certs/fullchain.pem:/etc/nginx/certs/fullchain.pem:ro
      - /path/to/certs/privkey.pem:/etc/nginx/certs/privkey.pem:ro

    networks:
      - proxy

networks:
  proxy:
    driver: bridge
```

> **Nota:** ajusta las rutas de los certificados y las variables de entorno (`SUBDOMAIN`, `DOMAIN`).

---

## 🔌 4. Configuración Nginx (Proxy Inverso)

Crea `nginx/conf.d/n8n.conf`:

```nginx
# Map para WebSocket/SSE
map $http_upgrade $connection_type {
  default   upgrade;
  ''        close;
}

server {
  listen 8080 ssl http2;
  server_name ${SUBDOMAIN}.${DOMAIN};

  ssl_certificate     /etc/nginx/certs/fullchain.pem;
  ssl_certificate_key /etc/nginx/certs/privkey.pem;

  client_max_body_size 50m;

  # 1) WebSocket bidireccional (/socket.io/)
  location /socket.io/ {
    proxy_pass         http://n8n:8080/socket.io/;
    proxy_http_version 1.1;
    proxy_set_header   Upgrade           $http_upgrade;
    proxy_set_header   Connection        $connection_type;
    proxy_set_header   Host              $host;
    proxy_set_header   X-Real-IP         $remote_addr;
    proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header   X-Forwarded-Proto $scheme;
    proxy_read_timeout   86400s;
    proxy_send_timeout   86400s;
    proxy_buffering      off;
    proxy_cache          off;
  }

  # 2) SSE o WebSocket unidireccional (/rest/push)
  location /rest/push {
    proxy_pass         http://n8n:8080/rest/push;
    proxy_http_version 1.1;
    proxy_set_header   Upgrade           $http_upgrade;
    proxy_set_header   Connection        $connection_type;
    proxy_set_header   Host              $host;
    proxy_set_header   X-Real-IP         $remote_addr;
    proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header   X-Forwarded-Proto $scheme;
    proxy_read_timeout   86400s;
    proxy_send_timeout   86400s;
    proxy_buffering      off;
    proxy_cache          off;
    chunked_transfer_encoding off;
  }

  # 3) UI, API REST y assets (el resto)
  location / {
    proxy_pass         http://n8n:8080;
    proxy_http_version 1.1;
    proxy_set_header   Upgrade           $http_upgrade;
    proxy_set_header   Connection        $connection_type;
    proxy_set_header   Host              $host;
    proxy_set_header   X-Real-IP         $remote_addr;
    proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header   X-Forwarded-Proto $scheme;
    proxy_read_timeout   3600s;
    proxy_send_timeout   3600s;
    proxy_buffering      off;
    proxy_cache          off;
  }
}
```

> Sustituye `${SUBDOMAIN}` y `${DOMAIN}` en `server_name`.

---

## 🚀 5. Despliegue Final

```bash
# Bajar servicios anteriores
docker compose down

# Subir con nueva configuración
docker compose up -d

# (Opcional) Ver logs en vivo
docker compose logs -f n8n
docker compose logs -f nginx
```

---

Con esto tu instancia de **n8n v1.86.1** estará operativa con un proxy Nginx, conexión estable (SSE) y sin peticiones de telemetría innecesarias.