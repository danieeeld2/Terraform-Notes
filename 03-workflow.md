# 3. Core Terraform workflow

| Subtema | Descripción | Cubierto |
|---|---|---|
| 3a | Describe the Terraform workflow | ✅ |
| 3b | Initialize a Terraform working directory | ✅ |
| 3c | Validate a Terraform configuration | ✅ |
| 3d | Generate and review an execution plan for Terraform | ✅ |
| 3e | Apply changes to infrastructure with Terraform | ✅ |
| 3f | Destroy Terraform-managed infrastructure | ✅ |
| 3g | Apply formatting and style adjustments to a configuration | ✅ |

---

## 3a. El workflow de Terraform (write / plan / apply)

Fuente: "Core Terraform Workflow Overview"

Las 3 fases:

1. **Write** — autoría de infraestructura como código.
2. **Plan** — preview de cambios antes de aplicarlos.
3. **Apply** — provisiona infraestructura reproducible.

Este workflow se ve distinto según el contexto: **individuo**, **equipo**, o equipo con **HCP Terraform**.

### Como practicante individual

- **Write**: escribes la config en tu editor, normalmente en un repo versionado (`git init`, `vim main.tf`, `terraform init`). Se recomienda ejecutar `terraform plan` repetidamente mientras escribes, como feedback loop rápido para detectar errores de sintaxis y validar que la config evoluciona como esperas.
- **Plan**: cuando el resultado de iterar te convence, haces commit (`git add` + `git commit`). El comando de revisión final es `terraform apply`, porque **`apply` primero muestra el plan para confirmación** antes de tocar infraestructura.
- **Apply**: tras revisar el plan, confirmas escribiendo `yes` para que Terraform provisione la infraestructura real. Después es común hacer `git push` a un remoto para guardar el trabajo.
- Este workflow es un **loop**: cada cambio nuevo vuelve a empezar por Write.

### Trabajando en equipo

Se añaden pasos a cada fase para evitar pisarse entre compañeros:

- **Write**: cada persona trabaja en su propia **rama** (`git checkout -b ...`) para evitar colisiones y resolver conflictos vía el flujo normal de merge conflicts. A medida que crece el equipo/infra, crecen también las **variables sensibles** necesarias para correr un plan (API keys, certificados...). Para no tener que gestionar esto localmente en cada máquina (carga + riesgo de seguridad), es común migrar las operaciones de Terraform a un entorno de **CI compartido**. Esto alarga el ciclo de iteración, por lo que los "speculative plans" dejan de ser tan útiles como feedback loop individual — pero siguen siendo muy útiles **antes de aplicar o mergear** cambios.
- **Plan**: el output del plan es la oportunidad para que el equipo **revise el trabajo de los demás** — hacer preguntas, evaluar riesgos, detectar errores antes de aplicar cambios dañinos. El lugar natural para esto es junto a los **pull requests**: se adjunta o se genera automáticamente (vía CI) un **speculative plan** para que el equipo lo revise junto al PR. Además de revisar si el plan refleja la intención del autor, el equipo también puede decidir **si quiere que el cambio ocurra ahora** (ej. retrasar un merge si el cambio implica downtime, hasta programar una ventana de mantenimiento).
- **Apply**: tras aprobar y mergear el PR, es importante revisar el **plan final concreto** que corre contra la rama compartida y la última versión del state — este plan **puede diferir** del revisado en el PR (por orden de merge, cambios manuales recientes en la infra, etc.). En este punto el equipo se pregunta: ¿se espera disrupción de servicio?, ¿hay partes de alto riesgo?, ¿hay que avisar a alguien? A veces el equipo observa el apply en directo (screen share, o mirando el build log en CI).
- Igual que para el individuo, este workflow es un **loop** que se repite por cada cambio — con distinta frecuencia según el equipo (varias veces a la semana o varias veces al día).

### El workflow mejorado con HCP Terraform

HCP Terraform está diseñado para reforzar este mismo workflow core a escala de equipo/organización:

- **Write**: HCP Terraform ofrece un lugar **centralizado y seguro** para guardar variables de input y el state, y devuelve un feedback loop ágil de speculative plans a los autores de config, mediante la **integración CLI** (bloque `cloud` dentro del bloque `terraform`):

```hcl
terraform {
  cloud {
    organization = "my-org"
    hostname     = "app.terraform.io" # opcional, default app.terraform.io

    workspaces {
      tags = {
        layer  = "networking"
        source = "cli"
      }
    }
  }
}
```

  Tras configurar la integración, basta con una **API key de HCP Terraform** para que cada miembro edite config y corra speculative plans contra la última versión del state, usando las variables remotas.

