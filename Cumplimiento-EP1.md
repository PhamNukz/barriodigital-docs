# Cumplimiento EP1 — Caso BarrioDigital

Punto por punto de lo que piden la [pauta EP1](Requisitos%20de%20la%20Prueba/EP1_DSY1107_Estudiante_encargo.md)
y el [enunciado del caso](Requisitos%20de%20la%20Prueba/Caso_BarrioDigital.md), con la evidencia
concreta de dónde está resuelto. Las brechas se listan igual que lo cumplido.

**Verificado el 12-09-2026** contra el despliegue real en AWS · 42 tests · CI verde en los 9 repos.
Versión navegable: https://claude.ai/code/artifact/7da8a589-7b95-4b6b-a6ca-c830e7ed9e4b

---

## Indicador 1 — MSAL con Angular (60%)

| Criterio de la pauta | Cómo se resolvió | Estado |
|---|---|---|
| MSAL integrado y operativo | `@azure/msal-angular` v3 en `app.config.ts`: `PublicClientApplication` con authority del tenant, `redirectUri` del Gateway, caché en `localStorage` y `APP_INITIALIZER` llamando a `initialize()` (obligatorio en v3) | ✅ |
| Inicio y cierre de sesión funcionan | `loginPopup()` / `logoutPopup()` en `SessionService`. Probado end-to-end contra Azure AD con cuentas de los 4 roles. Incluye selector multicuenta con `prompt: 'select_account'` | ✅ |
| Los guards operan sin fallas | `MsalGuard` en `/requests` y `/catalog`, más `roleGuard(['Admin','Funcionario'])` en `/catalog`: no basta con estar autenticado, se exige el rol | ✅ |
| El `MsalInterceptor` opera sin fallas | `protectedResourceMap` asocia `<gateway>/api` con el scope `access_as_user`; adjunta `Bearer` en cada llamada y reabre el popup si falla la renovación silenciosa | ✅ |
| Se obtienen los tokens para el API Gateway | `acquireTokenSilent({ scopes: [apiScope] })`. Verificado en vivo: **200** con token válido, **401** sin token y con token mal formado. La topbar muestra la expiración y permite renovar | ✅ |
| Se leen **roles y scopes** desde los claims | `tokenInfoDe()` decodifica el payload del **access token** y extrae `roles`, `scp` y `exp` de una sola pasada; los scopes se muestran en la tarjeta de bienvenida | ✅ |

**El detalle que casi cuesta el indicador:** los App Roles se asignan contra el service principal de
la *API*, no el de la SPA, así que viajan en el **access token** y no en el `id_token`. Leyendo
`idTokenClaims` el rol siempre llegaba vacío pese a estar bien asignado en Azure. Documentado en
[Entra-ID-y-JWT.md](Entra-ID-y-JWT.md), junto con el otro tropiezo del mismo origen: en tokens v2 el
claim `aud` es el GUID pelado, no `api://<GUID>`.

---

## Indicador 2 — El BFF valida el token (40%)

Implementado en `ResourceServerSecurityConfig` + `AudienceValidator`, presentes en el BFF y en los
4 microservicios que reciben tráfico.

| Criterio de la pauta | Cómo se resolvió | Estado |
|---|---|---|
| Valida el **issuer** | `JwtDecoders.fromIssuerLocation()` + `JwtValidators.createDefaultWithIssuer()` con `issuer-uri = https://login.microsoftonline.com/<TENANT>/v2.0` | ✅ |
| Valida el **audience** | `AudienceValidator` propio encadenado al decoder; acepta el GUID de la API y `api://<GUID>` para tolerar tokens v1 y v2 | ✅ |
| Verifica la **firma** | `NimbusJwtDecoder` descarga el JWKS del issuer y valida RS256 contra la clave pública de Azure AD | ✅ |
| Verifica la **vigencia** | Validador por defecto de Spring Security (`exp`, `nbf`); la expiración es visible en la topbar para poder demostrarlo | ✅ |
| Autorización **por rol** | `JwtRolesConverter` mapea el claim `roles` a `ROLE_*` y `@PreAuthorize` los exige por endpoint | ✅ |
| **Códigos de error** adecuados | RFC 7807 (`ProblemDetail`) con `detail` legible en todos los servicios | ✅ |

