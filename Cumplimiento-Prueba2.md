# Cumplimiento Prueba 2 (EP3 + EP4) — Caso BarrioDigital

Punto por punto de lo que piden la [pauta del encargo EP3](Requisitos%20de%20la%20Prueba/Pauta_Encargo_Prueba2.md),
la [pauta de presentación EP4](Requisitos%20de%20la%20Prueba/Pauta_Presentacion_Prueba2.md) y el
[enunciado del caso](Requisitos%20de%20la%20Prueba/Caso_BarrioDigital.md), con la evidencia concreta de dónde está
resuelto. Las brechas se listan igual que lo cumplido.

**Estado al 15-10-2026**, verificado leyendo el código de los 9 repos. Las justificaciones de diseño están en
[Justificacion-Mensajeria.md](Justificacion-Mensajeria.md); el backlog accionable en
[Pendientes-Prueba2.md](Pendientes-Prueba2.md); el guion de la demo en
[Guion-Presentacion-Prueba2.md](Guion-Presentacion-Prueba2.md).

> **Las dos pautas comparten los indicadores 1 al 8.** Solo el 9 difiere: EP3 puntúa lo **implementado**,
> EP4 lo **demostrado en vivo**. Una fila `✅` que no se pueda demostrar vale **0** en EP4.

---

## Resumen ejecutivo

| Indicador | % | Estado |
|---|---|---|
| 1 — Nombres centralizados | 10% | ⚠️ Parcial |
| 2 — Beans Queue/Exchange/Binding | 10% | ✅ |
| 3 — Config aislada de la lógica | 8% | ✅ |
| 4 — Consumidores por dominio | 12% | ⚠️ Los 3 en una clase |
| 5 — **ACK explícito + errores** | **16%** | ❌ ACK automático |
| 6 — MS administrador REST | 10% | ❌ No existe |
| 7 — `RabbitAdminService` | 8% | ❌ No existe |
| 8 — Validación de entrada | 6% | ❌ No existe |
| 9 — **Matriz de funcionalidades** | **20%** | ⚠️ Este documento la inaugura |

**18% cumplido** (2, 3) · **22% parcial** (1, 4) · **40% ausente** (5, 6, 7, 8) · **20% en curso** (9).

El punto de partida real es mejor de lo que sugiere la tabla: la topología de RabbitMQ del Caso §8 —3 exchanges,
6 colas, 9 bindings y DLX— **ya está implementada** en `ms-barriodigital-notify`, y `mq/compose.yml` ya define
2 nodos. Lo que falta es, sobre todo, **el plano de control** (indicadores 6, 7, 8 = 24%) y **hacer explícito el
ACK** (16%).

---

## Indicador 1 — Nombres centralizados en `application.yml` o `@Configuration` (10%)

> Cerrado en código por el bloque A (rama `feat/A-centralizar-nombres` de `notify` y `requests`), pendiente de merge.

| Criterio de la pauta | Cómo se resolvió | Estado |
|---|---|---|
| Nombres de exchanges centralizados | `barriodigital.rabbitmq.exchanges.*` en `application.yml`, leídos por `RabbitProperties` (`@ConfigurationProperties`) | ✅ |
| Nombres de colas centralizados | `barriodigital.rabbitmq.colas.*` y `routing-keys.*`; `RabbitTopologyConfig` ya no tiene literales | ✅ |
| Sin valores sueltos en el código | `NotificationPublisher` toma exchange y routing keys de `RabbitProperties`; los `@RabbitListener` usan `queues = "${barriodigital.rabbitmq.colas.email}"`. Quedan solo los `tipo` del envelope (`"email.send"`…), que son el tipo de comando, no un nombre de topología | ✅ |
| Organización consistente en todos los MS | Mismo namespace `barriodigital.rabbitmq.*` y misma clase `RabbitProperties`, copiada en `notify` y `requests` (duplicación deliberada, como `security`) | ✅ |

---

## Indicador 2 — Beans `Queue`, `Exchange` y `Binding` por caso de uso (10%)