- **Plan**: HCP Terraform corre el plan **automáticamente al crear el pull request**, mostrando el estado del progreso y, al terminar, si hubo cambios — visible directamente desde la vista del PR. Para revisiones completas, un enlace lleva a la vista detallada del plan en HCP Terraform.
- **Apply**: tras el merge, HCP Terraform presenta el **plan concreto final** al equipo para discutirlo y aprobarlo. Una vez confirmado el apply, muestra el progreso **en directo** a quien quiera verlo.

---

### 💡 Puntos clave para el examen (3a)

- Workflow core = **Write → Plan → Apply**, y es un **loop** (se repite en cada cambio).
- `terraform apply` **siempre** muestra primero el plan y pide confirmación (`yes`) antes de tocar infraestructura real — por eso es el comando de "revisión final".
- En equipo: ramas de VCS + **speculative plans** adjuntos/automáticos en los PRs son el punto de colaboración/revisión clave.
- El plan final en `apply` (tras merge) puede diferir del plan visto en el PR, por cambios en el state/infra entre medias.
- HCP Terraform añade: state y variables centralizados y seguros, ejecución remota de plans, integración con PRs, y visibilidad del apply en tiempo real — todo mediante el bloque `cloud {}` en la config.

## 3b. `terraform init`

Fuente: "terraform init command" (Dependency Lock File ya cubierto en [02-fundamentals.md](02-fundamentals.md), 2a)

### Qué hace

- Inicializa un **working directory** que contiene archivos de configuración Terraform.
- Es el **primer comando** a ejecutar tras escribir una config nueva o clonar una existente desde VCS.
- Es **seguro ejecutarlo varias veces** — nunca borra tu configuración o state existente (puede dar errores en runs posteriores, pero no destruye nada).

### Uso: `terraform init [options]`

Opciones generales (aplican a varios de los pasos de inicialización):

- `-input=true`: pide input si hace falta; con `false`, da error si se requiere input.
- `-lock=false`: desactiva el locking de state durante operaciones relacionadas con el state.
- `-lock-timeout=<duration>`: tiempo que Terraform espera para adquirir un state lock (default `0s` → falla inmediatamente si el lock ya está tomado).
- `-no-color`: desactiva colores en el output.
- `-upgrade`: actualiza módulos y plugins como parte de su instalación (ver detalle abajo).
- `-json`: output en formato JSON legible por máquina.

### Copiar un módulo fuente (`-from-module`)

- Por defecto, `init` asume que el working directory **ya contiene** configuración.
- Con `-from-module=MODULE-SOURCE`, se puede correr `init` contra un **directorio vacío**: copia el módulo dado al directorio destino antes de los demás pasos de inicialización.
- Útil para: (1) checkout rápido de una config desde VCS, o (2) copiar una config de ejemplo como base para una nueva.
- Para uso rutinario, se recomienda mejor hacer el checkout de VCS por separado (con los comandos propios del VCS) y luego correr `init`.

### Backend initialization

- Durante `init`, Terraform consulta la **configuración de backend** del root module y la inicializa.
- Re-ejecutar `init` con un backend ya inicializado **actualiza** el working directory al nuevo backend — mediante `-reconfigure` **o** `-migrate-state`.
  - **`-migrate-state`**: intenta copiar el state existente al nuevo backend (puede pedir confirmación interactiva).
  - **`-force-copy`**: suprime esos prompts respondiendo "yes" automáticamente (activa también `-migrate-state`).
  - **`-reconfigure`**: ignora la configuración existente, **sin migrar** el state.
- `-backend=false`: salta la configuración de backend (usar solo si el directorio ya estaba inicializado con un backend concreto — algunos pasos de init requieren un backend inicializado).
- `-backend-config=...`: para **partial backend configuration**, cuando los settings del backend son dinámicos o sensibles y no se pueden poner estáticamente en el archivo de config.

### Child module installation

- Durante `init`, Terraform busca bloques `module` en la config y descarga el código fuente de esos módulos según su argumento `source`.
- Re-ejecutar `init` con módulos ya instalados solo instala los **módulos nuevos** añadidos desde el último init — **no** actualiza los ya instalados.
- `-upgrade`: fuerza actualizar todos los módulos al código fuente disponible más reciente.
- `-get=false`: salta la instalación de child modules (usar solo si ya estaba inicializado con sus child modules antes).
- Terraform evalúa la mayoría de input variables al crear el plan, así que **no están disponibles durante init**. Cualquier variable referenciada en `source`/`version` de un bloque `module` debe declarar `const = true`.

