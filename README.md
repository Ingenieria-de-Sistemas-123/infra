# Guía de Infra: Reverse Proxy, HTTPS y DuckDNS

Esta guía explica en profundidad cómo se ocultaron los servicios detrás de NGINX, cómo se habilitó HTTPS con Let’s Encrypt usando DuckDNS y cómo probar y depurar paso a paso los entornos dev y prod.

---
## 1. Objetivos
- Unificar el acceso externo a la plataforma usando un único dominio por entorno (dev y prod).
- Exponer únicamente los puertos 80 (HTTP) y 443 (HTTPS) en la VM.
- Ocultar servicios y bases de datos (no exponer puertos al host) y acceder vía red interna Docker.
- Obtener certificados confiables (Let’s Encrypt) y renovarlos automáticamente.
- Mantener paridad entre dev y prod para reducir sorpresas al desplegar.

---
## 2. Arquitectura Resumida

Entorno dev (similar a prod):
- Dominio: https://snippet-org-dev.duckdns.org
- Reverse proxy (NGINX) recibe todas las solicitudes y enruta por paths:
  - /api → permission-service
  - /snippets → snippet-service
  - /language → language-service
- Servicios Spring Boot escuchan internamente en puerto 8080 (expose). No hay mapeo de puertos al host.
- Bases de datos Postgres (permission-db, snippet-db) expuestas solo a la red interna Docker en 5432 (expose). No hay `ports`.
- Certbot renueva certificados y recarga NGINX periódicamente.

