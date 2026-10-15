# barriodigital-docs

Documentación del caso BarrioDigital (DSY1107). No tiene código — es el repo de
referencia para el resto del proyecto.

- [Hallazgos-Operacion.md](Hallazgos-Operacion.md) — lo que apareció al desplegar y correr el sistema de verdad, y las correcciones a lo que otros docs daban por bueno.

## Prueba 2 — RabbitMQ (EP3 encargo + EP4 presentación)

Etapa actual: incorporar colas, exchanges y DLQs para tareas asíncronas.
Ambas pautas comparten los 9 indicadores; solo difiere el último (EP3 puntúa lo
implementado, EP4 lo demostrado en vivo).

| Archivo | Qué encontrar ahí |
|---|---|
| [`Cumplimiento-Prueba2.md`](Cumplimiento-Prueba2.md) | **Punto por punto de las dos pautas, revisión por capas y la matriz de funcionalidades (indicador de 20%)** |
| [`Justificacion-Mensajeria.md`](Justificacion-Mensajeria.md) | Por qué una cola y no HTTP, por qué RabbitMQ *y* Kafka, por qué el administrador es un MS aparte, por qué ACK manual, por qué colas quorum |
| [`Guion-Presentacion-Prueba2.md`](Guion-Presentacion-Prueba2.md) | Guion cronometrado de la demo (los 11 ítems en 5–10 min), demo de falla a DLQ y preguntas probables con su respuesta |
| [`Pendientes-Prueba2.md`](Pendientes-Prueba2.md) | Backlog de la etapa, con el % de rúbrica que desbloquea cada ítem |

## Contenido general

| Archivo | Qué encontrar ahí |
|---|---|
| [`Cumplimiento-EP1.md`](Cumplimiento-EP1.md) | **Punto por punto de la pauta EP1 y del caso, con la evidencia de dónde está resuelto y las brechas abiertas** |
| [`arquitectura.md`](arquitectura.md) | Visión general de la arquitectura, componentes y por qué se decidió cada uno |
| [`Entra-ID-y-JWT.md`](Entra-ID-y-JWT.md) | Paso a paso de Azure AD (App Registrations, App Roles, gotchas del token) |
| [`AWS-Infraestructura.md`](AWS-Infraestructura.md) | VPC, subredes, Security Groups, EC2, API Gateway — paso a paso de consola |
| [`CI-CD.md`](CI-CD.md) | Workflows de GitHub Actions (`ci.yml`, `build-image.yml`) listos para copiar |
| [`GitHub-Projects.md`](GitHub-Projects.md) | Cómo se armó el GitHub Project de seguimiento |
| [`Pendientes.md`](Pendientes.md) | Checklist completo de tareas del EP1, con dueño y prioridad de cada una |
| [`Auditoria-Simplificacion.md`](Auditoria-Simplificacion.md) | Barrido de sobreingeniería: qué se cortó, qué se dejó a propósito y por qué |

## Los 10 repos del caso

- `frontend-barriodigital` — Angular + MSAL
- `ms-barriodigital-bff`, `ms-barriodigital-requests`, `ms-barriodigital-catalog`,
  `ms-barriodigital-notify`, `ms-barriodigital-audit`, `ms-barriodigital-report`
  — microservicios Spring Boot
- `ms-barriodigital-rabbitadmin` — plano de control de RabbitMQ (⚠️ Prueba 2, pendiente)
- `barriodigital-infra` — Docker Compose por EC2 (`apps/`, `mq/`, `kafka/`)
- `barriodigital-docs` — este repo
