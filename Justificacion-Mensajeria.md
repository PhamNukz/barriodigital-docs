# Justificación de la mensajería — Caso BarrioDigital

Por qué el sistema usa colas, por qué usa **dos** tecnologías de mensajería, y por qué cada decisión de diseño de
esta etapa es como es. Sigue el formato de [Auditoria-Simplificacion.md](Auditoria-Simplificacion.md): cada decisión
trae **el argumento, la alternativa descartada y el costo que aceptamos**. Un beneficio sin su costo declarado es
publicidad, no documentación.

Los argumentos salen del propio código y de cosas que ya nos pasaron en este proyecto, no de un folleto de RabbitMQ.

---

## §1 — Por qué una cola y no una llamada HTTP directa

Cuando un vecino cambia el estado de un trámite hay que enviarle un email, crear el ticket de la cuadrilla y generar
un PDF. La opción obvia sería llamar a esos servicios por HTTP dentro del mismo request. No lo hacemos, por tres
razones concretas.

**1. No bloquear el hilo HTTP.** Generar un PDF y hablar con un proveedor de email es trabajo lento y de terceros.
El publish ocurre en `requests/service/event/TramiteEventRelay.java`, enganchado a
`@TransactionalEventListener(phase = AFTER_COMMIT)`: **después** de que la transacción confirmó y **fuera** de la
respuesta al usuario. El vecino recibe su 201 sin esperar al PDF.

**2. Un broker caído no debe romper la operación** — y tenemos la prueba empírica de lo contrario. El README de
`barriodigital-infra` documenta que tras el ciclo stop/start de las EC2 del Learner Lab, RabbitMQ y Kafka quedaron
abajo, y como `requests` publicaba eventos al cambiar de estado, **los cambios de estado empezaron a responder 503**.
Esa es exactamente la falla que el desacople evita: el comando se encola y se procesa cuando el consumidor vuelva;
no viaja dentro del request del usuario.

**3. El API Gateway no ofrece push.** Está anotado en el propio frontend, en `requests.component.ts:222`:

> `// ponytail: polling cada 10 s en vez de WebSocket/SSE -- HTTP API Gateway no soporta WebSocket y corta SSE a los 30 s.`

Sin canal de push desde el servidor, la cola es el mecanismo de desacople disponible: el trabajo se hace en segundo
plano y la UI lo descubre cuando consulta.

**Costo que aceptamos:** consistencia eventual. Entre el 201 al vecino y el email enviado pasa un tiempo
indeterminado, y hay que diseñar para que un mensaje se pueda procesar **dos veces** (ver §7).

---

## §2 — Por qué RabbitMQ *y además* Kafka (no es redundancia)

La pregunta más probable del docente. La respuesta es que transportan cosas distintas.

| | RabbitMQ | Kafka |
|---|---|---|
| Qué transporta | **Comandos**: "manda este email", "genera este PDF" | **Hechos**: "el trámite 42 pasó a RESUELTO" |
| Modelo | Work queue: un consumidor lo ejecuta y lo confirma | Log inmutable: N consumidores lo leen a su ritmo |
| Quién decide el destino | El productor, vía routing key | Cada consumidor, vía su offset |
| Al fallar | Reintento **por mensaje**, luego DLQ | Avanzar o bloquear la partición |
| En este sistema | `cmd.direct` → `notify` | `requests.events` → `audit` + `report` |

**La prueba de que no son intercambiables** está en el manejo de fallos. Kafka ordena por partición y avanza por
offset: si el email del trámite 42 falla, para reintentarlo hay que **frenar toda la partición** detrás de él, o
saltárselo y perderlo. RabbitMQ confirma mensaje por mensaje: ese email se reintenta 3 veces y cae a
`q.cmd.email.dlq` **sin tocar** los otros comandos en vuelo.

Y al revés: si `audit` y `report` leyeran de una cola RabbitMQ, el primero que consumiera el mensaje lo quitaría del
otro. Necesitan ver **el mismo hecho** cada uno, que es precisamente lo que da un log con offsets independientes.

