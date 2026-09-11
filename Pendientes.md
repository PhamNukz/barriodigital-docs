# Pendientes — BarrioDigital EP1 (DSY1107)

Backlog único del proyecto. Cada ítem tiene prioridad y repo afectado. El paso a paso de cada área vive en su propio doc:

| Área | Doc con el detalle |
|---|---|
| Entra ID + tokens | [Entra-ID-y-JWT.md](Entra-ID-y-JWT.md) |
| AWS (VPC, SG, EC2, Oracle, API Gateway) | [AWS-Infraestructura.md](AWS-Infraestructura.md) |
| GitHub Projects | [GitHub-Projects.md](GitHub-Projects.md) |
| CI/CD + secrets | [CI-CD.md](CI-CD.md) |

**Prioridades:** `P0` = sin esto no hay nota (pauta EP1: MSAL 60% + BFF valida JWT 40%) · `P1` = pedido por el caso, no por la pauta EP1 · `P2` = deseable / fase 2.

**Orden sugerido:** A → B → C → D → E → F → G. Entra primero porque sin tokens reales no se puede probar nada más.

## Reparto

| Quién | Bloques | Por qué |
|---|---|---|
| **Francisco** | C (red), D (EC2, Oracle), E (API Gateway) | Todo lo de AWS; depende de la sesión Learner Lab |
| **Benjamín** | A (Entra), B (tokens/front), F (GitHub, Projects), G1–G4 (CI + imágenes GHCR), H | No necesita AWS; se puede avanzar en paralelo desde el día 1 |
| **Los dos** | A7, D9, E5, G5–G8 | Necesitan datos de ambos lados: EIP + IPs privadas (Francisco) y IDs de Entra (Benjamín) |

**Puntos de sincronización:**
1. Benjamín termina A1–A6 → le pasa a Francisco `TENANT_ID` y `API_CLIENT_ID` para el authorizer (E2).
2. Francisco termina D3 (EIP) y D4/D7/D8 (IPs privadas) → se juntan para D9, `.env` (A7), G5–G8 y el primer deploy.
3. Francisco termina E5 (Invoke URL) → Benjamín la pone en `environment.prod.ts` y la redirect URI de prod en la App Registration de la SPA (A4).

---

## Para terminar (estado al 2026-09-11)

**Hecho:** toda la infra AWS (C, D, E1–E4) está creada y corriendo; Entra (A1–A5, A8) y las imágenes GHCR (G1–G3) existen; `environment.prod.ts` ya apunta al Gateway. **Lo que falta es conectar las piezas.** Orden:

