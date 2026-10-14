# Adaptador: GitHub

La definición dice qué artefactos hay, cómo evolucionan y qué servicios existen. Este adaptador dice cómo se representan, se observan, se proyectan y se materializan en GitHub, en el repositorio Git que aloja y en su copia de trabajo local.

## Representación

| Artefacto | En GitHub | Identidad |
| --- | --- | --- |
| Issue | Una issue. Su tipo es la etiqueta `feature`, `bug` o `chore` | Su URL |
| Environment | La rama del caso, registrada en el comentario de control, y la copia de trabajo local en esa rama | El nombre de la rama |
| Plan | Comentarios en la issue que empiezan por `<!-- adw:plan -->`, uno por revisión. El vigente es el más reciente | La URL del comentario |
| Change | La rama del Environment y su PR, cuya descripción incluye `Closes <URL de la issue>` | La rama y la URL del PR |

Para ir del PR a su issue: `gh pr view <URL> --json closingIssuesReferences`. Para ir de la issue a su PR: lee la rama en el comentario de control y usa `gh pr list --repo OWNER/REPO --head <rama> --state all`.

## Eventos

El payload puede venir de un webhook, de GitHub Actions, de la API de eventos o directamente de un humano, en lenguaje natural: «quiero hacer la issue #12», «he añadido un comentario en la issue X». Si viene de un humano, el mensaje es el evento: identifica la issue o el PR que menciona. El trigger del proyecto (`adws/`) envía la foto JSON de una issue abierta y admitida cuyo turno es tuyo: sin `adw:waiting-for-human`, sin stage abierto y sin otro dueño. No dice qué ha cambiado: lo averiguas al reconstruir el estado.

Lo que ocurre en una issue afecta a la Issue y lo que ocurre en un PR, a su Change. Un comentario en la conversación de un PR llega como comentario de una issue que tiene `pull_request`: afecta al Change.

## Comentarios

- Todo comentario del agente empieza por una marca: `<!-- adw:control -->` en el comentario de control, `<!-- adw:plan -->` en los planes y `<!-- adw -->` en el resto. El agente y el humano suelen usar la misma cuenta, así que el autor no sirve para distinguirlos. Un comentario con marca lo ha producido el agente; uno sin ella, el humano.
- En un PR hay tres tipos de comentario y hay que leerlos todos: conversación y revisiones (`gh pr view <URL> --json comments,reviews`) y comentarios en línea (`gh api repos/OWNER/REPO/pulls/N/comments`).
- Si el humano es el autor del PR, GitHub no le deja pedir cambios ni aprobar: su revisión llega siempre como comentarios. Decide si pide cambios por lo que dice, no por el tipo de revisión.

## Control

- **Admisión:** la issue lleva la etiqueta `adw`. Si una petición directa admite la issue, pon la etiqueta.
- **Propiedad:** la asignación de la issue. El dueño es la cuenta de GitHub con la que trabaja el sistema (`@me`), y sin asignar significa libre. Para reclamarla, `gh issue edit <URL> --add-assignee @me` y relee: si hay otro asignado además de ti, quítate con `--remove-assignee @me` y no actúes. Dos máquinas con la misma cuenta cuentan como el mismo dueño. El humano libera el caso desasignando la issue.
- **Turno:** la issue lleva `adw:waiting-for-human` mientras el turno es del humano. Al cederlo, publica primero un comentario que diga qué esperas del humano y que quite esa etiqueta cuando termine, y después pon la etiqueta. El humano devuelve el turno quitándola. Si lo devuelve con una petición directa, quítala tú.
- **Comentario de control:** un único comentario en la issue que empieza por `<!-- adw:control -->`. Recoge la rama del caso y su base. Lo crea el dueño la primera vez que lo necesita y después lo edita en el sitio, sin publicar otro. Solo lo escribe el dueño, así que no hay escrituras concurrentes.

## Estado

**Proyección.** Se proyecta en la issue con etiquetas. Todas van en kebab-case: minúsculas y palabras separadas por guiones.