**Regla de decisión que queda para el futuro:** si el mensaje dice *haz algo* y tiene un solo dueño → RabbitMQ.
Si dice *esto pasó* y lo puede querer cualquiera, ahora o dentro de una semana → Kafka.

**Costo que aceptamos:** dos brokers que operar, dos modelos mentales y dos clústers en el presupuesto del Learner
Lab. Se justifica porque el Caso §1 los exige a ambos y porque los casos de uso son genuinamente distintos.

---

## §3 — Por qué el administrador es un microservicio aparte

El Caso §5 pide *"un microservicio administrador de RabbitMQ"*. El candidato tentador era `ms-barriodigital-notify`,
que ya es dueño de la topología y ya tiene la dependencia AMQP. **No lo hicimos**, y la razón es de diseño antes que
de pauta: `notify` es un worker del **plano de datos** y el administrador es **plano de control**.

### Beneficios de separarlos

**1. Radio de impacto acotado.** `DELETE /queues/{name}` es destructivo y privilegiado. En un proceso aparte, un bug
o un deploy del admin no toca al consumidor que está procesando notificaciones de vecinos reales. Y en el escenario
inverso —`notify` caído— **sigues teniendo capacidad de operar el broker**, justo cuando más la necesitas.

**2. Superficie de red mínima donde corresponde.** Hoy `notify` **no expone HTTP en absoluto**: su `pom.xml` no
tiene `spring-boot-starter-web`, solo habla AMQP. Darle un puerto HTTP con endpoints que crean y borran colas
convertiría al único componente sin superficie de red en el **más peligroso del sistema**. Separado, se gatea con
`hasRole('Admin')` y puede quedar fuera del API Gateway, alcanzable solo dentro de la VPC.

**3. Ciclos de vida independientes.** El consumidor escala con el volumen de mensajes; el admin es de uso ocasional
y humano. Juntos, escalar `notify` a 3 réplicas te deja **3 APIs capaces de borrar la misma cola** sin coordinación
entre ellas.

**4. La tensión declarativo/imperativo queda explícita.** Esto es lo más interesante del diseño. `notify` declara la
topología con `@Bean` y **la re-aplica en cada arranque**, de forma idempotente. El admin la **muta en runtime**.
Son dos modelos en conflicto, y tiene una consecuencia observable:

> Una cola borrada con `DELETE /admin/queues/q.cmd.email` **reaparece** cuando `notify` reinicia.

En un solo proceso esa contradicción queda escondida y sorprende a quien la encuentre. Separados es una frontera
documentable, con una regla que se dice en una frase: **la topología base es código, lo operativo es API.** El admin
sirve para crear colas nuevas, purgar, inspeccionar y reparar; no para redefinir la topología del sistema, que vive
en el repositorio y se versiona.

**5. Testeabilidad.** El admin se prueba mockeando `AmqpAdmin`, sin broker. El consumidor se prueba sin MockMvc.
Hoy `notify` son 5 clases sin `spring-security-test` ni web: meterle controladores arrastra todo ese andamiaje a un
servicio que no lo necesitaba.

### Contraindicaciones de no hacerlo

| Alternativa | Qué se pierde |
|---|---|
| **No implementarlo** | **24 de 100 puntos** de la pauta (indicadores 6, 7 y 8): la mayor concentración de puntos ausentes de la prueba |
| **Meterlo dentro de `notify`** | La pauta premia que exista *"un servicio dedicado"* y que los controladores *"no conozcan detalles internos de RabbitMQ"*. Ponerlo en el dueño de la topología difumina justo esa frontera. Además arrastra web + seguridad a un worker headless, rompiendo el *"sin afectar la lógica existente"* que es el desafío declarado del encargo |
| **Meterlo en el BFF** | Ya tiene web, seguridad y validación, pero mezclaría la administración del broker con el proxy de dominio, y el BFF no tiene dependencia AMQP. Contradice la separación de responsabilidades que el propio proyecto documenta |

### Costo que aceptamos