| Criterio de la pauta | Cómo se resolvió | Estado |
|---|---|---|
| Un bean por exchange | `DirectExchange cmdDirectExchange()`, `TopicExchange cmdTopicExchange()`, `DirectExchange cmdDeadLetterExchange()` en `notify/amqp/RabbitTopologyConfig.java` | ✅ |
| Un bean por cola, con su DLQ | 6 beans `Queue`: las 3 principales vía el helper `cola(nombre, dlqRoutingKey)` que adjunta `x-dead-letter-exchange` y `x-dead-letter-routing-key`, más las 3 DLQ | ✅ |
| Un bean por binding | 9 beans `Binding`: 3 direct + 3 topic + 3 de DLQ | ✅ |
| Cada ruta identificada y documentada en código | Comentarios de sección por dominio (`// ---- q.cmd.email ----`) y javadoc de clase que cita la sección 8 del caso | ✅ |

### La topología declarada (Caso §8)

| Cola | DLQ | Binding direct | Binding topic | Routing key a DLX |
|---|---|---|---|---|
| `q.cmd.email` | `q.cmd.email.dlq` | `email.send` | `email.*` | `email.dlq` |
| `q.cmd.crew` | `q.cmd.crew.dlq` | `crew.ticket` | `crew.#` | `crew.dlq` |
| `q.cmd.certificate` | `q.cmd.certificate.dlq` | `certificate.gen` | `certificate.*` | `certificate.dlq` |

Exchanges: `cmd.direct` (direct), `cmd.topic` (topic), `cmd.dead.dlx` (direct). Todos `durable`, no `auto-delete`.

**Dos detalles honestos:** las DLQ no tienen DLX ni TTL —son terminales, sin camino de redrive— y **nadie publica
hoy a `cmd.topic`**: los 3 bindings topic existen pero el único productor manda todo a `cmd.direct`.

---

## Indicador 3 — Config aislada de la lógica de negocio (8%)

| Criterio de la pauta | Cómo se resolvió | Estado |
|---|---|---|
| Configuración aislada en clases específicas | Todo vive en el paquete `amqp/`: `RabbitTopologyConfig` (topología), `NotificationPublisher` (envío), `CommandEnvelope` (contrato) | ✅ |
| Los servicios de negocio solo consumen componentes ya configurados | `TramiteService` no conoce RabbitMQ. El puente es `service/event/TramiteEventRelay.java`, que escucha eventos de dominio con `@TransactionalEventListener(AFTER_COMMIT)` y llama a métodos de negocio del publisher (`notificarEmail`, `notificarCuadrilla`, `generarCertificado`) | ✅ |
| Sin lógica de configuración en controladores o servicios de dominio | `TramiteController` no menciona AMQP. Ninguna clase de `web/` ni `service/` declara colas o exchanges | ✅ |

**El detalle que sostiene el indicador:** el publisher expone **verbos de negocio**, no primitivas de mensajería.
`TramiteEventRelay` decide *qué comando* corresponde a cada transición de estado; nunca *a qué exchange va*. Por eso
agregar el MS administrador no puede romper este indicador: el plano de control va en otro proceso
([por qué](Justificacion-Mensajeria.md)).

---

## Indicador 4 — Consumidores con `@RabbitListener` agrupados por dominio (12%)

| Criterio de la pauta | Cómo se resolvió | Estado |
|---|---|---|
| Consumidores con `@RabbitListener` | 3 métodos anotados en `notify/amqp/CommandListener.java` | ✅ |
| Agrupados por dominio funcional | **No**: los 3 dominios (email, cuadrilla, certificado) están en **una sola clase** | ❌ |
| Estructura que facilita el mantenimiento | Comprensible, pero el nivel 60% describe exactamente esto: *"se encuentran mezclados dentro de un mismo paquete o clase sin una separación clara por dominios"* | ⚠️ |

**La brecha:** es la corrección más barata de toda la pauta —separar una clase en tres— y vale 12%.

---

## Indicador 5 — ACK explícito y manejo de errores (16%) · el más pesado

