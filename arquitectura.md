# Arquitectura — BarrioDigital

## Flujo de llamadas seguras (sección 6 del caso)

```mermaid
flowchart LR
    U[Vecino / Funcionario / Admin / Auditor] -->|login MSAL| AAD[Azure AD]
    U -->|Bearer access_token| GW[AWS API Gateway - JWT Authorizer]
    GW --> BFF[ms-barriodigital-bff]
    BFF -->|revalida JWT + rol| REQ[ms-barriodigital-requests]
    BFF -->|revalida JWT + rol| CAT[ms-barriodigital-catalog]
    REQ --> ORA1[(Oracle - requests)]
    CAT --> ORA2[(Oracle - catalog)]
```

## Eventos y mensajería (secciones 8 y 9)

```mermaid
flowchart LR
    REQ[ms-barriodigital-requests] -->|comando: email/crew/certificate| DIRECT[cmd.direct]
    DIRECT --> QE[q.cmd.email]
    DIRECT --> QC[q.cmd.crew]
    DIRECT --> QCert[q.cmd.certificate]
    QE -->|falla -> DLQ| DLX[cmd.dead.dlx]
    QC -->|falla -> DLQ| DLX
    QCert -->|falla -> DLQ| DLX
    QE --> NOTIFY[ms-barriodigital-notify]
    QC --> NOTIFY
    QCert --> NOTIFY

    REQ -->|requests.events| KAFKA[(Kafka)]
    KAFKA --> AUDIT[ms-barriodigital-audit]
    KAFKA --> REPORT[ms-barriodigital-report]
    AUDIT --> ORA3[(Oracle - audit)]
    REPORT --> ORA4[(Oracle - report)]
```

## Database per Service

Cada microservicio de dominio (`requests`, `catalog`, `audit`, `report`) tiene su propio esquema Oracle — nadie escribe en la base de otro. `audit` y `report` solo se enteran de lo que pasa en `requests` a través del tópico `requests.events`, nunca leyendo su base directamente.

## Repos

| Repo | Contenido |
|---|---|
| `frontend-barriodigital` | Angular + MSAL |
| `ms-barriodigital-bff` | Valida JWT como el API Gateway, orquesta requests/catalog |
| `ms-barriodigital-requests` | Trámites, máquina de estados, cupos, productor RabbitMQ/Kafka |
| `ms-barriodigital-catalog` | Tipos de trámite y cupos |
| `ms-barriodigital-notify` | Consumidor RabbitMQ (email, cuadrilla, certificado) |
| `ms-barriodigital-audit` | Consumidor Kafka, timeline de auditoría (solo lectura) |
| `ms-barriodigital-report` | Consumidor Kafka, KPIs (solo lectura) |
| `infra` | 3 `docker-compose.yml`: apps / mq / kafka |

## Despliegue AWS (sección 7 del caso)

```mermaid
flowchart LR
    GW[API Gateway<br/>JWT Authorizer] -->|EIP:8080| APPS
    subgraph PUB[Subred pública 10.0.1.0/24]
        APPS[ec2-apps<br/>frontend · bff · requests · catalog<br/>notify · audit · report]
    end
    subgraph PRIV[Subred privada 10.0.2.0/24]
        MQ[ec2-mq<br/>RabbitMQ x2]
        KAFKA[ec2-kafka<br/>ZK x3 + Kafka x3]
        DB[(ec2-db<br/>Oracle Free · 4 esquemas)]
    end
    APPS -->|5672| MQ
    APPS -->|9092-9094| KAFKA
    APPS -->|1521| DB
```

Solo `ec2-apps` tiene IP pública; `sg-mq`, `sg-kafka` y `sg-db` aceptan tráfico únicamente desde `sg-apps`. El BFF queda expuesto en 8080 porque API Gateway no tiene IPs fijas — por eso el BFF **revalida** el JWT completo en vez de confiar en el Gateway (defensa en profundidad). La alternativa con VPC Link + ALB interno, y por qué no la usamos, está en [AWS-Infraestructura.md](AWS-Infraestructura.md#por-qué-vpc-link-sería-más-seguro-y-por-qué-no-lo-usamos).

## Documentos

| Doc | Qué tiene |
|---|---|
| [Pendientes.md](Pendientes.md) | Backlog único con prioridades, mapeado a la pauta EP1 |
| [Entra-ID-y-JWT.md](Entra-ID-y-JWT.md) | App Registrations, roles, los 3 tokens, qué valida cada capa, gotcha `aud` |
| [AWS-Infraestructura.md](AWS-Infraestructura.md) | VPC, subredes, Security Groups, EC2, Oracle, API Gateway, VPC Link |
| [GitHub-Projects.md](GitHub-Projects.md) | Cómo funciona Projects y cómo lo usamos |
| [CI-CD.md](CI-CD.md) | Workflows listos (CI, imagen a GHCR, deploy SSH), secrets |
| [Requisitos de la Prueba/](Requisitos%20de%20la%20Prueba/) | Caso y pauta originales |