### Códigos de error verificados contra el despliegue

| Situación | Código | Quién responde |
|---|---|---|
| Sin token | 401 | Authorizer del API Gateway (no llega al BFF) |
| Token mal formado o vencido | 401 | Authorizer / Resource Server |
| Rol insuficiente | 403 | BFF y microservicio |
| Transición de estado inválida | 409 | ms-requests |
| Cupo diario agotado al admitir | 409 | ms-requests |
| Nombre de tipo duplicado | 409 | ms-catalog |
| Microservicio no disponible | 503 | BFF |
| Recurso inexistente | 404 | Microservicio |

---

## Instrucciones específicas del encargo

| Requisito | Cómo se resolvió | Estado |
|---|---|---|
| Backend compila, buenas prácticas y pruebas básicas | **42 tests** en los 6 microservicios, corriendo en GitHub Actions (`mvn -B verify`) en cada push. Usan H2 y JWT simulado: no requieren Oracle ni Azure | ✅ |
| Frontend completo, modular, sin errores, vistas funcionales | Angular 17 standalone; CI con `npm ci && npm run build`. Home/login, Trámites y Catálogo operativas | ✅ |
| Integración con BD cloud (entidades, repositorios, propiedades) | Oracle 23 Free en EC2 privada, **un usuario por microservicio** (Database per Service), JPA + Spring Data + migraciones Flyway | ✅ |
| Filtros que validan el JWT del IDaaS | Cadena de Spring Security OAuth2 Resource Server, `STATELESS`, en el BFF y los 4 microservicios expuestos | ✅ |
| Frontend con login IDaaS y JWT en las llamadas | Login con Azure AD y `Bearer` adjunto por el `MsalInterceptor` | ✅ |
| `.gitignore` correcto por tecnología | En los 9 repos: `target/`, `node_modules/`, `.env`, `*.pem`, `.idea/`. Ningún secreto versionado; el repo trae `.env.example` | ✅ |
| Formato: microservicios Java/Spring Boot + Angular por GitHub | 6 microservicios Spring Boot 3.3 (Java 17) + frontend Angular + infra + docs = **9 repositorios**, con CI e imagen Docker a GHCR | ✅ |

---

## Caso §3 — Alcance funcional mínimo

| Módulo | Regla clave | Cómo se resolvió | Estado |
|---|---|---|---|
| Gestión de trámites | No se puede pasar a EN_TERRENO sin ADMITIR | Máquina de estados en `EstadoTramite`; transición inválida → excepción → **409**. Con test dedicado | ✅ |
| Catálogo | El cupo diario disminuye al admitir | Al pasar a ADMITIDO se cuentan las admisiones del **día civil de Santiago**; si se agotó responde 409. La UI muestra el cupo por fila y bloquea el botón | ✅ |
| Notificaciones | Envío asíncrono (cola) | RabbitMQ, publicando después del commit y **fuera del hilo del request**: un broker caído no rompe la operación | ✅ |
| Reportería | Datos por streaming sin bloquear el core | Backend completo (consume `requests.events`, expone KPIs). **Falta la pantalla** `/reports` | ⚠️ Backend listo |
| Auditoría | Solo lectura | Backend completo (consume Kafka, persiste timeline). **Falta la pantalla** `/audit` | ⚠️ Backend listo |

---

## Caso §5 — Microservicios y endpoints

Los 7 servicios del enunciado existen y están desplegados: `bff`, `requests`, `catalog`, `notify`,
`audit`, `report`, más la infraestructura de RabbitMQ y Kafka.

| Endpoint esencial | Estado | Nota |
|---|---|---|
| `POST /api/requests` | ✅ | Rol Vecino o Funcionario |
| `GET /api/requests/{id}` | ✅ | Un vecino solo accede a los suyos |
| `PUT /api/requests/{id}/status` | ✅ | Valida transición y cupo |
| `GET /api/requests?status=&from=&to=` | ✅ | Acepta los nombres del caso y también `estado/desde/hasta` |
| `GET /api/catalog/procedures` | ✅ | Cualquier autenticado |
| `POST /api/catalog/procedures` | ✅ | Solo Admin |
| `PUT /api/catalog/procedures/{id}` | ✅ | Requisitos y cupo, editable desde la UI |
| `GET /api/report/kpis?range=last24h` | ✅ | Backend; sin pantalla todavía |
| `GET /api/report/top-procedures?range=last7d` | ✅ | Backend; sin pantalla todavía |

