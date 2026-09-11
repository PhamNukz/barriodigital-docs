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

Detalle de cada pieza, roles y estado de avance: [`../README.md`](../README.md).
