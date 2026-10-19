# Cumplimiento Prueba 2 (EP3 + EP4) — Caso BarrioDigital

Punto por punto de lo que piden la [pauta del encargo EP3](Requisitos%20de%20la%20Prueba/Pauta_Encargo_Prueba2.md),
la [pauta de presentación EP4](Requisitos%20de%20la%20Prueba/Pauta_Presentacion_Prueba2.md) y el
[enunciado del caso](Requisitos%20de%20la%20Prueba/Caso_BarrioDigital.md), con la evidencia concreta de dónde está
resuelto. Las brechas se listan igual que lo cumplido.

**Estado al 18-10-2026, después del despliegue**, verificado contra `origin/main` de los 10 repos y en vivo en
`ec2-apps` y `ec2-mq` (contenedores, `RestartCount`, `cluster_status`, `list_queues`, logs de `notify`). La primera versión de este
documento se escribió sobre clones locales atrasados y daba por rotas cosas que ya estaban en `main` (p. ej.
`GET /requests/cupos`); las diferencias entre lo supuesto y lo medido están en
[Hallazgos-Operacion.md](Hallazgos-Operacion.md). Las justificaciones de diseño están en
[Justificacion-Mensajeria.md](Justificacion-Mensajeria.md); el backlog accionable en
[Pendientes-Prueba2.md](Pendientes-Prueba2.md); el guion de la demo en
[Guion-Presentacion-Prueba2.md](Guion-Presentacion-Prueba2.md); dónde mirar y cómo validar cada indicador, en
[Guia-Validacion-Rubrica.md](Guia-Validacion-Rubrica.md).

> **Las dos pautas comparten los indicadores 1 al 8.** Solo el 9 difiere: EP3 puntúa lo **implementado**,
> EP4 lo **demostrado en vivo**. Una fila `✅` que no se pueda demostrar vale **0** en EP4.

---

## Resumen ejecutivo

| Indicador | % | Estado |
|---|---|---|
| 1 — Nombres centralizados | 10% | ✅ Bloque A mergeado |
| 2 — Beans Queue/Exchange/Binding | 10% | ✅ (ahora colas quorum) |
| 3 — Config aislada de la lógica | 8% | ✅ |
| 4 — Consumidores por dominio | 12% | ✅ 3 listeners en `amqp/listener/` (B) |
| 5 — **ACK explícito + errores** | **16%** | ✅ ACK manual con `AckManual` (C) |
| 6 — MS administrador REST | 10% | ✅ `ms-barriodigital-rabbitadmin` desplegado |
| 7 — `RabbitAdminService` | 8% | ✅ |
| 8 — Validación de entrada | 6% | ✅ |
| 9 — **Matriz de funcionalidades** | **20%** | ✅ 17/17 = 100% del alcance RabbitMQ (tramo 100) |

**Los 9 indicadores en su tramo máximo** según esta evaluación de lo implementado (EP3). Para EP4, que puntúa lo
demostrado, faltan I2 (token con rol `Admin`) y el ensayo.

**Estado al 18-10-2026:** B, C, C8, E1–E2, G4 y D11 están mergeados en `main` con CI verde y las imágenes
publicadas en GHCR. Desplegadas en `ec2-apps` el mismo día (8 contenedores, 0 reinicios) y verificadas contra el clúster: los 3 flujos
procesados, la DLQ con `log.error` y `x-death`, y `/login` en 200 por el Gateway. Queda I2 (Entra) para demostrar
con un token `Admin`, y el ensayo.

---

## Indicador 1 — Nombres centralizados en `application.yml` o `@Configuration` (10%)

> Cerrado por el bloque A: `feat/A-centralizar-nombres` mergeada en `notify` y `requests` el 15-10-2026.

| Criterio de la pauta | Cómo se resolvió | Estado |
|---|---|---|
| Nombres de exchanges centralizados | `barriodigital.rabbitmq.exchanges.*` en `application.yml`, leídos por `RabbitProperties` (`@ConfigurationProperties`) | ✅ |
| Nombres de colas centralizados | `barriodigital.rabbitmq.colas.*` y `routing-keys.*`; `RabbitTopologyConfig` ya no tiene literales | ✅ |
| Sin valores sueltos en el código | `NotificationPublisher` toma exchange y routing keys de `RabbitProperties`; los `@RabbitListener` usan `queues = "${barriodigital.rabbitmq.colas.email}"`. El `type` del envelope también sale de `RabbitProperties` (es la routing key canónica del comando). Verificable: `grep -rnE '"(q\.cmd|cmd\.|email\.|crew\.|certificate\.)' src/main` no devuelve nada en `notify` ni en `requests` | ✅ |
| Organización consistente en todos los MS | Mismo namespace `barriodigital.rabbitmq.*` y misma clase `RabbitProperties`, copiada en `notify` y `requests` (duplicación deliberada, como `security`) | ✅ |

