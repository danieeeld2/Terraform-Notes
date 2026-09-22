# 8. HCP Terraform

| Subtema | Descripción | Cubierto |
|---|---|---|
| 8a | Use HCP Terraform to create infrastructure | ✅ |
| 8b | Describe HCP Terraform collaboration and governance features | ✅ |
| 8c | Describe how to organize and use HCP Terraform workspaces and projects | ✅ |
| 8d | Configure and use HCP Terraform integration | ✅ |

_(Nota: 8c es tema nuevo en el examen 004)_

Fuentes: "Workspaces", "What is HCP Terraform?", "What is Terraform Enterprise?", "Remote operations", "Explorer for workspace visibility", "HCP Terraform private registry overview", "Change requests overview", "Policy enforcement overview", "Projects Overview", "Health" (Drift detection, Continuous validation), "Teams overview", "Run triggers", "Variable sets", "Use HCP Terraform with the Terraform CLI", "Connect to HCP Terraform", "Migrate Terraform state to HCP Terraform or Terraform Enterprise", "terraform login command"

---

## 8a. Crear infraestructura con HCP Terraform (workspaces, workflow, remote operations)

### Qué es HCP Terraform

- Aplicación que ayuda a equipos a **usar Terraform juntos**: gestiona runs en un entorno consistente y fiable, con acceso fácil a **state y secretos compartidos**, controles de acceso para aprobar cambios, un **private registry** de módulos, controles de **policy** detallados, etc.
- Disponible como servicio hosteado en **app.terraform.io**. Equipos pequeños (≤5 usuarios) pueden usarlo gratis: VCS, variables compartidas, entorno remoto estable, remote state seguro.
- **HCP Terraform Standard Edition**: añade audit logging, continuous validation, drift detection automático.
- (Antes se llamaba "Terraform Cloud" — renombrado a **HCP Terraform** en abril 2024, misma funcionalidad).
- **Terraform Enterprise**: distribución **self-hosted** de HCP Terraform, para organizaciones con necesidades avanzadas de seguridad/compliance — instancia privada con las features avanzadas de HCP Terraform.
- **HCP Europe**: permite gestionar infraestructura con recursos hosteados/gestionados/facturados por separado, para cumplir requisitos de residencia de datos europeos.

### Workspaces — concepto

- Un **workspace** = grupo de recursos de infraestructura gestionados por Terraform. Es el equivalente en HCP Terraform de un **working directory persistente** en local (config + state + variables).

| Componente | Terraform local | HCP Terraform |
|---|---|---|
| Configuración | En disco | Repo VCS enlazado, o subida periódica vía API/CLI |
| Variables | `.tfvars`, args CLI, o env del shell | En el workspace |
| State | En disco o backend remoto | En el workspace |
| Credenciales/secretos | Env del shell o prompts | En el workspace, como variables sensibles |

- Datos adicionales que guarda cada workspace:
  - **State versions**: backups de states previos (historial de cambios, recuperación ante problemas).
  - **Run history**: registro de toda la actividad de runs (resúmenes, logs, referencia al cambio que originó el run, comentarios).
- Cada workspace muestra un **resource count** (recursos en su state — incluye managed resources y data sources).

### Runs remotos (repaso de contexto)

- Con **remote operations habilitadas** (default), HCP Terraform ejecuta los runs en **VMs desechables propias**, usando la config/variables/state del workspace.

### HCP Terraform workspaces vs Terraform CLI workspaces — ⚠️ diferencia clave

| | HCP Terraform workspaces | Terraform CLI workspaces |
|---|---|---|
| Obligatoriedad | **Requeridos** — representan todas las colecciones de infra de la organización | Opcionales |
| Rol | Componente central del **RBAC** (permisos por workspace) | Aíslan varios state files dentro del **mismo** working directory |
| Sin ellos | No se puede gestionar nada en HCP Terraform sin al menos 1 workspace | Terraform CLI no requiere crearlos |

### Planificar y organizar workspaces

