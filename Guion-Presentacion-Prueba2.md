# Guion de la presentación — EP4 (Prueba 2)

La [pauta de EP4](Requisitos%20de%20la%20Prueba/Pauta_Presentacion_Prueba2.md) exige **11 ítems de demo** en
**5 a 10 minutos**, con *"orden e hilo conductor"* y *"lenguaje técnico"*. Eso son **~30 a 50 segundos por ítem**:
el tiempo es parte de la pauta, así que esto no se improvisa.

Este guion recorre los 11 ítems **en un solo hilo narrativo** —*un vecino ingresa un trámite y el sistema reacciona*—
en vez de leer la pauta como lista. Un solo flujo suena a sistema; una lista suena a checklist.

**Presupuesto: 8:15 de contenido**, dejando ~1:45 de colchón hasta el máximo de 10 min para preguntas, tropiezos y
el inevitable "se demoró en cargar".

---

## Antes de empezar — bloqueadores

Al 15-10-2026 el clúster MQ, los packages GHCR públicos y el despliegue de `ec2-apps` ya están cerrados y
verificados. Queda un bloqueador, y no es código:

| Bloqueador | Qué es | Bloquea los ítems |
|---|---|---|
| **I2** (A1–A6, A9 de EP1) | Entra ID: App Registrations, App Roles, consent y usuarios con rol asignado | 5, 6 y la demo del administrador (rol `Admin`) |

El ítem 10 ya tiene productor: E1 publica `email.resuelto` y `crew.visita.agendada` a `cmd.topic`.

Ver [Pendientes-Prueba2.md](Pendientes-Prueba2.md). **Hacer un ensayo completo con cronómetro al menos un día
antes**: es ahí donde se descubre qué fila de la matriz no se puede demostrar — y en EP4 esa fila vale 0.

## Preparación: pestañas y terminales abiertos de antemano

No se abre nada durante la presentación. Todo listo y ya cargado:

| # | Qué | Dónde |
|---|---|---|
| 1 | Portal de Entra ID, en `barriodigital-api` → Users and groups | navegador |
| 2 | Consola de AWS, en API Gateway → `barriodigital-api` → Authorizers | navegador |
| 3 | El frontend, **sin sesión iniciada** | https://dnddhzpvgg.execute-api.us-east-1.amazonaws.com |
| 4 | Management UI de RabbitMQ, pestaña *Exchanges* | http://localhost:15672 (usuario de `mq/.env`; `guest` ya da 401) |
| 5 | Terminal A, con sesión en `ec2-apps` | `ssh apps` |
| 6 | Terminal B, con sesión en `ec2-mq` | `ssh mq` |
| 7 | Terminal C, para los `curl` | local |
| 8 | [Cumplimiento-Prueba2.md](Cumplimiento-Prueba2.md), en la sección de la matriz | editor o navegador |

Los dos túneles SSH, levantados antes de empezar:

```bash
ssh -L 15672:127.0.0.1:15672 -L 15673:127.0.0.1:15673 mq
```

Y dejar los logs de `notify` corriendo en Terminal A, porque es la evidencia visual del consumo:

```bash
docker compose -f ~/barriodigital-infra/apps/compose.yml logs -f notify
```

---

## El guion

### 0:00 – 0:30 · Apertura (30 s)

> "BarrioDigital es la plataforma de trámites de una municipalidad con 20 oficinas comunales. Angular al frente,
> seis microservicios Spring Boot detrás de un API Gateway de AWS, Oracle con un esquema por servicio, y **dos
> transportes de mensajería con propósitos distintos**: RabbitMQ para comandos, Kafka para hechos. Lo que voy a
> mostrar es el camino completo de un trámite, desde el login hasta el PDF que se genera en segundo plano."

Mostrar el diagrama de [arquitectura.md](arquitectura.md). **No** enumerar los 6 microservicios uno por uno: se
nombran cuando aparezcan.

### 0:30 – 1:30 · Identidad · ítems 5 y 6 (60 s)

**Pestaña 1 — Entra ID.** Mostrar el tenant, la App Registration `barriodigital-api`, los **4 App Roles**
(Admin, Funcionario, Vecino, Auditor) y los usuarios con rol asignado en *Users and groups*. → **ítem 5**

