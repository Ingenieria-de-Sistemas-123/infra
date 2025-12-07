# Guía del Permission Service en la infraestructura

Este documento resume cómo se integra el microservicio `permission-service` dentro del stack de `infra`, qué contenedores lo acompañan, qué variables necesita y qué endpoints expone. La idea es poder seguir el flujo completo “desde el compose hasta la API”.

---

## 1. Rol del servicio

- Define y consulta los permisos sobre snippets (`OWNER`, `SHARED`) y resuelve si un usuario puede ejecutar una acción puntual.
- Expone endpoints para sincronizar la cuenta de un usuario autenticado con Auth0 y para autorizar acciones (`/api/me`, `/api/me/sync`, `/api/permissions/**`).
- Valida los JWT emitidos por Auth0 mediante `SPRING_SECURITY_OAUTH2_RESOURCESERVER_JWT_ISSUER_URI` y el audience `AUTH0_AUDIENCE`.

Los controladores están en `permission-service/src/main/java/com/snippetsearcher/permission/controller` e incluyen:

| Clase | Endpoint base | Responsabilidad |
|-------|---------------|-----------------|
| `PermissionController` | `/api/permissions` | CRUD de permisos y consultas por usuario/snippet. |
| `AuthorizationController` | `/api/permissions/authorize` | Evalúa si una acción está permitida. |
| `MeController` | `/api/me` | Sincroniza/retorna el usuario autenticado contra la DB local. |
| `StatusController` | `/actuator/health` | Healthcheck simple utilizado por Docker/NGINX. |

---

## 2. Contenedores según `docker-compose`

### 2.1 Entorno local (`docker-compose.local.yml`)

| Servicio | Imagen | Detalles |
|----------|--------|----------|
| `permission-db-local` | `postgres:16` | Variables en `env/local/permission-db.env`, expone `15432:5432` para debug, healthcheck `pg_isready`. |
| `permission-service-local` | `permission-service` | Requiere construir la imagen con `docker build -t permission-service ../permission-service`. Monta variables desde `env/local/permission-service.env`, expone `18080:8080`, depende de la DB y tiene healthcheck `wget …/actuator/health`. |
| `reverse-proxy-local` | `nginx:alpine` | Publica `http://localhost:8080`. El bloque `/api/` de `nginx/conf.d/local.conf` reenvía al `permission-service`. `printscript-ui` y el resto de servicios pasan por este proxy. |

Comando típico para dejarlo corriendo:

```bash
docker compose -f docker-compose.local.yml up -d permission-db permission-service reverse-proxy
```

### 2.2 Entorno dev (`docker-compose.dev.yml`)

- Usa la imagen publicada en GHCR `ghcr.io/ingenieria-de-sistemas-123/permission-service:dev`.
- No expone puertos al host, sólo `expose: 8080`. Todo el tráfico entra a través de `reverse-proxy-dev`, que publica `https://snippet-org-dev.duckdns.org` y enruta `/api/` hacia `permission-service`.
- Parametriza credenciales vía `.env.dev` (variables `SPRING_DATASOURCE_URL_PERMISSIONDB`, `SPRING_SECURITY_OAUTH2_RESOURCESERVER_JWT_ISSUER_URI`, `AUTH0_AUDIENCE`, etc.).

### 2.3 Entorno prod (`docker-compose.prod.yml`)

- Imagen `ghcr.io/ingenieria-de-sistemas-123/permission-service:prod`.
- Corre en la red `ss-prod`, con `permission-db` persistiendo en el volumen `permission_pg_data_prod`.
- Healthcheck idéntico y restart policy `unless-stopped`.
- Proxy público (`reverse-proxy`) publica `https://snippet-org-prod.duckdns.org` y también delega `/api/` hacia `permission-service`.

---

## 3. Variables y configuración