- Recomendación: **descomponer configuraciones monolíticas** grandes en piezas más pequeñas, cada una en su propio workspace, con permisos/responsabilidades delegadas. Ej: separar `networking-prod`, `app1-prod`, `monitoring-prod`, con equipos distintos gestionando cada uno.
- Beneficios: permite trabajar **en paralelo** (como microservicios), y facilita **reutilizar** configuraciones para otros entornos (`app1-dev`, etc.).
- En Terraform Enterprise, los admins pueden limitar el **nº máximo de workspaces** por organización.

### Crear workspaces e importar infraestructura existente

- Se crean vía: **UI de HCP Terraform**, **Workspaces API**, o la **CLI integration**.
- Se puede **buscar infraestructura existente** e importarla al workspace para gestionarla con Terraform.

---

### 💡 Puntos clave para el examen (8a)

- Un workspace de HCP Terraform = equivalente a un working directory local persistente (config + state + variables + secretos).
- **HCP Terraform workspaces son obligatorios** (y son la unidad de RBAC); los **Terraform CLI workspaces son opcionales** y sirven para aislar states dentro del mismo directorio.
- Se recomienda **descomponer configs monolíticas** en varios workspaces pequeños con ownership delegado, en vez de un único workspace gigante.
- HCP Terraform Standard añade: audit logging, continuous validation, drift detection automático.
- Terraform Enterprise = versión self-hosted de HCP Terraform.

---

## 8b. Colaboración y gobernanza (Explorer, private registry, change requests, policy enforcement, projects, health, teams)

### Explorer (visibilidad de workspaces)

- Superficie información valiosa de toda la organización: **Workspaces, Modules, Providers, Terraform versions**.
- Resultados en tabla, ordenable por columna; clic en un campo con link → vista más específica (ej. nº de módulos de un workspace → lista de esos módulos).
- Requiere permiso de organización: **Organization owner** o **View all workspaces** (o superior).
- Casos de uso predefinidos: top módulos/providers por uso, últimas versiones de Terraform en uso, workspaces sin VCS, **workspaces con checks fallidos** (continuous validation), **workspaces con drift**, runs por status, workspaces más recientemente actualizados, etc.
- **Query builder**: condiciones de filtro custom (campo + operador + valor), combinables con **AND lógico** entre múltiples condiciones.
- Se pueden **guardar vistas** (Save view / Save as) para repetir queries — HCP Terraform **no guarda resultados/histórico**, solo la query (se re-ejecuta al abrir).

### Private Registry

- Funciona como el Terraform Registry público, pero para compartir **providers y módulos dentro de la organización**.
- **Public providers/modules**: hosteados en el registry público; HCP Terraform los puede **sincronizar automáticamente** al registry privado de la organización — permite marcar cuáles están "recomendados" y centralizar su documentación.
- **Private providers/modules**: hosteados solo en el registry privado, disponibles solo para miembros de la organización (en Terraform Enterprise, también compartibles entre organizaciones configuradas para ello).
- Se pueden usar **policies Sentinel** para gestionar su uso (ej. exigir que todos los non-root modules sean del registry privado/público propio, o exigir versiones recientes).

### Change Requests

- Backlog de **action items** registrados en un workspace — permite a admins notificar a equipos cuando un workspace requiere acción (ej. actualizar una versión de módulo deprecada, fixes de seguridad/compliance).
- Se pueden crear directamente desde una query del **Explorer**.
- Al completarse, se pueden **archivar**.
- Disponible en ediciones **Standard y Premium**.

### Policy Enforcement

- Reglas para validar que los plans de Terraform cumplen normas de seguridad/best practices. Aplicable a **workspaces** y **Stacks**.
- 3 frameworks:
  - **Terraform policy** (beta): framework nativo basado en HCL. Soporta workspaces **y** Stacks.
  - **Sentinel**: policy-as-code de HashiCorp. Solo **workspaces**.
  - **OPA** (Open Policy Agent, lenguaje Rego). Solo **workspaces**.
- Casos de uso: verificar compliance de seguridad/best practices, o forzar estándares de workflow (ej. "no deployments en viernes").
- Workflow: **autor** de policies (custom o pre-escritas por HashiCorp, ej. PCI DSS) → **crear un policy set** (archivo de config en un repo VCS, conectado a la organización) → **revisar resultados** (falla según su **enforcement level**, puede parar el run; se puede *override* con permisos adecuados).
- Un policy set solo puede contener policies de **un único framework**, pero se pueden aplicar **varios policy sets** (de frameworks distintos) al mismo workspace/Stack.
- Se pueden aplicar globalmente o a proyectos/workspaces/Stacks/deployments específicos.
- Recomendación: guardar las policies en **VCS** (policy-as-code: estandarización, seguridad, auditabilidad) en vez de solo en la UI.