**Pestaña 3 — el frontend.** Hacer login con Microsoft. El popup de OIDC, y de vuelta en la app la tarjeta de
bienvenida que muestra **usuario, roles y los scopes delegados del token**. → **ítem 6**

> "El frontend usa MSAL v3 con OAuth 2.0 / OpenID Connect. El detalle que vale la pena: los App Roles se asignan
> sobre el service principal de la **API**, no de la SPA, así que viajan en el **access token**, no en el id_token.
> Leyendo `idTokenClaims` el rol llegaba siempre vacío."

Ese detalle es la mejor señal de que se entiende el modelo y no se copió un tutorial.

### 1:30 – 2:30 · El borde: API Manager · ítems 1, 2, 3, 4 (60 s)

**Pestaña 2 — API Gateway.** La instancia `barriodigital-api` corriendo (→ **ítem 1**), la ruta
`ANY /api/{proxy+}` con su integración HTTP hacia `http://<EIP>:8080/api/{proxy}` (→ **ítem 2**), y el authorizer
JWT `entra-jwt` con su issuer y audience (→ **ítem 4**).

**Terminal C** — las dos caras del authorizer, una al lado de la otra:

```bash
curl -i https://dnddhzpvgg.execute-api.us-east-1.amazonaws.com/api/catalog/procedures
```

> "**401** sin token. El Gateway corta antes de llegar al backend."

Y de vuelta en el frontend, navegar a `/catalog`: carga con datos. → **ítem 3**

> "La misma ruta, con el Bearer que puso el `MsalInterceptor`: **200**. Y el BFF **revalida** el token completo —
> firma contra el JWKS, issuer, audience, expiración, scope y rol — en vez de confiar en el Gateway. Defensa en
> profundidad: el Gateway no tiene IPs fijas, así que el BFF no puede asumir que todo lo que le llega ya pasó por ahí."

### 2:30 – 3:15 · Despliegue en la nube · ítem 7 (45 s)

**Terminal A — `ec2-apps`:**

```bash
docker compose -f ~/barriodigital-infra/apps/compose.yml ps
```

> "Ocho contenedores en `ec2-apps`: frontend, BFF, los cinco de dominio y el administrador de RabbitMQ. Todos con
> `restart: always`, y eso es a propósito: con `unless-stopped`, un contenedor detenido porque se apagó el daemon no
> vuelve cuando el daemon revive, y eso es exactamente el ciclo stop/start de las EC2 del Learner Lab. Nos pasó."

Mostrar la tabla de las 4 EC2 de [AWS-Infraestructura.md](AWS-Infraestructura.md): solo `ec2-apps` tiene IP pública;
`sg-mq`, `sg-kafka` y `sg-db` solo aceptan tráfico desde `sg-apps`. → **ítem 7**

### 3:15 – 4:15 · El trámite y el comando asíncrono (60 s)

**Frontend.** Crear un trámite como Vecino. Señalar el aviso de cupo disponible. Cambiar sesión a Funcionario y
avanzar el estado a **ADMITIDO**, luego a **RESUELTO**.

**Terminal A** — los logs de `notify` reaccionando en vivo:

```
[EMAIL/PUSH] tramite=43 tipo=ADMITIDO -> vecino notificado
[CERTIFICADO] PDF generado para tramite=43 motivo=certificado de resolucion
```

> "Aquí está el punto del encargo. El cambio de estado respondió **201 al instante**: no esperó al email ni al PDF.
> El publish ocurre en un `@TransactionalEventListener` con fase `AFTER_COMMIT` — después de que la transacción
> confirmó y **fuera** del hilo del request. Si el broker está caído, el trámite se guarda igual. Eso **no** era así
> con Kafka: cuando el broker se cayó tras un reinicio del lab, los cambios de estado empezaron a responder 503."

### 4:15 – 5:30 · Topología: exchanges, colas y DLQ · ítems 9 y 10 (75 s)

**Pestaña 4 — Management UI.**

*Exchanges*: `cmd.direct` (**direct**), `cmd.topic` (**topic**) y `cmd.dead.dlx`. → **ítem 10**