| Criterio de la pauta | Estado actual | Estado |
|---|---|---|
| Cada consumidor gestiona ACK de forma explícita | **No.** El modo es el `AcknowledgeMode.AUTO` por defecto de Spring: no hay `acknowledge-mode` en ningún `application.yml`, ningún listener recibe `Channel` ni `deliveryTag`, y no hay `basicAck`/`basicNack` en el proyecto | ❌ |
| Rutas diferenciadas entre reintentos, NACK y DLQ | **No.** Una excepción provoca NACK del contenedor y RabbitMQ dead-letterea por `x-dead-letter-exchange`. Funciona, pero la decisión es implícita | ❌ |
| Distinguir error recuperable de no recuperable | **No.** No existe `AmqpRejectAndDontRequeueException`, ni `ErrorHandler` propio, ni `RepublishMessageRecoverer`. Un payload inválido se reintenta igual que un timeout | ❌ |
| Registrar en logs los mensajes no entregados | **No.** Los `log.info` existentes son del camino feliz; no hay `log.error` en la ruta de fallo. La pauta lo pide literal: *"enviándolos a DLQ **y registrándolos en logs**"* | ❌ |
| Reintentos | Parcial: `notify/application.yml` tiene `listener.simple.retry: {enabled: true, max-attempts: 3}` con backoff por defecto | ⚠️ |
| Idempotencia | `ProcessedEventsGuard` deduplica por `eventId`; cada listener hace short-circuit si el evento ya se vio | ✅ |

**Dónde cae hoy:** el nivel **60%** lo describe con precisión incómoda — *"Los consumidores realizan ACK automáticos
o parciales. El manejo de errores existe pero es limitado"*. El salto al 100% exige ACK manual, y es el único
indicador donde el texto de la pauta no deja alternativa.

El techo de la idempotencia está declarado en el propio código (`ProcessedEventsGuard`): es un `Set` en memoria de
una sola instancia; con réplicas hay que pasar a tabla o Redis.

---

## Indicadores 6, 7 y 8 — El microservicio administrador (24%)

| Criterio de la pauta | Estado actual | Estado |
|---|---|---|
| 6 — Endpoints REST (`POST /queues`, `DELETE /queues/{name}`) | **No existe.** Cero ocurrencias de `RabbitAdmin` o `AmqpAdmin` en código; ningún endpoint en los 9 repos crea, borra, lista o purga colas | ❌ |
| 7 — `RabbitAdminService` dedicado | **No existe.** La topología se administra solo de forma declarativa, con `@Bean` aplicados al arrancar `notify` | ❌ |
| 8 — Validación de parámetros con DTO | **No existe** (depende del MS anterior) | ❌ |

La administración del broker hoy es **manual, por la Management UI**, alcanzada por túnel SSH
(`ssh -L 15672:10.0.1.254:15672 apps`). El Caso §5 pide el MS explícitamente: *"más un microservicio administrador
de RabbitMQ y otro de Kafka"*.

**Decisión tomada:** repo nuevo `ms-barriodigital-rabbitadmin`, puerto 8086, sin DB, en `ec2-apps`
(**no requiere una EC2 nueva**). El razonamiento de plano de control vs. plano de datos, con sus beneficios y la
contraindicación de meterlo en `notify`, está en [Justificacion-Mensajeria.md](Justificacion-Mensajeria.md).

---

## Indicador 9 — Matriz de funcionalidades (20%)

### Cómo se cuenta

El porcentaje es *(funcionalidades operativas ÷ total del alcance declarado) × 100*, y **las parciales no cuentan**.
El alcance se enumera sin duplicar: los 9 endpoints del Caso §5 son *la implementación* de los 5 módulos del §3,
así que se verifican aparte pero **no entran al denominador**. El denominador son **20 funcionalidades**:
5 módulos + 6 pantallas + 3 flujos de mensajería + 3 tópicos Kafka + 3 exigencias nuevas de EP3.

### A. Módulos funcionales (Caso §3)