### Plugin installation

- Durante `init`, Terraform busca referencias directas e indirectas a **providers** en la config e instala sus plugins.
- Para providers publicados en el Terraform Registry (público o de terceros), `init` los encuentra, descarga e instala automáticamente.
- Tras instalar, Terraform escribe la info de los providers seleccionados en el **dependency lock file** — se recomienda commitearlo para que futuros `init` seleccionen exactamente las mismas versiones.
- Opciones relevantes:
  - **`-upgrade`**: actualiza todos los plugins previamente seleccionados a la versión más nueva que cumpla los version constraints, **ignorando** el lock file.
  - `-get-plugins=false`: (obsoleta desde 0.13, eliminada en 0.15 — sustituida por `provider_installation` y `plugin_cache_dir` en la config CLI).
  - `-plugin-dir=PATH`: fuerza instalar plugins **solo** desde el directorio indicado (como si fuera un `filesystem_mirror`) — útil para overrides puntuales (ej. testear un build local de un provider).
  - `-lockfile=MODE`: modo del dependency lock file. Valor válido: **`readonly`** → suprime cambios al lock file pero **verifica checksums** contra lo ya registrado; incompatible con `-upgrade`. Útil si el lock file lo gestiona una herramienta de terceros y quieres controlar explícitamente cuándo cambia.

### `terraform init` en automatización

- En pipelines de CI/CD es habitual orquestar `terraform init` en automatización, para consistencia entre runs (ej. cachear plugins localmente y evitar reinstalar en cada run).

### Directorio de configuración distinto (`-chdir`)

- En versiones antiguas (≤v0.13) se podía pasar un path de directorio en vez de current working directory a `terraform apply` — **deprecado y eliminado en v0.15**.
- Alternativa actual: opción global **`-chdir=<dir>`** (aplica a todos los comandos), hace que Terraform trate ese directorio como el working directory para todo lo que normalmente leería/escribiría.
- Si además se necesita que el subdirectorio `.terraform` se escriba en otro sitio distinto al working directory, se usa la variable de entorno **`TF_DATA_DIR`**.

---

### 💡 Puntos clave para el examen (3b)

- `terraform init` = primer comando tras escribir/clonar config; **idempotente y seguro** de re-ejecutar (nunca borra config/state).
- Hace 3 cosas principalmente: **backend init**, **child module installation**, **plugin (provider) installation**.
- `-upgrade` ignora el dependency lock file y busca la versión más nueva compatible (tanto para módulos como para providers).
- Cambiar de backend requiere `-reconfigure` (sin migrar state) o `-migrate-state` (migra state al nuevo backend).
- `-backend=false` y `-get=false` saltan pasos de inicialización (solo si ya se había inicializado antes).
- `-chdir=<dir>` es la forma moderna (reemplaza el uso legacy de pasar un directorio como argumento) de apuntar Terraform a otro working directory; `TF_DATA_DIR` controla dónde se escribe `.terraform`.

## 3c. `terraform validate`

Fuente: "terraform validate command"

### Qué hace

- Valida los **archivos de configuración de un directorio**: comprueba si la config es **sintácticamente válida** e **internamente consistente**, **independientemente** de las variables proporcionadas o del state existente.
- **No valida** servicios remotos: ni el remote state, ni las APIs de los providers.
- Útil sobre todo para verificación general de **módulos reutilizables** (nombres de atributos correctos, tipos de valor correctos, etc.).
- Es seguro ejecutarlo automáticamente (ej. como check al guardar en un editor, o como test step en CI para un módulo reutilizable).

### Requisito previo

- Necesita un **working directory inicializado**, con los plugins y módulos referenciados ya instalados.
- Para inicializar solo con fines de validación, sin acceder a ningún backend configurado:

```bash
terraform init -backend=false
```

- Si quieres verificar la config **en el contexto de un run concreto** (workspace concreto, valores de variables concretos, etc.) hay que usar `terraform plan` en su lugar — `plan` **incluye una validación implícita**.

### Uso: `terraform validate [options]`

- `-var 'NAME=VALUE'`: fija el valor de una input variable del root module. Se puede repetir para varias variables.
- `-var-file=FILENAME`: fija valores desde un archivo `.tfvars`. Se puede repetir para varios archivos.
- `-json`: output en JSON legible por máquina (desactiva color siempre) — pensado para integraciones (editores de texto, sistemas automatizados).
- `-no-color`: sin colores en el output.

### Formato de salida JSON (`-json`)