---

## Indicador 2 — Beans `Queue`, `Exchange` y `Binding` por caso de uso (10%)

| Criterio de la pauta | Cómo se resolvió | Estado |
|---|---|---|
| Un bean por exchange | `DirectExchange cmdDirectExchange()`, `TopicExchange cmdTopicExchange()`, `DirectExchange cmdDeadLetterExchange()` en `notify/amqp/RabbitTopologyConfig.java` | ✅ |
| Un bean por cola, con su DLQ | 6 beans `Queue`: las 3 principales vía el helper `cola(nombre, dlqRoutingKey)` que adjunta `x-dead-letter-exchange` y `x-dead-letter-routing-key`, más las 3 DLQ | ✅ |
| Un bean por binding | 9 beans `Binding`: 3 direct + 3 topic + 3 de DLQ | ✅ |
| Cada ruta identificada y documentada en código | El javadoc de `RabbitTopologyConfig` trae la tabla de rutas (productor → exchange → binding → cola → DLQ), y cada sección del archivo (`// ---- email: q.cmd.email <- …`) documenta las suyas | ✅ |

### La topología declarada (Caso §8)

| Cola | DLQ | Binding direct | Binding topic | Routing key a DLX |
|---|---|---|---|---|
| `q.cmd.email` | `q.cmd.email.dlq` | `email.send` | `email.*` | `email.dlq` |
| `q.cmd.crew` | `q.cmd.crew.dlq` | `crew.ticket` | `crew.#` | `crew.dlq` |
| `q.cmd.certificate` | `q.cmd.certificate.dlq` | `certificate.gen` | `certificate.*` | `certificate.dlq` |

Exchanges: `cmd.direct` (direct), `cmd.topic` (topic), `cmd.dead.dlx` (direct). Todos `durable`, no `auto-delete`.
Las 6 colas son **quorum** (A5) y en `ec2-mq` tienen réplica en `rabbit@rabbitmq1` y `rabbit@rabbitmq2`
(`rabbitmq-queues quorum_status`, verificado el 15-10-2026).

**Detalle honesto:** las DLQ no tienen DLX ni TTL; son terminales, sin camino de redrive. Desde E1, `requests`
publica a `cmd.topic` `email.resuelto` (calza con `email.*`) y `crew.visita.agendada` (calza con `crew.#`); el
binding `certificate.*` sigue sin productor.

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
| Consumidores con `@RabbitListener` | Un `@RabbitListener` por cola | ✅ |
| Agrupados por dominio funcional | `EmailCommandListener`, `CrewCommandListener` y `CertificateCommandListener` en `notify/amqp/listener/`, una clase por dominio | ✅ |
| Estructura que facilita el mantenimiento | La decisión de ACK común vive en `amqp/AckManual`; cada listener solo valida su payload y ejecuta su acción. `CommandListenersTest` cubre los tres | ✅ |

---

## Indicador 5 — ACK explícito y manejo de errores (16%) · el más pesado

| Criterio de la pauta | Estado actual | Estado |
|---|---|---|
| Cada consumidor gestiona ACK de forma explícita | `acknowledge-mode: manual` y `default-requeue-rejected: false`. Cada listener recibe `Channel` y `DELIVERY_TAG` y delega en `AckManual.procesar`, que hace `basicAck`/`basicNack` | ✅ |
| Rutas diferenciadas entre reintentos, NACK y DLQ | Éxito o duplicado → `basicAck` · error recuperable → `basicNack(requeue=true)` hasta `max-intentos` (3) entregas, contadas con `x-delivery-count` de la cola quorum · agotadas o error permanente → `basicNack(requeue=false)` → DLX → DLQ | ✅ |
| Distinguir error recuperable de no recuperable | `ComandoNoRecuperableException` (payload sin campos obligatorios, vía `AckManual.exigir`) va directo a la DLQ sin gastar reintentos; cualquier otra excepción se trata como recuperable | ✅ |
| Registrar en logs los mensajes no entregados | `log.error` en cada NACK con `eventId`, cola y motivo (`NACK->DLQ …`, `NACK->requeue …`) | ✅ |
| Lógica de error documentada | Javadoc de `AckManual`, de `ComandoNoRecuperableException` y de cada listener | ✅ |
| Pruebas | `AckManualTest` (ack, requeue, DLQ por permanente, DLQ por reintentos agotados, duplicado, sin `eventId`, falla simulada) con `Channel` mockeado | ✅ |
| Verificado en vivo (18-10) | Contra el clúster: payload incompleto → `NACK->DLQ`; `simularFalla` → `NACK->requeue` 1/3 y 2/3 → `NACK->DLQ` 3/3 (confirma `x-delivery-count` en RabbitMQ 3.13); JSON inválido → rechazado por el contenedor → DLQ. Los mensajes de la DLQ traen `x-death` con `reason: rejected` | ✅ |
| Idempotencia | `ProcessedEventsGuard` deduplica por `eventId`; al reencolar se libera el `eventId` para que el reintento no se tome por duplicado | ✅ |

