# barriodigital-docs

Documentación del caso BarrioDigital (DSY1107, entrega EP1). No tiene código —
es el repo de referencia para el resto del proyecto.

## Contenido

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

## Los 9 repos del caso

- `frontend-barriodigital` — Angular + MSAL
- `ms-barriodigital-bff`, `ms-barriodigital-requests`, `ms-barriodigital-catalog`,
  `ms-barriodigital-notify`, `ms-barriodigital-audit`, `ms-barriodigital-report`
  — microservicios Spring Boot
- `barriodigital-infra` — Docker Compose por EC2 (`apps/`, `mq/`, `kafka/`)
- `barriodigital-docs` — este repo