Puede que Terraform encuentre un error **antes** de empezar la validación (por tanto no sujeto al formato JSON) — el software externo debe estar preparado para recibir stdout que no sea JSON válido y tratarlo como error genérico.

Objeto JSON de nivel superior:

- **`format_version`** (string): versión del formato de salida (ej. `"1.0"`). Minor version = cambios compatibles hacia atrás; major version = cambios incompatibles.
- **`valid`** (bool): resumen del resultado — `true` si la config es válida, `false` si hay errores.
- **`error_count`** (number): nº de errores detectados. Si `valid=true`, siempre es 0.
- **`warning_count`** (number): nº de warnings (no invalidan la config, pero son avisos a revisar).
- **`diagnostics`** (array de objetos): cada uno describe un error o warning, con:
  - `severity`: `"error"` o `"warning"`.
  - `summary`: descripción corta del problema (como un "heading").
  - `detail`: mensaje opcional con más detalle (puede ser multi-párrafo).
  - `range`: referencia a la parte del código fuente afectada (`filename`, `start`, `end` con `byte`/`line`/`column`) — puede faltar si no aplica.
  - `snippet`: extracto del código fuente relacionado (`context`, `code`, `start_line`, `highlight_start_offset`, `highlight_end_offset`, `values`).
  - `values` dentro de `snippet`: objetos con `traversal` (ej. `var.instance_count`) y `statement`, útiles para identificar qué valor exacto causó el error (especialmente con `for_each` o construcciones similares).

---

### 💡 Puntos clave para el examen (3c)

- `terraform validate` = solo **sintaxis + consistencia interna** de la config — **no** toca remote state ni APIs de providers (a diferencia de `plan`, que sí).
- Requiere `terraform init` previo (al menos `-backend=false`) porque necesita plugins/módulos instalados.
- `terraform plan` hace una validación implícita **en contexto** (con variables/workspace reales) — `validate` es más rápido y genérico, ideal para CI de módulos reutilizables.
- `-json` da salida machine-readable con `valid`, `error_count`, `warning_count`, `diagnostics[]`.

## 3d. `terraform plan` y el grafo de recursos

Fuentes: "Dependency Graph" (Resource Graph), "terraform plan command"

### Dependency Graph (Resource Graph)

- Terraform construye un **grafo de dependencias** y lo usa para operaciones como generar plans y hacer refresh del state. (Tema avanzado — no imprescindible para usar Terraform, pero cae en el examen.)

**Tipos de nodos del grafo:**

- **Resource Node**: representa un único recurso. Si tiene `count`, hay un resource node **por cada count**. Lleva adjunta la config, el diff y el state de ese recurso.
- **Provider Configuration Node**: representa el momento en que se configura completamente un provider (ej. pasarle credenciales de AWS).
- **Resource Meta-Node**: representa un **grupo** de recursos, sin acción propia — es por conveniencia para dependencias y para un grafo más legible. Solo aparece en recursos con `count` > 1.

Se pueden visualizar todos estos nodos con `terraform graph`.

**Construcción del grafo** (pasos secuenciales):

1. Se añaden **resource nodes** según la configuración (con el diff/state adjunto si existe).
2. Se mapean recursos a **provisioners** (si tienen alguno) — esto se hace después de crear todos los resource nodes, para que recursos con el mismo tipo de provisioner puedan compartir su implementación.
3. Se crean **edges** (aristas) a partir de dependencias explícitas (`depends_on`).
4. Si hay state, se añaden recursos **"orphan"** (presentes en el state pero ya no en la config) — nunca tienen config asociada.
5. Se mapean recursos a **providers**: se crean provider configuration nodes y edges de dependencia (el recurso depende de que su provider esté configurado).
6. Se parsean las **interpolaciones** en la config de recursos/providers para determinar dependencias (una referencia a un atributo de otro recurso crea una dependencia).
7. Se crea un **root node** que apunta a todos los recursos, para que el grafo tenga una única raíz (se ignora al recorrerlo).
8. Si hay un diff, los recursos que se van a **destruir** se dividen en **dos nodos**: uno que destruye y otro que crea (si se recrea) — porque el orden de destrucción suele ser distinto al de creación y no se puede representar en un solo nodo.
9. Se **valida** que el grafo no tenga ciclos y tenga una única raíz.

**Recorrido del grafo (walking):**

