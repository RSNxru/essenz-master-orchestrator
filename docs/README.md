# Documentación de Essenz Master Orchestrator

Infraestructura unificada del ecosistema Essenz — **Nivel 1 (Fundación), cierre**.

| Documento | Contenido |
|---|---|
| [01 · Arquitectura](01-arquitectura.md) | Qué es el orquestador, los 11 contenedores, la red interna y las decisiones de diseño |
| [02 · API Gateway](02-gateway.md) | El proxy inverso Nginx explicado línea a línea: rutas, rewrites y los dos problemas reales que resuelve |
| [03 · Red y variables cruzadas](03-red-y-variables.md) | Cómo se encuentran los servicios entre sí y cómo conectar uno nuevo |
| [04 · Operaciones](04-operaciones.md) | Arranque, comandos del día a día, tests y solución de problemas |

## Vista rápida

- **Punto de entrada único:** `http://localhost:8080` (gateway Nginx)
- **Rutas públicas:** `/api/auth/` · `/api/notifications/` · `/api/events/` · `/health/auth/` · `/health/events/`
- **Levantar todo:** `docker compose up -d --build` → 11 contenedores: gateway + 3 APIs + 2 workers + n8n + 2 PostgreSQL + 2 Redis
- **Accesos directos de desarrollo:** auth `:8000` · notifications `:8001` · events `:8002` · n8n `:5678`
- **Los tres microservicios** viven en sus propios repos: [essenz-auth](https://github.com/RSNxru/essenz-auth) · [essenz-notifications](https://github.com/RSNxru/essenz-notifications) · [essenz-events](https://github.com/RSNxru/essenz-events)
