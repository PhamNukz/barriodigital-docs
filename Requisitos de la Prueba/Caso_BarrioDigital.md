# Caso BarrioDigital: plataforma para trámites comunales y atención vecinal

## 1. Contexto 
Los vecinos dejan solicitudes en papel o por redes sociales. No hay un número de seguimiento ni evidencia de quién cambió el estado. Un municipio junto a una red de 20 oficinas comunales y juntas de vecinos necesita una plataforma unificada para:

* Ingresar trámites por la web, administrar cupos diarios por tipo de solicitud y coordinar cuadrillas en terreno.
* Notificar al vecino (email/push) y a la cuadrilla (ticket de visita).
* Generar un panel de operaciones en tiempo real (trámites por hora, tiempo de resolución, estados activos).
* Auditar eventos del trámite (quién ingresó, admitió, visitó o resolvió una solicitud).

**Lo que la red exige:**
* Login corporativo con Azure AD (IDaaS).
* Frontend Angular con MSAL y autorización por rol (Admin, Operador del dominio, Cliente del dominio).
* Backend Spring Boot con microservicios detrás de AWS API Gateway, protegido con el JWT de Azure AD.
* Despliegue en AWS EC2 con Docker / Docker Compose.
* Mensajería asíncrona con RabbitMQ (tareas / colas de trabajo) y streaming con Kafka (con Zookeeper) para analítica y auditoría en tiempo real.

## 2. Actores y roles

| Rol | Responsabilidad |
| :--- | :--- |
| **Admin** | Define tipos de trámite, cupos y ve KPIs comunales. |
| **Funcionario (Operador)** | Admite solicitudes, asigna cuadrilla y cierra el trámite. |
| **Vecino (Cliente)** | Ingresa y sigue sus solicitudes. |
| **Auditor** | Consulta el timeline. Solo lectura. Relevante para transparencia. |

## 3. Alcance funcional mínimo

| Módulo | Descripción | Actores | Reglas clave |
| :--- | :--- | :--- | :--- |
| **Gestión de trámites** | CRUD de trámites y cambio de estado (INGRESADO → ADMITIDO → EN_GESTIÓN → EN_TERRENO → RESUELTO / RECHAZADO) | Vecino, Funcionario | No se puede pasar a EN_TERRENO sin ADMITIR |
| **Catálogo** | CRUD de tipos de trámite y cupos diarios disponibles | Admin | El cupo diario disminuye al admitir el trámite |
| **Notificaciones** | Email/push al vecino y ticket de visita a la cuadrilla | Funcionario, Vecino | Envío asíncrono (cola) |
| **Reportería** | Panel de KPIs: trámites por hora, tiempo de resolución*, estados activos | Admin | Datos por streaming (Kafka) sin bloquear el core |
| **Auditoría** | Timeline de eventos del trámite | Auditor | Solo lectura |

## 4. Seguridad e identidad (IDaaS Azure + API Gateway)
* **App Registration “BarrioDigital”**: `clientId`, `redirectUri`, `authority = https://login.microsoftonline.com/<TENANT_ID>/`
* **MSAL Angular**: proteger rutas y adjuntar `Bearer <access_token>` a cada request.
* **AWS API Gateway (HTTP API) con JWT Authorizer**: issuer `https://login.microsoftonline.com/<TENANT_ID>/v2.0` y audiences `api://<API_CLIENT_ID>`.
* **Spring Security**: validar el JWT con `security.oauth2.resourceserver.jwt.issuer-uri` y comprobar que el rol puede usar el endpoint llamado.

## 5. Microservicios de dominio

| Servicio | Dominio | DB | Responsabilidad | Exposición |
| :--- | :--- | :--- | :--- | :--- |
| **ms-barriodigital-requests** | Trámites | Oracle | CRUD trámites, estados, coordinación de cupos y notificación | `/api/requests/*` |
| **ms-barriodigital-catalog** | Tipos de trámite / cupos | Oracle | CRUD tipos, cupos y requisitos | `/api/catalog/*` |
| **ms-barriodigital-notify** | Notificaciones | sin DB | Procesa envío email/webpush y ticket de cuadrilla vía RabbitMQ | no público (consumidor RabbitMQ) |
| **ms-barriodigital-audit** | Auditoría / timeline | Oracle | Consume Kafka y persiste eventos | `/api/audit/*` (read-only) |
| **ms-barriodigital-report** | KPIs / analytics | Oracle | Agregaciones y endpoints de lectura (consume Kafka) | `/api/report/*` (read-only) |

Debes incluir además **ms-barriodigital-bff** (Spring Boot + Spring Security) como BFF detrás del API Gateway, más un microservicio administrador de RabbitMQ y otro de Kafka, según la pauta de cada evaluación.