| # | Quién | Qué | Bloquea a |
|---|---|---|---|
| 1 | **Benjamín** | **G4** — poner **Public** los 7 packages en https://github.com/PhamNukz?tab=packages (Package settings → Danger Zone → Change visibility). Solo el dueño puede. | El deploy: `ec2-apps` no puede hacer `docker pull` de packages privados |
| ~~2~~ ✅ | **Francisco** | **Gateway sirve el frontend** — rutas `ANY /` y `ANY /{proxy+}` → `http://100.60.226.205/{proxy}` sin authorizer (paso 7 de AWS-Infraestructura.md). Entra exige `https` en redirect URIs y `http://<EIP>` no sirve. | El login |
| 2b | **Benjamín** | **A4** — en Entra, App Registration `barriodigital-spa` → Authentication → plataforma SPA → agregar `https://dnddhzpvgg.execute-api.us-east-1.amazonaws.com` como redirect URI y como front-channel logout URL | El login desde la URL pública |
| 3 | **Benjamín** | **A6** — confirmar que los 4 usuarios de prueba tienen rol asignado en Enterprise Applications → `barriodigital-api` → Users and groups | Sin `roles` en el token el BFF responde 403 |
| 4 | **Benjamín** | **F4/F5** — dar acceso a Francisco al GitHub Project (Project → ⚙ → Manage access; ser collaborator no basta) y confirmar que los issues de los ítems ya hechos se cierran | Solo evidencia |
| 5 | **Francisco** | Verificar en Actions del frontend que `build-image` corrió verde tras el push de `environment.prod.ts` | Que el front apunte al Gateway |
| 6 | **Francisco** | Re-aplicar los compose nuevos (tienen `restart` y heaps): `scp kafka/compose.yml kafka:~/kafka/` + `ssh kafka` → `cd kafka && docker compose up -d`; igual para `mq`. Luego verificar clusters: `ssh mq` → `docker compose exec rabbitmq1 rabbitmqctl cluster_status` (2 running nodes); `ssh kafka` → `docker compose exec kafka1 kafka-topics.sh --bootstrap-server 10.0.1.29:9092 --create --topic smoke --partitions 3 --replication-factor 3` (si falla, los brokers no se ven entre sí por la IP del host) | Mensajería |
| 6b | **Francisco** | **Deploy** en `ec2-apps` (EC2 prendidas): `scp -r apps apps:~/` → `ssh apps` → `cd apps && cp .env.example .env` → completar `AAD_ISSUER_URI` (`https://login.microsoftonline.com/db9e57fc-5bb8-44fc-8d2f-caf0060c79da/v2.0`), `AAD_API_CLIENT_ID` (`cdf23af8-9ba5-483b-a5ba-03e23c43b101`), las 4 `*_DB_PASSWORD` (lo demás ya viene en el example) → `docker compose up -d` → `docker compose ps` (7 `Up`) | Todo lo demás |
| 7 | **Francisco** | **E6/B1** — evidencia con `curl`: sin token → 401 (Gateway), con token → 200, sin rol → 403 (BFF). Token: login en `https://dnddhzpvgg.execute-api.us-east-1.amazonaws.com`, F12 → Network → header `Authorization` de cualquier llamada al Gateway | Pauta indicador 2 (40%) |
| 8 | **Benjamín** | **A9/B2** — login real en `https://dnddhzpvgg.execute-api.us-east-1.amazonaws.com`, decodificar el token en jwt.ms, confirmar `iss`/`aud`/`scp`/`roles` | Pauta indicador 1 (60%) |
| 9 | **Los dos** | Capturas para la presentación: consola AWS (VPC, EC2, SG, Gateway con authorizer), los 3 `curl`, jwt.ms, tablero del Project | Entrega |
| 10 | **Francisco** | **D11** — apagar las 4 EC2 al terminar cada sesión | Presupuesto |

**Opcional si sobra tiempo (P1/P2):** G6–G8 (deploy automático por SSH desde Actions), B3 (guard por rol en el front), F7 (branch protection), G9 (badges), F8 (READMEs). Ninguno afecta la nota de EP1.

**Ya no aplica:** A7 quedó cubierto por `environment.ts` (Benjamín) + `.env.example` (Francisco); G5 hecho (compose usa `image: ghcr.io/...`; `compose.local.yml` para desarrollo).

---

## A. Microsoft Entra ID (P0)

- [ ] **A1** `P0` `Benjamín` Crear App Registration **`barriodigital-api`** (Expose an API → Application ID URI `api://<API_CLIENT_ID>`, scope `access_as_user`). — Entra
- [ ] **A2** `P0` `Benjamín` En `barriodigital-api` → App roles: crear `Admin`, `Funcionario`, `Vecino`, `Auditor` (allowed member type: Users/Groups, value = mismo nombre). — Entra
- [ ] **A3** `P0` `Benjamín` En `barriodigital-api` → Manifest: `"requestedAccessTokenVersion": 2`. Sin esto el `iss` del access token es v1 (`sts.windows.net`) y el authorizer del API Gateway lo rechaza. — Entra
- [ ] **A4** `P0` `Benjamín` Crear App Registration **`barriodigital-spa`** (plataforma SPA, redirect `http://localhost:4200` y luego la URL pública del front; logout URL igual). — Entra
- [ ] **A5** `P0` `Benjamín` En `barriodigital-spa` → API permissions → agregar `barriodigital-api / access_as_user` → **Grant admin consent**. — Entra
- [ ] **A6** `P0` `Benjamín` Enterprise Applications → `barriodigital-api` → Users and groups → asignar al menos 1 usuario por rol (4 usuarios de prueba). **Sin esto el claim `roles` no aparece en el token** y el backend responde 403 a todo. — Entra
- [x] **A7** `P0` `Francisco + Benjamín` Copiar `TENANT_ID`, `SPA_CLIENT_ID`, `API_CLIENT_ID` a `frontend-barriodigital/src/environments/environment.ts` y a `barriodigital-infra/apps/.env`. — frontend, infra
- [x] **A8** `P0` `Benjamín` **Bug `aud`**: en tokens v2.0 el claim `aud` es el **GUID** de la API (`<API_CLIENT_ID>`), no `api://<API_CLIENT_ID>`. Hoy `application.yml` de bff/requests/catalog/audit/report solo acepta `api://…`. Fix: listar ambos en `barriodigital.security.audiences` (`${AAD_API_CLIENT_ID}` y `api://${AAD_API_CLIENT_ID}`) y dejar `AAD_API_CLIENT_ID` como GUID pelado en `.env.example`. — 5 MS + infra
- [ ] **A9** `P0` `Benjamín` Probar login real: decodificar el access token en https://jwt.ms y verificar `iss` (…/v2.0), `aud`, `scp = access_as_user`, `roles`, `exp`. — frontend

