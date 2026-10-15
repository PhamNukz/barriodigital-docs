# Cumplimiento Prueba 2 (EP3 + EP4) — Caso BarrioDigital

Punto por punto de lo que piden la [pauta del encargo EP3](Requisitos%20de%20la%20Prueba/Pauta_Encargo_Prueba2.md),
la [pauta de presentación EP4](Requisitos%20de%20la%20Prueba/Pauta_Presentacion_Prueba2.md) y el
[enunciado del caso](Requisitos%20de%20la%20Prueba/Caso_BarrioDigital.md), con la evidencia concreta de dónde está
resuelto. Las brechas se listan igual que lo cumplido.

**Estado al 15-10-2026, después del despliegue**, verificado contra `origin/main` de los 10 repos y en vivo en
`ec2-apps` y `ec2-mq` (contenedores, `RestartCount`, `cluster_status`, `list_queues`). La primera versión de este
documento se escribió sobre clones locales atrasados y daba por rotas cosas que ya estaban en `main` (p. ej.
`GET /requests/cupos`); las diferencias entre lo supuesto y lo medido están en
[Hallazgos-Operacion.md](Hallazgos-Operacion.md). Las justificaciones de diseño están en
[Justificacion-Mensajeria.md](Justificacion-Mensajeria.md); el backlog accionable en
[Pendientes-Prueba2.md](Pendientes-Prueba2.md); el guion de la demo en
[Guion-Presentacion-Prueba2.md](Guion-Presentacion-Prueba2.md).

> **Las dos pautas comparten los indicadores 1 al 8.** Solo el 9 difiere: EP3 puntúa lo **implementado**,
> EP4 lo **demostrado en vivo**. Una fila `✅` que no se pueda demostrar vale **0** en EP4.

---

## Resumen ejecutivo

| Indicador | % | Estado |
|---|---|---|
| 1 — Nombres centralizados | 10% | ✅ Bloque A mergeado |
| 2 — Beans Queue/Exchange/Binding | 10% | ✅ (ahora colas quorum) |
| 3 — Config aislada de la lógica | 8% | ✅ |
| 4 — Consumidores por dominio | 12% | ⚠️ Los 3 en una clase |
| 5 — **ACK explícito + errores** | **16%** | ❌ ACK automático |
| 6 — MS administrador REST | 10% | ✅ `ms-barriodigital-rabbitadmin` desplegado |
| 7 — `RabbitAdminService` | 8% | ✅ |
| 8 — Validación de entrada | 6% | ✅ |
| 9 — **Matriz de funcionalidades** | **20%** | ⚠️ 12/20 = 60% |

**52% cumplido** (1, 2, 3, 6, 7, 8) · **12% parcial** (4) · **16% ausente** (5) · **20% en curso** (9).

Lo que queda es **código de Benjamín en `notify`**: separar los listeners (12%) y el ACK manual (16%). La
infraestructura —clúster de 2 nodos, colas quorum replicadas, los 8 contenedores de `ec2-apps`— está desplegada y
verificada.

---

## Indicador 1 — Nombres centralizados en `application.yml` o `@Configuration` (10%)

> Cerrado por el bloque A: `feat/A-centralizar-nombres` mergeada en `notify` y `requests` el 15-10-2026.

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
Las 6 colas son **quorum** (A5) y en `ec2-mq` tienen réplica en `rabbit@rabbitmq1` y `rabbit@rabbitmq2`
(`rabbitmq-queues quorum_status`, verificado el 15-10-2026).

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

| Criterio de la pauta | Cómo se resolvió | Estado |
|---|---|---|
| 6 — Endpoints REST (`POST /queues`, `DELETE /queues/{name}`) | `QueueAdminController`, `ExchangeAdminController` y `BindingAdminController` bajo `/admin/**` | ✅ |
| 7 — `RabbitAdminService` dedicado | `service/RabbitAdminService.java` sobre `AmqpAdmin`; los controladores no conocen RabbitMQ | ✅ |
| 8 — Validación de parámetros con DTO | Records en `web/AdminDtos.java` con `@NotBlank`/`@Pattern`, `@Valid` en los controladores y `ApiExceptionHandler` con `ProblemDetail` | ✅ |

