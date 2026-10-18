# Traefik Core

Reverse proxy ماژولار برای Docker. روی سرور **کنار** پوشهٔ پروژه می‌نشیند؛ تنظیمات از `.env`. Composeهای هر پروژه داخل همان پروژه می‌مانند — این ریپو فقط Traefik است.

## چیدمان روی سرور

```
/srv/
├── traefik-core/                 # این ریپو
│   ├── .env
│   ├── docker-compose.yml
│   ├── dynamic/
│   └── letsencrypt/
└── my-project/                   # پروژهٔ تو (composeها مال خودشه)
    ├── docker-compose.yml
    ├── docker-compose.admin.yml  # اختیاری
    └── ...
```

```
اینترنت → Traefik (:80/:443) → شبکه TRAEFIK_NETWORK
                              ├─ سرویس‌های compose اصلی
                              └─ سرویس‌های compose ادمین / …
```

## راه‌اندازی Traefik

```bash
cd /srv/traefik-core
cp .env.example .env
# ACME_EMAIL و در صورت نیاز TRAEFIK_NETWORK را پر کن
touch letsencrypt/acme.json && chmod 600 letsencrypt/acme.json
docker compose up -d
```

همهٔ تنظیمات استاتیک از `.env` خوانده می‌شوند.

## اتصال از داخل پروژه

در هر `docker-compose` پروژه، شبکهٔ ساخته‌شده توسط Traefik را به‌صورت external بگیر و label بزن. `TRAEFIK_NETWORK` باید با `.env` این core یکی باشد.

```yaml
services:
  web:
    # ...
    networks:
      - proxy
    labels:
      - traefik.enable=true
      - traefik.docker.network=${TRAEFIK_NETWORK:-traefik}
      - traefik.http.routers.web.rule=Host(`${WEB_HOST}`)
      - traefik.http.routers.web.entrypoints=websecure
      - traefik.http.routers.web.tls=true
      - traefik.http.routers.web.tls.certresolver=letsencrypt
      - traefik.http.routers.web.middlewares=security-headers@file
      - traefik.http.services.web.loadbalancer.server.port=${WEB_PORT:-80}

networks:
  proxy:
    name: ${TRAEFIK_NETWORK:-traefik}
    external: true
```

اول Traefik را بالا بیاور، بعد composeهای پروژه را.

## متغیرهای `.env` (core)

| Variable | Default | نقش |
|----------|---------|-----|
| `TRAEFIK_VERSION` | `v3.6.1` | تگ ایمیج |
| `TRAEFIK_CONTAINER_NAME` | `traefik` | نام کانتینر |
| `TRAEFIK_RESTART` | `unless-stopped` | سیاست ری‌استارت |
| `TRAEFIK_NETWORK` | `traefik` | شبکه مشترک با پروژه‌ها |
| `HTTP_PORT` / `HTTPS_PORT` | `80` / `443` | پورت روی هاست |
| `TRAEFIK_WEB_PORT` / `TRAEFIK_WEBSECURE_PORT` | `80` / `443` | پورت داخل کانتینر |
| `HTTP_REDIRECT_TO` | `websecure` | مقصد ریدایرکت HTTP |
| `HTTP_REDIRECT_SCHEME` | `https` | اسکیما ریدایرکت |
| `LOG_LEVEL` | `INFO` | سطح لاگ |
| `DASHBOARD_ENABLED` | `true` | API/dashboard |
| `DASHBOARD_INSECURE` | `false` | داشبورد بدون auth روی entrypoint |
| `DOCKER_ENDPOINT` | `unix:///var/run/docker.sock` | endpoint داکر |
| `EXPOSED_BY_DEFAULT` | `false` | فقط سرویس‌های `traefik.enable=true` |
| `FILE_WATCH` | `true` | واچ `dynamic/` |
| `ACME_EMAIL` | *(required)* | ایمیل Let's Encrypt |
| `ACME_CA_SERVER` | production LE | برای تست: staging URL |

## Middlewareها

داخل `dynamic/middlewares/` — هر فایل یک ماژول؛ حذف فایل = خاموش.

| نام | Label | کاربرد |
|-----|-------|--------|
| `security-headers` | `security-headers@file` | HSTS، frame deny، مخفی‌کردن Server |
| `compress` | `compress@file` | فشرده‌سازی |
| `rate-limit` | `rate-limit@file` | محدودیت نرخ |

```yaml
- traefik.http.routers.web.middlewares=security-headers@file,compress@file
```

## نکات

- فقط کانتینرهایی با `traefik.enable=true` expose می‌شوند.
- `traefik.docker.network` را برابر `TRAEFIK_NETWORK` بگذار.
- TLS پیش‌فرض: TLS 1.3 (`dynamic/tls.yml`).

## منابع

- [Traefik](https://doc.traefik.io/traefik/)
- [Docker provider](https://doc.traefik.io/traefik/providers/docker/)
