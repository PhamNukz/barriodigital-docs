# GitHub Projects — cómo usarlo en BarrioDigital

## Qué es (en 5 líneas)

- **Issue** = una tarea, vive *dentro de un repo* (`ms-barriodigital-bff#12`).
- **Milestone** = un hito con fecha, agrupa issues *de un solo repo*.
- **Project (v2)** = un tablero/tabla/roadmap que agrupa issues y PRs **de varios repos** y les agrega campos propios (estado, prioridad, área, sprint). Vive a nivel de usuario u organización, no de repo.

Como tenemos 9 repos, el Project es lo único que nos deja ver todo el trabajo en un solo lugar. Cada ítem de [Pendientes.md](Pendientes.md) se convierte en un issue en el repo que corresponda, y el Project los muestra todos.

Es lo que el profe quiere ver: un Kanban con tarjetas moviéndose de *Todo → In Progress → Done*, cada tarjeta ligada a un issue y ese issue cerrado por un PR.

## Paso a paso

### 1. Crear el proyecto
1. https://github.com/PhamNukz → pestaña **Projects** → *New project*.
2. Template **Board** (o *Team planning*, trae más campos; Board basta).
3. Name `BarrioDigital EP1` → Create.
4. Settings (⚙ arriba a la derecha) → visibilidad **Private** (solo colaboradores) o Public si quieren mostrarlo en la entrega. → *Manage access* → agregar a la pareja como **Admin**.

### 2. Campos personalizados
En la vista, click en `+` al final de las columnas de la tabla (o ⚙ → *Fields*):

| Campo | Tipo | Opciones |
|---|---|---|
| `Status` | ya existe | `Todo`, `In Progress`, `Done` (+ agregar `Blocked`) |
| `Área` | Single select | `Entra`, `AWS`, `GitHub`, `CI/CD`, `Backend`, `Frontend`, `Docs` |
| `Prioridad` | Single select | `P0`, `P1`, `P2` |
| `Repo` | (automático — el issue ya sabe de qué repo es) | — |

Vistas útiles (pestañas arriba, *New view*):
- **Board por Status** (la default).
- **Tabla agrupada por Área**: *Group by → Área*, *Sort by → Prioridad*.
- **Solo P0**: *Filter* → `prioridad:P0 -status:Done`.

### 3. Workflows (automatizaciones)
⚙ → **Workflows**:
- *Item added to project* → Status = `Todo`. (on)
- *Item closed* → Status = `Done`. (on)
- *Pull request merged* → Status = `Done`. (on)
- **Auto-add to project**: *Edit* → elegir cada uno de los 9 repos → filtro `is:issue,pr is:open` → Enable. Así cualquier issue nuevo cae solo al tablero. (Auto-add permite varios repos en el plan free desde 2024; si solo deja uno, agregar los issues a mano con `+ Add item` → pegar la URL del issue.)

### 4. Labels (una vez por repo)
En cada repo → Issues → Labels → crear: `area:entra`, `area:aws`, `area:github`, `area:cicd`, `area:backend`, `area:frontend`, `area:docs`, `P0`, `P1`, `P2`. O con `gh` (ver abajo) en loop.

### 5. Crear los issues desde Pendientes.md
Regla: **1 ítem = 1 issue**, en el repo que dice la columna "repo afectado" (los de Entra/AWS que no tocan código van a `barriodigital-docs` o `barriodigital-infra`). Título = el código + resumen (`A8 — aceptar aud GUID y api://GUID`). Cuerpo = copiar el texto del ítem + link al doc con el paso a paso.

Tip: en la vista Board, `+ Add item` → escribir texto → Enter crea un *draft* (nota sin repo). Después, `⋯ → Convert to issue → elegir repo`. Es la forma más rápida de volcar la lista.

### 6. Flujo diario
1. Tomo una tarjeta → me asigno (*Assignees*) → la muevo a `In Progress`.
2. Creo rama `feat/A8-aud-guid` en el repo.
3. PR con `Closes #<n>` en la descripción → al mergear, el issue se cierra y la tarjeta pasa sola a `Done`.
4. Ítems que no son código (crear VPC, App Registration): cerrar el issue a mano con un comentario + captura de pantalla como evidencia. Esas capturas después sirven para la presentación.

### 7. Milestone
En `barriodigital-docs` → Issues → Milestones → `Entrega EP1` con la fecha de entrega. Asignarlo a los P0. (Los milestones son por repo; si quieren uno "global", el campo `Prioridad` del Project cumple el mismo rol. No hace falta crear el milestone en los 9 repos.)

## Automatizar con `gh` CLI (opcional)

Requiere `gh auth login` con scope `project` (`gh auth refresh -s project,repo`).

```bash
OWNER=PhamNukz
REPOS="frontend-barriodigital ms-barriodigital-bff ms-barriodigital-requests ms-barriodigital-catalog ms-barriodigital-notify ms-barriodigital-audit ms-barriodigital-report barriodigital-infra barriodigital-docs"

# labels en los 9 repos
for r in $REPOS; do
  for l in area:entra area:aws area:github area:cicd area:backend area:frontend area:docs; do
    gh label create "$l" --repo $OWNER/$r --color 0E8A16 --force
  done
  gh label create P0 --repo $OWNER/$r --color B60205 --force
  gh label create P1 --repo $OWNER/$r --color FBCA04 --force
  gh label create P2 --repo $OWNER/$r --color C5DEF5 --force
done

# proyecto
gh project create --owner $OWNER --title "BarrioDigital EP1"
gh project list --owner $OWNER            # anotar el NUMBER
PROJ=1

# un issue de ejemplo y agregarlo al proyecto
URL=$(gh issue create --repo $OWNER/ms-barriodigital-bff \
  --title "A8 — aceptar aud GUID y api://GUID" \
  --label P0 --label area:backend \
  --body "Ver barriodigital-docs/Entra-ID-y-JWT.md#gotcha-aud-v1-vs-v2")
gh project item-add $PROJ --owner $OWNER --url "$URL"
```

Para cargar todos los ítems en lote: hacer un CSV `repo,titulo,labels,body` y un `while read` sobre las dos últimas líneas. No vale la pena scriptear más que eso.

## Errores comunes
- Crear el Project **dentro de un repo** (Repo → Projects → New): funciona, pero solo ve issues de ese repo. Crearlo desde el perfil de usuario.
- Olvidar dar acceso a la pareja al Project: ser collaborator del repo **no** da acceso al Project, son permisos separados.
- Tarjetas draft que nunca se convierten en issue: no tienen PR que las cierre, quedan huérfanas. Convertir siempre.