## B. Tokens / JWT — cobertura (P0)

Los 3 tokens que existen en este flujo y quién los usa (detalle en [Entra-ID-y-JWT.md](Entra-ID-y-JWT.md)):

| Token | Para qué | Quién lo maneja | Estado |
|---|---|---|---|
| `id_token` | Identidad del usuario + `roles` para mostrar/ocultar en la UI | `roles.ts` (`idTokenClaims`) | ✅ implementado |
| `access_token` | Autorización hacia la API (`aud`, `scp`, `roles`, `exp`) — viaja como `Authorization: Bearer …` | `MsalInterceptor` → API Gateway → BFF → MS | ✅ implementado, falta probar con token real |
| `refresh_token` | Renovar el access token sin volver a loguear (24 h en SPA) | MSAL internamente (`acquireTokenSilent`) | ✅ automático, falta manejar fallo |

> "Bearer" **no es un token**: es el esquema del header HTTP. No falta ningún tipo de token adicional: no aplica `client_credentials` (no hay servicio-a-servicio sin usuario) ni `on-behalf-of` (el BFF reenvía el mismo bearer del usuario, ver `DomainClients.java`).

- [ ] **B1** `P0` `Benjamín` Verificar que el backend valida las 6 cosas de la pauta: firma (JWKS del issuer), `iss`, `aud`, `exp`, `scp`, rol por endpoint. Ya está en `ResourceServerSecurityConfig` + `AudienceValidator` + `SecurityConfig`; solo hay que **evidenciarlo** con `curl` (200 / 401 / 403) para la presentación. — bff
- [ ] **B2** `P0` `Benjamín` Frontend: manejar `InteractionRequiredAuthError` cuando falla la renovación silenciosa (refresh vencido) → volver a `loginPopup`. Hoy `MsalInterceptor` lo hace por defecto con `InteractionType.Popup`; confirmar en consola que ocurre. — frontend
- [ ] **B3** `P1` `Benjamín` Guard por rol en el front: `/catalog` solo `Admin`/`Funcionario` (hoy solo `MsalGuard`, cualquier autenticado entra). Crear `roleGuard` funcional que lea `rolesDe(msal)`. — frontend
- [ ] **B4** `P2` `Benjamín` Mostrar expiración del token (`exp`) y botón "renovar" en la topbar — útil para la demo. — frontend

## C. AWS — Red y zonas de seguridad (P0)

