# =============================================================
# Essenz Master Orchestrator — API Gateway
# Nginx alpine con la configuración de enrutamiento horneada en
# la imagen (sin volúmenes: la imagen es autocontenida y portable).
# =============================================================
FROM nginx:1.27-alpine

# Reemplaza el server por defecto por el enrutador de Essenz
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Nota: no se puede hacer `nginx -t` en build — los upstreams
# (essenz_auth, etc.) solo resuelven dentro de essenz_network.

EXPOSE 80