---

## Caso §6 — Pantallas

Brecha principal: **3 de 6 pantallas construidas**. Las que faltan consumen microservicios que sí
existen y responden.

| Pantalla | Ruta | Estado |
|---|---|---|
| Login | `/login` | ⚠️ El login vive en la Home (`/`), no en ruta propia |
| Dashboard | `/dashboard` | ❌ Pendiente (EP2) |
| Trámites | `/requests` | ✅ Listar, crear, cambiar estado, filtros y cupos |
| Catálogo | `/catalog` | ✅ Tipos y cupos, con edición en línea para Admin |
| Reportería | `/reports` | ❌ Pendiente — el backend de KPIs ya responde |
| Auditoría | `/audit` | ❌ Pendiente — el backend de timeline ya responde |

---

## Caso §7 — Despliegue

| Instancia | Contiene | Red |
|---|---|---|
| `ec2-apps` | frontend, bff, requests, catalog, notify, audit, report | Subred pública, IP elástica |
| `ec2-mq` | RabbitMQ, clúster de 2 nodos con Management UI | Subred privada |
| `ec2-kafka` | 3 ZooKeeper + 3 brokers Kafka + Kafka UI | Subred privada |
| `ec2-db` | Oracle 23 Free, 4 usuarios (uno por servicio) | Subred privada |

Un `compose.yml` por instancia. Security Groups con solo los puertos necesarios; solo `ec2-apps`
tiene IP pública y a las privadas se entra por bastión. ✅

---

## Caso §8 — Topología RabbitMQ

Las 6 colas (3 flujos + 3 DLQ), 3 exchanges y 9 bindings del enunciado, en `ms-barriodigital-notify`,
que es el dueño de la topología. ✅

| Cola | DLQ | Binding direct | Binding topic |
|---|---|---|---|
| `q.cmd.email` | `q.cmd.email.dlq` | `email.send` | `email.*` |
| `q.cmd.crew` | `q.cmd.crew.dlq` | `crew.ticket` | `crew.#` |
| `q.cmd.certificate` | `q.cmd.certificate.dlq` | `certificate.gen` | `certificate.*` |

Exchanges `cmd.direct`, `cmd.topic` y `cmd.dead.dlx`. Envelope común con
`type, eventId, timestamp, traceId, correlationId` y reintentos antes de derivar a la DLQ.

---

## Caso §9 — Topología Kafka

| Tópico | Part. | Repl. | Estado |
|---|---|---|---|
| `requests.events` | 3 | 3 | ✅ Verificado en el clúster; lo consumen audit y report |
| `audit.timeline` | 3 | 3 | ❌ Nadie lo produce; audit persiste en Oracle desde `requests.events` |
| `*.DLT` | 3 | 3 | ❌ Sin dead-letter topics por consumidor |

---

## Brechas conocidas

Ninguna afecta los dos indicadores ponderados de EP1, que se califican solo sobre MSAL y la
validación del JWT en el BFF. Se listan igual porque el caso sí las pide.

1. **Tres pantallas sin construir** (`/dashboard`, `/reports`, `/audit`). Los microservicios que
   las alimentan están desplegados y responden; falta la vista. Registrado como **H2** en
   [Pendientes.md](Pendientes.md), diferido a EP2.
2. **Kafka: falta `audit.timeline` y los `*.DLT`.** El flujo funciona sobre `requests.events`
   (3 particiones, 3 réplicas) y audit/report lo consumen.
3. **Preflight CORS: `OPTIONS` responde 401.** El authorizer del Gateway intercepta también
   OPTIONS. **No afecta la app desplegada**: frontend y API comparten origen, así que el navegador
   nunca dispara un preflight cross-origin.
4. **Clases de seguridad duplicadas en 5 repos.** Decisión consciente (ver
   [Auditoria-Simplificacion.md](Auditoria-Simplificacion.md)); el riesgo de divergencia está
   cubierto por un workflow que compara las copias y falla si difieren.