- Recorrido **depth-first** estándar, y **en paralelo**: un nodo se procesa en cuanto todas sus dependencias ya se han procesado.
- El paralelismo se limita con un **semáforo** para no sobrecargar la máquina — por defecto, hasta **10 nodos concurrentes**.
- Configurable con el flag **`-parallelism`** en `plan`, `apply` y `destroy` — uso avanzado, normalmente no hace falta tocarlo.
- Nota: algunos providers (ej. AWS) gestionan el rate limiting de su API con backoff/retry a nivel de su propio cliente API — `-parallelism` **no** está pensado para solucionar rate limiting directamente.

### `terraform plan`

Genera un **execution plan** que permite previsualizar los cambios que Terraform va a hacer. Por defecto, al crear un plan Terraform:

1. **Lee el estado actual** de los objetos remotos ya existentes, para que el state esté actualizado.
2. **Compara** la config actual con el state previo, anotando diferencias.
3. **Propone** un conjunto de acciones de cambio que, si se aplican, harán que los objetos remotos coincidan con la config.

- `plan` **no ejecuta** los cambios propuestos — solo los muestra.
- Si no hace falta ningún cambio, `terraform plan` reporta que no hay acciones que tomar.
- `terraform apply` (sin plan previo guardado) genera automáticamente un nuevo plan y pide aprobación.
- **`-out=FILE`**: guarda el plan generado en un archivo, que luego se puede pasar a `terraform apply` como argumento extra. Workflow en dos pasos, pensado sobre todo para **automatización**.
- Sin `-out`, se genera un **speculative plan**: describe el efecto sin intención real de aplicarlo — útil en equipos para verificar el efecto de un cambio antes de mandarlo a code review (aunque el plan final tras el merge puede diferir si hubo otros cambios mientras tanto).

### Modos de planificación (Planning Modes)

Disponibles tanto en `plan` como en `apply`. Son **mutuamente excluyentes**.

- **Normal mode** (default): cambia el sistema remoto para que coincida con la config.
- **Destroy mode** (`-destroy`): plan cuyo objetivo es **destruir todos** los objetos remotos existentes, dejando el state vacío. Equivale a `terraform destroy`. Útil para entornos de desarrollo transitorios.
  - Nota: en Terraform ≤v0.15, `-destroy` solo estaba soportado por `plan`, no por `apply` — para aplicar había que usar `terraform destroy`.
- **Refresh-only mode** (`-refresh-only`, disponible desde v0.15.4): plan cuyo único objetivo es **actualizar el state** (y outputs del root module) para reflejar cambios hechos a objetos remotos **fuera** de Terraform (ej. tras responder a un incidente manualmente).

### Opciones de planificación (Planning Options)

Disponibles en `plan` y `apply`:

- **`-invoke=action.<TYPE>.<LABEL>`**: genera plan para invocar una **action** concreta, excluyendo el resto de la config del plan.
- **`-refresh=false`**: desactiva la sincronización por defecto del state con los objetos remotos antes de comprobar cambios de config. Más rápido (menos requests a la API), pero Terraform **ignora cambios externos** → plan potencialmente incompleto/incorrecto. **No se puede combinar** con `-refresh-only`.
- **`-replace=ADDRESS`**: instruye a Terraform a **reemplazar** la instancia del recurso indicado (útil para objetos degradados, patrones de infraestructura inmutable). Repetible para varios recursos. **No compatible con `-destroy`**. Disponible desde v0.15.2 (antes: `terraform taint`).
- **`-target=ADDRESS`**: enfoca la planificación **solo** en las instancias que coincidan con la address y lo que dependa de ellas. **Uso excepcional** (recuperarse de errores, workarounds) — ver "Resource Targeting" abajo.
- **`-var 'NAME=VALUE'`** / **`-var-file=FILENAME`**: fijar input variables (igual que en `validate`).

### Input variables por línea de comandos — detalles de sintaxis

- Cuidado con espacios alrededor del `=` → error (`-var "length = 2"` falla).
- Unix shell: usar comillas simples `'name=value'`. Si el valor lleva comilla simple, hay que escaparla cerrando/reabriendo la comilla.
- Windows: usar **cmd.exe** (no PowerShell — no puede pasar comillas literales correctamente a programas externos) con comillas dobles `"name=value"`.
- Para tipos no primitivos (list, map, set, `any`), hay que escribir una expresión válida del lenguaje Terraform, con el quoting/escaping necesario para el shell. Ej: `-var 'name=["a", "b", "c"]'`.
- Se recomienda usar `-var-file` en vez de `-var` para evitar la complejidad de parsing del shell.

### Resource Targeting (`-target`)