> "Dos tipos de exchange porque hacen cosas distintas. `direct` entrega por coincidencia exacta: `email.send` llega
> a `q.cmd.email` y a ningún otro lado — es un comando con destinatario conocido. `topic` entrega por patrón:
> al resolver un trámite publico `email.resuelto` y lo recibe cualquier cola enlazada con `email.*`; al pasar a
> terreno publico `crew.visita.agendada` y calza con `crew.#`. **El productor no sabe quién consume.** Ojo con el
> patrón: `*` es exactamente una palabra y `#` es cero o más."

Mostrar un mensaje real atravesando `cmd.topic`, no solo el binding dibujado: pasar un trámite a `RESUELTO` y
enseñar el *message rate* de `cmd.topic` y el `[EMAIL/PUSH] … tipo=RESUELTO` en el log de `notify`.

*Queues*: las 6 — 3 principales y 3 DLQ. Abrir `q.cmd.email` y mostrar en *Features* el
`x-dead-letter-exchange: cmd.dead.dlx`. → **ítem 9**

| Cola | DLQ | Binding direct | Binding topic |
|---|---|---|---|
| `q.cmd.email` | `q.cmd.email.dlq` | `email.send` | `email.*` |
| `q.cmd.crew` | `q.cmd.crew.dlq` | `crew.ticket` | `crew.#` |
| `q.cmd.certificate` | `q.cmd.certificate.dlq` | `certificate.gen` | `certificate.*` |

> "Tres flujos, cada uno con su DLQ: 3 exchanges, 6 colas, 9 bindings. Y todo esto está declarado como beans
> `Queue`, `Exchange` y `Binding` en una clase `@Configuration`, con los nombres centralizados en
> `application.yml` — ningún nombre de cola escrito a mano en la lógica de negocio."

### 5:30 – 6:15 · El clúster · ítems 8 y 11 (45 s)

**Terminal B — `ec2-mq`:**

```bash
docker compose ps
docker compose exec rabbitmq1 rabbitmqctl cluster_status
```

Señalar los dos nodos en *Running Nodes*: `rabbit@rabbitmq1` y `rabbit@rabbitmq2`. → **ítems 8 y 11**

> "Dos nodos en clúster, unidos por cookie Erlang compartida y peer discovery por configuración. Las colas son
> **quorum**, no clásicas: eso importa porque una cola clásica vive en un solo nodo y el clúster compartiría
> metadata pero no mensajes — el segundo nodo no tendría copia de nada. El mirroring clásico está deprecado en 3.13,
> así que quorum es la única opción con futuro."

Si hay tiempo y valentía: la Management UI del nodo 2 en `:15673`, mostrando el mismo estado.

### 6:15 – 7:15 · La demo de falla: ACK, NACK y DLQ (60 s) ← **el indicador de 16%**

Esto es lo que la pauta premia de verdad. Desde la Management UI (*Exchanges → cmd.direct → Publish message*),
routing key `email.send`, *Properties* `content_type` = `application/json`, y **dos** mensajes con `eventId` nuevos:

1. **Permanente** — `{"type":"email.send","eventId":"demo-5","data":{"tramiteId":1}}` (sin `tipoNotificacion`).
2. **Recuperable** — `{"type":"email.send","eventId":"demo-6","data":{"tramiteId":1,"tipoNotificacion":"X","simularFalla":true}}`.

**Terminal A** — el log real (verificado el 18-10):

```
ERROR … AckManual : NACK->DLQ cola=q.cmd.email eventId=demo-5 motivo=payload sin 'tipoNotificacion'
ERROR … AckManual : NACK->requeue cola=q.cmd.email eventId=demo-6 motivo=error recuperable, intento 1/3: …
ERROR … AckManual : NACK->requeue cola=q.cmd.email eventId=demo-6 motivo=error recuperable, intento 2/3: …
ERROR … AckManual : NACK->DLQ cola=q.cmd.email eventId=demo-6 motivo=reintentos agotados (3/3): …
```