Repo `ms-barriodigital-rabbitadmin`, puerto 8086, sin DB, en `ec2-apps` (sin EC2 nueva), con `hasRole('Admin')` en
`/admin/**`. Tests: `AdminApiTest` (validación y seguridad) y `RabbitAdminServiceTest`. Desplegado el 15-10-2026,
0 reinicios. El razonamiento plano de control / plano de datos está en
[Justificacion-Mensajeria.md](Justificacion-Mensajeria.md); los detalles de operación (por qué `GET /admin/queues`
recibe los nombres, por qué una cola borrada reaparece) en [Hallazgos-Operacion.md §7](Hallazgos-Operacion.md).

> **Para demostrarlo en vivo hace falta un token con rol `Admin`**, y eso depende de I2 (App Roles en Entra).

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
| Catálogo | El cupo diario descuenta al admitir | `ms-barriodigital-catalog` + `requests` | ✅ | CRUD y validación de cupo (`CupoExcedidoException` → 409). `GET /requests/cupos` existe desde el 12-09 (`TramiteController.cuposDeHoy`) y `RequestsApiSecurityTest.la_ruta_cupos_no_la_captura_el_id` lo protege |
| Notificaciones | Envío asíncrono (cola) | `requests` (productor) + `notify` (consumidor) | ⚠️ | Contra el broker real: notify tiene 1 consumidor en cada cola y la DLQ recibió mensajes con `x-death` completo. Falta el ACK manual (C) para que la ruta de error sea explícita, y los consumidores solo escriben log (no hay proveedor de email/PDF) |
| Reportería | Datos por streaming sin bloquear el core | `ms-barriodigital-report` | ✅ | Consume `requests.events`, `KpiService` agrega y la pantalla `/reports` (G1) los muestra |
| Auditoría | Solo lectura | `ms-barriodigital-audit` | ✅ | `AuditEventListener` consume Kafka con dedupe por `eventId`; la pantalla `/audit` (G2) muestra el timeline con filtros |

**4 de 5 operativos.**

### B. Pantallas (Caso §6)

| Pantalla | Rol | Componente | Estado | Evidencia / qué falta |
|---|---|---|---|---|
| `/login` | Público | `frontend-barriodigital` | ⚠️ | El login funciona (MSAL v3, `loginPopup`) pero vive en `HomeComponent` en `/`, no en ruta propia |
| `/dashboard` | Todo autenticado | `frontend-barriodigital` | ✅ | `DashboardComponent` con `MsalGuard` (G3) |
| `/requests` | Admin, Funcionario, Vecino | `frontend-barriodigital` | ✅ | `RequestsComponent` con `MsalGuard`: formulario, tabla filtrable, avance de estado, refresco cada 10 s |
| `/catalog` | Admin, Funcionario | `frontend-barriodigital` | ✅ | `CatalogComponent` con `MsalGuard` + `roleGuard(['Admin','Funcionario'])`, edición en línea para Admin |
| `/reports` | Admin | `frontend-barriodigital` | ✅ | `ReportsComponent` con `roleGuard(['Admin'])` (G1) |
| `/audit` | Admin, Auditor | `frontend-barriodigital` | ✅ | `AuditComponent` con `roleGuard(['Admin','Auditor'])` (G2) |

**5 de 6 operativas.** Solo falta `/login` como ruta propia (G4). Las vistas por rol se demuestran recién con I2.

### C. Flujos de mensajería RabbitMQ (Caso §8)