El techo de la idempotencia está declarado en el propio código (`ProcessedEventsGuard`): es un `Set` en memoria de
una sola instancia; con réplicas hay que pasar a tabla o Redis.

---

## Indicadores 6, 7 y 8 — El microservicio administrador (24%)

| Criterio de la pauta | Cómo se resolvió | Estado |
|---|---|---|
| 6 — Endpoints REST (`POST /queues`, `DELETE /queues/{name}`) | `QueueAdminController`, `ExchangeAdminController` y `BindingAdminController` bajo `/admin/**` | ✅ |
| 7 — `RabbitAdminService` dedicado | `service/RabbitAdminService.java` sobre `AmqpAdmin`; los controladores no conocen RabbitMQ | ✅ |
| 8 — Validación de parámetros con DTO | Records en `web/AdminDtos.java` con `@NotBlank`/`@Size`/`@Pattern`; `@Valid` en el cuerpo y `@Validated` para los nombres de la ruta y de `?names=`; el prefijo reservado `amq.` se rechaza; `ApiExceptionHandler` devuelve `ProblemDetail` (400 con el campo y el motivo, 409 al redeclarar con otros argumentos). 17 tests en `AdminApiTest` | ✅ |

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
Los tramos son proporcionales al máximo (90%): 72% de funcionalidades da el 80% del puntaje, 54% el 60% y 27% el 30%.

**Alcance de la Prueba 2: el sistema BarrioDigital con mensajería RabbitMQ.** La pauta de esta etapa es 100%
RabbitMQ (colas, exchanges, DLQ y el microservicio administrador), así que el alcance que se mide es el sistema
completo que el usuario usa —módulos y pantallas— más todo lo de RabbitMQ. Los **tópicos Kafka** del Caso §9 son la
infraestructura de eventos de la etapa anterior y **quedan fuera del alcance de esta prueba**; se listan al final,
sin contar, para que el estado se vea igual.

El alcance se enumera sin duplicar: los 9 endpoints del Caso §5 son *la implementación* de los 5 módulos del §3,
así que se verifican aparte pero **no entran al denominador**. El denominador son **17 funcionalidades**:
5 módulos + 6 pantallas + 3 flujos RabbitMQ + 3 exigencias nuevas de EP3.

### A. Módulos funcionales (Caso §3)

| Módulo | Regla clave del caso | Componente responsable | Estado | Evidencia / qué falta |
|---|---|---|---|---|
| Gestión de trámites | No se puede pasar a `EN_TERRENO` sin admitir | `ms-barriodigital-requests` | ✅ | Máquina de estados en `EstadoTramite`; transición inválida → excepción → **409**. `EstadoTramiteTest` cubre el grafo |
| Catálogo | El cupo diario descuenta al admitir | `ms-barriodigital-catalog` + `requests` | ✅ | CRUD y validación de cupo (`CupoExcedidoException` → 409). `GET /requests/cupos` existe desde el 12-09 (`TramiteController.cuposDeHoy`) y `RequestsApiSecurityTest.la_ruta_cupos_no_la_captura_el_id` lo protege |
| Notificaciones | Envío asíncrono (cola) | `requests` (productor) + `notify` (consumidor) | ✅ | Productor por `cmd.direct` y `cmd.topic`, 3 listeners con ACK manual y DLQ con log. Contra el broker real: 1 consumidor en cada cola. Los consumidores simulan el envío con log estructurado (no hay proveedor de email/PDF) |
| Reportería | Datos por streaming sin bloquear el core | `ms-barriodigital-report` | ✅ | Consume `requests.events`, `KpiService` agrega y la pantalla `/reports` (G1) los muestra |
| Auditoría | Solo lectura | `ms-barriodigital-audit` | ✅ | `AuditEventListener` consume Kafka con dedupe por `eventId`; la pantalla `/audit` (G2) muestra el timeline con filtros |

