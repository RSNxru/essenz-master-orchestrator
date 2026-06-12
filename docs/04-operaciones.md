# 04 · Operaciones

## Primer arranque

El orquestador asume esta estructura (cada microservicio es su propio repo, clonado al lado):

```powershell
git clone https://github.com/RSNxru/essenz-master-orchestrator EssenzCompany
cd EssenzCompany
git clone https://github.com/RSNxru/essenz-auth
git clone https://github.com/RSNxru/essenz-notifications
git clone https://github.com/RSNxru/essenz-events

# Cada proyecto necesita su .env (copiar desde su .env.example y ajustar)
copy essenz-auth\.env.example essenz-auth\.env
copy essenz-notifications\.env.example essenz-notifications\.env
copy essenz-events\.env.example essenz-events\.env

docker compose up -d --build
```

**Si los stacks individuales están corriendo, bájalos primero** — comparten los puertos de desarrollo:

```powershell
docker compose -f essenz-auth/docker-compose.yml down
docker compose -f essenz-notifications/docker-compose.yml down
docker compose -f essenz-events/docker-compose.yml down
```

(`down` no borra datos: los volúmenes persisten. Ojo: el stack maestro usa **sus propios** volúmenes con prefijo `essenz_`, separados de los de cada stack individual.)

## Verificar que todo vive

```powershell
docker compose ps        # 11 contenedores Up (las bases/brokers, "healthy")

curl http://localhost:8080/                  # mapa de rutas del gateway
curl http://localhost:8080/health/auth/      # {"service":"essenz-auth","status":"ok",...}
curl http://localhost:8080/health/events/    # {"status":"ok","checks":{"database":true,"broker":true}}
```

Prueba end-to-end completa (evento por el gateway, procesado asíncrono):

```powershell
curl -X POST http://localhost:8080/api/events/ -H "Content-Type: application/json" `
     -d '{\"event_type\":\"user.registered\",\"source\":\"essenz-auth\",\"payload\":{\"user_id\":1}}'
# → 202 con "status":"RECEIVED" y un id

curl http://localhost:8080/api/events/<id>/
# → unos segundos después: "status":"PROCESSED"
```

## Comandos del día a día

| Tarea | Comando |
|---|---|
| Levantar / aplicar cambios del compose | `docker compose up -d` |
| Rebuild de un servicio | `docker compose up -d --build events_web` |
| Logs en vivo | `docker compose logs -f gateway events_worker` |
| Reiniciar un worker tras editar handlers | `docker compose restart events_worker` |
| Shell de Django | `docker compose exec events_web python manage.py shell` |
| Bajar todo (conserva datos) | `docker compose down` |
| Bajar todo y **borrar datos** | `docker compose down -v` |

Los servicios web montan el código del host como volumen: editar un `.py` recarga solo (runserver). **Los workers de Celery no se recargan solos** — reinícialos tras cambiar tareas o handlers.

## Tests

```powershell
docker compose exec events_web python manage.py test    # 20 tests de essenz-events
docker compose exec auth_web python manage.py test      # 17 tests de essenz-auth
```

## Superusuario del admin (essenz-events)

`events_web` ejecuta `python manage.py ensure_superuser` en cada arranque. Es **idempotente**: crea el superusuario solo si no existe, así que no rompe reinicios (a diferencia de `createsuperuser --noinput`, que falla si ya existe). Lee del `.env` de essenz-events:

```
DJANGO_SUPERUSER_USERNAME=admin
DJANGO_SUPERUSER_EMAIL=admin@essenz.local
DJANGO_SUPERUSER_PASSWORD=...    # sin esta variable NO crea nada (a propósito)
```

Admin de events: `http://localhost:8002/admin/`.

## Solución de problemas

| Síntoma | Causa probable | Remedio |
|---|---|---|
| `port is already allocated` al levantar | Un stack individual sigue corriendo | `docker compose down` en la carpeta de ese proyecto |
| 400 Bad Request entre servicios | URL con guion bajo (`essenz_events`) | Usa el alias con guion: `essenz-events` (ver [02 · Gateway](02-gateway.md)) |
| El gateway entrega una ruta al servicio equivocado | IPs congeladas (no debería pasar con el resolver dinámico) | `docker compose restart gateway` |
| 502 en una ruta del gateway | El backend de esa ruta está caído o arrancando | `docker compose logs <servicio>`; espera el healthcheck |
| Evento se queda en `RECEIVED` | El worker de events no corre o no ve el broker | `docker compose logs events_worker`; `restart events_worker` |
| Cambié un handler y no pasa nada | Celery no recarga código | `docker compose restart events_worker` |
| `DisallowedHost` en logs de un Django | Petición con `Host` no permitido (solo producción) | Añade el host a `DJANGO_ALLOWED_HOSTS` del `.env` |