### Health (Drift Detection + Continuous Validation)

- HCP Terraform puede hacer **health assessments** automáticos en un workspace, comparando la infra real con la config.
- Dos tipos:
  - **Drift detection**: ¿la infra real coincide con la config?
  - **Continuous validation**: ¿las custom conditions (`precondition`/`postcondition`/`check`) siguen pasando tras el provisioning?
- Disponible en ediciones **Standard y Premium**.

**Requisitos del workspace:**

- Terraform **0.15.4+** solo para drift detection; **1.3.0+** para drift detection **y** continuous validation.
- Modo de ejecución **Remote** o **Agent**.
- El último run debe haber sido **exitoso** (si el último acabó en error/cancelado/descartado, se pausan los health assessments hasta el siguiente apply exitoso). Debe haber al menos un apply exitoso (no se evalúan workspaces sin infra real).

**Permisos:**

- Ver el estado de salud → acceso de lectura al workspace.
- Cambiar settings de salud a nivel **organización** → **organization owner**.
- Cambiar settings de salud de un **workspace** o disparar assessments **on-demand** → **admin de ese workspace**.

**Scheduling:**

- Se puede forzar (enforce) a nivel organización (sobreescribe settings por workspace) o dejar que cada workspace **opte** individualmente.
- El siguiente assessment se programa a un intervalo fijo desde el **apply o assessment más reciente** (lo que sea más reciente).
- Un nuevo run **cancela** el assessment en curso (no interfiere con runs).
- Los **on-demand assessments** solo desde la UI, requieren rol admin del workspace; resetean la programación automática.

**Drift detection en detalle:**

- **Configuration drift** ≠ **state drift**. Configuration drift = cambios externos que **invalidan** la config (detectado por drift detection). State drift = cambios externos que **no** la invalidan (se remedia con **refresh-only mode**, no con drift detection).
- Formas de resolver drift:
  - **Overwrite drift**: nuevo plan + apply para revertir la infra a coincidir con la config.
  - **Update Terraform configuration**: modificar la config para incorporar el cambio y evitar que se revierta en el próximo apply.

**Continuous validation en detalle:**

- Evalúa `precondition`, `postcondition` y `check` blocks — se recomienda usar **`check` blocks** para monitorización post-apply.
- ⚠️ **Falsos positivos**: el health assessment genera un **speculative plan**; si la config usa data sources cuyos valores cambian entre el último run y el assessment, el plan especulativo reflejará esos cambios (sin modificar la infra real) — puede dar una alerta de "falla" aunque la infra no haya cambiado todavía. Mitigación: usar data sources **scoped** que consulten la config real del recurso en vez de un "último valor computado".

### Projects

- Organizan **workspaces y Stacks** en grupos, con un **set de permisos separado** por proyecto — más granular que permisos de organización, más amplio que permisos por workspace individual.
- Cada workspace/Stack pertenece a **exactamente un proyecto**. Por defecto, todo va al **"Default Project"** (se puede renombrar, no se puede borrar).
- Permiso "**Manage Workspaces**" → crear/gestionar workspaces (van al Default Project); para crearlos en **otros** proyectos hace falta además "**Manage Projects & Workspaces**" o rol admin de ese proyecto.
- Permiso "**Manage all Projects**" → ver/editar/borrar/asignar acceso a **todos** los proyectos de la organización.
- **Execution mode** por proyecto: por defecto hereda el de la organización, pero se puede sobreescribir: **Organization Default**, **Remote** (runs en infra de HCP Terraform/TFE), **Local** (runs en máquinas propias, solo se guarda/sincroniza state — Stacks **no** soportan Local), **Agent** (vía HCP Terraform agents).

### Teams