| Módulo | Regla clave del caso | Componente responsable | Estado | Evidencia / qué falta |
|---|---|---|---|---|
| Gestión de trámites | No se puede pasar a `EN_TERRENO` sin admitir | `ms-barriodigital-requests` | ✅ | Máquina de estados en `EstadoTramite`; transición inválida → excepción → **409**. `EstadoTramiteTest` cubre el grafo |
| Catálogo | El cupo diario descuenta al admitir | `ms-barriodigital-catalog` + `requests` | ⚠️ | CRUD y **validación** de cupo operativos (`TramiteService.verificarCupoDisponible` → `CupoExcedidoException` → 409). Pero **falta `GET /requests/cupos`**: el BFF y el frontend lo llaman y no existe |
| Notificaciones | Envío asíncrono (cola) | `requests` (productor) + `notify` (consumidor) | ⚠️ | Publisher, 3 consumidores, DLQ e idempotencia completos en código. **Nunca ejecutado contra un broker real**, y los consumidores solo escriben log (no hay proveedor de email/PDF) |
| Reportería | Datos por streaming sin bloquear el core | `ms-barriodigital-report` | ⚠️ | Backend completo: consume `requests.events`, `KpiService` agrega y expone KPIs. **Falta la pantalla `/reports`** |
| Auditoría | Solo lectura | `ms-barriodigital-audit` | ⚠️ | Backend completo: `AuditEventListener` consume Kafka con dedupe por `eventId` y persiste el timeline. **Falta la pantalla `/audit`** |

**1 de 5 operativo.**

### B. Pantallas (Caso §6)

| Pantalla | Rol | Componente | Estado | Evidencia / qué falta |
|---|---|---|---|---|
| `/login` | Público | `frontend-barriodigital` | ⚠️ | El login funciona (MSAL v3, `loginPopup`) pero vive en `HomeComponent` en `/`, no en ruta propia |
| `/dashboard` | Todo autenticado | — | ❌ | No existe. La más barata de las tres que faltan: landing según rol con datos que ya responden |
| `/requests` | Admin, Funcionario, Vecino | `frontend-barriodigital` | ✅ | `RequestsComponent` con `MsalGuard`: formulario, tabla filtrable, avance de estado, refresco cada 10 s |
| `/catalog` | Admin, Funcionario | `frontend-barriodigital` | ✅ | `CatalogComponent` con `MsalGuard` + `roleGuard(['Admin','Funcionario'])`, edición en línea para Admin |
| `/reports` | Admin | — | ❌ | No existe. `GET /api/report/kpis` y `/top-procedures` ya responden |
| `/audit` | Admin, Auditor | — | ❌ | No existe. `GET /api/audit/timeline` ya responde, con filtros |

**2 de 6 operativas.** Es el punto más débil del alcance, y el de mejor relación esfuerzo/puntaje: son vistas Angular
sobre endpoints que ya funcionan, sin lógica de negocio nueva.

### C. Flujos de mensajería RabbitMQ (Caso §8)

| Flujo | Cola / DLQ | Productor → Consumidor | Estado | Evidencia / qué falta |
|---|---|---|---|---|
| Email / push al vecino | `q.cmd.email` / `.dlq` | `NotificationPublisher.notificarEmail` → `CommandListener.email` | ⚠️ | Código completo, disparado en `ADMITIDO`, `EN_TERRENO` y `RESUELTO`. Sin verificar contra broker |
| Ticket de visita a la cuadrilla | `q.cmd.crew` / `.dlq` | `notificarCuadrilla` → `CommandListener.crew` | ⚠️ | Ídem. Se dispara en `EN_TERRENO` |
| Generación de certificado PDF | `q.cmd.certificate` / `.dlq` | `generarCertificado` → `CommandListener.certificate` | ⚠️ | Ídem. Se dispara al crear (comprobante) y en `RESUELTO` (certificado) |

**0 de 3 operativos**, porque *operativo* exige haberlo visto funcionar y el broker nunca se levantó.
`TramiteEventRelayTest` verifica la **intención** (qué comando corresponde a cada transición), nunca el cable.

### D. Tópicos Kafka (Caso §9)

| Tópico | Part. / Repl. | Estado | Nota |
|---|---|---|---|
| `requests.events` | 3 / 3 | ✅ | Lo produce `RequestsEventPublisher`, lo consumen `audit` y `report` |
| `audit.timeline` | 3 / 3 | ❌ | Nadie lo produce; `audit` persiste en Oracle desde `requests.events` |
| `*.DLT` por consumidor | 3 / 3 | ❌ | Sin dead-letter topics |

