# Hallazgos de operación — lo que apareció al correr el sistema de verdad

Hasta el 15-10-2026 el README de `barriodigital-infra` decía textual que los tres
`docker-compose.yml` estaban validados **solo como sintaxis YAML** y que *"nadie los ha
corrido de verdad todavía"*. Ese día se desplegaron y verificaron el clúster de RabbitMQ y el
microservicio administrador. Este documento es **la diferencia entre lo que se suponía y lo
que resultó ser cierto**.

Vale leerlo antes de la presentación: varios de estos puntos son respuestas a preguntas
probables, y cuatro corrigen afirmaciones que otros documentos del repo todavía daban por buenas.

---

## 1. Correcciones a lo que decían los docs

| Dónde | Decía | Es |
|---|---|---|
| `Pendientes-Prueba2.md`, I1 | *"Sin esto `ec2-apps` no puede hacer `docker pull` y no hay despliegue"* | **Falso.** Los 7 packages estaban privados, pero `ec2-apps` tenía un `docker login ghcr.io` guardado y los contenedores se actualizaban igual. I1 era higiene, no bloqueador. El motivo real para cerrarlo es que ese login es un PAT y vive en una máquina |
| `Pendientes-Prueba2.md`, D9 | *"con 6 servicios a 384m ya va en ~3.8 GB: sin esto hay OOM kill en bucle"* | **Falso.** `-Xmx` es el máximo del heap, no una reserva. Medido: 3831 MB totales, **2186 disponibles**, y los 7 contenedores juntos ocupaban ~1079 MB reales. El octavo entró cómodo |
| `Justificacion-Mensajeria.md` §6 | que las colas quorum *"sobreviven a la pregunta ¿y si apagamos el nodo 1?"* | **Falso con 2 nodos.** Corregido en el bloque A6 |
| `AWS-Infraestructura.md` | que el puerto 22 estaba en `0.0.0.0/0` *"temporal hasta el 14-09"* | Se quedó así **un mes**, en un repo público que además publica la EIP. Cerrado el 15-10 |

---

## 2. Colas quorum con 2 nodos: el límite que hay que saber decir

Una cola quorum con 2 réplicas es un grupo Raft de tamaño 2, y **la mayoría es 2**. Replica
los mensajes —que es lo que arregla el hallazgo de que el segundo nodo no tenía copia de
nada— pero **no tolera la caída de un nodo**: si cae uno, la cola queda indisponible.

Y no solo para escrituras. El consumo también se detiene, porque confirmar un ack exige
comprometer una entrada en el log de Raft; si el nodo que sobrevive es *follower*, directamente
no hay líder. Tolerar un fallo exige **3 nodos** (mayoría 2 de 3).

> Respuesta honesta a *"¿y si apagamos el nodo 1?"*: los mensajes no se pierden, pero la cola
> no atiende hasta que el nodo vuelva.

---

## 3. El clúster de RabbitMQ

**El `entrypoint` viejo mentía sobre la salud.** Hacía `rabbitmq-server -detached; sleep 10;
join_cluster; tail -f /dev/null`, dejando `tail` como PID 1: si el broker moría, el contenedor
seguía reportando `Up` y `restart: always` nunca disparaba. Lo reemplazó `rabbitmq.conf` con
`peer_discovery_backend = classic_config`.

**El nodo 1 virgen quemaba 167 de sus 177 s de arranque buscando al nodo 2**, que por
`depends_on` tiene prohibido existir hasta que el nodo 1 esté `healthy`. Termina bien —se rinde
y sigue como standalone— pero con los defaults (30 intentos) el `start_period` de 60 s vencía
antes, compose abortaba el `up` y **el nodo 2 nunca arrancaba**. Con
`cluster_formation.discovery_retry_limit = 5` el arranque baja a 46 s.

| | Local (PC) | `ec2-mq` (t3.small) |
|---|---|---|
| Nodo 1 | 45,5 s | **20,0 s** |
| Nodo 2 | 11,9 s | 9,6 s |
| Ambos `healthy` | 71 s | ~45 s |