- Grupos de usuarios de HCP Terraform dentro de una organización. Un usuario en al menos un team de una org = miembro de esa org.
- Disponible en ediciones **Essentials, Standard, Premium**.
- Las organizaciones **otorgan permisos de workspace a teams** (correr runs, gestionar variables, leer/escribir state, etc.) — los teams solo tienen permisos dentro de **su** organización (un usuario puede pertenecer a varios teams/orgs).
- Gestión también vía API (Teams API, Team Members API, Team Tokens API, Team Access API) o el provider `tfe` (`tfe_team`, `tfe_team_members`, `tfe_team_access`).
- **API tokens de team**: no asociados a un usuario concreto.
- **Owners team**: existe en toda organización; sus miembros = "organization owners". El creador de la org es el primer miembro. En orgs gratuitas, límite de **5 miembros**; en de pago, sin límite. **No se puede** dejar vacío ni borrar el owners team.

---

### 💡 Puntos clave para el examen (8b)

- **Explorer**: vista de organización sobre workspaces/modules/providers/versions, con query builder y vistas guardables — requiere permiso *organization owner* o *view all workspaces*.
- **Private registry**: sincroniza módulos/providers públicos + hostea privados; se puede gobernar su uso con Sentinel.
- **Change requests**: backlog de acciones por workspace, creable desde Explorer (Standard/Premium).
- **Policy enforcement**: 3 frameworks (**Terraform policy** — workspaces+Stacks; **Sentinel** y **OPA** — solo workspaces); un policy set = un solo framework, pero se pueden combinar varios sets.
- **Health**: drift detection (config drift, distinto de state drift) + continuous validation (evalúa pre/postcondition y `check`); requiere Terraform 0.15.4+/1.3.0+ y modo Remote/Agent, y el último run exitoso.
- **Projects**: agrupan workspaces/Stacks, permisos más granulares que org, cada uno con **execution mode** propio (Organization Default/Remote/Local/Agent) — Stacks no soportan Local.
- **Teams**: unidad de RBAC; el **owners team** no puede quedar vacío ni eliminarse; límite de 5 en orgs gratuitas.

---

## 8c. Organización de workspaces y projects (run triggers, variable sets)

### Run Triggers

- Conectan un workspace a **una o más "source workspaces"** de la organización: cuando un run en una source workspace hace **apply exitoso**, se **encola automáticamente** un run en el workspace conectado.
- Hasta **20 source workspaces** por workspace conectado.
- Pensados para workspaces que dependen de info/infra producida por **otros** workspaces (ej. uso de data sources que leen valores que otro workspace puede cambiar) — hace explícita esa dependencia externa.
- Crear/borrar requiere **admin access** al workspace, y **permiso de lectura de runs** en la source workspace.
- **Auto-apply**: los runs disparados por un run trigger **no auto-aplican** por defecto, salvo que se active el setting específico "**Auto-apply run triggers**" (independiente del auto-apply normal del workspace).
- Los runs encolados por un trigger muestran info extra: link a la source workspace y al run que lo disparó.
- Para compartir datos entre workspaces: **`terraform_remote_state`** data source (requiere que el workspace origen permita explícitamente el acceso — remote state sharing), aunque se **recomienda** usar el data source **`tfe_outputs`** del provider TFE/HCP Terraform en su lugar — más seguro, no requiere acceso completo al state para leer solo los outputs.

### Variable Sets

- Colecciones **reutilizables** de variables aplicables a **múltiples workspaces y Stacks** a la vez.
- Propiedad de una **organización** o de un **project** (determina qué permisos hacen falta para gestionarlo).
- ⚠️ **No se evalúan** en workspaces con execution mode **Local**.
- Scope al crear:
  - **Organization-owned**: **Apply globally** (a todos los workspaces existentes y futuros) o **Apply to specific projects/Stacks/workspaces**.
  - **Project-owned**: **Apply to entire project** (todos los workspaces/Stacks actuales y futuros del proyecto) o a workspaces/Stacks específicos.