- Permite enfocar Terraform en un **subconjunto de recursos**, usando resource address syntax:
  - Address de una **instancia concreta** (ej. `aws_instance.example[0]` con `count`/`for_each`) → selecciona solo esa instancia.
  - Address de un **recurso completo** → selecciona todas sus instancias.
  - Address de una **instancia de módulo completa** → selecciona todos los recursos de ese módulo y sus child modules.
- Tras seleccionar las instancias objetivo, Terraform **extiende la selección** a todo lo que esas instancias dependan (directa o indirectamente).
- Pensado para **circunstancias excepcionales** (recuperarse de errores, workarounds de limitaciones) — **no recomendado** para uso rutinario, porque puede causar **drift no detectado** y confusión sobre el estado real vs la config.
- Alternativa recomendada a usar `-target` habitualmente: **dividir configuraciones grandes en varias más pequeñas** que se apliquen independientemente, usando **data sources** para acceder a info de recursos creados en otras configuraciones.

### Otras opciones de `terraform plan`

- `-compact-warnings`: muestra warnings en forma compacta (solo el summary), salvo que vayan acompañados de al menos un error.
- **`-detailed-exitcode`**: exit codes más granulares:
  - `0` = éxito, sin cambios (diff vacío)
  - `1` = error
  - `2` = éxito, con cambios (diff no vacío)
- `-generate-config-out=PATH` (experimental): si hay bloques `import` en la config, genera HCL para los recursos importados que aún no estén en la config, y lo escribe en un archivo nuevo (que no debe existir ya).
- `-input=false`: desactiva el prompt de input para variables sin valor asignado — útil en automatización no interactiva.
- `-json`: output JSON legible por máquina (implica `-input=false` → la config no puede tener variables sin asignar).
- `-lock=false` / `-lock-timeout=DURATION`: igual que en `init`.
- `-no-color`: sin colores.
- **`-out=FILENAME`**: guarda el plan generado en un archivo binario opaco, pasable luego a `apply`. Convención típica: `tfplan`. **No usar sufijo `.tf`** (Terraform lo interpretaría como archivo de config). El archivo contiene la **config completa y todos los valores**, incluidos datos sensibles **en texto plano** — tratar como artefacto potencialmente sensible.
- `-parallelism=n`: límite de operaciones concurrentes al recorrer el grafo (default 10).
- (Legacy, solo backend local) `-state`.

### `-chdir` (igual que en `init`)

- El uso legado de pasar un directorio como argumento posicional a `plan`/`apply` está **deprecado (v0.14) y eliminado (v0.15)** → usar la opción global **`-chdir=<dir>`**.

### Entender el output del plan

Símbolos que indican la acción sobre cada recurso:

| Símbolo | Acción | Descripción |
|---|---|---|
| `+` | Create | El recurso no existe aún — se creará |
| `-` | Destroy | Se destruirá este recurso |
| `~` | In-place update | Se actualiza sin destruir/recrear |
| `-/+` | Replace | Se destruye y se vuelve a crear |

Ejemplo de resumen final: `Plan: 2 to add, 1 to change, 2 to destroy.`

- Un resource se marca para **destroy** si ya no está en la configuración (comentario `# (because X is not in configuration)`).
- Un cambio que **"forces replacement"** (ej. cambiar el `ami` de una instancia) provoca `-/+` (destroy + create).

### Ejemplos de uso

```bash
terraform plan -var='env=prod'
terraform plan -var-file='my-vars.tfvars'
terraform plan -invoke='action.aws_lambda_invoke.test'
```

---

### 💡 Puntos clave para el examen (3d)

- `terraform plan`: **lee state actual → compara con config → propone acciones**. No aplica nada.
- Modos: **Normal** (default), **Destroy** (`-destroy`, = `terraform destroy`), **Refresh-only** (`-refresh-only`, solo sincroniza state) — mutuamente excluyentes.
- `-refresh=false` ahorra tiempo pero puede dar un plan **incompleto** al ignorar cambios externos.
- `-replace=ADDRESS` reemplaza un recurso concreto (sucesor de `terraform taint`); incompatible con `-destroy`.
- `-target=ADDRESS` limita el plan a un subconjunto + sus dependencias — **solo para casos excepcionales**, no rutina (riesgo de drift no detectado).
- `-out=FILE` guarda un plan binario ejecutable después con `apply` — **contiene secretos en claro**, tratar como sensible.
- `-detailed-exitcode`: `0`=sin cambios, `1`=error, `2`=cambios pendientes — muy usado en pipelines CI.
- Símbolos del plan: `+` create, `-` destroy, `~` update in-place, `-/+` destroy+recreate (replace).
- **Resource Graph**: nodos = Resource / Provider Configuration / Resource Meta-Node (para `count`); se construye en pasos (resources → provisioners → `depends_on` → orphans → providers → interpolaciones → root node → split destroy/create → validar sin ciclos).
- El grafo se recorre **depth-first y en paralelo**, con hasta **10 nodos concurrentes** por defecto (`-parallelism`).

