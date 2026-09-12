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
| 7 | `yagni` | Campo, columna y getter `cuadrillaAsignada` + su campo en el DTO de respuesta | Nada lo escribía nunca: se serializaba siempre `null` y el frontend ni lo declaraba. **Verificado antes de borrar**: 0 de 4 filas tenían valor. Se va con la migración `V2__drop_cuadrilla_asignada.sql` | requests |
| 8 | `delete` | Endpoint `GET /requests/{id}` + su proxy en el BFF | Sin consumidores: el listado ya cubre el caso y aplica la misma regla de acceso (un vecino solo ve lo suyo). Si EP2 necesita vista de detalle, son 6 líneas | requests, bff |

**Neto: -77 líneas de código muerto, 0 dependencias eliminables** (no hay ninguna
dependencia que el stdlib o la plataforma ya cubran; el `pom.xml` de cada
microservicio solo trae starters que se usan).

Aparte de esos cortes se **agregaron** ~90 líneas: la edición en línea del
catálogo, que es funcionalidad nueva y no parte del recorte (ver más abajo por
qué construirla era la forma correcta de cerrar uno de los hallazgos).

## Los tres diferidos, y cómo se cerraron

En la primera pasada estos tres quedaron sin aplicar porque cada uno tenía una
bifurcación (implementar vs. borrar) que no correspondía decidir a ciegas. Se
resolvieron después, cada uno por su lado:

- **`cuadrillaAsignada`** → **borrado**, no diferido. Lo que faltaba era el dato:
  se consultó la base de producción y la columna estaba vacía en las 4 filas, así
  que la migración no pierde nada. Si EP2 implementa la asignación de cuadrillas,
  se vuelve a agregar junto con su lógica, que es cuando corresponde.

- **`PUT /api/catalog/procedures/{id}`** → **el endpoint se queda y ahora se usa**:
  se construyó la edición en línea del catálogo (Admin puede cambiar requisitos,
  cupo diario y si el tipo está activo desde la tabla). Además resuelve un problema
  concreto de operación: subir el cupo diario de un tipo requería tocar la base.
  El hallazgo era correcto — faltaba la UI, no sobraba el endpoint.

- **`GET /api/requests/{id}`** → **borrado**. Aquí el argumento de "superficie REST
  legítima" no se sostuvo: el listado ya devuelve lo mismo y aplica la misma regla
  de autorización, así que el endpoint no aportaba nada que no existiera. Volver a
  agregarlo son 6 líneas el día que haya vista de detalle.

- **Clases de seguridad duplicadas en 5 repos** (`AzureAdProperties`,
  `AudienceValidator`, `JwtRolesConverter`, `ResourceServerSecurityConfig`) —
  **se mantiene la duplicación, pero deja de ser silenciosa.** Deduplicar exige
  publicar un artefacto compartido (no hay monorepo ni registro propio), lo que a
  esta escala agrega más maquinaria de la que ahorra. Pero el riesgo real de la
  duplicación no es el tamaño: es que una copia cambie y las otras no. Ahora el
  workflow [`verificar-seguridad-duplicada.yml`](.github/workflows/verificar-seguridad-duplicada.yml)
  compara las 5 copias de cada clase (en cada push a este repo, semanalmente y a
  demanda) y **falla si divergen o si falta alguna**. Verificado al momento de
  escribir esto: las 4 clases son byte a byte idénticas en los 5 repos.

## Lo que se revisó y salió limpio

- Ninguna clase CSS declarada sin uso en `styles.css`.
- Ninguna variable de `apps/.env.example` sin leer por algún servicio o por el compose.
- Todos los DTO de `audit` y `report` tienen usos reales.
- Ningún import Java sin uso fuera del hallazgo #6.
- Sin interfaces de una sola implementación, sin factories de un solo producto,
  sin wrappers que solo delegan.