**1 de 3.** Kafka queda **fuera del foco de Prueba 2 por decisión del equipo** (la pauta es 100% RabbitMQ), pero
sigue siendo alcance declarado del Caso, así que cuenta en el denominador. Ver *Brechas conocidas*.

### E. Exigencias nuevas de la pauta EP3

| Exigencia (texto de la pauta) | Componente | Estado | Evidencia / qué falta |
|---|---|---|---|
| "productores y consumidores que se conecten a colas RabbitMQ" | `requests` + `notify` | ⚠️ | Ambos existen y son los únicos 2 MS con `spring-boot-starter-amqp`. Pendiente verificarlos contra el broker |
| "manejar mensajes no entregados enviándolos a DLQ **y registrándolos en logs**" | `notify` | ⚠️ | La DLQ está declarada vía `x-dead-letter-exchange`, pero **no hay log en la ruta de fallo** |
| "microservicio administrador… crear y eliminar colas, exchanges y bindings" | — | ❌ | No existe |

**0 de 3.**

### Conteo consolidado

| Bloque | Operativas | Total |
|---|---|---|
| A — Módulos funcionales | 1 | 5 |
| B — Pantallas | 2 | 6 |
| C — Flujos RabbitMQ | 0 | 3 |
| D — Tópicos Kafka | 1 | 3 |
| E — Exigencias EP3 | 0 | 3 |
| **Total hoy** | **4** | **20** |

**Hoy: 4/20 = 20%** → por debajo del piso de 27%, es decir **0 puntos** en un indicador que vale 20%.

Proyección al cerrar el roadmap de [Pendientes-Prueba2.md](Pendientes-Prueba2.md):

| Escenario | Operativas | Ratio | Tramo |
|---|---|---|---|
| Hoy | 4 / 20 | 20% | **0 pts** |
| \+ clúster verificado y 3 flujos demostrados | 7 / 20 | 35% | 30 pts |
| \+ MS admin, ACK con logs, cupos | 11 / 20 | 55% | 60 pts |
| \+ `/reports`, `/audit`, `/dashboard`, `/login` | 15 / 20 | 75% | 80 pts |
| \+ los 5 módulos cerrados | **18 / 20** | **90%** | **100 pts** |
| \+ los 2 tópicos Kafka pendientes | 20 / 20 | 100% | 100 pts |

**Cerrar el roadbook completo deja el indicador justo en el 90%**, el borde del tramo máximo. No hay margen para
dejar fuera un bloque entero: las 4 pantallas y los 3 flujos de mensajería son obligatorios, no opcionales.

---

## Checklist de demo de EP4 (los 11 ítems)

| # | Ítem exigido | Estado | Evidencia / qué falta |
|---|---|---|---|
| 1 | Instancia de API Manager en la nube | ✅ | API Gateway HTTP API, ID `dnddhzpvgg`, `us-east-1` |
| 2 | Config del API Manager que llama al backend | ✅ | Ruta `ANY /api/{proxy+}` → `http://<EIP>:8080/api/{proxy}` |
| 3 | El frontend consume vía API Manager | ✅ | `environment.prod.ts` apunta al Invoke URL; el Gateway además sirve la SPA |
| 4 | El API Manager valida JWT | ✅ | Authorizer `entra-jwt` (issuer + audience). **401** capturado el 11-09-2026 sin token |
| 5 | Tenant IDaaS con usuarios registrados | ⚠️ | `Pendientes.md` A1–A6 y A9 siguen abiertos (`P0` desde EP1) |
| 6 | Frontend con OAuth 2.0 / OIDC → JWT válido | ✅ | MSAL v3 con `acquireTokenSilent`; los roles se leen del **access token** |
| 7 | Backend y frontend desplegados e integrados | ⚠️ | Nunca corrido end-to-end. Bloqueado por **G4**: los packages GHCR son privados |
| 8 | **2 nodos RabbitMQ en clúster** | ⚠️ | `mq/compose.yml` define `rabbitmq1` y `rabbitmq2` con cookie Erlang y `join_cluster`. **Nunca ejecutado** |
| 9 | **3 colas con sus DLQ funcionando** | ⚠️ | Declaradas (tabla del indicador 2). Nunca verificadas contra un broker |
| 10 | **Exchanges direct y topic configurados** | ⚠️ | Los 3 exchanges y 9 bindings existen, pero **nadie publica a `cmd.topic`** |
| 11 | **docker-compose en la nube levantando RabbitMQ** | ⚠️ | `deploy.yml` está limitado a `paths: ['apps/**']`; `mq/` se despliega a mano por `scp` y `ec2-mq` no tiene el repo clonado |