**5 de 5 operativos.**

### B. Pantallas (Caso §6)

| Pantalla | Rol | Componente | Estado | Evidencia / qué falta |
|---|---|---|---|---|
| `/login` | Público | `frontend-barriodigital` | ✅ | Ruta propia `/login` (G4), sin guard, con MSAL v3 `loginPopup`. Es alias de la portada: `/` no redirige para no romper el popup |
| `/dashboard` | Todo autenticado | `frontend-barriodigital` | ✅ | `DashboardComponent` con `MsalGuard` (G3) |
| `/requests` | Admin, Funcionario, Vecino | `frontend-barriodigital` | ✅ | `RequestsComponent` con `MsalGuard`: formulario, tabla filtrable, avance de estado, refresco cada 10 s |
| `/catalog` | Admin, Funcionario | `frontend-barriodigital` | ✅ | `CatalogComponent` con `MsalGuard` + `roleGuard(['Admin','Funcionario'])`, edición en línea para Admin |
| `/reports` | Admin | `frontend-barriodigital` | ✅ | `ReportsComponent` con `roleGuard(['Admin'])` (G1) |
| `/audit` | Admin, Auditor | `frontend-barriodigital` | ✅ | `AuditComponent` con `roleGuard(['Admin','Auditor'])` (G2) |

**6 de 6 operativas.** Las vistas por rol se demuestran recién con I2.

### C. Flujos de mensajería RabbitMQ (Caso §8)

| Flujo | Cola / DLQ | Productor → Consumidor | Estado | Evidencia / qué falta |
|---|---|---|---|---|
| Email / push al vecino | `q.cmd.email` / `.dlq` | `notificarEmail` (direct) y `difundirResuelto` (topic, `email.resuelto`) → `EmailCommandListener` | ✅ | Disparado en `ADMITIDO`, `EN_TERRENO` y `RESUELTO`. 18-10: `email.resuelto` por `cmd.topic` → `[EMAIL/PUSH] tramite=901 tipo=RESUELTO` en el log de notify |
| Ticket de visita a la cuadrilla | `q.cmd.crew` / `.dlq` | `difundirVisitaAgendada` (topic, `crew.visita.agendada`) → `CrewCommandListener` | ✅ | Se dispara en `EN_TERRENO`. 18-10: `crew.visita.agendada` por `cmd.topic` → `[CUADRILLA] ticket de visita creado` |
| Generación de certificado PDF | `q.cmd.certificate` / `.dlq` | `generarCertificado` (direct) → `CertificateCommandListener` | ✅ | Se dispara al crear (comprobante) y en `RESUELTO` (certificado). 18-10: `certificate.gen` por `cmd.direct` → `[CERTIFICADO] PDF generado` |

**3 de 3 operativos.** Verificado el 18-10-2026 contra el clúster real tras desplegar las imágenes nuevas: se
publicó en el broker el mismo envelope que arma `NotificationPublisher` y `notify` procesó los tres sin `NACK`
(dos por `cmd.topic`, uno por `cmd.direct`). El disparo desde la UI, al cambiar el estado de un trámite, necesita
un token con rol (I2) y se demuestra en el ensayo (H5). El productor está cubierto por `TramiteEventRelayTest` y
`NotificationPublisherTest`.

### D. Exigencias nuevas de la pauta EP3

| Exigencia (texto de la pauta) | Componente | Estado | Evidencia / qué falta |
|---|---|---|---|
| "productores y consumidores que se conecten a colas RabbitMQ" | `requests` + `notify` | ✅ | Conectados al clúster en `ec2-mq`: 1 consumidor por cola de trabajo (`list_queues`, 15-10-2026) |
| "manejar mensajes no entregados enviándolos a DLQ **y registrándolos en logs**" | `notify` | ✅ | `basicNack(requeue=false)` → DLX → DLQ, con `log.error` (`eventId`, cola y motivo) en cada rechazo (C5). Ver indicador 5 |
| "microservicio administrador… crear y eliminar colas, exchanges y bindings" | `ms-barriodigital-rabbitadmin` | ✅ | Ver indicadores 6–8 |

