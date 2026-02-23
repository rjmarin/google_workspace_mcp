# Deploy Extend — Google Workspace MCP Staging & Producción

Guía específica para el deploy del MCP en el entorno Extend. El `README.md` principal es el del proyecto upstream (taylorwilsdon/google_workspace_mcp).

---

## Archivos relevantes

| Archivo | Propósito |
|---|---|
| `docker-compose.staging.yml` | Compose para staging |
| `docker-compose.yml` | Compose original del proyecto (referencia) |
| `client_secret.json` | OAuth credentials de Google Cloud (**NO en git**) |
| `.env.staging` | Variables de entorno para staging (**NO en git**) |
| `.env.production` | Variables de entorno para producción (**NO en git**) |

---

## Paso 1 — Obtener `client_secret.json`

1. Ir a [console.cloud.google.com](https://console.cloud.google.com)
2. Seleccionar proyecto `canvas-landing-481511-e7`
3. APIs & Services → Credentials
4. Buscar el OAuth 2.0 Client `621426227634-8f6n40v7fk0ohrian49vbph6q7g5n4qs`
5. Descargar JSON → renombrar a `client_secret.json`
6. Colocar en `~/google_workspace_mcp/client_secret.json`

---

## Paso 2 — Crear `.env.staging` (NO está en git)

```bash
cat > ~/google_workspace_mcp/.env.staging << 'EOF'
# Google OAuth Credentials
GOOGLE_OAUTH_CLIENT_ID="621426227634-8f6n40v7fk0ohrian49vbph6q7g5n4qs.apps.googleusercontent.com"
GOOGLE_OAUTH_CLIENT_SECRET="<client_secret del JSON>"

# URLs de staging (HTTPS requerido por Google OAuth)
WORKSPACE_EXTERNAL_URL="https://mcp-preludio.extend.cl"
GOOGLE_OAUTH_REDIRECT_URI="https://mcp-preludio.extend.cl/oauth2callback"

# Storage backend — disk para persistir tokens entre reinicios
WORKSPACE_MCP_OAUTH_PROXY_STORAGE_BACKEND=disk
WORKSPACE_MCP_OAUTH_PROXY_DISK_DIRECTORY=/app/store_creds

# Debug (se puede desactivar en producción)
OAUTH2_ENABLE_DEBUG=true
OAUTH2_ENABLE_LEGACY_AUTH=true
EOF
```

---

## Paso 3 — Crear `.env.production` (para producción)

Igual que staging pero con las URLs de producción:

```bash
cat > ~/google_workspace_mcp/.env.production << 'EOF'
GOOGLE_OAUTH_CLIENT_ID="621426227634-8f6n40v7fk0ohrian49vbph6q7g5n4qs.apps.googleusercontent.com"
GOOGLE_OAUTH_CLIENT_SECRET="<client_secret del JSON>"

# URLs de producción
WORKSPACE_EXTERNAL_URL="https://mcp.preludio.extend.cl"
GOOGLE_OAUTH_REDIRECT_URI="https://mcp.preludio.extend.cl/oauth2callback"

WORKSPACE_MCP_OAUTH_PROXY_STORAGE_BACKEND=disk
WORKSPACE_MCP_OAUTH_PROXY_DISK_DIRECTORY=/app/store_creds
OAUTH2_ENABLE_LEGACY_AUTH=true
EOF
```

---

## Paso 4 — Configurar Google Cloud Console

Las redirect URIs autorizadas deben incluir **tanto staging como producción**:

En [console.cloud.google.com](https://console.cloud.google.com) → APIs & Services → Credentials → OAuth client `621426227634-...`:

**Authorized redirect URIs:**
```
https://mcp-preludio.extend.cl/oauth2callback
https://mcp-preludio.extend.cl/oauth/callback
https://mcp.preludio.extend.cl/oauth2callback
https://mcp.preludio.extend.cl/oauth/callback
```

**Authorized JavaScript origins:**
```
https://mcp-preludio.extend.cl
https://mcp.preludio.extend.cl
https://app-preludio.extend.cl
https://preludio.extend.cl
```

> **Importante**: Google OAuth requiere HTTPS obligatoriamente. No funciona con HTTP.

---

## Comandos de deploy

```bash
# Primera vez o tras cambios en el código
cd ~/google_workspace_mcp
docker compose -f docker-compose.staging.yml up -d --build

# Reiniciar sin rebuild
docker compose -f docker-compose.staging.yml up -d

# Ver logs
docker logs -f gws_mcp

# Verificar health
curl https://mcp-preludio.extend.cl/health
```

---

## Actualizar (pull de cambios)

```bash
cd ~/google_workspace_mcp
git pull https://<TOKEN>@github.com/rjmarin/google_workspace_mcp.git main
docker compose -f docker-compose.staging.yml up -d --build
```

---

## Cómo se integra con LibreChat

LibreChat se conecta al MCP via `streamable-http`. La URL está configurada en `~/extend-librechat/librechat.yaml`:

```yaml
mcpServers:
  g-suite:
    type: streamable-http
    url: http://gws_mcp:8000/mcp    # nombre del contenedor en Docker
    startup: false
    requiredOAuth: false
```

El flujo OAuth es:
1. Usuario en LibreChat activa el tool de G-Suite
2. LibreChat llama a `https://mcp-preludio.extend.cl/mcp`
3. MCP responde que requiere OAuth → redirige a Google
4. Google redirige al callback: `https://mcp-preludio.extend.cl/oauth2callback`
5. MCP guarda el token en `/app/store_creds` (volumen persistente)
6. LibreChat puede ejecutar tools de Google Workspace

---

## Solución de problemas

### `fetch failed` / `ECONNREFUSED` en LibreChat

LibreChat no puede conectar al MCP. Verificar:
1. Que `gws_mcp` está corriendo: `docker ps | grep gws_mcp`
2. Que `librechat.yaml` usa `http://gws_mcp:8000/mcp` (no `host.docker.internal`)
3. Que ambos contenedores están en `extend-network`

### Google devuelve `redirect_uri_mismatch`

La URI en `.env.staging` no está autorizada en Google Cloud Console. Agregar la URI exacta en la lista de Authorized redirect URIs.

### Tokens no persisten entre reinicios

El volumen `store_creds` debe estar declarado en `docker-compose.staging.yml`. Si se hizo `docker compose down -v`, los tokens se borraron y hay que volver a autenticar.

---

## Para producción

1. Crear `.env.production` con URLs `mcp.preludio.extend.cl`
2. Actualizar `docker-compose.staging.yml` para usar `env_file: .env.production` (o crear `docker-compose.production.yml`)
3. Asegurarse de que Google Cloud Console tiene las redirect URIs de producción autorizadas
4. El nginx de producción (`conf.d/mcp.preludio.extend.cl.conf`) debe apuntar a `gws_mcp:8000`