La EC2 arrancó **más rápido** que el PC: el entorno de Docker Desktop sobre WSL2 pesa más que
una t3.small con disco EBS.

**El `rabbitmq.conf` va en `conf.d/`, no como `/etc/rabbitmq/rabbitmq.conf`.** La imagen trae su
propio `conf.d/10-defaults.conf` (que incluye `loopback_users.guest = false`, por eso
`guest/guest` funcionaba remoto) y el entrypoint genera configuración ahí a partir de
`RABBITMQ_DEFAULT_USER/_PASS`. Montando en `20-cluster.conf` los fragmentos se mergean en orden
y nadie pisa a nadie. Verificado en los logs: ambos nodos cargan los dos archivos.

**El watermark de memoria hay que fijarlo.** El default es `0.4` **relativo**: en los 1905 MiB
que ve la t3.small, cada nodo se creería dueño de 762 MiB y entre los dos dejarían al SO sin
aire. Está en `vm_memory_high_watermark.absolute = 512MB`. Si las colas quorum hacen alarmar la
memoria, ese es el número a mover — y la alarma **bloquea publishers en silencio**, por eso
`rabbitmq-diagnostics -q check_local_alarms` está en la verificación del README.

---

## 4. Dos trampas de Windows a Linux

**CRLF.** `core.autocrlf=true` deja CRLF en el working tree y el `scp` se lo lleva tal cual.
YAML lo tolera; el parser de `rabbitmq.conf` **no**: deja el `\r` dentro del valor y declara el
nodo como `rabbit@rabbitmq2\r`. El clúster no forma y el error no apunta a nada. Cerrado con
`.gitattributes` y `* text=auto eol=lf`.

**`scp -r mq mq:~/` con `~/mq` ya existente copia DENTRO** y deja `~/mq/mq`. Pasó dos veces: al
desplegar había un `compose.yml` del 11-09 y una carpeta anidada huérfana, y lo que corría era
el compose viejo. Los tres bloques del README ahora borran el destino antes de copiar.

---

## 5. La caída del 15-10: cómo se rompió y por qué costó encontrarlo

Al regenerar el secret `APPS_ENV`, el contenido se copió **desde la terminal** y se perdieron
las **4 primeras líneas** del archivo: `AAD_ISSUER_URI`, `AAD_API_CLIENT_ID`,
`AAD_REQUIRED_SCOPE` y `CORS_ALLOWED_ORIGINS`. El siguiente `deploy` volcó ese secret a
`apps/.env` y **5 de los 7 servicios entraron en bucle de reinicio** con
`IllegalArgumentException: issuer cannot be empty`.

Tres cosas lo hicieron difícil de ver:

1. **`docker ps` mostraba todo `Up`.** Un contenedor que crashea y reinicia aparece como
   `Up 17 seconds` una y otra vez. El dato que delata es `RestartCount`, no el estado.
2. **`notify` sobrevivió** porque es headless y no usa Entra, así que "algunos funcionan"
   despistaba.
3. **`AAD_REQUIRED_SCOPE` seguía apareciendo con valor** dentro del contenedor, porque en
   `apps/compose.yml` tiene un default `${...:-access_as_user}`. Las otras dos no lo tienen y
   llegaban vacías. Parecía que la config estaba a medias cuando en realidad faltaba entera.

Y un detalle de Spring que explica el síntoma: en `application.yml` el valor es
`${AAD_ISSUER_URI:default}`, pero **el default no aplica cuando la variable existe y está
vacía** — solo cuando no existe. Compose define la variable siempre, así que el default nunca
entró en juego.

**Para la próxima:** volcar el `.env` a un archivo y copiar el archivo entero, nunca desde el
scroll de una terminal. Está escrito en el README de infra.