- Variables dentro del set: tipo (Terraform o environment), opcionalmente **sensitive**, nombre/valor/descripción.
- ⚠️ Error si declaras la **misma key** en múltiples **global** variable sets.
- **Overwrite**: se puede sobreescribir una variable de un set creando una variable **específica del workspace** con la misma key/tipo — se marca con flag **OVERWRITTEN**. Variables de un set también pueden sobreescribir a las de otro set con la misma key aplicado al mismo workspace (ver *variable precedence*).
- **Priority variable sets**: sus valores sobreescriben variables con la misma key definidas en scopes **más específicos** (incluidas las puestas por CLI flags o archivos `.tfvars`/`terraform.tfvars`).
- En **Stacks**: se accede vía bloque **`store`** en el archivo de deployment (`<NAME>.tfdeploy.hcl`), referenciando `store.varset.<VARSET_NAME>.<VARIABLE_NAME>` — en Stacks, la variable precedence normal **no aplica**: si un Stack referencia un varset, siempre usa esos valores.

**Seguridad de variables:**

- HCP Terraform **cifra** los valores de variable con el **transit backend de Vault** antes de guardarlos.
- Las **descripciones** de variable se guardan **en texto plano** — no poner info sensible ahí.
- Recomendación: pasar credenciales como **variables de entorno** en vez de variables Terraform cuando sea posible (los runs reciben el texto completo de todas las variables Terraform, pueden aparecer en logs/state si se envían a un output/parámetro de resource; los mocks de Sentinel también las incluyen).
- Las env vars no se guardan en el state, pero **sí pueden aparecer en logs** si `TF_LOG=TRACE`.
- **Dynamic credentials**: alternativa a credenciales estáticas para algunos providers — credenciales temporales por-run, elimina la necesidad de rotar secretos manualmente.
- Límites: `description` 512 caracteres, `key` 128 caracteres, `value` 256 KB. Soporta texto multi-línea y **HCL** (para variables Terraform, no para env vars) — con el checkbox HCL se pueden meter listas/maps.
- **Sensitive**: marcar como sensible la hace **write-only** — nadie (ni tú) puede ver su valor en la UI ni leerlo vía API. Se puede **actualizar** el valor, pero no otros atributos (para eso, borrar y recrear).

---

### 💡 Puntos clave para el examen (8c)

- **Run triggers**: hasta 20 source workspaces; disparan run en apply exitoso de la fuente; **no auto-aplican** salvo activar "Auto-apply run triggers" explícitamente.
- `tfe_outputs` (recomendado) vs `terraform_remote_state` (requiere acceso completo al state) para leer outputs de otro workspace.
- **Variable sets**: reutilizables entre workspaces/Stacks; owned por org o project; **no se evalúan en modo Local**.
- **Priority variable sets** sobreescriben incluso valores puestos por CLI o `.tfvars`.
- Variables **sensitive** = write-only, nadie puede leerlas después (ni por UI ni por API) — solo se pueden reescribir.
- En Stacks, los variable sets se referencian con el bloque **`store`**, y ahí la variable precedence normal no aplica.

---

## 8d. Integración: CLI con HCP Terraform, migrar state, remote operations, `terraform login`

### Remote Operations (detalle del workflow de runs)

- HCP Terraform ejecuta runs en **VMs desechables propias** — entorno consistente, habilita Sentinel, cost estimation, notificaciones, integración VCS, etc.
- Runs remotos disparables por: webhooks VCS, controles de la UI, llamadas API, o **Terraform CLI** (con CLI, el progreso se **streamea** al terminal, como si fuera local).
- **Desactivar remote operations**: cambiar el **Execution Mode** del workspace a **Local** — el workspace pasa a actuar solo como backend remoto de state; toda la ejecución ocurre en tus propias máquinas/CI. Se pierden features que dependen de ejecución remota (Sentinel, cost estimation, notificaciones).
- **HCP Terraform agents** (feature de pago): permiten a HCP Terraform comunicarse con infra privada/on-prem/aislada sin necesitar ingress público — el agente hace **polling** y ejecuta cambios localmente. Soportan **hooks** (programas custom en puntos estratégicos del run).

**Runs y workspaces:**

- Cada workspace tiene su propia **cola de runs**, procesados en orden. Un run nuevo se añade al final; si ya hay uno en curso, el nuevo **espera** (estado "pending") — ni siquiera se planifica hasta que termine el actual.
- Excepciones que **no bloquean** la cola: **plan-only runs**, y la fase de **planning** de **saved plan runs** (aplicar un saved plan sí bloquea como un run normal; TFE aún no soporta este workflow).
- Al iniciar un run, HCP Terraform **fija (lock)** la configuration version y los valores de variables usados — cambios posteriores solo afectan a runs **futuros**.