**3 de 3.**

### Conteo consolidado

| Bloque | Operativas | Total |
|---|---|---|
| A — Módulos funcionales | 5 | 5 |
| B — Pantallas | 6 | 6 |
| C — Flujos RabbitMQ | 3 | 3 |
| D — Exigencias EP3 | 3 | 3 |
| **Total** | **17** | **17** |

**Hoy (19-10): 17/17 = 100%** → tramo de **100 puntos**.

| Momento (mismo alcance de 17) | Operativas | Ratio | Tramo |
|---|---|---|---|
| 15-10 | 11 / 17 | 65% | 60 pts |
| 18-10: C, E1 y G4 mergeados | 14 / 17 | 82% | 80 pts |
| 18-10: \+ los 3 flujos vistos con éxito en el broker real | **17 / 17** | **100%** | **100 pts** |

**Margen:** si en la presentación una fila no se logra demostrar, queda 16/17 = 94% y sigue en el tramo de 100.
Con dos filas (15/17 = 88%) baja a 80. En EP4 cuenta lo demostrado: los 3 flujos hay que repetirlos en vivo (H5).

### Fuera del alcance de la Prueba 2: tópicos Kafka (Caso §9, no cuentan)

Se dejan a la vista para que el estado sea completo. `requests.events` es el que alimenta los módulos de
Reportería y Auditoría, que sí están en el alcance y funcionan.

| Tópico | Part. / Repl. | Componente responsable | Estado | Evidencia / qué falta |
|---|---|---|---|---|
| `requests.events` | 3 / 3 | `requests` (productor) → `audit`, `report` | ✅ | `kafka/RequestsEventPublisher` en requests; `AuditEventListener` en audit y `ReportEventListener` en report |
| `audit.timeline` | 3 / 3 | `audit` | ❌ | Nadie lo produce; `audit` persiste en Oracle desde `requests.events` |
| `*.DLT` por consumidor | 3 / 3 | `audit`, `report` | ❌ | Sin dead-letter topics |

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
| 10 | **Exchanges direct y topic configurados** | ✅ | `cmd.direct`, `cmd.topic` y `cmd.dead.dlx` en el broker. `requests` publica `email.resuelto` y `crew.visita.agendada` a `cmd.topic` (E1); el 18-10 ambos se enrutaron (`routed: true`) y `notify` los procesó |
| 11 | **docker-compose en la nube levantando RabbitMQ** | ✅ | `mq/compose.yml` corriendo en `ec2-mq`; despliegue manual por `scp`, documentado en el README de infra |

**10 verdes, 1 amarillo, 0 rojos.** El amarillo es I2 (Entra, portal).
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
| 7 | `spring.rabbitmq.host` a un solo host | ✅ `spring.rabbitmq.addresses` con `RABBITMQ_ADDRESSES=10.0.1.254:5672,10.0.1.254:5673` en `requests`, `notify` y `rabbitadmin` (C8) |

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

Cerradas el 18-10: ACK implícito (C), consumidores en una clase (B), nadie publicaba a `cmd.topic` (E1) y sin
failover del cliente (C8).

1. **Entra sin App Roles** (I2). Sin un token con rol `Admin` no se demuestran los indicadores 6–8 (24%), que ya
   están construidos y corriendo, ni las pantallas por rol. No es código, y es el bloqueador más caro que queda.
2. **Quorum con 2 nodos no tolera la caída de uno.** Replica los mensajes, pero con Raft de 2 la mayoría es 2 y la
   cola queda indisponible (para escribir y para consumir) si cae un nodo. Tolerancia a fallo exige 3 nodos. Por
   eso el failover del cliente (C8) reconecta al otro nodo, pero no hace disponible la cola.
3. **Kafka `audit.timeline` y `*.DLT` sin implementar.** Fuera del alcance de la Prueba 2, que es 100% RabbitMQ:
   no entran a la matriz, pero están listados en ella. Conviene decirlo antes de que lo pregunten, no después.
4. **Reintento sin backoff.** El NACK con requeue devuelve el mensaje a la cola quorum y se reentrega de inmediato:
   los 3 intentos ocurren en milisegundos. Sirve para la clasificación de errores que pide la pauta; para fallos
   transitorios reales haría falta un retraso (cola de espera con TTL).
5. **Riesgo de la nota individual.** EP4 se califica individual pero se presenta en grupo: **ambos integrantes
   tienen que poder defender** el administrador (escrito por Francisco) y el ACK manual (escrito por Benjamín).