Efecto colateral de la reparación: los respaldos (`.env.bueno`, `.env.roto-*`) quedaron dentro
de un checkout de un repo **público**. Un `git add -A` ahí habría commiteado las contraseñas de
Oracle. Movidos fuera y cerrado el patrón en `.gitignore` con `apps/.env.*`.

---

## 6. Cerrar el puerto 22 mata el deploy automático

`deploy.yml` usa `appleboy/ssh-action` desde un runner de GitHub, que tiene **IP dinámica**. El
rango de GitHub Actions son miles de CIDR y un Security Group admite 60 reglas: no hay forma de
dejarlo entrar. Al restringir el 22 a `/32` (I4), el workflow dejó de poder conectarse.

La decisión fue **pasar el despliegue de `apps/` a manual**, igual que el de `mq/`:
**ningún indicador de EP3 ni ninguno de los 11 ítems de EP4 menciona CI/CD**, así que la
automatización no rinde puntos. Se quitó el disparo por `push` para que los merges no dejaran
`main` en rojo.

Los puertos **80 y 8080 se quedan en `0.0.0.0/0` a propósito**: los llama el API Gateway, que no
tiene IP fija. La defensa ahí no es el Security Group sino la aplicación — el BFF valida el JWT
por sí mismo, así que golpear 8080 directo **salta el Gateway pero no la autenticación**. Es la
respuesta a *"¿y qué impide saltarse el API Manager?"*.

---

## 7. El microservicio administrador

**`GET /admin/queues` recibe los nombres de las colas.** AMQP 0-9-1 **no tiene operación de
listar colas**: enumerar es del plano HTTP de management (`:15672/api/queues`), no del
protocolo. Como el servicio administra por `AmqpAdmin`, el listado inspecciona las colas que se
le nombren (`?names=a,b`) y sin parámetro usa `barriodigital.rabbitadmin.colas-conocidas`.

**Una cola borrada por la API reaparece cuando `notify` reinicia.** `notify` declara la
topología con `@Bean` y la re-aplica en cada arranque; el administrador la muta en runtime. No
es un bug: *la topología base es código, lo operativo es API*.

**`/actuator/health` da 503 si el broker no responde.** `spring-boot-starter-amqp` agrega su
health indicator y un administrador que no alcanza el broker está DOWN de verdad. Importa si
alguna vez se le pone healthcheck al compose: quedaría `unhealthy` justo cuando más se lo quiere
a mano.

**Arranca en 47 s en `ec2-apps` contra 4,6 s en un PC**, con ocho JVM levantando a la vez en una
t3.medium. Cualquier `start_period` tiene que contemplarlo.

---

## 8. Integrar tres pantallas construidas en paralelo

Las tres se construyeron por separado y cada una compilaba sola. Al integrarlas apareció lo que
ninguna podía ver:

- **Nadie enlazó `/dashboard`**: cada una agregó su propio enlace de navegación, y la landing
  quedaba alcanzable solo escribiendo la URL.
- **El `ng build` del conjunto es la única verificación que vale.** Tres builds verdes por
  separado no dicen nada sobre el merge.
- Los conflictos fueron todos de tipo *"quedarse con ambos"*: imports y una línea de ruta por
  pantalla. Repartir por archivo funcionó.

---

## 9. Evidencia capturada

- **`cluster_status`** con los dos nodos en *Running Nodes*, sin alarmas ni particiones, ambas
  Management UI en 200 y login real con el usuario nuevo (y `guest` devolviendo 401) — ítem 8
  de EP4.
- **40 mensajes de DLQ** exportados con su `x-death` completo (`reason: rejected`, cola
  original, exchange, `eventId`, `traceId`) — ítem H4. Se rescataron con
  `ackmode=ack_requeue_true` antes del `down -v` que aplicó las credenciales nuevas.
- **Topología reconstruida sola**: tras borrar los volúmenes, `notify` redeclaró los 3
  exchanges, las 6 colas y los 9 bindings al conectar. El indicador 2 se sostiene solo.