**Workspace locks:**

- Un run en curso **bloquea el workspace** automáticamente. Plan-only runs y la fase de planning de saved plans pueden **ignorar** el lock. Un usuario/team también puede bloquear manualmente el workspace (ej. para mantenimiento).

**3 workflows principales para iniciar runs:**

1. **UI/VCS-driven** (modo principal).
2. **API-driven** (más flexible, requiere tooling propio).
3. **CLI-driven** (usa el CLI estándar de Terraform).

**Plan y Apply:**

- Siempre se planifica primero, luego se usa el output del plan para el apply.
- Por defecto espera **aprobación humana** antes de aplicar — se puede configurar **auto-apply**. Algunos plans **no pueden auto-aplicarse** (ej. los encolados por run triggers, o iniciados por usuarios sin permiso de apply).
- Si el plan **no tiene cambios**, no se intenta aplicar — el run termina como "**Planned and finished**" (salvo que se use el modo *allow empty apply*).

**Speculative plans:**

- Plan-only, no pueden aplicar cambios; **no esperan** a que termine la cola (no afectan infra real).
- 3 formas de generarlos: PRs en workspaces VCS-backed (con link en el PR), `terraform plan` con la CLI integration configurada, o vía la Runs API marcando la configuration version como *speculative*.
- Se pueden **reintentar** (Retry Run) si fallan por un factor externo — requiere permiso para encolar plans; solo plans **fallidos o cancelados**.

**Saved plans** (requiere CLI **v1.6.0+** para usarlos desde HCP Terraform):

- `terraform plan -out <FILE>` → planificar y guardar; `terraform apply <FILE>` → aplicar el guardado; `terraform show <FILE>` → inspeccionar antes de aplicar.
- Afectan la cola de runs de forma distinta a los runs normales, y a veces se **descartan automáticamente**.

**Run states**: pending → plan → policy check → apply → completion (algunos estados requieren confirmación).

**Import en HCP Terraform:**

- Se recomienda usar **bloques `import`** (Terraform 1.5+) para importar en HCP Terraform.
- HCP Terraform **no soporta ejecución remota** para el comando `terraform import` — el workspace actúa solo como backend de state, toda la ejecución ocurre **localmente**.
- Como corre local, las **variables de entorno del workspace no están disponibles** — cualquier env var que necesite el provider debe definirse en tu entorno local.

### CLI integration — bloque `cloud`

- La integración CLI permite usar `terraform plan`/`apply` **ejecutándose remotamente** en HCP Terraform por defecto, con el log streameado al terminal local — acceso a variables cifradas en el workspace, cost estimates, policy checking, desde el flujo normal del CLI.
- Workspaces en modo **Local**: HCP Terraform solo guarda state (comportamiento de backend estándar).
- Disponible desde **Terraform 1.1.0+** y **Terraform Enterprise 202201-1+**. Versiones anteriores usan el **backend `remote`**.

**Pasos para conectar:**

1. **Proveer credenciales** (recomendado: `terraform login`, o un user token en la config).
2. **Definir connection settings**: bloque **`cloud`** dentro de `terraform {}`:
   ```hcl
   terraform {
     cloud {
       organization = "my-org"
       hostname     = "app.terraform.io" # opcional, default app.terraform.io

       workspaces {
         project = "networking-development"
         tags = {
           layer  = "networking"
           source = "cli"
         }
       }
     }
   }
   ```
   - `organization`: org de HCP Terraform.
   - `workspaces.tags`: mapa de tags (o lista de strings, legado) — enlaza a workspaces existentes con esos tags; si no hay ninguno, el CLI ofrece crear uno nuevo con esos tags al inicializar.
   - `workspaces.name`: nombre de un workspace existente — **mutuamente excluyente** con `tags`.
   - `workspaces.project`: nombre de un proyecto existente — asocia con workspaces de ese proyecto que hagan match por `name`/`tags`.
3. **Inicializar** (`terraform init`). Por defecto sube una copia de la config al hacer `plan`/`apply` — se puede excluir archivos con **`.terraformignore`**.
4. **Migrar state** (opcional) — ver abajo.