**5 verdes, 6 amarillos, 0 rojos.** Ninguno es imposible: los 6 amarillos son ejecución y evidencia, no desarrollo
desde cero. El guion cronometrado que los recorre está en
[Guion-Presentacion-Prueba2.md](Guion-Presentacion-Prueba2.md).

---

## Verificación por capas

### Arquitectura en la nube
4 EC2 (`ec2-apps` pública con EIP · `ec2-mq`, `ec2-kafka`, `ec2-db` privadas), API Gateway HTTP API con JWT
authorizer, sin ECS/RDS/ALB por costo. `sg-mq` ya permite 5672 y 15672 desde `sg-apps`, y 4369/25672 dentro de
`sg-mq`: **el MS administrador no necesita ninguna regla ni instancia nueva.**

**El límite real es la RAM.** `ec2-apps` es t3.medium (**4 GB**) y hoy corre 6 servicios Java a `-Xmx384m` —el ancla
`&aad` de `apps/compose.yml:26`— más nginx. Con el overhead de JVM eso ya son ~3.8 GB de 4. Un 7mo servicio Java
heredando 384m provoca **OOM kill**, y con `restart: always` eso es un bucle de reinicios. Al admin hay que darle
`-Xmx192m` y conviene bajar `notify` a lo mismo (es headless, 5 clases, sin web ni JPA).

### Contenedor RabbitMQ
`barriodigital-infra/mq/compose.yml` levanta 2 nodos `rabbitmq:3.13-management`. Problemas, en orden de gravedad:

| # | Hallazgo | Impacto |
|---|---|---|
| 1 | Todas las colas son `QueueBuilder.durable(...)` **clásicas** → viven en un solo nodo | El clúster comparte metadata, no mensajes: el 2º nodo no tiene copia de nada. Hoy "2 nodos" es cosmético |
| 2 | `rabbitmq2` no publica puertos | Sin UI ni AMQP alcanzables en el segundo nodo |
| 3 | El `entrypoint` reemplaza el del image y deja `tail -f /dev/null` como PID 1 | Si el broker muere el contenedor sigue `Up` y `restart: always` no dispara. `rabbitmq2` tampoco tiene healthcheck |
| 4 | Join por `sleep 10` + `join_cluster`, no peer discovery | Frágil; sin `set -e` sobrevive al reinicio por accidente |
| 5 | Sin `rabbitmq.conf`, `definitions.json` ni bind mount donde ponerlos | La topología solo existe después de que arranca `notify`; un clúster nuevo muestra UI vacía |
| 6 | Credenciales `guest/guest`, cookie hardcodeada, sin `mq/.env.example` | `apps/` y `kafka/` sí tienen su `.env.example` |
| 7 | `spring.rabbitmq.host` apunta a un solo host | Sin failover al nodo 2. `RABBITMQ_PORT` lo lee Spring pero compose nunca lo pasa |

### Backend
Spring Boot 3.3.5 / Java 17 en los 6 MS, `cl.duoc.barriodigital` como groupId. Solo `requests` y `notify` tienen
`spring-boot-starter-amqp`. `notify` es headless: sin web, sin seguridad, sin DB — 5 clases en total.
Los dos transportes están separados por propósito: **RabbitMQ para comandos** (`cmd.direct` → `notify`) y
**Kafka para hechos** (`requests.events` → `audit` + `report`).

### Base de datos
4 esquemas Oracle (`requests`, `catalog`, `audit`, `report`) sobre el mismo `DB_HOST` con usuario distinto —Database
per Service por esquemas—, Flyway `V1__init.sql` en cada uno y `ddl-auto: validate`. El requisito de la pauta
(*"entidades, repositorios y propiedades de conexión"*) **está cumplido**. El MS administrador **no lleva DB**: es
plano de control, stateless.