### Endpoints esenciales (ejemplos)

**ms-barriodigital-requests**
```http
POST /api/requests (crear trámite)
GET /api/requests/{id}
PUT /api/requests/{id}/status  body: { "status": "INGRESADO|ADMITIDO|EN_GESTIÓN|EN_TERRENO|RESUELTO|RECHAZADO" }
GET /api/requests?status=...&from=...&to=...
```

**ms-barriodigital-catalog**
```http
GET /api/catalog/procedures
POST /api/catalog/procedures
PUT /api/catalog/procedures/{id} (requisitos/cupo)
```

**ms-barriodigital-report**
```http
GET /api/report/kpis?range=last24h
GET /api/report/top-procedures?range=last7d
```

## 6. Pantallas propuestas

| Pantalla | Ruta | Roles | Función |
| :--- | :--- | :--- | :--- |
| **Login** | `/login` | público | MSAL. Botón «Iniciar sesión con Microsoft». |
| **Dashboard** | `/dashboard` | todos los autenticados | Admin: KPIs comunales. Funcionario: cola de admisión y terreno. Vecino: últimos trámites y estado. |
| **Trámites** | `/requests` | Admin, Funcionario, Vecino | Listar, crear (vecino o funcionario) y cambiar estado (funcionario/admin). |
| **Catálogo de trámites** | `/catalog` | Admin, Funcionario | Tipos de solicitud y cupos diarios. |
| **Reportería** | `/reports` | Admin | Trámites por hora, tiempo de resolución, tipos más demandados. |
| **Auditoría** | `/audit` | Admin, Auditor | Trazabilidad del trámite. Filtros: usuario, fechas, tipo de evento. |

*El flujo de llamadas seguras es siempre: JWT → API Gateway → ms-barriodigital-bff → microservicio de dominio.*

## 7. Despliegue (EC2 + Docker Compose)
* **ec2-apps**: requests-svc, catalog-svc, notify-svc, report-svc, audit-svc
* **ec2-mq**: RabbitMQ (clúster de 2 nodos para las evaluaciones) con Management UI.
* **ec2-kafka**: Zookeeper (3 nodos) + Kafka (3 brokers) + Kafka UI.
* Un `compose.yml` para apps, otro para mq y otro para kafka.
* **Security Groups**: abrir solo los puertos necesarios (AMQP/5672, Kafka/9092, HTTP APIs internas).

**Sugerencia de repositorios GitHub**
```text
/frontend-barriodigital (Angular + MSAL)
/ms-barriodigital-bff (Spring Boot, Spring Security)
/ms-barriodigital-requests (Spring Boot)
/ms-barriodigital-catalog (Spring Boot)
/ms-barriodigital-notify (Spring Boot, consumer RabbitMQ)
/ms-barriodigital-report (Spring Boot, consumer Kafka)
/ms-barriodigital-audit (Spring Boot, consumer Kafka)
/infra
  ├── /apps/compose.yml
  ├── /mq/compose.yml
  └── /kafka/compose.yml
/docs/
```

## 8. Topología RabbitMQ (6 colas: 3 flujos + 3 DLQ)

| Cola principal | Propósito | DLQ | Binding direct | Binding topic |
| :--- | :--- | :--- | :--- | :--- |
| **q.cmd.email** | Email/push al vecino (admitido, visita agendada, resuelto) | `q.cmd.email.dlq` | `email.send` | `email.*` |
| **q.cmd.crew** | Ticket de visita a la cuadrilla en terreno | `q.cmd.crew.dlq` | `crew.ticket` | `crew.#` |
| **q.cmd.certificate** | Generación de PDF (comprobante de ingreso o certificado de resolución) | `q.cmd.certificate.dlq` | `certificate.gen` | `certificate.*` |

* **Exchanges**: `cmd.direct` (direct), `cmd.topic` (topic) y `cmd.dead.dlx` (direct, para DLQ).
* **Buenas prácticas**: envelope común (type, eventId, timestamp, traceId, correlationId), ACK/NACK explícitos, idempotencia y métricas de tasa de DLQ.

## 9. Topología Kafka

| Tópico | Particiones | Réplicas | Política | Retención | Propósito |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **requests.events** | 3 | 3 | delete | 3–7 días | Fuente de verdad de eventos de la trámite. Alimenta reportería y auditoría. |
| **audit.timeline** | 3 | 3 | compact,delete | 14–30 días | Historial quién / qué / cuándo / desde dónde. |
| **\*.DLT** (por consumidor) | 3 | 3 | delete | 7–14 días | Mensajes que fallaron tras N reintentos, con metadatos de error. |