- Cada stage abierto es `stage:` más el nombre del stage (`ReviewResponse` → `stage:review-response`). Es también el candado del caso.
- Cada milestone alcanzado es el artefacto, dos puntos y el milestone (`changes_requested` del Change → `change:changes-requested`).
- El tipo de la issue es `feature`, `bug` o `chore`. Si la issue ya trae una, respétala.
- Los milestones terminales se observan sin etiqueta: `issue:closed` es la issue cerrada, `change:merged` es el PR integrado y `change:discarded`, el PR cerrado sin integrar.
- `environment:ready` y `environment:released` no se proyectan: describen la máquina, no el caso. Se observan en local: la rama actual, que esté al día con `origin` y si `http://localhost:3004/healthz` responde.

**Catálogo.** Son las únicas etiquetas que usa el adaptador. No inventes otras ni cambies su forma.

| Familia | Etiquetas |
| --- | --- |
| Control | `adw`, `adw:waiting-for-human` |
| Tipo | `feature`, `bug`, `chore` |
| Stages | `stage:clarification`, `stage:setup`, `stage:teardown`, `stage:planning`, `stage:implementation`, `stage:publication`, `stage:review-response` |
| Issue | `issue:clarified`, `issue:info-requested` |
| Plan | `plan:ready`, `plan:question-raised` |
| Change | `change:implemented`, `change:published`, `change:review-answered`, `change:changes-requested` |
| Fallos | `issue:failed`, `environment:failed`, `plan:failed`, `change:failed` |

**Observación.** Las etiquetas son el último estado proyectado. Contrástalas con lo observable: comentarios humanos posteriores al último comentario del agente, comentarios de plan, commits sin subir, si el PR contiene la cabeza de la rama.

**Escritura.** Cambia etiquetas con `gh issue edit <URL> --add-label … --remove-label …`, sin reemplazar el resto. Si una etiqueta no existe, créala con `gh label create`. Al abrir un stage, pon su etiqueta antes de nada. En lo demás, publica primero y cambia las etiquetas al final: quien vea una etiqueta nueva ya encontrará en GitHub lo que anuncia.

**Reintentos.** Si un stage falla, proyecta `<artifact>:failed`, comenta el fallo y cede el turno. El humano reintenta devolviéndolo: al quitar `adw:waiting-for-human`, se invalida `failed` y quitas su etiqueta. Si una ejecución se interrumpió y dejó una etiqueta `stage:*`, el humano la quita para reintentar.

## Servicios

El plan vive en la issue, no en la rama. `plans/issue-N.md` es solo una copia de trabajo para las skills: nunca se commitea.

| Servicio | En GitHub |
| --- | --- |
| `clarify` | Publica las preguntas como comentario en la issue. Si la clasificas, pon la etiqueta del tipo. |
| `setup_environment` | Si el comentario de control no tiene rama, asígnala como `<tipo>/issue-N-<slug>` (el slug sale del título, en minúsculas y con guiones), con la rama por defecto del repositorio como base (`gh repo view OWNER/REPO --json defaultBranchRef`), y regístrala. Si la rama no existe en `origin`, `git fetch origin <base>` y `git switch -c <rama> origin/<base>`. Si existe, `git fetch origin <rama>`, `git switch <rama>` y `git merge --ff-only origin/<rama>`. Después ejecuta `env:start`. |
| `teardown_environment` | Ejecuta `env:stop`. |
| `write_plan` | El plan se escribe en `plans/issue-N.md`. Publícalo en la issue como comentario que empiece por `<!-- adw:plan -->`, indicando la revisión y las peticiones que recoge, y borra el fichero. |
| `implement` | Antes de lanzarlo, escribe el plan vigente en `plans/issue-N.md`. Bórralo cuando termine: no forma parte del Change. |
| `publish` | Hace commit con un mensaje convencional según el tipo (`feat:`, `fix:` o `chore:`) y `git push -u origin <rama>`. Si no hay PR, lo abre con `gh pr create --repo OWNER/REPO --base <base> --head <rama>`, con `Closes <URL de la issue>` y el enlace al plan vigente en la descripción, y comenta el enlace en la issue. Si ya existe, el push lo actualiza. |
| `answer_review` | Responde con comentarios en el PR. |
| `report_failure` | Comenta en la issue qué stage ha fallado y por qué, y que el humano quite `adw:waiting-for-human` para reintentar. |

El workflow compuesto se publica como comentario en la issue o en el PR, según dónde haya ocurrido el evento.