Un repo más con su `Dockerfile`, `ci.yml`, `build-image.yml` y entrada en `apps/compose.yml`, y una **sexta copia**
de las 4 clases de `cl.duoc.barriodigital.security`, lo que obliga a actualizar
`.github/workflows/verificar-seguridad-duplicada.yml` (hoy clona 5 repos hardcodeados). Costo conocido y acotado: el
proyecto ya decidió a conciencia que duplicar esas 4 clases cuesta menos que publicar y mantener un artefacto Maven.

**No cuesta una EC2 nueva.** Va en `ec2-apps` como 8vo contenedor en el puerto **8086**, igual que `notify`, y alcanza el broker por el
mismo camino que ya usan `requests` y `notify`: AMQP 5672 → `10.0.1.254`. `sg-mq` ya permite 5672 y 15672 desde
`sg-apps`, así que **no hay ningún recurso AWS que crear**. Lo que sí hay que dimensionar es el heap: `ec2-apps` es
t3.medium (4 GB) y ya corre 6 servicios Java a `-Xmx384m`, así que el admin va con `-Xmx192m`.

---

## §4 — Por qué ACK manual y no automático

El ACK automático de Spring confirma el mensaje cuando el método del listener retorna sin excepción, y deja que una
excepción provoque el NACK y el dead-lettering. Funciona. Es lo que el proyecto tiene hoy, y es la práctica que
Spring recomienda por defecto.

**El problema es que oculta la decisión.** Con ACK automático no hay forma de distinguir estos dos casos:

| Caso | Qué corresponde hacer |
|---|---|
| El proveedor de email está caído (timeout) | **Reintentar**: el mensaje es válido, el entorno falló |
| El payload no trae `tramiteId` | **No reintentar nunca**: reintentarlo 1.000 veces dará el mismo error |

Con ACK automático ambos recorren el mismo camino y se reintentan igual. Con ACK manual la decisión es **una línea
de código visible y auditable**: `basicAck` al terminar bien, `basicNack(requeue=true)` para lo recuperable,
`basicNack(requeue=false)` para lo permanente —que va directo a la DLQ sin gastar reintentos.

Es además el único camino al 100% del indicador más pesado de la pauta (16%), que pide textualmente *"Cada consumidor
gestiona ACK de forma explícita"* y *"decisiones claras entre reintentos, NACK y envío a DLQ"*.

**Costo que aceptamos:** el ACK manual es una obligación que el framework deja de cumplir por ti. Un `basicAck`
olvidado en una rama deja el mensaje en estado *unacked* indefinidamente —ni procesado ni devuelto a la cola— hasta
que el consumidor se desconecte. Por eso cada rama lleva su test.

---

## §5 — Por qué direct *y* topic

Tener dos exchanges de comando parece redundante hasta que se mira para qué sirve cada uno.

**`cmd.direct`** entrega por coincidencia exacta de routing key. `email.send` llega a `q.cmd.email` y a ningún otro
lado. Es lo correcto para un **comando puntual con un destinatario conocido**: el productor sabe exactamente quién
debe ejecutarlo.

**`cmd.topic`** entrega por patrón. Los bindings declarados son `email.*`, `crew.#` y `certificate.*`. Eso permite
publicar `email.estado.resuelto` y que **cualquier consumidor interesado en `email.*` lo reciba sin que el productor
sepa que existe**. Es el mecanismo de extensión: mañana se agrega un consumidor de métricas que escucha
`email.estado.*` y `requests` no cambia ni una línea.

**La brecha honesta de hoy:** los 3 bindings topic existen, pero **nadie publica a `cmd.topic`** — el único
productor manda todo a `cmd.direct`. El argumento de extensibilidad es correcto y no se puede demostrar, lo cual en
EP4 equivale a no tenerlo (*"las funcionalidades que no se logran demostrar no se contabilizan"*). Por eso esta
etapa agrega un flujo real que atraviese el topic exchange: sin un mensaje cruzándolo, la respuesta a *"¿y quién
publica ahí?"* es *"nadie"*.

---

## §6 — Por qué colas quorum y no clásicas espejadas

