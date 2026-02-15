# ARCHITECTURE.md

## Objetivo
Definir un monorepo Nx con:
- `apps/web` (Angular)
- `apps/api` (NestJS)
- `docker-compose` con 2 servicios: `web` (Nginx) y `api` (Nest)

Alcance: arquitectura y scaffolding mínimo. No incluye features de negocio.

## Supuestos mínimos
- Node.js LTS 20.x
- npm 10.x
- Docker y Docker Compose v2
- SO de desarrollo: indiferente (Windows/Linux/macOS)
- Puerto público web: `8080`
- Puerto interno API: `3000`
- El frontend consume API vía `/api` proxied por Nginx en runtime de contenedor

## Árbol de carpetas propuesto
```text
.
+- apps/
¦  +- web/
¦  ¦  +- src/
¦  ¦  +- project.json
¦  ¦  +- ...
¦  +- api/
¦     +- src/
¦     +- project.json
¦     +- ...
+- libs/
¦  +- (vacío inicialmente; reservado para código compartido)
+- docker/
¦  +- web/
¦  ¦  +- Dockerfile
¦  ¦  +- nginx.conf
¦  +- api/
¦     +- Dockerfile
+- docker-compose.yml
+- nx.json
+- package.json
+- tsconfig.base.json
+- ARCHITECTURE.md
```

## Comandos Nx exactos (generación inicial)

### 1) Crear workspace Nx integrado
```bash
npx create-nx-workspace@latest agents-ia --workspaceType=integrated --preset=apps --nxCloud=skip --packageManager=npm --interactive=false
```

### 2) Entrar al repo
```bash
cd agents-ia
```

### 3) Generar app Angular `web`
```bash
npx nx add @nx/angular@latest
npx nx g @nx/angular:application web --directory=apps --standalone=true --routing=true --style=scss --e2eTestRunner=none --unitTestRunner=jest
```

### 4) Generar app NestJS `api`
```bash
npx nx add @nx/nest@latest
npx nx g @nx/nest:application api --directory=apps --frontendProject=web --e2eTestRunner=none --unitTestRunner=jest
```

### 5) Validación rápida de targets
```bash
npx nx show project web
npx nx show project api
```

Notas:
- Si la versión de Nx cambia defaults/flags, mantener nombres de proyecto `web` y `api`.
- En algunas versiones el flag `--directory=apps` puede variar. Objetivo estructural: `apps/web` y `apps/api`.

## Decisiones Docker

### 1) `docker/web/Dockerfile` (multi-stage)
Estrategia:
- Stage `builder`: instala dependencias y compila `web` con Nx.
- Stage `runtime`: imagen `nginx:alpine` sirviendo estáticos.

Contenido propuesto:
```dockerfile
# docker/web/Dockerfile
FROM node:20-alpine AS builder
WORKDIR /workspace

COPY package*.json ./
COPY nx.json tsconfig.base.json ./
COPY apps ./apps
COPY libs ./libs

RUN npm ci
RUN npx nx build web --configuration=production

FROM nginx:1.27-alpine AS runtime
COPY docker/web/nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=builder /workspace/dist/apps/web /usr/share/nginx/html

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### 2) `docker/web/nginx.conf`
Objetivos:
- SPA fallback (`try_files ... /index.html`)
- reverse proxy `/api` hacia servicio `api:3000`

Contenido propuesto:
```nginx
server {
  listen 80;
  server_name _;

  root /usr/share/nginx/html;
  index index.html;

  location / {
    try_files $uri $uri/ /index.html;
  }

  location /api/ {
    proxy_pass http://api:3000/;
    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
  }
}
```

### 3) `docker/api/Dockerfile`
Estrategia:
- Build Nest con Nx en stage builder
- Runtime Node liviano con output compilado

Contenido propuesto:
```dockerfile
# docker/api/Dockerfile
FROM node:20-alpine AS builder
WORKDIR /workspace

COPY package*.json ./
COPY nx.json tsconfig.base.json ./
COPY apps ./apps
COPY libs ./libs

RUN npm ci
RUN npx nx build api --configuration=production

FROM node:20-alpine AS runtime
WORKDIR /app
ENV NODE_ENV=production

COPY package*.json ./
RUN npm ci --omit=dev

COPY --from=builder /workspace/dist/apps/api ./dist

EXPOSE 3000
CMD ["node", "dist/main.js"]
```

### 4) `docker-compose.yml`
Solo 2 servicios (`web`, `api`) como requerido.

Contenido propuesto:
```yaml
services:
  api:
    build:
      context: .
      dockerfile: docker/api/Dockerfile
    container_name: agents_api
    expose:
      - "3000"
    environment:
      - NODE_ENV=production

  web:
    build:
      context: .
      dockerfile: docker/web/Dockerfile
    container_name: agents_web
    depends_on:
      - api
    ports:
      - "8080:80"
```

## Flujo de ejecución esperado
- Local dev sin Docker:
  - `npx nx serve web`
  - `npx nx serve api`
- Contenedores:
  - `docker compose up --build`
  - Abrir `http://localhost:8080`
  - Requests a `/api/*` salen por Nginx hacia `api:3000`

## Checklist DoD

### Build
- [ ] `npx nx build web --configuration=production` exitoso
- [ ] `npx nx build api --configuration=production` exitoso
- [ ] `docker compose build` exitoso

### Lint
- [ ] `npx nx lint web` exitoso
- [ ] `npx nx lint api` exitoso

### Test
- [ ] `npx nx test web` exitoso
- [ ] `npx nx test api` exitoso

### Run
- [ ] `npx nx serve web` levanta en dev
- [ ] `npx nx serve api` levanta en dev
- [ ] `docker compose up` levanta ambos servicios
- [ ] `GET http://localhost:8080` responde app Angular
- [ ] `GET http://localhost:8080/api/health` (o endpoint equivalente) responde desde Nest

## Riesgos y TODOs explícitos
- TODO: Confirmar versión exacta de Nx y ajustar flags de generators si cambian entre releases.
- TODO: Definir estrategia de variables de entorno (`.env`, `ConfigModule`, secrets).
- TODO: Agregar endpoint health-check estable en `api` (`/api/health`).
- TODO: Incorporar `.dockerignore` para reducir contexto de build.
- TODO: Definir CI (pipeline con `nx affected`, caching y policy de calidad).
- TODO: Decidir política de CORS para entorno local sin proxy (si aplica).
- TODO: Definir librerías compartidas iniciales en `libs/` (DTOs, contratos).

## Decisiones fuera de alcance
- Autenticación/autorización
- Base de datos y migraciones
- Observabilidad (logs estructurados, métricas, trazas)
- Deploy productivo (K8s, ECS, etc.)