| Flujo | Cola / DLQ | Productor → Consumidor | Estado | Evidencia / qué falta |
|---|---|---|---|---|
| Email / push al vecino | `q.cmd.email` / `.dlq` | `NotificationPublisher.notificarEmail` → `CommandListener.email` | ⚠️ | Disparado en `ADMITIDO`, `EN_TERRENO` y `RESUELTO`. Cola quorum con 1 consumidor en el broker real |
| Ticket de visita a la cuadrilla | `q.cmd.crew` / `.dlq` | `notificarCuadrilla` → `CommandListener.crew` | ⚠️ | Ídem. Se dispara en `EN_TERRENO` |
| Generación de certificado PDF | `q.cmd.certificate` / `.dlq` | `generarCertificado` → `CommandListener.certificate` | ⚠️ | Ídem. Se dispara al crear (comprobante) y en `RESUELTO` (certificado) |

**0 de 3 operativos todavía.** El cable ya existe: broker arriba, consumidores conectados y 40 mensajes de DLQ
capturados con su `x-death` ([Hallazgos §9](Hallazgos-Operacion.md)). Para contarlos falta **ver un comando
procesado con éxito** en el ensayo (H5), y conviene hacerlo después del ACK manual (C), porque eso cambia los
listeners.

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
| "productores y consumidores que se conecten a colas RabbitMQ" | `requests` + `notify` | ✅ | Conectados al clúster en `ec2-mq`: 1 consumidor por cola de trabajo (`list_queues`, 15-10-2026) |
| "manejar mensajes no entregados enviándolos a DLQ **y registrándolos en logs**" | `notify` | ⚠️ | La DLQ funciona (mensajes con `x-death` capturados), pero **no hay `log.error` en la ruta de fallo** (C5) |
| "microservicio administrador… crear y eliminar colas, exchanges y bindings" | `ms-barriodigital-rabbitadmin` | ✅ | Ver indicadores 6–8 |

**2 de 3.**

### Conteo consolidado

| Bloque | Operativas | Total |
|---|---|---|
| A — Módulos funcionales | 4 | 5 |
| B — Pantallas | 5 | 6 |
| C — Flujos RabbitMQ | 0 | 3 |
| D — Tópicos Kafka | 1 | 3 |
| E — Exigencias EP3 | 2 | 3 |
| **Total hoy** | **12** | **20** |

**Hoy: 12/20 = 60%** → tramo de **60 puntos**. (En la primera versión de este documento era 4/20.)

Proyección con lo que queda de [Pendientes-Prueba2.md](Pendientes-Prueba2.md):

| Escenario | Operativas | Ratio | Tramo |
|---|---|---|---|
| Hoy | 12 / 20 | 60% | 60 pts |
| \+ ACK manual con logs (C) → DLQ con logs y módulo de notificaciones | 14 / 20 | 70% | 60 pts |
| \+ los 3 flujos vistos con éxito en el ensayo | 17 / 20 | 85% | 80 pts |
| \+ `/login` como ruta propia (G4) | **18 / 20** | **90%** | **100 pts** |
| \+ los 2 tópicos Kafka pendientes | 20 / 20 | 100% | 100 pts |

**El tramo máximo exige cerrar C, demostrar los 3 flujos y G4.** Sin cualquiera de los tres el indicador se queda
en 80. Kafka sigue fuera del foco de esta prueba.

---

## Checklist de demo de EP4 (los 11 ítems)

| # | Ítem exigido | Estado | Evidencia / qué falta |
|---|---|---|---|
| 1 | Instancia de API Manager en la nube | ✅ | API Gateway HTTP API, ID `dnddhzpvgg`, `us-east-1` |
| 2 | Config del API Manager que llama al backend | ✅ | Ruta `ANY /api/{proxy+}` → `http://<EIP>:8080/api/{proxy}` |
| 3 | El frontend consume vía API Manager | ✅ | `environment.prod.ts` apunta al Invoke URL; el Gateway además sirve la SPA |
| 4 | El API Manager valida JWT | ✅ | Authorizer `entra-jwt` (issuer + audience). **401** capturado el 11-09-2026 sin token |
| 5 | Tenant IDaaS con usuarios registrados | ⚠️ | `Pendientes.md` A1–A6 y A9 siguen abiertos (I2 de Prueba 2, Benjamín). Sin App Roles no hay token con rol `Admin` |
| 6 | Frontend con OAuth 2.0 / OIDC → JWT válido | ✅ | MSAL v3 con `acquireTokenSilent`; los roles se leen del **access token** |
| 7 | Backend y frontend desplegados e integrados | ✅ | 8 contenedores en `ec2-apps` (frontend + 7 servicios), todos con `RestartCount=0`. Packages GHCR públicos (I1) |
| 8 | **2 nodos RabbitMQ en clúster** | ✅ | `cluster_status`: `rabbit@rabbitmq1` y `rabbit@rabbitmq2` en *Running Nodes* |
| 9 | **3 colas con sus DLQ funcionando** | ✅ | 6 colas **quorum**, replicadas en los 2 nodos; DLQ con mensajes `x-death` capturados |
| 10 | **Exchanges direct y topic configurados** | ⚠️ | `cmd.direct`, `cmd.topic` y `cmd.dead.dlx` en el broker, pero **nadie publica a `cmd.topic`** (E1) |
| 11 | **docker-compose en la nube levantando RabbitMQ** | ✅ | `mq/compose.yml` corriendo en `ec2-mq`; despliegue manual por `scp`, documentado en el README de infra |