## 3e. `terraform apply`

Fuente: "terraform apply command"

### Qué hace

- Ejecuta las operaciones **propuestas** en un plan de Terraform.
- Uso: `terraform apply [options] [plan file]`

### Dos modos de uso

**1. Automatic Plan Mode** (sin pasar un plan file):

- Terraform genera automáticamente un nuevo execution plan (como si corrieras `terraform plan`), **te pide aprobación**, y ejecuta las operaciones indicadas.
- Acepta todos los **planning modes** (`-destroy`, `-refresh-only`) y **planning options** (`-replace`, `-target`, `-var`, `-var-file`, etc.) de `terraform plan`, para personalizar el plan generado.
- **`-auto-approve`**: aplica el plan **sin pedir confirmación**.
  - ⚠️ Incluye operaciones destructivas (ej. borrar recursos). Si se usa en automatización, se recomienda correr primero un `plan` y revisarlo.
  - Recomendación: si usas `-auto-approve`, asegúrate de que nadie más pueda cambiar tu infra fuera del workflow de Terraform (para minimizar drift/cambios impredecibles).

**2. Saved Plan Mode** (pasando un **plan file** guardado, ej. de `terraform plan -out=FILE`):

- Terraform ejecuta las operaciones del plan guardado **sin pedir confirmación** (pasar el plan file ya se interpreta como la aprobación).
- Pensado para workflows de **automatización** (two-step: `plan -out` → revisar → `apply <planfile>`).
- Se puede inspeccionar el plan guardado antes de aplicarlo con `terraform show`.
- **No se pueden especificar** planning modes/options adicionales al usar un plan guardado — esas decisiones ya están "congeladas" en el archivo.

### Apply Options (propias del comando `apply`)

- **`-auto-approve`**: salta la aprobación interactiva. Terraform la **ignora** si le pasas un plan file guardado (porque eso ya es la aprobación).
- `-compact-warnings`: igual que en `plan`.
- **`-input=false`**: desactiva **todos** los prompts interactivos — incluida la aprobación del plan. Si no hay forma de aprobar, la operación **falla** (asume que no quieres aplicar). Para no interactivo, ver "Running Terraform in Automation".
- `-invoke=action.<TYPE>.<LABEL>`: invoca una **action** concreta del provider.
- **`-json`**: output JSON. Implica `-input=false` → requiere además `-auto-approve` **o** un plan guardado (si no, no hay forma de aprobar).
- `-lock=false` / `-lock-timeout=DURATION`: igual que en `plan`/`init`.
- `-no-color`.
- `-parallelism=n` (default 10).
- `-replace=resource`: reemplaza una instancia concreta en vez de update/no-op (solo sin plan guardado).
- Todos los **planning modes/options** de `plan` — solo disponibles si **no** hay plan file guardado.
- (Legacy, solo backend local): `-state`, `-state-out`, `-backup`.

### `-chdir`

- Igual que en `init`/`plan`: pasar un directorio como argumento posicional a `apply` está deprecado (v0.14) y eliminado (v0.15) → usar `-chdir=<dir>` (global) y `TF_DATA_DIR` si hace falta reubicar `.terraform`.

### Errores durante un `apply`

Si Terraform o un provider encuentra un error durante el apply:

1. **Loguea** el error y lo reporta en consola.
2. **Actualiza el state file** con los cambios que sí se hicieron a los recursos.
3. **Libera el lock** del state.
4. **Termina** la operación de apply.

- La infraestructura puede quedar en un **estado inválido** tras un apply fallido — Terraform **no hace rollback automático** de un apply parcialmente completado.
- Tras resolver el error, hay que **volver a aplicar** la configuración para llevar la infra al estado deseado.

### Ejemplos

```bash
terraform apply -var='env=prod'
terraform apply -var-file='my-vars.tfvars'
terraform apply -invoke='action.aws_lambda_invoke.test'
# con for_each/count en actions:
terraform apply -invoke='action.aws_lambda_invoke.test["first-member-of-a-set"]'
```

---

### 💡 Puntos clave para el examen (3e)