El clúster de 2 nodos de `mq/compose.yml` hoy es **cosmético**: todas las colas se declaran con
`QueueBuilder.durable(...)`, que crea colas **clásicas**. Una cola clásica vive en **un solo nodo**; el clúster
comparte metadata, no mensajes. El segundo nodo es miembro del clúster y no tiene copia de ningún mensaje, así que
si cae el primero no hay nada que tomar el relevo.

La forma histórica de arreglarlo era el *classic queue mirroring* vía política de HA. **Está deprecado en RabbitMQ
3.13 —la versión que usamos— y se elimina en 4.0.** El reemplazo son las **colas quorum**: replicación basada en
Raft, con semántica de consistencia explícita y mayoría para confirmar escrituras. Es la recomendación oficial y la
única opción con futuro.

Importa para la pauta porque EP4 pide demostrar *"dos nodos RabbitMQ configurados y su integración en un clúster"*.
Mostrar `cluster_status` con dos nodos cumple la letra; que una cola tenga réplica en ambos cumple la intención, y
sobrevive a la pregunta *"¿y si apagamos el nodo 1?"*.

**Costo que aceptamos:** las colas quorum tienen más overhead de memoria y disco que las clásicas, y
`ec2-mq` es una t3.small de 2 GB con los dos nodos encima. Para el volumen de un ejercicio de curso es irrelevante,
pero conviene no fingir que es gratis.

---

## §7 — Idempotencia, y su techo declarado

RabbitMQ garantiza entrega *at-least-once*: un mensaje puede llegar **más de una vez** —reintento tras un timeout,
reconexión del consumidor, redelivery tras un NACK. Un sistema que manda emails no puede tratar eso como detalle.

La defensa es un `eventId` único por mensaje, parte del envelope común que exige el Caso §8:

```java
public record CommandEnvelope(String type, String eventId, Instant timestamp,
        String traceId, String correlationId, Map<String, Object> data) {}
```

Cada consumidor consulta `ProcessedEventsGuard.esNuevo(eventId)` antes de actuar y hace short-circuit si ya lo vio.
El `correlationId` lleva el `tramiteId`, lo que permite seguir todos los comandos de un mismo trámite en los logs.

**El techo está declarado en el propio código**, no escondido:

```java
// ponytail: dedupe en memoria de una sola instancia. Si notify-svc escala a
// mas de una replica, esto deja de servir y hay que pasar a una tabla/Redis
private final Set<String> vistos = ConcurrentHashMap.newKeySet();
```

Dos límites reales, dichos antes de que los pregunten: es **por instancia** (con dos réplicas de `notify` cada una
tiene su propio set y un mensaje puede procesarse dos veces), y es **sin eviction** (el set crece durante toda la
vida del proceso). Para una instancia única y un trámite por minuto no es un problema; para producción real la
siguiente parada es una tabla con el `eventId` como clave única, que además sobrevive a los reinicios.

Vale la pena notar que `audit` hace lo mismo del lado Kafka, pero **contra la base de datos**
(`repo.findByEventId(...)` antes de insertar): ahí la idempotencia sí es durable y multi-instancia, porque existía
una tabla donde apoyarla.

---

## Resumen de decisiones

| Decisión | Alternativa descartada | Argumento en una línea |
|---|---|---|
| Cola para notificaciones y PDF | Llamada HTTP en el request | Un broker caído ya nos costó 503 en producción |
| RabbitMQ **y** Kafka | Solo uno de los dos | Comandos necesitan reintento por mensaje; hechos necesitan fan-out con replay |
| Admin en repo propio | Dentro de `notify` | Plano de control separado del plano de datos; `DELETE /queues` no vive junto al consumidor |
| ACK manual | ACK automático | Es la única forma de distinguir error recuperable de permanente |
| direct **y** topic | Solo direct | El topic permite agregar consumidores sin tocar el productor |
| Colas quorum | Mirroring clásico | Deprecado en 3.13; sin replicación el 2º nodo no sirve de nada |
| Dedupe por `eventId` | Confiar en entrega única | RabbitMQ es at-least-once por diseño |