**9 verdes, 2 amarillos, 0 rojos.** Los dos amarillos son trabajo de Benjamín: I2 (Entra) y E1 (publicar al topic).
Una sola captura desde `ec2-mq` cubre los ítems 8, 9 y 10. El guion está en
[Guion-Presentacion-Prueba2.md](Guion-Presentacion-Prueba2.md).

---

## Verificación por capas

### Arquitectura en la nube
4 EC2 (`ec2-apps` pública con EIP · `ec2-mq`, `ec2-kafka`, `ec2-db` privadas), API Gateway HTTP API con JWT
authorizer, sin ECS/RDS/ALB por costo. `sg-mq` permite 5672 y 15672 desde `sg-apps`, y 4369/25672 dentro de
`sg-mq`, y desde el 15-10-2026 también el **5673** (AMQP del nodo 2) desde `sg-apps`, probado desde `ec2-apps`. El
8080 de `sg-apps` tiene que quedar en `0.0.0.0/0`: restringido a una IP, el Gateway da 503 en todo `/api/*` ([Hallazgos §10](Hallazgos-Operacion.md)).

**RAM:** la alarma de OOM de la primera versión era falsa. `-Xmx` es un máximo, no una reserva: medido en
`ec2-apps`, 3831 MB totales con ~1,4 GB disponibles corriendo los 8 contenedores. `notify` y `rabbitadmin` van igual
a `-Xmx192m` (D8, D9). Sin swap (D10 no aplicado). Detalle en [Hallazgos-Operacion.md §1](Hallazgos-Operacion.md).

### Contenedor RabbitMQ
`barriodigital-infra/mq/compose.yml` levanta 2 nodos `rabbitmq:3.13-management`. Los hallazgos de la primera
revisión, y cómo quedaron:

| # | Hallazgo original | Estado |
|---|---|---|
| 1 | Colas **clásicas** → el 2º nodo no tenía copia de nada | ✅ Colas quorum (A5), replicadas en los 2 nodos. Con 2 nodos no toleran la caída de uno ([Hallazgos §2](Hallazgos-Operacion.md)) |
| 2 | `rabbitmq2` no publicaba puertos | ✅ `5673:5672` y `15673:15672` (F2) |
| 3 | `entrypoint` con `tail -f /dev/null` como PID 1 | ✅ Eliminado; healthcheck en ambos nodos |
| 4 | Join por `sleep 10` + `join_cluster` | ✅ `rabbitmq.conf` con `classic_config` (F1) |
| 5 | Sin `rabbitmq.conf` ni bind mount | ✅ `mq/rabbitmq.conf` montado en `conf.d/` |
| 6 | `guest/guest`, cookie hardcodeada, sin `.env.example` | ✅ `mq/.env.example` (F4); `guest` devuelve 401 |
| 7 | `spring.rabbitmq.host` a un solo host | ⚠️ Sigue igual: **C8**, pendiente |