**Pestaña 4** — `q.cmd.email.dlq` con **2 mensajes** más. *Get messages* con *Ack mode* = *Nack message requeue
true* (para no sacarlos) y mostrar el header `x-death` (`reason: rejected`, cola y exchange de origen).

`simularFalla` es una falla inyectable: los consumidores simulan el envío con un log y nunca fallan solos, así que
sin ella la ruta de reintentos solo se vería en los tests. Si preguntan, decirlo así.

> "Cada consumidor gestiona el ACK **de forma explícita**: recibe el `Channel` y el `deliveryTag`, y decide. Si
> termina bien, `basicAck`. Si falla por algo **permanente**, como este payload sin `tipoNotificacion`,
> `basicNack` sin requeue: va directo a la DLQ sin gastar reintentos, porque reintentarlo daría el mismo error. Si
> falla por algo **recuperable** —como este proveedor caído— `basicNack` con requeue: se reintenta hasta tres
> entregas, que cuenta el `x-delivery-count` de la cola quorum, y recién ahí va a la DLQ. Y todo queda en el log,
> no solo en la cola."

> "Lo otro que hay aquí es **idempotencia**: RabbitMQ garantiza entrega *at-least-once*, así que un mensaje puede
> llegar dos veces. Cada comando trae un `eventId` y el consumidor lo descarta si ya lo procesó."

### 7:15 – 7:45 · El microservicio administrador (30 s) ← **indicadores 6, 7 y 8**

**Terminal C** — las tres caras de la API, seguidas:

```bash
curl -i -X POST localhost:8086/admin/queues -H 'Content-Type: application/json' -d '{"nombre":""}'
```
→ **400** con mensaje legible: *el nombre de la cola no puede estar vacío*.

```bash
curl -i -X POST localhost:8086/admin/queues -H "Authorization: Bearer $TOKEN_VECINO" -d '...'
```
→ **403**: no tiene rol Admin.

```bash
curl -s -X POST localhost:8086/admin/queues -H "Authorization: Bearer $TOKEN_ADMIN" \
  -H 'Content-Type: application/json' -d '{"nombre":"q.demo","durable":true}'
```
→ **201**, y la cola aparece en la Management UI. Luego `DELETE /admin/queues/q.demo` y desaparece.

> "Es un microservicio **aparte**, y eso es deliberado. `notify` es plano de datos: consume mensajes y no expone
> HTTP. El administrador es plano de control. `DELETE /queues` no puede vivir en el mismo proceso que está
> procesando las notificaciones de los vecinos: si escalo el consumidor a tres réplicas, tendría tres APIs capaces
> de borrar la misma cola. Los controladores llaman a `RabbitAdminService` y no conocen detalles de RabbitMQ."

### 7:45 – 8:15 · Matriz y cierre (30 s)

**Pestaña 8** — la matriz de [Cumplimiento-Prueba2.md](Cumplimiento-Prueba2.md). Recorrer los bloques, no las filas:
módulos, pantallas, flujos de mensajería, cada uno con su estado, componente responsable y evidencia.

> "La matriz declara el alcance de esta prueba —el sistema con su mensajería RabbitMQ: 17 funcionalidades, las 17
> operativas— con el estado de cada una, quién la implementa y dónde está la evidencia. Lo que queda fuera también
> está dicho: los tópicos Kafka de la etapa anterior aparecen listados aparte, sin contar, porque esta etapa es de
> RabbitMQ."

> "El resumen: la mensajería desacoplada funciona **sin haber tocado la lógica de negocio existente**. El servicio
> de trámites no sabe que RabbitMQ existe; publica eventos de dominio y un relay los traduce a comandos."

Esa última frase es el enunciado del encargo devuelto como conclusión.

---

## Preguntas probables

Respuestas de una o dos frases. El desarrollo completo está en
[Justificacion-Mensajeria.md](Justificacion-Mensajeria.md).

