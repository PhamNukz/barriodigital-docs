# Guía de validación de la rúbrica — Prueba 2 (EP3 + EP4)

Para cada uno de los 9 indicadores: **qué pide el nivel 100%**, **el concepto** que hay que saber explicar,
**dónde mirar en el código**, **cómo validarlo** (tests, broker y app) y **qué decir** si preguntan. Todo lo que
dice "verificado" se corrió contra el sistema real el 18-10-2026. Los números de línea son de esa fecha.

El estado detallado está en [Cumplimiento-Prueba2.md](Cumplimiento-Prueba2.md); el guion cronometrado de la demo, en
[Guion-Presentacion-Prueba2.md](Guion-Presentacion-Prueba2.md). Esta guía es el respaldo técnico de los dos.

Atajos a los repos (todos en `main`):

| Alias | Repo |
|---|---|
| **notify** | [ms-barriodigital-notify](https://github.com/PhamNukz/ms-barriodigital-notify) — consumidor y **dueño de la topología** |
| **requests** | [ms-barriodigital-requests](https://github.com/PhamNukz/ms-barriodigital-requests) — productor |
| **rabbitadmin** | [ms-barriodigital-rabbitadmin](https://github.com/PhamNukz/ms-barriodigital-rabbitadmin) — microservicio administrador |
| **infra** | [barriodigital-infra](https://github.com/PhamNukz/barriodigital-infra) — `mq/compose.yml` (clúster) y `apps/compose.yml` |

---

## 0. Preparación (5 minutos antes)

1. **EC2 encendidas** (Learner Lab) y comprobar que la app responde:
   `https://dnddhzpvgg.execute-api.us-east-1.amazonaws.com/login` → 200.
2. **Terminal 1, logs de notify** (aquí se ve todo lo que pasa en los consumidores):
   ```bash
   ssh apps
   cd ~/barriodigital-infra/apps && docker compose logs -f --no-log-prefix notify
   ```
3. **Terminal 2, túnel a la Management UI** de los dos nodos:
   ```bash
   ssh -L 15672:127.0.0.1:15672 -L 15673:127.0.0.1:15673 mq
   ```
   Abrir http://localhost:15672 (nodo 1) y http://localhost:15673 (nodo 2). El usuario es `RABBITMQ_DEFAULT_USER`
   de `~/mq/.env` en `ec2-mq`; `guest` ya no entra.
4. **Token Admin**, solo para el administrador y depende de I2 (Entra): iniciar sesión en la app con el usuario
   Admin → DevTools → *Application* → *Local Storage* → la entrada cuyo nombre contiene `accesstoken` → copiar
   el campo `secret`. Dura ~1 h; el botón **↻ Renovar** de la app pide otro.

**Para publicar un mensaje a mano** (se usa en varios indicadores): Management UI → *Exchanges* → elegir el
exchange → *Publish message* → *Routing key*, en *Properties* poner `content_type` = `application/json`, y
pegar el *Payload*. **Cada `eventId` tiene que ser nuevo** en cada ensayo: `notify` recuerda los que ya vio
(idempotencia) hasta reiniciarse.

---

## Indicador 1 — Nombres centralizados (10%)

**100% pide:** todos los nombres de colas, exchanges y bindings en `application.yml` o en una clase de
configuración única, ningún valor suelto en el código y la misma organización en todos los microservicios.

**Concepto.** `@ConfigurationProperties` liga un árbol del YAML a un `record` tipado. Cambiar un nombre es
editar el YAML: no se recompila nada.

**Dónde mirar**
- notify: [`application.yml:17`](https://github.com/PhamNukz/ms-barriodigital-notify/blob/main/src/main/resources/application.yml#L17): bloque `barriodigital.rabbitmq` (exchanges, colas, routing keys).
- requests: [`application.yml:47`](https://github.com/PhamNukz/ms-barriodigital-requests/blob/main/src/main/resources/application.yml#L47): el mismo bloque, más `email-resuelto` y `crew-visita-agendada` ([línea 72](https://github.com/PhamNukz/ms-barriodigital-requests/blob/main/src/main/resources/application.yml#L72)), que solo publica él.
- [`notify/amqp/RabbitProperties.java:9`](https://github.com/PhamNukz/ms-barriodigital-notify/blob/main/src/main/java/cl/duoc/barriodigital/notify/amqp/RabbitProperties.java#L9) y su copia en [requests](https://github.com/PhamNukz/ms-barriodigital-requests/blob/main/src/main/java/cl/duoc/barriodigital/requests/amqp/RabbitProperties.java#L10).
- Los listeners referencian la cola por placeholder: `@RabbitListener(queues = "${barriodigital.rabbitmq.colas.email}")`.

**Cómo validarlo**
```bash
# en notify y en requests: no devuelve nada, porque src/main no tiene ningún nombre de topología literal
grep -rnE '"(q\.cmd|cmd\.|email\.|crew\.|certificate\.)' src/main
```

**Qué decir.** "El único lugar donde aparece `q.cmd.email` es el YAML. El código lee `p.colas().email()`, y hasta
el `type` del mensaje se toma de las properties. Los dos servicios usan el mismo namespace y la misma clase."

---

## Indicador 2 — Beans `Queue`, `Exchange` y `Binding` por caso de uso (10%)

**100% pide:** clases de configuración con beans para cada caso de uso, y cada ruta identificada y documentada en
el código.

**Concepto.** Con `RabbitAdmin` (que Spring Boot crea solo), todo bean `Queue`, `Exchange` o `Binding` se
**declara en el broker al arrancar**. La declaración es idempotente: si ya existe igual, no pasa nada. Por eso
`notify` es el dueño de la topología y la recrea si alguien la borra.

**Dónde mirar** — [`notify/amqp/RabbitTopologyConfig.java`](https://github.com/PhamNukz/ms-barriodigital-notify/blob/main/src/main/java/cl/duoc/barriodigital/notify/amqp/RabbitTopologyConfig.java)
- [Línea 21](https://github.com/PhamNukz/ms-barriodigital-notify/blob/main/src/main/java/cl/duoc/barriodigital/notify/amqp/RabbitTopologyConfig.java#L21): **tabla de rutas** en el javadoc: productor → exchange → binding → cola → DLQ.
- 3 exchanges (`cmd.direct`, `cmd.topic` [l. 55](https://github.com/PhamNukz/ms-barriodigital-notify/blob/main/src/main/java/cl/duoc/barriodigital/notify/amqp/RabbitTopologyConfig.java#L55), `cmd.dead.dlx`).
- `cola(...)` [l. 64](https://github.com/PhamNukz/ms-barriodigital-notify/blob/main/src/main/java/cl/duoc/barriodigital/notify/amqp/RabbitTopologyConfig.java#L64): cola **quorum** con `x-dead-letter-exchange` y `x-dead-letter-routing-key`.
- Una sección por dominio, [desde la l. 76](https://github.com/PhamNukz/ms-barriodigital-notify/blob/main/src/main/java/cl/duoc/barriodigital/notify/amqp/RabbitTopologyConfig.java#L76): cola + DLQ + binding direct + binding topic + binding DLQ.
- Total: **3 exchanges, 6 colas y 9 bindings**.
- En requests, [`RabbitTopologyConfig.java:33-38`](https://github.com/PhamNukz/ms-barriodigital-requests/blob/main/src/main/java/cl/duoc/barriodigital/requests/amqp/RabbitTopologyConfig.java#L33): declara solo los dos exchanges a los que publica.

**Cómo validarlo en el broker** (verificado el 18-10)
```bash
ssh mq
cd ~/mq && docker compose exec rabbitmq1 rabbitmqctl list_queues name type consumers messages
docker compose exec rabbitmq1 rabbitmqctl list_bindings source_name destination_name routing_key
```
Esperado: 6 colas `quorum`; las 3 principales con 1 consumidor; 9 bindings con origen en los 3 exchanges (más los
bindings por defecto del exchange `""`, que crea RabbitMQ). En la Management UI: *Queues* → `q.cmd.email` →
*Features* muestra `DLX: cmd.dead.dlx` y `DLK: email.dlq`.

**Qué decir.** "Cada caso de uso del Caso §8 (email, cuadrilla, certificado) tiene su cola, su DLQ y sus bindings
como beans. La tabla del javadoc dice qué productor usa cada ruta."

---

## Indicador 3 — Configuración aislada de la lógica de negocio (8%)

**100% pide:** la configuración de mensajería en clases propias; los servicios de negocio solo consumen
componentes ya configurados, sin configuración en controladores ni servicios de dominio.

**Concepto.** El dominio **publica eventos de Spring** (`ApplicationEventPublisher`), no mensajes AMQP. Un
*relay* los escucha **después del commit** (`@TransactionalEventListener(AFTER_COMMIT)`) y llama a verbos de
negocio del publisher. Así nunca se notifica un cambio que después se revierte.

**Dónde mirar**
- [`requests/service/TramiteService.java:86`](https://github.com/PhamNukz/ms-barriodigital-requests/blob/main/src/main/java/cl/duoc/barriodigital/requests/service/TramiteService.java#L86) y [l. 98](https://github.com/PhamNukz/ms-barriodigital-requests/blob/main/src/main/java/cl/duoc/barriodigital/requests/service/TramiteService.java#L98): solo `events.publishEvent(...)`.
- [`requests/service/event/TramiteEventRelay.java:62-79`](https://github.com/PhamNukz/ms-barriodigital-requests/blob/main/src/main/java/cl/duoc/barriodigital/requests/service/event/TramiteEventRelay.java#L62): qué estado dispara qué comando.
- [`requests/amqp/NotificationPublisher.java`](https://github.com/PhamNukz/ms-barriodigital-requests/blob/main/src/main/java/cl/duoc/barriodigital/requests/amqp/NotificationPublisher.java#L34): verbos de negocio (`notificarEmail`, `difundirResuelto`…); el único `convertAndSend` está en la [l. 71](https://github.com/PhamNukz/ms-barriodigital-requests/blob/main/src/main/java/cl/duoc/barriodigital/requests/amqp/NotificationPublisher.java#L71).

**Cómo validarlo**
```bash
# en requests: 0 coincidencias en servicio y controlador
grep -ci "amqp\|rabbit" src/main/java/cl/duoc/barriodigital/requests/service/TramiteService.java \
                        src/main/java/cl/duoc/barriodigital/requests/web/TramiteController.java
```

**Qué decir.** "`TramiteService` no sabe que existe RabbitMQ. El relay decide *qué comando* corresponde a cada
estado, nunca *a qué exchange va*: eso lo sabe solo el paquete `amqp/`."

---

## Indicador 4 — Consumidores con `@RabbitListener` agrupados por dominio (12%)

**100% pide:** todos los consumidores con `@RabbitListener`, organizados por dominio funcional, con una estructura
que facilite el mantenimiento.

**Dónde mirar** — paquete [`notify/amqp/listener/`](https://github.com/PhamNukz/ms-barriodigital-notify/tree/main/src/main/java/cl/duoc/barriodigital/notify/amqp/listener)
- `EmailCommandListener`, `CrewCommandListener` y `CertificateCommandListener`: una clase por dominio y una cola cada una.
- Cada listener solo valida su payload (`AckManual.exigir`) y hace su acción. La decisión de ACK común está en `AckManual`, así que no se repite.
- Test: [`CommandListenersTest`](https://github.com/PhamNukz/ms-barriodigital-notify/blob/main/src/test/java/cl/duoc/barriodigital/notify/amqp/listener/CommandListenersTest.java).

**Cómo validarlo en vivo.** Management UI → *Queues*: cada cola de trabajo tiene **1 consumidor**. Al publicar en
cada cola, el log muestra la clase de su dominio: `EmailCommandListener : [EMAIL/PUSH]…`,
`CrewCommandListener : [CUADRILLA]…`, `CertificateCommandListener : [CERTIFICADO]…`.

**Qué decir.** "Agregar un dominio nuevo es una clase nueva en `listener/` con su cola en el YAML. No se toca
ninguna de las otras."

---

## Indicador 5 — ACK explícito y manejo de errores (16%) ★ el más pesado

**100% pide:** cada consumidor confirma (ACK) de forma explícita; rutas diferenciadas para reintentos, NACK y
DLQ; la lógica de error documentada y consistente.

**Conceptos**
- `acknowledge-mode: manual`: el broker no da el mensaje por procesado hasta que el código llama a `basicAck`.
- `basicNack(tag, false, requeue)`: con `requeue=true` vuelve a la cola; con `false` RabbitMQ lo manda al
  **DLX** (`x-dead-letter-exchange`) y de ahí a la DLQ, con un header `x-death` que dice de dónde vino y por qué.
- **Error recuperable** (proveedor caído, timeout): vale la pena reintentar. **Error permanente** (payload mal
  formado): reintentar no lo arregla, así que va directo a la DLQ.
- `x-delivery-count`: header que las **colas quorum** agregan en cada reentrega. Con él se cuentan los intentos
  sin guardar estado.
- **Idempotencia**: un duplicado (mismo `eventId`) se confirma con ACK sin reprocesar, porque ya está procesado y
  no fallido.

**Dónde mirar**
- [`notify/application.yml:14-20`](https://github.com/PhamNukz/ms-barriodigital-notify/blob/main/src/main/resources/application.yml#L14): `acknowledge-mode: manual`, `default-requeue-rejected: false`, `max-intentos: 3`.
- [`notify/amqp/AckManual.java`](https://github.com/PhamNukz/ms-barriodigital-notify/blob/main/src/main/java/cl/duoc/barriodigital/notify/amqp/AckManual.java), **el corazón**:
  - javadoc l. 11-26: las tres rutas, y el caso del mensaje que no se puede convertir.
  - [`procesar` l. 61](https://github.com/PhamNukz/ms-barriodigital-notify/blob/main/src/main/java/cl/duoc/barriodigital/notify/amqp/AckManual.java#L61): sin `eventId` → DLQ (l. 65); duplicado → `basicAck` (l. 71); no recuperable → `basicNack(…, false)` (l. 78-79); recuperable → `intento >= maxIntentos` (l. 83) ? DLQ : `guard.liberar` + `basicNack(…, true)` (l. 88-91); éxito → `basicAck` (l. 95).
  - Cada NACK deja un `log.error` con `eventId`, cola y motivo.
- [`ComandoNoRecuperableException`](https://github.com/PhamNukz/ms-barriodigital-notify/blob/main/src/main/java/cl/duoc/barriodigital/notify/amqp/ComandoNoRecuperableException.java): la clasificación hecha explícita.
- Listener, [`EmailCommandListener.java:33-39`](https://github.com/PhamNukz/ms-barriodigital-notify/blob/main/src/main/java/cl/duoc/barriodigital/notify/amqp/listener/EmailCommandListener.java#L33): recibe `Channel`, `DELIVERY_TAG` y `x-delivery-count`.
- Tests: [`AckManualTest`](https://github.com/PhamNukz/ms-barriodigital-notify/blob/main/src/test/java/cl/duoc/barriodigital/notify/amqp/AckManualTest.java), una prueba por rama con `Channel` mockeado.

**Cómo validarlo en vivo** (los 5 casos verificados el 18-10). Publicar en `cmd.direct` con routing key
`email.send` y mirar el log de notify:

| Caso | Payload | Log esperado | Resultado en el broker |
|---|---|---|---|
| Éxito | `{"type":"email.send","eventId":"g-1","data":{"tramiteId":1,"tipoNotificacion":"ADMITIDO"}}` | `[EMAIL/PUSH] tramite=1 tipo=ADMITIDO` | ACK, la cola queda en 0 |
| Duplicado | el mismo de arriba, otra vez | `email.send duplicado, se ignora (eventId=g-1)` | ACK, no se reprocesa |
| Permanente | `{"type":"email.send","eventId":"g-2","data":{"tramiteId":1}}` | `NACK->DLQ … motivo=payload sin 'tipoNotificacion'` | 1 mensaje en `q.cmd.email.dlq` |
| Recuperable | `{"type":"email.send","eventId":"g-3","data":{"tramiteId":1,"tipoNotificacion":"X","simularFalla":true}}` | `NACK->requeue … intento 1/3`, `… 2/3`, `NACK->DLQ … reintentos agotados (3/3)` | reintenta 2 veces y va a la DLQ |
| No convertible | payload `esto no es json` | `WARN … Fatal message conversion error; message rejected` | a la DLQ, rechazado por el contenedor |

`simularFalla` es una **falla inyectable** ([`AckManual.java:55`](https://github.com/PhamNukz/ms-barriodigital-notify/blob/main/src/main/java/cl/duoc/barriodigital/notify/amqp/AckManual.java#L55)).
Existe porque los consumidores simulan el envío con un log y nunca fallan solos; así la ruta de reintentos se
puede ver en vivo, no solo en tests.

Después: *Queues* → `q.cmd.email.dlq` → *Get messages* (Ack mode: *Nack message requeue true*, para no
sacarlos) → el header `x-death` muestra `reason: rejected`, `queue: q.cmd.email` y `exchange` de origen.

**Qué decir.** "El ACK lo decide el código, no Spring. Hay tres salidas: éxito o duplicado es ACK; un error
permanente va a la DLQ sin gastar reintentos; un error recuperable se reencola hasta 3 entregas, contadas con el
`x-delivery-count` de la cola quorum, y después va a la DLQ. Toda salida a la DLQ deja un `log.error` con el motivo."

**Preguntas probables**
- *¿Por qué no `retry` de Spring?* Porque el retry de Spring reintenta dentro del mismo hilo y oculta la decisión.
  Con ACK manual la decisión se ve en el código, y el broker es el que cuenta las entregas.
- *¿Y si el reintento es inmediato?* Lo es: es una limitación declarada (Cumplimiento, brecha 4). Para fallos
  transitorios reales haría falta una cola de espera con TTL.
- *¿Por qué `default-requeue-rejected: false`?* Para que cualquier rechazo que no esté contemplado vaya a la DLQ
  y nunca quede en un bucle infinito.

---

## Indicador 6 — El administrador expone una API REST (10%)

**100% pide:** API REST clara y coherente: rutas, verbos HTTP y nombres de recursos bien definidos, siguiendo
buenas prácticas y documentada.

**Dónde mirar** — rabbitadmin, paquete [`web/`](https://github.com/PhamNukz/ms-barriodigital-rabbitadmin/tree/main/src/main/java/cl/duoc/barriodigital/rabbitadmin/web)

| Verbo | Ruta | Código | Clase |
|---|---|---|---|
| `POST` | `/admin/queues` | 201 | `QueueAdminController` l. 36 |
| `DELETE` | `/admin/queues/{nombre}` | 204 / 404 | l. 43 |
| `GET` | `/admin/queues[?names=a,b]` | 200 | l. 56 |
| `POST` | `/admin/queues/{nombre}/purge` | 200 | l. 63 |
| `POST` / `DELETE` | `/admin/exchanges[/{nombre}]` | 201 / 204 | `ExchangeAdminController` |
| `POST` / `DELETE` | `/admin/bindings` | 201 / 204 | `BindingAdminController` |

- Documentación: tabla de API y de códigos de error en el [README](https://github.com/PhamNukz/ms-barriodigital-rabbitadmin#api).
- Seguridad: [`SecurityConfig.java:26`](https://github.com/PhamNukz/ms-barriodigital-rabbitadmin/blob/main/src/main/java/cl/duoc/barriodigital/rabbitadmin/config/SecurityConfig.java#L26) exige `hasRole('Admin')` para todo `/admin/**`, en un solo lugar.

**Cómo validarlo.** El puerto 8086 **no** se abre en `sg-apps` (es plano de control), así que se llama desde
`ec2-apps`:
```bash
ssh apps
curl -s -o /dev/null -w "%{http_code}\n" localhost:8086/admin/queues          # 401: sin token (verificado)
T='<access token del usuario Admin>'
curl -s -H "Authorization: Bearer $T" localhost:8086/admin/queues              # 200: las 6 colas con mensajes y consumidores
curl -s -X POST -H "Authorization: Bearer $T" -H 'content-type: application/json' \
     localhost:8086/admin/queues -d '{"nombre":"q.demo"}'                        # 201
curl -s -X POST -H "Authorization: Bearer $T" -H 'content-type: application/json' \
     localhost:8086/admin/bindings -d '{"cola":"q.demo","exchange":"cmd.topic","routingKey":"email.*"}'   # 201
```
**Demo de cierre (topic + administrador juntos).** Publicar `email.resuelto` en `cmd.topic`: el mensaje llega a
`q.cmd.email` **y** a `q.demo`. Es un consumidor nuevo creado por API, sin tocar el productor. Después limpiar:
```bash
curl -s -X DELETE -H "Authorization: Bearer $T" localhost:8086/admin/queues/q.demo   # 204
```

**Qué decir.** "Recursos en plural, verbos HTTP con su significado y códigos correctos: 201 al crear, 204 al
borrar, 404 si no existe. Purgar es una acción sobre la cola, por eso es un subrecurso `POST …/purge`."

---

## Indicador 7 — `RabbitAdminService` encapsula la lógica (8%)

**100% pide:** toda la creación, actualización y eliminación de colas y exchanges en un servicio dedicado; los
controladores llaman métodos de alto nivel sin conocer RabbitMQ.

**Dónde mirar**
- [`service/RabbitAdminService.java`](https://github.com/PhamNukz/ms-barriodigital-rabbitadmin/blob/main/src/main/java/cl/duoc/barriodigital/rabbitadmin/service/RabbitAdminService.java): `crearCola` (l. 42), `eliminarCola` (54), `purgarCola` (61), `listar` (71), `crearExchange` (82), `eliminarExchange` (86), `crearBinding` (94), `eliminarBinding` (98). Es el **único** lugar que usa `AmqpAdmin`, `QueueBuilder` y `ExchangeBuilder`.
- Devuelve `InfoCola` (record propio), no tipos de Spring AMQP.

**Cómo validarlo**
```bash
# en rabbitadmin: los controladores no importan nada de AMQP
grep -l "org.springframework.amqp" src/main/java/cl/duoc/barriodigital/rabbitadmin/web/*Controller.java   # sin salida
```
Test: [`RabbitAdminServiceTest`](https://github.com/PhamNukz/ms-barriodigital-rabbitadmin/blob/main/src/test/java/cl/duoc/barriodigital/rabbitadmin/RabbitAdminServiceTest.java), con `AmqpAdmin` mockeado.

**Qué decir sobre "actualización".** "En AMQP una cola no se modifica una vez declarada: redeclararla con otros
argumentos es un error del broker. Las actualizaciones que existen son purgar y cambiar bindings, y están en el
servicio. Cambiar argumentos exige borrar y crear."

---

## Indicador 8 — Validación de entrada (6%)

**100% pide:** se validan todos los parámetros con DTO y anotaciones; no se pueden crear colas con nombre vacío
ni con configuraciones inválidas; los mensajes de error son claros.

**Dónde mirar**
- [`web/AdminDtos.java:20`](https://github.com/PhamNukz/ms-barriodigital-rabbitadmin/blob/main/src/main/java/cl/duoc/barriodigital/rabbitadmin/web/AdminDtos.java#L20): `NOMBRE_VALIDO`, con letras, dígitos y `. _ : -`, sin espacios y sin el prefijo reservado `amq.`. Records con `@NotBlank`, `@Size(max=255)` y `@Pattern`; el tipo de exchange restringido a `direct|topic|fanout|headers`.
- `@Validated` en `QueueAdminController` y `ExchangeAdminController`: también se validan `{nombre}` de la ruta y `?names=`.
- [`web/ApiExceptionHandler.java`](https://github.com/PhamNukz/ms-barriodigital-rabbitadmin/blob/main/src/main/java/cl/duoc/barriodigital/rabbitadmin/web/ApiExceptionHandler.java): `ProblemDetail` con `title` y `detail` legibles. 400 de cuerpo (l. 25), 400 de ruta (l. 34), 404 (l. 48) y **409** al redeclarar con otros argumentos (l. 65).
- Tests: 17 en [`AdminApiTest`](https://github.com/PhamNukz/ms-barriodigital-rabbitadmin/blob/main/src/test/java/cl/duoc/barriodigital/rabbitadmin/AdminApiTest.java).

**Cómo validarlo** (con el token de arriba)

| Petición | Esperado |
|---|---|
| `POST /admin/queues {"nombre":""}` | 400 · `nombre: el nombre de la cola es obligatorio` |
| `POST /admin/queues {"nombre":"cola con espacios"}` | 400 · `… (sin espacios) …` |
| `POST /admin/queues {"nombre":"amq.mia"}` | 400 · `… no puede empezar con 'amq.' …` |
| `POST /admin/exchanges {"nombre":"x","tipo":"cuantico"}` | 400 · `el tipo debe ser uno de: direct, topic, fanout, headers` |
| `DELETE /admin/queues/cola!mala` | 400 · el nombre de la ruta también se valida |
| `POST /admin/queues {"nombre":"q.cmd.email"}` | **409** · el broker rechaza redeclarar la cola existente con otros argumentos (verificado: `inequivalent arg 'x-dead-letter-exchange'`) y la cola sigue intacta |

**Qué decir.** "Un nombre inválido nunca llega al broker: se rechaza con 400 diciendo qué campo y por qué. Y si
la configuración choca con lo que ya existe, el 409 trae el motivo exacto del broker."

---

## Indicador 9 — Matriz de funcionalidades (20%)

**100% pide:** **90% o más** de las funcionalidades del alcance implementadas y operativas; cada una con estado,
componente responsable y evidencia verificable en el repositorio. Las parciales no cuentan. Los tramos son: 72%
de funcionalidades → 80% del puntaje; 54% → 60%; 27% → 30%.

**Dónde mirar** — [Cumplimiento-Prueba2.md, *Indicador 9*](Cumplimiento-Prueba2.md#indicador-9--matriz-de-funcionalidades-20)
- 5 tablas (módulos, pantallas, flujos RabbitMQ, tópicos Kafka y exigencias de EP3), todas con **Estado**,
  **Componente** y **Evidencia**.
- Conteo: **18 / 20 = 90%** → tramo 100. Los 2 que faltan son tópicos Kafka, fuera del foco de esta prueba
  (100% RabbitMQ) y declarados como brecha.
- Enlazada desde el README de los 9 repos de código e infra (H2).

**Cómo validar los 3 flujos RabbitMQ** (las filas que más se discuten)
- *En la app* (necesita I2): Vecino crea un trámite → `[CERTIFICADO] … comprobante de ingreso`. Funcionario lo
  admite → `[EMAIL/PUSH] … ADMITIDO`. Lo pasa a terreno → `[EMAIL/PUSH] … VISITA_AGENDADA` + `[CUADRILLA]` (por
  **topic**). Lo resuelve → `[EMAIL/PUSH] … RESUELTO` (por **topic**) + `[CERTIFICADO] … certificado de resolucion`.
- *Sin token* (verificado el 18-10): publicar desde la Management UI

| Exchange | Routing key | Payload |
|---|---|---|
| `cmd.topic` | `email.resuelto` | `{"type":"email.send","eventId":"m-1","data":{"tramiteId":1,"tipoNotificacion":"RESUELTO"}}` |
| `cmd.topic` | `crew.visita.agendada` | `{"type":"crew.ticket","eventId":"m-2","data":{"tramiteId":1,"direccion":"Calle 1"}}` |
| `cmd.direct` | `certificate.gen` | `{"type":"certificate.gen","eventId":"m-3","data":{"tramiteId":1,"motivo":"certificado de resolucion"}}` |

**Ojo con el margen.** 18/20 es exactamente el 90%. Si en la presentación alguna fila no se logra demostrar, baja a
85% (tramo 80). Por eso el ensayo (H5) tiene que pasar por **todas** las filas `✅`.

---

## Ítems de demo de EP4 relacionados con RabbitMQ

| Ítem | Cómo mostrarlo |
|---|---|
| 8 · 2 nodos en clúster | `docker compose exec rabbitmq1 rabbitmqctl cluster_status` → *Running Nodes*: `rabbit@rabbitmq1`, `rabbit@rabbitmq2`. O la pestaña *Overview* de cualquiera de las dos UI |
| 9 · 3 colas con sus DLQ | *Queues*: 6 colas `quorum` con miembros en los 2 nodos (`docker compose exec rabbitmq1 rabbitmq-queues quorum_status q.cmd.email`) |
| 10 · Exchanges direct y topic | *Exchanges*: `cmd.direct`, `cmd.topic` (con *message rate* > 0 después de publicar `email.resuelto`) y `cmd.dead.dlx` |
| 11 · docker-compose en la nube | `ssh mq` → `cd ~/mq && docker compose ps` (2 servicios `healthy`) y mostrar `mq/compose.yml` + `mq/rabbitmq.conf` (peer discovery `classic_config`) |
| Failover del cliente (C8) | `apps/compose.yml`: `RABBITMQ_ADDRESSES=10.0.1.254:5672,10.0.1.254:5673`. Si preguntan: con 2 nodos la cola quorum **no** tolera la caída de uno (la mayoría Raft es 2); el cliente reconecta, pero la cola queda indisponible. Hacen falta 3 nodos |

## Comandos para correr todos los tests

```bash
cd ms-barriodigital-notify && mvn -B verify        # 11 tests: AckManualTest, CommandListenersTest
cd ms-barriodigital-requests && mvn -B verify      # 24 tests: NotificationPublisherTest, TramiteEventRelayTest…
cd ms-barriodigital-rabbitadmin && mvn -B verify   # 29 tests: AdminApiTest, RabbitAdminServiceTest
```