**Bug encontrado y confirmado:** `GET /requests/cupos` **no existe**. `TramiteController` solo declara
`@GetMapping`, `/{id}`, `POST` y `/{id}/status`, pero el BFF lo proxea (`DomainClients.cuposDeHoy()`) y el frontend
lo llama (`requests.component.ts:250`). La petición cae en `@GetMapping("/{id}")`, falla al convertir `"cupos"` a
`Long`, y el componente lo traga en silencio (`error: () => this.cupos.clear()`). **Los badges de cupo y el aviso
"quedan N cupos" nunca se muestran.** La lógica de cupo sí funciona al admitir; lo que falta es exponerla.

### Frontend
Angular 17.3 standalone, MSAL v3, roles leídos del **access token** (no del `id_token` — los App Roles se asignan
sobre el service principal de la API). Lo que pide la pauta —modular, sin errores de compilación, vistas
funcionales, login IDaaS, JWT en las llamadas— **está cumplido**. Faltan 4 de 6 pantallas del Caso §6.
El único mecanismo de actualización viva es un poll de 10 s, con su restricción documentada en código: el HTTP API
Gateway no soporta WebSocket y corta SSE a los 30 s.

### Higiene de repositorios
La pauta pide *"solo debe subirse a los repositorios lo que corresponda a la tecnología utilizada, configurando los
`.gitignore` necesarios"*. **Verificado el 15-10-2026:** `git ls-files` no devuelve ni un archivo bajo `target/` ni
`node_modules/` en ningún repo. Los `.gitignore` de los MS cubren `target/`, `*.class`, `*.jar`, IDEs y `.env`; el
del frontend cubre `/dist`, `/node_modules`, `.angular/cache` y `environment.local.ts`. ✅

---

## Brechas conocidas

1. **El plano de control no existe** (indicadores 6, 7, 8 = **24%**). Es la mayor concentración de puntos ausentes.
   Se resuelve con `ms-barriodigital-rabbitadmin`; no requiere EC2 nueva.
2. **El ACK es implícito** (**16%**). El único indicador donde el texto de la pauta no admite alternativa al ACK
   manual.
3. **La capa MQ nunca se ha ejecutado.** El README de infra lo dice: los 3 compose están validados solo como
   sintaxis YAML. Hasta que `rabbitmqctl cluster_status` muestre los dos nodos, los ítems 8, 9 y 11 de EP4 y los 3
   flujos de la matriz son promesas.
4. **Nadie publica a `cmd.topic`.** Los bindings topic existen sin productor: el ítem 10 de EP4 se puede *mostrar*
   pero no *demostrar*.
5. **El segundo nodo no replica.** Colas clásicas en un clúster de 2 nodos = cero redundancia real. Se cierra con
   colas quorum (el mirroring clásico está deprecado en 3.13). Ojo con el alcance: con 2 nodos quorum **replica los
   mensajes pero no tolera la caída de un nodo** (Raft de 2, mayoría = 2; la cola queda indisponible para escrituras).
   Tolerancia a fallo requiere 3 nodos.
6. **4 de 6 pantallas faltan o están fuera de ruta**, y son lo que decide el indicador de 20%.
7. **`GET /requests/cupos` roto**, con el frontend fallando en silencio.
8. **Kafka `audit.timeline` y `*.DLT` sin implementar.** Decisión consciente: la pauta de Prueba 2 es 100%
   RabbitMQ. Cuesta 2 puntos de denominador en la matriz y conviene decirlo antes de que lo pregunten, no después.
9. **Bloqueadores heredados de EP1, ambos `P0`:** **G4** (los packages GHCR siguen privados, así que `ec2-apps` no
   puede hacer `docker pull`) y **A1–A6/A9** (Entra ID). Sin ellos no hay demo end-to-end, y ninguno es trabajo de
   código.
10. **Riesgo de la nota individual.** EP4 se califica individual pero se presenta en grupo. El reparto actual separa
    AWS de Entra/frontend: **ambos integrantes tienen que poder defender RabbitMQ**, el clúster y el ACK.