Flujo de red (ejemplo dev):
1. Cliente hace petición a https://snippet-org-dev.duckdns.org/snippets/...
2. NGINX (reverse-proxy-dev) acepta en 443 y termina TLS.
3. NGINX reescribe/encamina la ruta hacia snippet-service (http://snippet-service:8080/).
4. snippet-service responde; NGINX devuelve respuesta al cliente.
5. Para DB, snippet-service conecta directamente a snippet-db:5432 dentro de la red Docker (sin pasar por NGINX).

---
## 3. Componentes y Rol de Cada Uno
- DuckDNS: Provee dominio dinámico apuntando a la IP pública de la VM.
- NGINX: Reverse proxy que:
  - Termina TLS (carga certificados).
  - Aplica redirección HTTP→HTTPS.
  - Encamina paths a servicios internos.
  - Añade headers de seguridad (HSTS, X-Frame-Options, etc.).
- Certbot: Cliente ACME que emite y renueva certificados Let's Encrypt usando desafío HTTP-01 (webroot).
- Servicios backend (permission, snippet, language): Aplicaciones Spring Boot; exponen actuator health internamente.
- Bases de datos (Postgres): Persistencia; accesibles solo para sus servicios vía hostnames de contenedor.

---
## 4. Decisiones Clave y Razones
1. Un único dominio con paths en lugar de subdominios: Menor complejidad, un solo certificado por entorno, menos problemas CORS.
2. Ocultar puertos internos con `expose` en vez de `ports`: Reduce superficie de ataque y replica comportamiento real de producción.
3. Certbot webroot en contenedor separado: Evita detener NGINX y simplifica renovación automática.
4. Healthchecks internos vía wget curl a actuator/health: Permite a Docker saber si el servicio está listo sin exponer el puerto al host.
5. Headers de seguridad: Mejora score de seguridad básico y evita clickjacking, sniffing de contenido.

---
## 5. Despliegue Paso a Paso (Dev)

Precondiciones:
- DuckDNS con `snippet-org-dev.duckdns.org` apuntando a la IP pública.
- Puertos 80 y 443 abiertos en firewall.

Pasos:
```bash
# 1. Levantar stack
docker compose -f docker-compose.dev.yml pull
docker compose -f docker-compose.dev.yml up -d --remove-orphans

# 2. Emitir certificado inicial (solo primera vez)
docker compose -f docker-compose.dev.yml run --rm certbot certbot certonly --webroot -w /var/www/certbot -d snippet-org-dev.duckdns.org --email ing.sistemas.123.encrypt@gmail.com --agree-tos --no-eff-email

# 3. Recargar NGINX para cargar el nuevo cert
docker compose -f docker-compose.dev.yml exec reverse-proxy-dev nginx -s reload

# 4. Verificar salud del proxy
curl -k https://snippet-org-dev.duckdns.org/actuator/health

# 5. Verificar rutas a servicios
curl -k https://snippet-org-dev.duckdns.org/snippets/actuator/health || true
curl -k https://snippet-org-dev.duckdns.org/api/actuator/health || true
curl -k https://snippet-org-dev.duckdns.org/language/actuator/health || true
```
(Actuator/health por path depende de que cada servicio exponga ese endpoint en su contexto; si no, probar un endpoint real de la API.)

---
## 6. Despliegue Paso a Paso (Prod)

Precondiciones:
- DuckDNS con `snippet-org-prod.duckdns.org` apuntando a la IP pública.
- Puertos 80 y 443 abiertos.

Pasos:
```bash
# 1. Levantar stack
docker compose -f docker-compose.prod.yml pull
docker compose -f docker-compose.prod.yml up -d --remove-orphans

# 2. Emitir certificado inicial
docker compose -f docker-compose.prod.yml run --rm certbot certbot certonly --webroot -w /var/www/certbot -d snippet-org-prod.duckdns.org --email ing.sistemas.123.encrypt@gmail.com --agree-tos --no-eff-email

# 3. Recargar NGINX
docker compose -f docker-compose.prod.yml exec reverse-proxy nginx -s reload

# 4. Pruebas
curl -k https://snippet-org-prod.duckdns.org/actuator/health
curl -k https://snippet-org-prod.duckdns.org/api/actuator/health || true
curl -k https://snippet-org-prod.duckdns.org/snippets/actuator/health || true
```

---
## 7. Renovación de Certificados
- Contenedor `certbot` / `certbot-dev` ejecuta cada ~12h: `certbot renew --webroot ...` y luego `nginx -s reload`.
- Revisión manual:
```bash
docker logs certbot-dev --tail 100
docker logs certbot --tail 100
```
- Forzar renovación (solo para pruebas cerca del vencimiento):
```bash
docker compose -f docker-compose.dev.yml run --rm certbot certbot renew --dry-run
```

---
## 8. Debugging Profundo

### 8.1 Ver estado de contenedores
```bash
docker compose -f docker-compose.dev.yml ps
docker compose -f docker-compose.dev.yml logs -f reverse-proxy-dev
```

### 8.2 Revisar salud interna de servicios (sin proxy)
```bash
docker compose -f docker-compose.dev.yml exec snippet-service curl -fsS http://localhost:8080/actuator/health
docker compose -f docker-compose.dev.yml exec language-service curl -fsS http://localhost:8080/actuator/health
docker compose -f docker-compose.dev.yml exec permission-service curl -fsS http://localhost:8080/actuator/health
```

### 8.3 Revisar conectividad a DB
```bash
docker compose -f docker-compose.dev.yml exec permission-service bash -c 'apt-get update && apt-get install -y postgresql-client || true; psql -h permission-db -U "$SPRING_DATASOURCE_USERNAME" -d "$SPRING_DATASOURCE_URL"'
```
(Adaptar si la imagen no tiene bash/apt; alternativa: ejecutar psql dentro del contenedor de la DB.)

### 8.4 Simular fallo de servicio
- Parar un servicio:
```bash
docker compose -f docker-compose.dev.yml stop snippet-service
```
- Ver respuesta del proxy (debería 502 en /snippets/).
- Reiniciar:
```bash
docker compose -f docker-compose.dev.yml start snippet-service
```

### 8.5 Analizar logs de NGINX
- Accesos:
```bash
docker compose -f docker-compose.dev.yml exec reverse-proxy-dev tail -f /var/log/nginx/access.log
```
- Errores:
```bash
docker compose -f docker-compose.dev.yml exec reverse-proxy-dev tail -f /var/log/nginx/error.log
```

### 8.6 Errores comunes
- 502: servicio caído o puerto incorrecto → verificar healthchecks y `proxy_pass`.
- 404 en ACME: root incorrecto en `/.well-known/acme-challenge/` → revisar que ruta apunte a `/var/www/certbot`.
- `certbot renew` no recarga NGINX: comprobar que comando del contenedor incluye `nginx -s reload` y que el binario existe.

---
## 9. Paridad Dev/Prod y Buenas Prácticas
- Estructura idéntica entre `docker-compose.dev.yml` y `docker-compose.prod.yml` facilita reproducir errores.
- Ocultar DB y servicios reduce exposición y prepara para escenarios Multi-Host con una red overlay.
- Renovación automática previene expiración y downtime.
- Logs centralizados en NGINX permiten analizar tráfico externo sin entrar a cada servicio.

---
## 10. Limpieza y Mantenimiento
```bash
# Eliminar contenedores y volúmenes (dev)
docker compose -f docker-compose.dev.yml down -v

# Prune de imágenes antiguas
docker image prune -f
```

---
## 11. Checklist Rápido de Verificación (Dev)
1. DNS apunta a IP pública (ping snippet-org-dev.duckdns.org).
2. Puerto 80 accesible (curl http://snippet-org-dev.duckdns.org -I devuelve 301).
3. Certificado emitido (archivos en /etc/letsencrypt/live/snippet-org-dev.duckdns.org dentro del contenedor NGINX).
4. HTTPS responde (curl -k https://snippet-org-dev.duckdns.org/actuator/health).
5. Rutas funcionan (/api, /snippets, /language).
6. Renovación en logs (docker logs certbot-dev muestra intentos cada 12h).

---
## 12. Extensiones Futuras (Ideas)
- Rate limiting por ruta en NGINX.
- Integrar WAF o ModSecurity.
- Métricas y dashboards (Prometheus/Grafana) detrás del proxy.
- Canary releases usando distintos upstreams.

---
## 13. Referencias
- Let’s Encrypt Docs: https://letsencrypt.org/docs/
- NGINX Reverse Proxy: https://nginx.org/en/docs/
- Certbot Webroot: https://eff-certbot.readthedocs.io/en/stable/using.html#webroot

---
## 14. Conclusión
La infraestructura implementada estandariza el acceso, mejora la seguridad y simplifica despliegues. El diseño por paths y un solo dominio reduce complejidad inicial sin cerrar puertas a futuros escalados.

---
## 15. Configuración DuckDNS y VM (Dev)

### 15.1 Registrar dominio en DuckDNS
1. Inicia sesión en https://www.duckdns.org/.
2. En la sección de dominios, crea `snippet-org-dev` (esto genera `snippet-org-dev.duckdns.org`).
3. Copia el token de tu cuenta (lo necesitarás si automatizas actualización IP).

### 15.2 Apuntar dominio a la IP pública
1. Obtén la IP pública de tu VM (por ejemplo `curl ifconfig.me`).
2. En DuckDNS, ingresa esa IP en el campo de actualización del dominio y guarda.
3. Verifica propagación:
   ```bash
   ping snippet-org-dev.duckdns.org
   nslookup snippet-org-dev.duckdns.org
   ```

### 15.3 (Opcional) Actualización dinámica de IP
Si la IP cambia (residencial o reinicios):
```bash
curl "https://duckdns.org/update?domains=snippet-org-dev&token=<TOKEN>&ip="
```
Puedes agregarlo a un cron (cada 5 minutos) para mantener el DNS actualizado.

### 15.4 Firewall / Seguridad en la VM
1. Abre puertos 80 y 443 hacia Internet:
   - En Linux con ufw:
     ```bash
     sudo ufw allow 80/tcp
     sudo ufw allow 443/tcp
     sudo ufw enable
     sudo ufw status
     ```
   - En proveedores cloud (AWS, Azure, etc.): agrega reglas en el security group / NSG.
2. Verifica que el puerto 80 responde:
   ```bash
   curl -I http://snippet-org-dev.duckdns.org || curl -I http://<IP_PUBLICA>
   ```

### 15.5 Preparar entorno Docker
1. Instala Docker y Docker Compose (plugin):
   - Verifica:
     ```bash
     docker version
     docker compose version
     ```
2. Clona el repo infra en la VM:
   ```bash
   git clone <URL_REPO_INFRA>.git
   cd infra
   git checkout dev
   ```

### 15.6 Variables de entorno
1. Crea archivo `.env.dev.dev` si no existe con valores reales:
   ```env
   DB_NAME=snippets
   DB_USERNAME=snippets_user
   DB_PASSWORD=snippets_pass
   DB_PERMISSION_NAME=permission
   DB_PERMISSION_USER=permission_user
   DB_PERMISSION_PASSWORD=permission_pass
   SPRING_DATASOURCE_URL_SNIPPET=jdbc:postgresql://snippet-db:5432/snippets
   SPRING_DATASOURCE_USERNAME_SNIPPET=snippets_user
   SPRING_DATASOURCE_PASSWORD_SNIPPET=snippets_pass
   SPRING_DATASOURCE_URL_PERMISSIONDB=jdbc:postgresql://permission-db:5432/permission
   SPRING_DATASOURCE_USERNAME_PERMISSIONDB=permission_user
   SPRING_DATASOURCE_PASSWORD_PERMISSIONDB=permission_pass
   ```
2. No incluir secretos sensibles en el repo público.

### 15.7 Levantar stack y emitir certificado (Dev)
Ejecuta la secuencia descrita en la sección 5.

### 15.8 Validaciones finales
Checklist rápido:
- `curl -k https://snippet-org-dev.duckdns.org/actuator/health` devuelve status UP.
- Los servicios responden bajo sus paths.
- `docker compose -f docker-compose.dev.yml ps` muestra health `healthy` para DBs y servicios.
- Certificados presentes: `docker compose -f docker-compose.dev.yml exec reverse-proxy-dev ls /etc/letsencrypt/live/snippet-org-dev.duckdns.org`

---
## 16. Refinamientos de Producción Aplicados

Se ajustó `docker-compose.prod.yml` para mejorar claridad y separación:
- Red `ss-prod` dedicada para evitar colisiones con dev.
- Volúmenes renombrados con sufijo `_prod`:
  - `pg_data_prod`, `permission_pg_data_prod` para persistencia de cada DB.
  - `nginx_conf_prod`, `certbot_www_prod`, `letsencrypt_prod` para proxy y certificados.
- Uso de archivo `.env.prod` (debes crearlo) para credenciales y configuraciones de producción separadas de dev.
- Healthchecks en ambas bases y servicios para garantizar orden de arranque y detectar fallos tempranos.

### 16.1 Archivo .env.prod (ejemplo)
Crea `./.env.prod` (no lo subas si contiene secretos reales):
```env
DB_NAME=snippets
DB_USERNAME=snippets_prod
DB_PASSWORD=CHANGEME_STRONG
DB_PERMISSION_NAME=permission
DB_PERMISSION_USER=permission_prod
DB_PERMISSION_PASSWORD=CHANGEME_STRONG
SPRING_DATASOURCE_URL_SNIPPET=jdbc:postgresql://snippet-db:5432/snippets
SPRING_DATASOURCE_USERNAME_SNIPPET=snippets_prod
SPRING_DATASOURCE_PASSWORD_SNIPPET=CHANGEME_STRONG
SPRING_DATASOURCE_URL_PERMISSIONDB=jdbc:postgresql://permission-db:5432/permission
SPRING_DATASOURCE_USERNAME_PERMISSIONDB=permission_prod
SPRING_DATASOURCE_PASSWORD_PERMISSIONDB=CHANGEME_STRONG
```
Ajusta usuarios, contraseñas y nombres según políticas de seguridad.

### 16.2 Tags de Imágenes
Actualmente las imágenes usan `:dev` como tag. Para producción deberías cambiar a un tag estable (`:prod`, `:latest`, o SHA específico). Ejemplo:
```yaml
image: ghcr.io/ingenieria-de-sistemas-123/snippet-service:prod
```
Repite para cada servicio. Esto desacopla el ciclo de desarrollo de las versiones desplegadas.

### 16.3 Emisión de Certificado en Prod
Mismo flujo que dev, cambiando el dominio:
```bash
docker compose -f docker-compose.prod.yml run --rm certbot certbot certonly --webroot -w /var/www/certbot -d snippet-org-prod.duckdns.org --email ing.sistemas.123.encrypt@gmail.com --agree-tos --no-eff-email
```
Recarga:
```bash
docker compose -f docker-compose.prod.yml exec reverse-proxy nginx -s reload
```

### 16.4 Validación Post-Despliegue Prod
Checklist:
1. DNS correcto: `ping snippet-org-prod.duckdns.org`.
2. Redirección HTTP→HTTPS: `curl -I http://snippet-org-prod.duckdns.org` devuelve 301.
3. Salud proxy: `curl -k https://snippet-org-prod.duckdns.org/actuator/health`.
4. Servicios detrás de paths responden.
5. Renovación aparece en `docker logs certbot` tras algunas horas.

### 16.5 Limpieza y Backup
- Respaldar volúmenes críticos antes de actualizaciones mayores:
```bash
docker stop $(docker ps -q)
# Ejemplo backup volumen Postgres
docker run --rm -v pg_data_prod:/data -v $(pwd):/backup alpine tar -czf /backup/pg_data_prod_$(date +%F).tar.gz -C /data .
```
- Restaurar (ejemplo):
```bash
docker run --rm -v pg_data_prod:/data -v $(pwd):/backup alpine sh -c "tar -xzf /backup/pg_data_prod_YYYY-MM-DD.tar.gz -C /data"
```

---
## 17. Manejo Seguro de Secretos y Variables (.env)

### 17.1 No versionar archivos .env con credenciales
Los archivos `.env.dev.dev` y `.env.prod` no deben subirse al repo si contienen credenciales reales. Usa GitHub Actions Secrets para inyectarlos en la VM en runtime.

### 17.2 GitHub Actions Secrets usados (ejemplo)
Configura en Settings > Secrets and variables > Actions:
- `DB_NAME`, `DB_USERNAME`, `DB_PASSWORD`
- `DB_PERMISSION_NAME`, `DB_PERMISSION_USER`, `DB_PERMISSION_PASSWORD`
- `VM_HOST`, `VM_USER`, `DEPLOY_SSH_KEY` (llave privada de despliegue)
- `GH_PAT` (token para pull de imágenes si necesario)

El workflow CD genera un archivo `.env.prod` o `.env.dev.dev` dinámico antes de `docker compose up`. No deja rastros en el repo.

### 17.3 Rotación de secretos
- Cambia credenciales DB cada X meses y vuelve a actualizar secrets.
- Revoca tokens antiguos (`GH_PAT`) y crea nuevos con scopes mínimos (read:packages).

### 17.4 Uso de digests para imágenes
Para mayor inmutabilidad:
```yaml
image: ghcr.io/ingenieria-de-sistemas-123/snippet-service@sha256:<digest>
```
Obtén el digest con:
```bash
docker pull ghcr.io/ingenieria-de-sistemas-123/snippet-service:prod
docker inspect --format='{{index .RepoDigests 0}}' ghcr.io/ingenieria-de-sistemas-123/snippet-service:prod
```
Reemplaza el tag por el digest al fijar una versión exacta.

### 17.5 Minimizar privilegios
- Usa un usuario PostgreSQL distinto por base (ya aplicado: snippet vs permission).
- Limita roles: no otorgar superuser a las cuentas de aplicación.

### 17.6 Limpieza automática
Agrega prune controlado (ya está en CD). Considera un job mensual que respalde volúmenes antes de prune extendido.

### 17.7 Archivos de ejemplo .env
Se agregaron plantillas:
- `.env.dev.dev.example`
- `.env.prod.example`
Copia y renombra la que corresponda, reemplazando valores `CHANGEME`. No subas los archivos reales con credenciales. Preferí usar Secrets en CI/CD.