- `apply` sin plan file = genera plan + pide aprobación (salvo `-auto-approve`).
- `apply` con plan file guardado = ejecuta directamente, **sin** poder añadir planning modes/options (ya están fijados en el archivo).
- `-auto-approve` se **ignora** si pasas un plan file (pasar el archivo ya es la aprobación).
- Un apply fallido **no hace rollback**: el state se actualiza con lo que sí se aplicó, y hay que corregir y re-aplicar.
- `-json` requiere `-auto-approve` o un plan guardado (si no, no hay cómo aprobar sin input interactivo).

---

## 3f. `terraform destroy`

Fuente: "terraform destroy command"

### Qué hace

- **Deprovisiona todos los objetos** gestionados por una configuración de Terraform.
- Típicamente no se usa en objetos de producción de larga vida, pero es muy útil para limpiar **infraestructura efímera** de desarrollo.

### Uso: `terraform destroy [options]`

- Es literalmente un **alias de conveniencia** de: `terraform apply -destroy`
- Por eso acepta casi todas las opciones de `apply`, pero:
  - **No acepta** un argumento de plan file.
  - **Fuerza** el modo de planificación "destroy".
- Nota: `-destroy` como opción de `apply` solo existe desde **Terraform v0.15.2**. En versiones anteriores, había que usar `terraform destroy` directamente para conseguir ese efecto.

### Plan especulativo de destroy

Para ver el efecto de destruir **sin ejecutarlo**:

```bash
terraform plan -destroy
```

Corre `plan` en modo destroy, mostrando los cambios propuestos sin aplicarlos.

### Destruir un recurso concreto

Con `-target`, igual que en `plan`/`apply`, se puede destruir un recurso concreto y sus dependencias:

```bash
terraform destroy -target aws_instance.example
```

---

### 💡 Puntos clave para el examen (3f)

- `terraform destroy` ≈ `terraform apply -destroy` (alias de conveniencia).
- `terraform plan -destroy` = preview de qué se destruiría, sin ejecutar nada.
- `-target` también funciona en `destroy` para limitar el alcance (mismas advertencias que en `plan`: uso excepcional).

---

## 3g. `terraform fmt`

Fuente: "terraform fmt command"

### Qué hace

- **Formatea** el contenido de los archivos de configuración Terraform para que coincida con el **formato y estilo canónico**.
- Aplica un subconjunto de las convenciones de estilo del lenguaje Terraform, más ajustes menores de legibilidad.
- Los comandos de Terraform que **generan** configuración producen archivos que ya siguen el estilo de `terraform fmt` — conviene seguir ese mismo estilo en los archivos escritos a mano, por consistencia.
- El formato canónico puede cambiar ligeramente entre versiones de Terraform — se recomienda correr `terraform fmt` proactivamente tras actualizar de versión.
- Nuevas reglas de formato **no se consideran breaking changes**.
- Es una herramienta **intencionadamente opinionada**, **sin opciones de personalización** del estilo en sí — su objetivo es la consistencia entre distintos codebases de Terraform, aunque el estilo elegido no guste a todo el mundo. Si no te convence, puedes optar por no usarlo (o usar una herramienta de formateo de terceros — pero entonces conviene aplicarla también a los archivos autogenerados por Terraform, para mantener consistencia).

### Uso: `terraform fmt [options] [target...]`

- Por defecto, escanea el **directorio actual** en busca de archivos de configuración.
- El `target` puede ser: un directorio, un archivo concreto, o **stdin** (pasando un guion solo `-`).

**Flags:**

| Flag | Descripción |
|---|---|
| `-list=false` | No lista los archivos con inconsistencias de formato |
| `-diff` | Muestra el diff de los cambios de formato |
| `-write=false` | No sobreescribe los archivos (implícito con `-check` o si el input viene de STDIN) |
| `-check` | Comprueba si el input **ya está formateado**. Exit status `0` si está bien formateado; no-cero + lista de archivos mal formateados si no |
| `-no-color` | Sin colores en el output |
| `-recursive` | Procesa también los archivos de **subdirectorios** (por defecto solo el directorio indicado/actual) |

---

### 💡 Puntos clave para el examen (3g)

- `terraform fmt` = formateo automático de estilo canónico HCL — **sin opciones de personalización de estilo**.
- `-check`: modo "verificar sin modificar", útil en CI (exit code no-cero si algo está mal formateado) — típicamente combinado con `-recursive` y `-diff`.
- Por defecto **solo** el directorio actual — hace falta `-recursive` para subdirectorios.
- No confundir con `terraform validate` (sintaxis/consistencia) — `fmt` es puramente **estilo/formato**, no valida nada semánticamente.