| Pregunta | Respuesta |
|---|---|
| **¿Por qué RabbitMQ y Kafka a la vez? ¿No es redundante?** | Transportan cosas distintas: RabbitMQ lleva **comandos** con reintento y DLQ **por mensaje**; Kafka lleva **hechos** con fan-out y replay. Si un email falla, en Kafka habría que frenar la partición entera para reintentarlo. |
| **¿Por qué el administrador es un microservicio aparte?** | Plano de control vs. plano de datos. `DELETE /queues` no vive junto al consumidor que procesa notificaciones reales, y `notify` no expone HTTP: darle un puerto lo convertiría en la superficie más peligrosa del sistema. |
| **¿Quién publica en `cmd.topic`?** | `requests`, al difundir cambios de estado: `email.resuelto` (cae en `email.*`) y `crew.visita.agendada` (key jerárquica, cae en `crew.#`). `*` es una palabra y `#` cero o más. Permite sumar consumidores sin tocar el productor. |
| **Si borras una cola con el admin y reinicia `notify`, ¿qué pasa?** | **Reaparece.** `notify` re-declara la topología base en cada arranque. La regla es: la topología base es código y se versiona; el admin es para operar — crear colas nuevas, purgar, inspeccionar, reparar. |
| **¿Por qué ACK manual y no automático?** | El automático funciona pero oculta la decisión: no distingue un timeout (reintentar) de un payload inválido (no reintentar nunca). Con ACK manual esa decisión es una línea de código visible. |
| **¿El segundo nodo tiene copia de los mensajes?** | Sí, porque las colas son **quorum** (Raft). Con colas clásicas el clúster compartiría solo metadata y el segundo nodo no tendría nada. |
| **¿Qué pasa si se cae el broker?** | El trámite se guarda igual: el publish es `AFTER_COMMIT` y fuera del hilo del request. Se pierde la notificación de esa ejecución, no la operación. |
| **¿Cómo evitan enviar dos veces el mismo email?** | `eventId` único en el envelope + verificación antes de actuar. Hoy es en memoria y por instancia: con réplicas hay que moverlo a una tabla, y está anotado en el código. |
| **¿Por qué el BFF revalida el token si el Gateway ya lo validó?** | El Gateway no tiene IPs fijas, así que el BFF queda expuesto en 8080 y no puede asumir que todo lo que le llega pasó por ahí. Defensa en profundidad. |
| **¿Por qué no usan WebSocket para actualizar la UI?** | El HTTP API Gateway no soporta WebSocket y corta SSE a los 30 s. Por eso hay polling de 10 s, y está comentado en el código. |

---

## Reparto: quién dice qué

**EP4 se califica de forma individual** aunque la presentación sea grupal, y la pauta dice que *"cada estudiante
fundamenta la propuesta desarrollada"*. El reparto histórico de [Pendientes.md](Pendientes.md) separa AWS
(Francisco) de Entra/frontend (Benjamín): **eso es un riesgo aquí**, porque la pauta de esta etapa es 100%
mensajería y el docente puede preguntarle de RabbitMQ a cualquiera de los dos.

| Tramo | Quién habla | Pero **ambos** deben poder responder |
|---|---|---|
| 0:00–0:30 Apertura | cualquiera | la arquitectura general |
| 0:30–1:30 Identidad (5, 6) | Benjamín | por qué los roles van en el access token |
| 1:30–2:30 API Manager (1–4) | Francisco | por qué el BFF revalida el JWT |
| 2:30–3:15 Despliegue (7) | Francisco | por qué `restart: always` |
| 3:15–4:15 Trámite y comando | Benjamín | por qué el publish es AFTER_COMMIT |
| 4:15–5:30 Topología (9, 10) | **los dos** | direct vs. topic |
| 5:30–6:15 Clúster (8, 11) | Francisco | por qué quorum y no clásicas |
| 6:15–7:15 Demo de falla | **los dos** | ACK manual, recuperable vs. permanente |
| 7:15–7:45 MS administrador | Francisco | plano de control vs. plano de datos |
| 7:45–8:15 Matriz y cierre | cualquiera | las brechas y por qué se decidieron |

**Regla:** presenta quien construyó, porque en una demo cronometrada no hay margen para dudar. La revisión
cruzada de PR es lo que permite que el otro **responda preguntas** sobre esa parte.

Las cuatro filas en negrita son **no negociables para los dos**: concentran los indicadores de 16% y 24%, y son el
tema de la etapa.
