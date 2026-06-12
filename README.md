# Essenz Master Orchestrator

Infraestructura unificada del **Nivel 1 (Fundación)** del ecosistema Essenz.
No reescribe ningún microservicio: usa los build contexts de cada proyecto
y los une con un API Gateway y una red interna de Docker.

> **Documentación completa en [`docs/`](docs/README.md)**: arquitectura,
> gateway línea a línea, red y variables cruzadas, y operaciones.
> Los microservicios viven en sus propios repos:
> [essenz-auth](https://github.com/RSNxru/essenz-auth) ·
> [essenz-notifications](https://github.com/RSNxru/essenz-notifications) ·
> [essenz-events](https://github.com/RSNxru/essenz-events)

```
                        ┌──────────────────────────┐
  cliente ──:8080──────▶│   essenz_gateway (Nginx) │
                        └────────────┬─────────────┘
                 ┌───────────────────┼───────────────────┐
        /api/auth/          /api/notifications/      /api/events/
                 ▼                   ▼                   ▼
        ┌──────────────┐   ┌─────────────────────┐  ┌──────────────┐
        │ essenz_auth  │   │ essenz_notifications│  │ essenz_events│
        │ :8000        │   │ :8000               │  │ :8000        │
        ├──────────────┤   ├─────────────────────┤  ├──────────────┤
        │ auth_db (PG) │   │ notifications_redis │  │ events_db(PG)│
        └──────────────┘   │ notifications_worker│  │ events_redis │
                           │ n8n :5678           │  │ events_worker│
                           └─────────────────────┘  └──────────────┘
                    ────── red interna: essenz_network ──────
```

## Estructura de carpetas

```
EssenzCompany/
├── docker-compose.yml      # ← Compose MAESTRO (este stack)
├── gateway/
│   ├── Dockerfile          # nginx:1.27-alpine + conf horneada
│   └── nginx.conf          # enrutamiento por path
├── essenz-auth/            # Django/DRF + PostgreSQL
├── essenz-notifications/   # Django + Celery + Redis + n8n
└── essenz-events/          # Django + Celery + Redis + PostgreSQL
```

## Arranque

```powershell
# 1. Baja los stacks individuales si están corriendo (comparten puertos)
docker compose -f essenz-auth/docker-compose.yml down
docker compose -f essenz-notifications/docker-compose.yml down
docker compose -f essenz-events/docker-compose.yml down

# 2. Levanta TODO el ecosistema
docker compose up -d --build
```

## Mapa de rutas del gateway (puerto público: 8080)

| Ruta pública                  | Servicio interno                                    |
|-------------------------------|-----------------------------------------------------|
| `/api/auth/...`               | `essenz_auth:8000/api/v1/auth/...`                  |
| `/api/notifications/...`      | `essenz_notifications:8000/api/notifications/...`   |
| `/api/events/...`             | `essenz_events:8000/api/v1/events/...`              |
| `/health/auth/`               | `essenz_auth:8000/health/`                          |
| `/health/events/`             | `essenz_events:8000/health/`                        |
| `/`                           | JSON con el mapa de rutas                           |

El admin de Django y Swagger usan rutas absolutas, así que no pasan por el
gateway; en dev siguen accesibles directo: auth `:8000`, notifications
`:8001`, events `:8002`, n8n `:5678`. Las bases de datos quedan expuestas
solo para inspección (auth `:5432`, events `:5433`, redis `:6379`/`:6380`).

## Variables de entorno cruzadas (cómo un servicio encuentra a otro)

Regla del stack maestro: **cada proyecto conserva su `.env` intacto** (sigue
funcionando standalone) y el compose maestro sobrescribe SOLO las variables
de interconexión vía `environment:`, que en Compose tiene prioridad sobre
`env_file:`. Dentro de `essenz_network` los servicios se resuelven por
nombre DNS — nunca uses `host.docker.internal` ni `localhost` entre
contenedores.

> **Importante — guion bajo vs guion:** Django rechaza cabeceras `Host`
> con `_` (RFC 1034/1035), así que `http://essenz_events:8000` responde
> 400 aunque el DNS resuelva. Por eso cada web tiene un **alias de red
> con guion** (`essenz-auth`, `essenz-notifications`, `essenz-events`):
> usa siempre el alias en URLs servicio-a-servicio. El `container_name`
> con `_` se conserva para `docker logs/exec` y para clientes no-Django.

| Servicio        | Variable                           | Valor en el stack maestro                                      | Para qué |
|-----------------|------------------------------------|----------------------------------------------------------------|----------|
| auth            | `POSTGRES_HOST`                    | `auth_db`                                                      | su propia DB |
| auth            | `ESSENZ_EVENTS_URL`                | `http://essenz-events:8000/api/v1/events/`                     | publicar eventos (Nivel 2) |
| notifications   | `CELERY_BROKER_URL` / `REDIS_URL`  | `redis://notifications_redis:6379/0`                           | su broker |
| notifications   | `N8N_WEBHOOK_URL`                  | `http://n8n:5678/webhook/essenz-events`                        | orquestar flujos |
| events          | `POSTGRES_HOST`                    | `events_db`                                                    | su propia DB |
| events          | `CELERY_BROKER_URL`                | `redis://events_redis:6379/0`                                  | su broker |
| events          | `ESSENZ_NOTIFICATIONS_WEBHOOK_URL` | `http://essenz-notifications:8000/api/notifications/send/`     | despachar notificaciones (Nivel 2) |

Para añadir una nueva variable cruzada: declárala en el bloque
`environment:` del servicio consumidor en el `docker-compose.yml` maestro,
apuntando al **alias con guion** del servicio destino (`essenz-auth`,
`essenz-notifications`, `essenz-events`) o al **nombre de servicio** para
la infraestructura (`auth_db`, `events_redis`, `n8n`, …).

> Nota: `ESSENZ_EVENTS_URL` ya queda inyectada en auth para el Nivel 2,
> pero auth todavía no la consume (la publicación de `user.registered`
> hacia el hub es trabajo de Nivel 2).

## Superusuario del admin (essenz-events)

Al arrancar, `events_web` ejecuta `python manage.py ensure_superuser`
(comando idempotente: crea el admin solo si no existe). Credenciales en
`essenz-events/.env`:

```
DJANGO_SUPERUSER_USERNAME=admin
DJANGO_SUPERUSER_EMAIL=admin@essenz.local
DJANGO_SUPERUSER_PASSWORD=...   # sin esta variable NO se crea nada
```

## Tests

```powershell
# essenz-events (20 tests: API, tasks, handlers, ensure_superuser)
docker compose exec events_web python manage.py test

# essenz-auth (17 tests)
docker compose exec auth_web python manage.py test
```

## Smoke test del ecosistema

```powershell
# Gateway vivo
curl http://localhost:8080/

# Salud de los servicios a través del gateway
curl http://localhost:8080/health/auth/
curl http://localhost:8080/health/events/

# Publicar un evento a través del gateway
curl -X POST http://localhost:8080/api/events/ -H "Content-Type: application/json" `
     -d '{"event_type":"user.registered","source":"essenz-auth","payload":{"user_id":1}}'
```