**`.terraformignore`**: reglas tipo `.gitignore` (comentarios `#`, líneas en blanco ignoradas, `/` al final = directorio, `!` para negar patrón). Por defecto, sin este archivo, Terraform excluye igualmente `.git/` y `.terraform/` (salvo `.terraform/modules`).

### Migrar state a HCP Terraform / Terraform Enterprise

**3 formas:**

1. **CLI**: añadir el bloque `cloud` (con `hostname`, `organization`, `workspaces.tags` o `.name`) y correr `terraform init` — crea los workspaces especificados si no existen.
2. **API**: codificar el state en **base64** + generar hash **MD5**, crear el workspace si no existe (`POST /organizations/:org/workspaces`), **bloquear** el workspace (`POST .../actions/lock`), subir el state (`POST .../state-versions` con el md5 y el state codificado), y **desbloquear** (`POST .../actions/unlock`).
3. **`tf-migrate`** (herramienta CLI separada, no viene con HCP Terraform, hay que descargarla aparte) — migración automática, incluso en bulk, o para migrar un **workspace a un Stack**.

**Requisitos**: Terraform **v1.1+** para el bloque `cloud` (v1.0 o anterior → usar backend `remote`). Antes de migrar: **parar todas las operaciones** de Terraform asociadas a esos state files (lock/borrar jobs de CI, restringir acceso al backend). Solo migrar a workspaces que **nunca hayan corrido un run**.

**Migrar desde local/state backend**: `terraform init` con prompts del CLI. HCP Terraform **exige nombre único** por workspace — puede pedir renombrar (convención sugerida: `<COMPONENT>-<ENVIRONMENT>-<REGION>`, ej. `networking-prod-us-east`).

**Migrar desde backend `remote`**: sustituir `backend "remote" {...}` por `cloud {...}` — sigue usando los mismos workspaces. Si usabas `prefix` (multi-workspace), sustituir por `tags`.

### `terraform login`

- Obtiene un **API token** para HCP Terraform/Terraform Enterprise/otro host compatible.
- Solo funciona en escenarios **interactivos** (abre un navegador). Para automatización no interactiva: configurar credenciales manualmente en el **CLI configuration file**.
- Uso: `terraform login [hostname]` — sin hostname, asume `app.terraform.io`. Para **HCP Europe**: `terraform login app.terraform.io/eu`.
- Por defecto guarda el token en **texto plano** en `credentials.tfrc.json` local (avisa antes de guardarlo, se puede cancelar). Alternativa: un **credentials helper program** para guardar/recuperar credenciales de otro sistema (ej. secrets manager de la organización).
- Funciona con cualquier servidor que soporte el **login protocol** (incluye HCP Terraform y Terraform Enterprise).

---

### 💡 Puntos clave para el examen (8d)

- Bloque **`cloud {}`** dentro de `terraform {}` = integración CLI moderna (Terraform 1.1+); versiones anteriores usaban `backend "remote"`.
- `workspaces.name` y `workspaces.tags` son **mutuamente excluyentes** en el bloque `cloud`.
- Migrar de `backend "remote"` a `cloud` conserva los mismos workspaces; `prefix` → hay que convertirlo a `tags`.
- `terraform import` **siempre se ejecuta localmente** en HCP Terraform (nunca remoto) — env vars del workspace no disponibles, hay que definirlas localmente. Se recomienda usar bloques `import` en su lugar.
- Cola de runs por workspace: **secuencial**, salvo **plan-only runs** y la fase de planning de **saved plans**, que no bloquean.
- Speculative plans = solo plan, no bloquean cola, no tocan infra real; 3 formas de generarlos (PR, `terraform plan` con CLI integration, API).
- Saved plans requieren **CLI 1.6.0+** para usarse desde HCP Terraform.
- Ejecución local (`Execution Mode = Local`) = se pierden Sentinel, cost estimation, notificaciones — el workspace solo guarda/sincroniza state.
- `terraform login` = solo interactivo (abre navegador); token guardado en claro en `credentials.tfrc.json` salvo que uses un credentials helper.
- Migración de state: CLI (bloque `cloud` + init), API (base64+MD5, lock→upload→unlock), o `tf-migrate` (bulk / migrar a Stack) — nunca migrar a un workspace que ya haya corrido un run.