- [x] **C1** `P0` `Francisco` Crear VPC `barriodigital-vpc` `10.0.0.0/16` en `us-east-1`. — AWS
- [x] **C2** `P0` `Francisco` Subred **pública** `10.0.0.0/24` (`us-east-1a`, auto-assign public IP ON) y subred **privada** `10.0.1.0/24` (`us-east-1a`). — AWS
- [x] **C3** `P0` `Francisco` Internet Gateway adjunto a la VPC; route table `rt-public` con `0.0.0.0/0 → igw`, asociada a la pública. La privada queda con la route table por defecto (solo `10.0.0.0/16 local`), **sin NAT Gateway** (costo + Learner Lab). — AWS
- [x] **C4** `P0` `Francisco` Security Groups (tabla completa en [AWS-Infraestructura.md](AWS-Infraestructura.md#security-groups)):
  - `sg-apps` → in: 22 (mi IP), 8080 (0.0.0.0/0, API Gateway), 80 (0.0.0.0/0, frontend). out: all.
  - `sg-mq` → in: 5672 y 15672 desde `sg-apps`; 4369 y 25672 desde `sg-mq` (cluster). out: all.
  - `sg-kafka` → in: 9092-9094 y 8085 desde `sg-apps`; 2181, 2888, 3888 desde `sg-kafka`. out: all.
  - `sg-db` → in: 1521 desde `sg-apps`; 22 desde `sg-apps` (bastion). out: all.
- [x] **C5** `P0` `Francisco` Ubicación (ya decidida, dejar documentada): **pública** = `ec2-apps` (frontend, bff, requests, catalog, notify, audit, report). **privada** = `ec2-mq`, `ec2-kafka`, `ec2-db`. Solo `ec2-apps` tiene IP pública. — docs
- [x] **C6** `P1` `Francisco` Documentar en `arquitectura.md` por qué elegimos BFF expuesto vs VPC Link (ver sección en AWS-Infraestructura.md). — docs

## D. AWS — EC2 (P0)

- [x] **D1** `P0` `Francisco` Key pair `barriodigital-key` (descargar `.pem`, compartir con la pareja por canal privado, nunca a git). — AWS
- [x] **D2** `P0` `Francisco` `ec2-apps` — Ubuntu 22.04, `t3.small`, subred pública, `sg-apps`, user-data Docker (script en AWS-Infraestructura.md). — AWS
- [x] **D3** `P0` `Francisco` **Elastic IP** asociada a `ec2-apps` (Learner Lab cambia la IP pública en cada stop/start; el API Gateway y `environment.prod.ts` apuntan a esta EIP). — AWS
- [x] **D4** `P0` `Francisco` `ec2-db` — Ubuntu 22.04, `t3.medium` (Oracle Free necesita ≥ 2 GB RAM), disco 30 GB, subred privada, `sg-db`. Como no tiene internet: **crear primero en la subred pública** para instalar Docker + `docker pull gvenzl/oracle-free:23-slim`, luego crear AMI y lanzarla en la privada (una EC2 no se puede mover de subred: hay que relanzar desde la AMI). — AWS
- [x] **D5** `P0` `Francisco` En `ec2-db`: levantar Oracle con volumen + script `init/01-schemas.sql` que crea los 4 usuarios (`barriodigital_requests|catalog|audit|report`) con `CREATE SESSION, CREATE TABLE, CREATE SEQUENCE, UNLIMITED TABLESPACE`. — infra
- [x] **D6** `P0` `Francisco` Corregir `DB_SERVICE=FREEPDB1` (Oracle Free 23c) en `barriodigital-infra/apps/.env.example` (hoy dice `XEPDB1`, que es de XE 21c). — infra
- [x] **D7** `P0` `Francisco` `ec2-mq` — Ubuntu 22.04, `t3.small`, privada, `sg-mq`. Misma técnica AMI (Docker + `docker pull rabbitmq:3.13-management` en pública, luego relanzar en privada). — AWS
- [x] **D8** `P0` `Francisco` `ec2-kafka` — Ubuntu 22.04, `t3.medium` (3 ZK + 3 brokers + UI ≈ 3 GB RAM), privada, `sg-kafka`. Misma técnica AMI. — AWS
- [x] **D9** `P0` `Francisco + Benjamín` **Compose multi-host**: `apps/compose.yml` asume que Rabbit y Kafka están en la misma red Docker (`rabbitmq1`, `kafka1:9092`). En 3 EC2 distintas eso no resuelve. Cambiar a variables: `RABBITMQ_HOST=${MQ_PRIVATE_IP}`, `KAFKA_BOOTSTRAP_SERVERS=${KAFKA_PRIVATE_IP}:9092,${KAFKA_PRIVATE_IP}:9093,${KAFKA_PRIVATE_IP}:9094` y en `kafka/compose.yml` `KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://${KAFKA_PRIVATE_IP}:909X` (puerto host mapeado). Quitar `external: true` de la red en mq/kafka (cada EC2 tiene su propia red). — infra
- [x] **D10** `P1` `Francisco` Acceso a las privadas por bastion: `ssh -J ubuntu@<EIP-apps> ubuntu@10.0.1.X`. Túnel para UIs: `ssh -L 15672:10.0.1.X:15672 ubuntu@<EIP-apps>`. — docs
- [ ] **D11** `P2` `Francisco` Recordatorio: apagar las 4 EC2 al terminar cada sesión (Learner Lab tiene presupuesto en US$ limitado). — infra

## E. AWS — API Gateway (P0)

- [x] **E1** `P0` `Francisco` Crear **HTTP API** `barriodigital-api` (no REST API: el JWT authorizer nativo es de HTTP API). — AWS
- [x] **E2** `P0` `Francisco` Authorizer tipo JWT: issuer `https://login.microsoftonline.com/<TENANT_ID>/v2.0`, audience `<API_CLIENT_ID>` (GUID, ver A8). Identity source `$request.header.Authorization`. — AWS
- [x] **E3** `P0` `Francisco` Ruta `ANY /api/{proxy+}` → integración HTTP URI `http://<EIP-apps>:8080/api/{proxy}` con el authorizer adjunto. — AWS
- [x] **E4** `P0` `Francisco` CORS en el API: origins = URL del front (y `http://localhost:4200` para probar), headers `Authorization, Content-Type`, methods `GET,POST,PUT,DELETE,OPTIONS`. — AWS
- [ ] **E5** `P0` `Francisco + Benjamín` Stage `$default` con auto-deploy. Copiar la Invoke URL a `environment.prod.ts → apiBaseUrl` y el origen del front a `CORS_ALLOWED_ORIGINS` del BFF. — frontend, infra
- [ ] **E6** `P0` `Francisco` Probar: `curl` sin token → 401 del Gateway; con token válido → 200; con token de un usuario sin rol → 403 del BFF (evidencia para la pauta, indicador 2). — docs

## F. GitHub (P1)

- [x] **F1** `P0` `Benjamín` Crear `Dockerfile` en `ms-barriodigital-bff` (8080), `ms-barriodigital-requests` (8081), `ms-barriodigital-catalog` (8082): copiar el de `ms-barriodigital-audit` cambiando `EXPOSE`. — 3 MS
- [x] **F2** `P0` `Benjamín` Crear `Dockerfile` en `frontend-barriodigital`: etapa `node:20` → `npm ci && npm run build -- --configuration production`, etapa `nginx:alpine` sirviendo `dist/frontend-barriodigital/browser` con `try_files $uri /index.html` (SPA). — frontend
- [x] **F3** `P1` `Benjamín` `.gitignore`: agregar `.idea/` en `barriodigital-infra` (aparece sin trackear); confirmar `target/`, `node_modules/`, `.env`, `*.pem` en todos. — 9 repos
- [ ] **F4** `P1` `Benjamín` Agregar a la pareja como **collaborator** (Settings → Collaborators) en los 9 repos. — GitHub
- [ ] **F5** `P1` `Benjamín` Crear **GitHub Project** `BarrioDigital EP1` (Board) a nivel de usuario, linkear los 9 repos, campos `Área` y `Prioridad`, workflow auto-add + auto-close. Guía: [GitHub-Projects.md](GitHub-Projects.md). — GitHub
- [ ] **F6** `P1` `Benjamín` Crear un **issue por ítem** de este archivo (labels `area:aws`, `area:entra`, `area:github`, `area:backend`, `area:frontend`, `area:docs`, `P0/P1/P2`) y agregarlos al proyecto. Milestone `Entrega EP1`. — GitHub
- [ ] **F7** `P2` `Benjamín` Branch protection en `main` (require PR + status check `ci`) en los 7 repos de código. — GitHub
- [ ] **F8** `P1` `Benjamín` README de cada repo con: qué es, cómo correr local, variables de entorno. — 9 repos

## G. CI/CD (P1)

Guía completa con los YAML listos: [CI-CD.md](CI-CD.md).

- [ ] **G1** `P1` `Benjamín` `.github/workflows/ci.yml` en los 6 MS Java: `mvn -B verify` con JDK 17 + cache Maven, on push/PR. Los tests ya corren con H2 y JWT mockeado (`application-test.yml`), no necesitan Oracle ni Azure. — 6 MS
- [ ] **G2** `P1` `Benjamín` `.github/workflows/ci.yml` en frontend: `npm ci && npm run build`. — frontend
- [ ] **G3** `P1` `Benjamín` `.github/workflows/build-image.yml` en los 7 repos de código: on push `main` → `docker build` + push a `ghcr.io/phamnukz/<repo>:latest` y `:<sha>`. Usa `GITHUB_TOKEN` (permiso `packages: write`), **sin secrets extra**. — 7 repos
- [ ] **G4** `P1` `Benjamín` Hacer públicos los packages en GHCR (Package settings → Change visibility) para que `ec2-apps` haga `docker pull` sin login. Alternativa: PAT `read:packages` y `docker login ghcr.io` una vez en la EC2. — GitHub
- [x] **G5** `P1` `Francisco + Benjamín` `apps/compose.yml`: reemplazar `build: ../../<repo>` por `image: ghcr.io/phamnukz/<repo>:latest`. Dejar `compose.override.yml` con los `build:` para desarrollo local. — infra
- [ ] **G6** `P1` `Francisco + Benjamín` `.github/workflows/deploy.yml` en `barriodigital-infra`: on push `main` + `workflow_dispatch` → SSH a `ec2-apps` (`appleboy/ssh-action`) → escribe `.env` desde secret → `git pull` → `docker compose pull && docker compose up -d`. — infra
- [ ] **G7** `P1` `Francisco + Benjamín` Secrets en `barriodigital-infra` (Settings → Secrets → Actions): `EC2_HOST` (EIP), `EC2_USER` (`ubuntu`), `EC2_SSH_KEY` (clave privada **dedicada al deploy**, no la `.pem` del key pair), `APPS_ENV` (contenido completo del `.env`). — GitHub
- [ ] **G8** `P1` `Francisco + Benjamín` Generar la clave dedicada: `ssh-keygen -t ed25519 -f deploy_key -C barriodigital-deploy`, agregar la pública a `~/.ssh/authorized_keys` de `ec2-apps`, la privada al secret. — AWS, GitHub
- [ ] **G9** `P2` `Benjamín` Badge de CI en cada README. — 7 repos
- [x] **G10** `P2` `Benjamín` `mq` y `kafka` se levantan a mano una vez por SSH (no cambian con cada push); documentar el comando en el README de infra. — infra

## H. Código pendiente detectado (no bloquea EP1)

- [x] **H1** `P2` `Benjamín` `TramiteService` ya publica a RabbitMQ (`NotificationPublisher`) y Kafka (`RequestsEventPublisher`) — hecho en commit `85d7879` de requests. Pendiente menor: `apps/compose.yml` no pasa `RABBITMQ_USER/PASSWORD` al servicio `requests` (usa guest/guest por defecto, coincide con mq). — requests
- [ ] **H2** `P2` `Benjamín` Rutas del caso que no existen en el front: `/login`, `/dashboard`, `/reports`, `/audit` (solo hay `/`, `/requests`, `/catalog`). La pauta EP1 solo evalúa MSAL + BFF; dejar para EP2. — frontend
- [ ] **H3** `P2` `Benjamín` `AzureAdProperties`, `AudienceValidator`, `JwtRolesConverter`, `ResourceServerSecurityConfig` están copiadas idénticas en 5 repos (ver comentario `ponytail:` en `AzureAdProperties.java`). Si se tocan (A8), tocar las 5. — 5 MS
- [x] **H4** `P2` `Benjamín` `arquitectura.md` linkeaba `../README.md` que no existe; reemplazado por índice de docs + sección Despliegue AWS. — docs

---

## Mapa pauta EP1 → ítems

| Indicador de la pauta | % | Ítems que lo cubren |
|---|---|---|
| 1. MSAL + Angular: login/logout, guards, interceptor, tokens para el API Gateway, roles/scopes desde claims | 60% | A1–A9, B1–B3, E1–E5 |
| 2. BFF valida issuer, audience, firma, vigencia; autorización por rol; códigos de error | 40% | A8, B1, E6 (+ ya implementado en `SecurityConfig` del BFF) |
| Instrucciones generales: compila, `.gitignore`, DB cloud configurada, entrega vía GitHub | — | D4–D6, F1–F3, G1–G2 |
