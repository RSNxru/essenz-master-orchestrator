# 03 · Red y variables cruzadas

## La red interna

Todos los contenedores viven en una única red bridge:

```yaml
networks:
  essenz_network:
    name: essenz_network
    driver: bridge
```

Dentro de ella, el DNS embebido de Docker resuelve tres tipos de nombre:

| Nombre | Ejemplo | Cuándo usarlo |
|---|---|---|
| Nombre de servicio | `auth_db`, `events_redis`, `n8n` | Infraestructura (bases, brokers, n8n) |
| `container_name` | `essenz_auth`, `essenz_events` | `docker logs` / `docker exec` y clientes no-Django |
| **Alias con guion** | `essenz-auth`, `essenz-events` | **URLs HTTP servicio-a-servicio** (Django rechaza `Host` con `_`) |

Regla práctica: para una URL `http://...` entre servicios, **siempre el alias con guion**. Para todo lo demás, el nombre que prefieras.

Nunca uses `host.docker.internal` ni `localhost` entre contenedores: el primero es un hack que sale de la red interna y vuelve a entrar por el host; el segundo apunta al propio contenedor.

## Cómo se configuran las variables cruzadas

El principio: **cada proyecto conserva su `.env` intacto** — sigue siendo válido para correr el servicio standalone con su propio compose. El compose maestro sobrescribe *solo* las variables de interconexión usando `environment:`, que en Docker Compose **tiene prioridad sobre `env_file:`**:

```yaml
events_web:
  env_file:
    - ./essenz-events/.env          # todo lo propio del servicio
  environment:                       # solo lo que cambia en el stack maestro
    POSTGRES_HOST: events_db
    CELERY_BROKER_URL: redis://events_redis:6379/0
    ESSENZ_NOTIFICATIONS_WEBHOOK_URL: http://essenz-notifications:8000/api/notifications/send/
```

Los workers reciben **el mismo bloque** que su web mediante anclas de YAML (`&events_cross_env` / `*events_cross_env`): los handlers corren en el worker, así que necesitan las mismas URLs. Definirlas una vez evita que diverjan.

### El mapa completo

| Servicio | Variable | Valor en el stack maestro | Para qué |
|---|---|---|---|
| auth | `POSTGRES_HOST` | `auth_db` | su propia base |
| auth | `ESSENZ_EVENTS_URL` | `http://essenz-events:8000/api/v1/events/` | publicar eventos (Nivel 2) |
| notifications | `REDIS_URL` / `CELERY_BROKER_URL` | `redis://notifications_redis:6379/0` | su broker |
| notifications | `CELERY_RESULT_BACKEND` | `redis://notifications_redis:6379/1` | resultados |
| notifications | `N8N_WEBHOOK_URL` | `http://n8n:5678/webhook/essenz-events` | disparar flujos |
| events | `POSTGRES_HOST` | `events_db` | su propia base |
| events | `CELERY_BROKER_URL` | `redis://events_redis:6379/0` | su broker |
| events | `CELERY_RESULT_BACKEND` | `redis://events_redis:6379/1` | resultados |
| events | `ESSENZ_NOTIFICATIONS_WEBHOOK_URL` | `http://essenz-notifications:8000/api/notifications/send/` | despachar notificaciones (Nivel 2) |

> `ESSENZ_EVENTS_URL` y `ESSENZ_NOTIFICATIONS_WEBHOOK_URL` ya están inyectadas y apuntan a endpoints reales, pero **el código que las consume es Nivel 2**: hoy auth no publica eventos y el handler de events solo registra en bitácora.

## Receta: conectar un servicio nuevo al ecosistema

Supongamos que llega `essenz-billing` (Nivel 2). Pasos en el compose maestro:

1. **Servicios**: añade `billing_web` (build `./essenz-billing`), su `billing_db` y, si usa Celery, `billing_redis` + `billing_worker`.
2. **Red**: conéctalos a `essenz_network`; al web dale `container_name: essenz_billing` y el alias `essenz-billing`.
3. **Variables cruzadas**: en `environment:` de `billing_web` (y su worker), apunta a quien necesite — por ejemplo `ESSENZ_EVENTS_URL: http://essenz-events:8000/api/v1/events/` para publicar `order.paid`.
4. **Gateway**: en `nginx.conf`, añade la variable `set $billing_backend essenz_billing;` y su `location /api/billing/ { rewrite ... ; proxy_pass http://$billing_backend:8000; }`, y agrega el servicio al `depends_on` del gateway. Rebuild: `docker compose up -d --build gateway`.
5. **Consumidores**: si events debe reaccionar a sus eventos, registra los handlers en `apps/events/handlers.py` de essenz-events (`"order.paid": handle_order_paid` o el dominio `"order"` completo).

Cinco pasos, cero cambios en los servicios existentes — esa es la prueba de que el desacoplamiento funciona.