| Variable | Descripción | Fuente |
|----------|-------------|--------|
| `SPRING_DATASOURCE_URL/USERNAME/PASSWORD` | Conexion a `permission-db`. | `env/local/permission-service.env`, `.env.dev`, `.env.prod`. |
| `SPRING_SECURITY_OAUTH2_RESOURCESERVER_JWT_ISSUER_URI` | URL del issuer de Auth0 (p.ej. `https://dev-7t5evioih4aq30t7.us.auth0.com/`). | Mismos `.env`. |
| `AUTH0_AUDIENCE` | API Audience configurada en Auth0 (`https://snippet-api`). | Mismos `.env`. |
| `POSTGRES_*` | Inicialización de la DB. | `env/local/permission-db.env` o `.env.*` |

Consejos:

- Para cambiar credenciales, modifica los archivos `.env` correspondientes y recrea los contenedores (`docker compose … up -d permission-db permission-service`).
- La UI (`printscript-ui`) espera que el backend esté detrás del proxy, por lo que normalmente no se invoca `http://localhost:18080` directamente salvo debugging.

---

## 4. Flujo de red

1. El navegador accede a `http://localhost:8080` (local) o al dominio DuckDNS (dev/prod).
2. `reverse-proxy` recibe la petición y aplica las reglas de `nginx/conf.d/*.conf`.
3. Rutas:
   - `/api/**` → `permission-service:8080` (permisos, `/me`, `/authorize`).
   - `/snippets/**`, `/language/**` → otros servicios.
4. `permission-service` valida el JWT de Auth0, sincroniza al usuario (`/api/me/sync`) y responde.

---

## 5. Endpoints principales

| Método | Ruta | Descripción |
|--------|------|-------------|
| `GET /actuator/health` | Health simple (usado por Docker/NGINX). |
| `POST /api/me/sync` | Crea/actualiza el usuario autenticado en la DB y devuelve su registro. |
| `GET /api/me` | Retorna los datos del usuario autenticado (crea en caso de no existir). |
| `POST /api/permissions` | Crea un permiso `OWNER` o `SHARED`. |
| `DELETE /api/permissions/{snippetId}/{userId}` | Elimina un permiso existente. |
| `GET /api/permissions/{snippetId}/author` | Devuelve el owner del snippet. |
| `GET /api/permissions/user/{userId}?type=` | Lista permisos filtrando opcionalmente por tipo. |
| `POST /api/permissions/authorize` | Endpoint centralizado para que otros servicios consulten si una acción está permitida. |

Todos requieren un JWT válido (Resource Server) excepto el healthcheck.

---

## 6. Operaciones habituales

- **Construir la imagen local:** `docker build -t permission-service ../permission-service`.
- **Levantar sólo este servicio + DB:** `docker compose -f docker-compose.local.yml up -d permission-db permission-service`.
- **Ver logs:** `docker compose -f docker-compose.local.yml logs -f permission-service`.
- **Revisar salud manualmente:** `curl http://localhost:18080/actuator/health`.
- **Acceder a la DB local:** `psql postgres://permission:permission@localhost:15432/permission`.

---

## 7. Persistencia y migraciones

- La base usa Postgres 16. En local no se definió volumen (se pierde al hacer `docker compose down -v`). En prod sí (`permission_pg_data_prod`).
- Las migraciones Flyway viven en `permission-service/src/main/resources/db/migration` y se ejecutan automáticamente al iniciar la app.

---

## 8. Integración con otros servicios

- **printscript-ui:** realiza `POST /api/me/sync` al cargar, luego consume `/api/permissions/**` según corresponda.
- **snippet-service:** consulta `/api/permissions/authorize` antes de permitir operaciones sobre snippets.
- **nginx:** depende de que `permission-service` esté `healthy` para exponer los dominios públicos; si falla el healthcheck, el proxy no levanta.

Con este panorama deberías poder seguir el rastro completo: construir la imagen, levantar los contenedores, entender qué variables setear y cómo viaja cada request hasta terminar en los endpoints del `permission-service`.
