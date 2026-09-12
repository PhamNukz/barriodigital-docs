# Auditoría de simplificación — 2026-09-12

Barrido de todo el árbol (9 repos, ~3.800 líneas de código propio) buscando
**sobreingeniería**: código muerto, abstracciones con una sola implementación,
flexibilidad especulativa y configuración que nadie lee.

Fuera de alcance de esta pasada: bugs de correctitud, seguridad y rendimiento
(esos van en revisión aparte — los de esta sesión están registrados en los
commits de cada repo).

## Método

- Cada método público contrastado contra sus referencias en **todos** los repos.
- Cada endpoint del BFF contrastado contra lo que realmente llama el frontend
  (el BFF es el único cliente).
- Métodos de repositorio, beans, DTOs, clases CSS, variables de `.env.example`
  y campos de `environment.ts` contrastados contra sus usos.

## Hallazgos aplicados

| # | Tag | Qué se cortó | Por qué | Repo |
|---|-----|--------------|---------|------|
| 1 | `delete` | Endpoint `GET /requests/tipos/{tipoId}/cupo` + método de servicio + proxy del BFF + su test | Quedó sin consumidores al introducir `GET /requests/cupos`, que entrega lo mismo para todos los tipos en una sola llamada | requests, bff |
| 2 | `delete` | Bean `cmdTopicExchange` (`cmd.topic`) en `RabbitTopologyConfig` | `requests` solo publica a `cmd.direct`; el dueño de la topología de `cmd.topic` es `notify`, que la declara y bindea por su cuenta | requests |
| 3 | `delete` | `rolesDe()` en `auth/roles.ts` | Quedó como envoltorio de `tokenInfoDe()` sin ningún llamador cuando `SessionService` pasó a leer roles y expiración de una sola decodificación | frontend |
| 4 | `delete` | `ReportEventRepository.findAllByTramiteIdOrderByTimestampAsc` | Query derivada declarada y nunca invocada | report |
| 5 | `delete` | Flag `environment.production` (ambos archivos) | Nadie lo lee: `angular.json` reemplaza el archivo completo vía `fileReplacements`, así que el flag no decide nada | frontend |
| 6 | `delete` | `import java.util.Map` sin uso | — | notify |

**Neto: -57 líneas, 0 dependencias eliminables** (no hay ninguna dependencia que
el stdlib o la plataforma ya cubran; el `pom.xml` de cada microservicio solo
trae starters que se usan).

## Hallazgos NO aplicados, y por qué

- **`yagni: cuadrillaAsignada`** (`requests`) — columna, campo, getter y campo del
  DTO de respuesta que **nada escribe nunca**: se serializa siempre `null`, y el
  frontend ni siquiera lo declara. Es especulativo, pero quitarlo exige una
  migración Flyway que borre una columna en producción, y la "cuadrilla" es
  vocabulario del caso (el comando `crew.ticket` de la sección 8 ya existe). El
  costo/riesgo de la migración supera las ~4 líneas que ahorra. Se deja anotado
  para EP2: o se implementa la asignación, o se borra junto con el comando.
- **`GET /api/requests/{id}` y `PUT /api/catalog/procedures/{id}`** — sin llamador
  desde el frontend, pero son superficie REST legítima del caso (ver un trámite
  puntual, editar un tipo de trámite). Lo que falta es la UI, no sobra el
  endpoint. Borrarlos sería quitar funcionalidad pedida.
- **Clases de seguridad duplicadas en 5 repos** (`AzureAdProperties`,
  `AudienceValidator`, `JwtRolesConverter`, `ResourceServerSecurityConfig`) —
  ya registrado como H3 en [Pendientes.md](Pendientes.md). Deduplicar exige
  publicar un artefacto compartido (no hay monorepo ni registro de artefactos
  propio), lo que agrega más maquinaria de la que ahorra a esta escala. Se
  mantiene la duplicación consciente, documentada con un comentario en el
  código: **si se toca una, hay que tocar las cinco.**

## Lo que se revisó y salió limpio

- Ninguna clase CSS declarada sin uso en `styles.css`.
- Ninguna variable de `apps/.env.example` sin leer por algún servicio o por el compose.
- Todos los DTO de `audit` y `report` tienen usos reales.
- Ningún import Java sin uso fuera del hallazgo #6.
- Sin interfaces de una sola implementación, sin factories de un solo producto,
  sin wrappers que solo delegan.