### Backend
Spring Boot 3.3.5 / Java 17 en los 7 MS, `cl.duoc.barriodigital` como groupId. `requests`, `notify` y
`rabbitadmin` tienen `spring-boot-starter-amqp`. `notify` es headless: sin web, sin seguridad, sin DB.
Los dos transportes están separados por propósito: **RabbitMQ para comandos** (`cmd.direct` → `notify`) y
**Kafka para hechos** (`requests.events` → `audit` + `report`).

### Base de datos
4 esquemas Oracle (`requests`, `catalog`, `audit`, `report`) sobre el mismo `DB_HOST` con usuario distinto —Database
per Service por esquemas—, Flyway `V1__init.sql` en cada uno y `ddl-auto: validate`. El requisito de la pauta
(*"entidades, repositorios y propiedades de conexión"*) **está cumplido**. El MS administrador **no lleva DB**: es
plano de control, stateless.

**Corrección:** la primera versión de este documento daba `GET /requests/cupos` por inexistente. Era un clon
local atrasado 13 commits: en `main` existe desde el 12-09-2026 (`TramiteController.cuposDeHoy` →
`TramiteService.cuposDeHoy`), y `RequestsApiSecurityTest.la_ruta_cupos_no_la_captura_el_id` verifica que
`/{id}` no le gane la ruta. G5 no tenía nada que arreglar.

### Frontend
Angular 17.3 standalone, MSAL v3, roles leídos del **access token** (no del `id_token` — los App Roles se asignan
sobre el service principal de la API). Lo que pide la pauta —modular, sin errores de compilación, vistas
funcionales, login IDaaS, JWT en las llamadas— **está cumplido**. Están 5 de 6 pantallas del Caso §6; solo falta
`/login` como ruta propia.
El único mecanismo de actualización viva es un poll de 10 s, con su restricción documentada en código: el HTTP API
Gateway no soporta WebSocket y corta SSE a los 30 s.

### Higiene de repositorios
La pauta pide *"solo debe subirse a los repositorios lo que corresponda a la tecnología utilizada, configurando los
`.gitignore` necesarios"*. **Verificado el 15-10-2026:** `git ls-files` no devuelve ni un archivo bajo `target/` ni
`node_modules/` en ningún repo. Los `.gitignore` de los MS cubren `target/`, `*.class`, `*.jar`, IDEs y `.env`; el
del frontend cubre `/dist`, `/node_modules`, `.angular/cache` y `environment.local.ts`. ✅

---

## Brechas conocidas

1. **El ACK es implícito** (**16%**). El único indicador donde el texto de la pauta no admite alternativa al ACK
   manual. Bloque C, Benjamín.
2. **Los 3 consumidores siguen en una clase** (**12%**). Bloque B, Benjamín: va antes de C.
3. **Entra sin App Roles** (I2). Sin un token con rol `Admin` no se demuestran los indicadores 6–8 (24%), que ya
   están construidos y corriendo, ni las pantallas por rol. No es código, y es el bloqueador más caro que queda.
4. **Nadie publica a `cmd.topic`.** Los bindings topic existen sin productor: el ítem 10 de EP4 se puede *mostrar*
   pero no *demostrar*. Bloque E.
5. **Quorum con 2 nodos no tolera la caída de uno.** Replica los mensajes, pero con Raft de 2 la mayoría es 2 y la
   cola queda indisponible (para escribir y para consumir) si cae un nodo. Tolerancia a fallo exige 3 nodos.
6. **Sin failover del cliente** (C8): los MS solo conocen el nodo 1 (el 5673 ya está abierto en `sg-mq`; falta el cambio en los `application.yml`).
7. **Kafka `audit.timeline` y `*.DLT` sin implementar.** Decisión consciente: la pauta de Prueba 2 es 100%
   RabbitMQ. Cuesta 2 puntos de denominador en la matriz y conviene decirlo antes de que lo pregunten, no después.
8. **Riesgo de la nota individual.** EP4 se califica individual pero se presenta en grupo: **ambos integrantes
   tienen que poder defender** el administrador (escrito por Francisco) y el ACK manual (escrito por Benjamín).
