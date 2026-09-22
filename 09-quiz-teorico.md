# Quiz teórico — Terraform Associate 004

Preguntas de repaso organizadas por tema (1-8), en el mismo formato que el examen oficial:
**Verdadero/Falso**, **Opción múltiple** (1 respuesta) y **Respuesta múltiple** (varias respuestas, se indica cuántas).

Cómo usarlo: piensa tu respuesta antes de abrir el desplegable "Ver respuesta". No mires antes de responder — el objetivo es autoevaluarte, no memorizar la posición de la respuesta correcta.

Índice: [Tema 1](#tema-1--iac-con-terraform) · [Tema 2](#tema-2--terraform-fundamentals) · [Tema 3](#tema-3--core-terraform-workflow) · [Tema 4](#tema-4--terraform-configuration) · [Tema 5](#tema-5--terraform-modules) · [Tema 6](#tema-6--terraform-state-management) · [Tema 7](#tema-7--maintain-infrastructure) · [Tema 8](#tema-8--hcp-terraform)

---

## Tema 1 — IaC con Terraform

**1.1 (V/F)** Terraform sigue un enfoque mutable de infraestructura: siempre modifica los recursos existentes in-place en vez de recrearlos.

<details><summary>Ver respuesta</summary>

❌ **Falso**. Terraform sigue un enfoque **inmutable**: cuando un cambio no se puede aplicar in-place, destruye y recrea el recurso, en vez de mutarlo. Ver [01-iac.md](01-iac.md).
</details>

---

**1.2 (Opción múltiple)** ¿Cuáles son las tres fases del core Terraform workflow, en orden?

- ⬜ Init, Validate, Apply
- ⬜ Write, Plan, Apply
- ⬜ Plan, Write, Destroy
- ⬜ Write, Apply, Plan

<details><summary>Ver respuesta</summary>

✅ **Write, Plan, Apply**. `init` y `validate` son pasos dentro del workflow, pero las 3 fases conceptuales son Write → Plan → Apply.
</details>

---

**1.3 (Respuesta múltiple — elige 2)** ¿Qué garantiza a Terraform el uso de un **resource graph**?

- ⬜ Que los recursos no dependientes se puedan crear/modificar en paralelo
- ⬜ Que nunca haga falta un state file
- ⬜ Que determine el orden correcto de creación/destrucción según dependencias
- ⬜ Que el `plan` no muestre nunca cambios

<details><summary>Ver respuesta</summary>

✅ Paralelizar recursos no dependientes.
✅ Determinar el orden correcto según dependencias.
❌ El resource graph no elimina la necesidad de state.
❌ No tiene relación con si el plan muestra cambios o no.
</details>

---

**1.4 (V/F)** Terraform requiere que todos los recursos de una configuración pertenezcan al mismo cloud provider.

<details><summary>Ver respuesta</summary>

❌ **Falso**. Terraform es agnóstico de plataforma — puedes combinar múltiples providers (multi-cloud, hybrid cloud) en la misma configuración con el mismo workflow.
</details>

---

## Tema 2 — Terraform fundamentals

**2.1 (Opción múltiple)** ¿Qué comando genera o actualiza el dependency lock file (`.terraform.lock.hcl`)?

- ⬜ `terraform validate`
- ⬜ `terraform init`
- ⬜ `terraform plan`
- ⬜ `terraform fmt`

<details><summary>Ver respuesta</summary>

✅ **`terraform init`**. También se actualiza con `terraform init -upgrade` si se buscan versiones más nuevas.
</details>

---

**2.2 (V/F)** El dependency lock file rastrea también las versiones exactas de los módulos remotos usados en la configuración.

<details><summary>Ver respuesta</summary>

❌ **Falso**. El lock file solo rastrea **providers**. Para módulos remotos, Terraform siempre usa la versión más reciente que cumpla el constraint, salvo que fijes un constraint exacto (`=`).
</details>

---

**2.3 (Respuesta múltiple — elige 2)** ¿Qué elementos identifican a un provider?

- ⬜ Su local name
- ⬜ Su número de versión de Terraform Core
- ⬜ Su source address
- ⬜ Su alias por defecto

<details><summary>Ver respuesta</summary>

✅ **Local name** (usado dentro del módulo).
✅ **Source address** (identificador global: `[hostname/]namespace/type`).
❌ La versión de Terraform Core no identifica al provider.
❌ El alias es opcional y solo aplica si hay múltiples configuraciones del mismo provider.
</details>

---

**2.4 (Opción múltiple)** Un módulo raíz declara `version = "~> 1.0.4"` para un provider. ¿Qué versiones son aceptables?

- ⬜ Cualquier versión 1.x.x
- ⬜ Solo la versión exacta 1.0.4
- ⬜ Versiones `>= 1.0.4` y `< 1.1.0` (solo patch releases dentro de 1.0.x)
- ⬜ Cualquier versión mayor o igual a 1.0.4

<details><summary>Ver respuesta</summary>

✅ **`>= 1.0.4` y `< 1.1.0`**. El operador `~>` con tres componentes (`1.0.4`) fija major y minor, permitiendo solo incrementar el patch.
</details>

---

**2.5 (V/F)** Si defines dos bloques `provider "aws"` con alias, y ninguno de los dos omite el argumento `alias`, un `resource` sin el meta-argumento `provider` usará automáticamente el primer bloque declarado.

<details><summary>Ver respuesta</summary>

❌ **Falso**. Si todos los bloques de un provider usan alias (ninguno es el "default" sin alias), Terraform crea una **configuración default vacía implícita**. El resource fallará si el provider requiere argumentos obligatorios.
</details>

---

**2.6 (Opción múltiple)** ¿Por qué necesita Terraform un state file, según la documentación oficial de "Purpose of Terraform State"?

- ⬜ Solo por razones históricas, ya no es realmente necesario
- ⬜ Para mapear la configuración con objetos reales, guardar metadata de dependencias, y cachear atributos por rendimiento
- ⬜ Únicamente para guardar los valores de los outputs
- ⬜ Solo lo necesitan los backends remotos, no el backend local

<details><summary>Ver respuesta</summary>

✅ Las tres razones: mapeo config↔mundo real, metadata (dependencias, especialmente al borrar un recurso de la config), y cache de performance.
</details>

---

## Tema 3 — Core Terraform workflow

**3.1 (V/F)** `terraform validate` comprueba que la configuración es válida consultando el estado real de los recursos en el proveedor cloud.

<details><summary>Ver respuesta</summary>

❌ **Falso**. `validate` solo comprueba sintaxis y consistencia interna — **no** toca servicios remotos ni el state. Eso lo hace `terraform plan`.
</details>

---

**3.2 (Opción múltiple)** Ejecutas `terraform apply` sin haber generado un plan guardado previamente. ¿Qué ocurre?

- ⬜ Terraform aplica cambios sin generar ningún plan
- ⬜ Terraform genera un plan automáticamente, pide aprobación, y lo aplica
- ⬜ Terraform da error, hace falta ejecutar `plan` primero manualmente
- ⬜ Terraform solo actualiza el state sin tocar infraestructura

<details><summary>Ver respuesta</summary>

✅ **Genera un plan automáticamente y pide aprobación** (salvo `-auto-approve`).
</details>

---

**3.3 (Respuesta múltiple — elige 2)** ¿Qué opciones de `terraform plan`/`apply` son **mutuamente excluyentes** entre sí?

- ⬜ `-destroy`
- ⬜ `-refresh-only`
- ⬜ `-var`
- ⬜ `-lock-timeout`

<details><summary>Ver respuesta</summary>

✅ **`-destroy`** y **`-refresh-only`** son planning modes mutuamente excluyentes (solo uno activo a la vez).
❌ `-var` y `-lock-timeout` son opciones normales, compatibles con cualquier modo.
</details>

---

**3.4 (V/F)** `terraform destroy` es equivalente a ejecutar `terraform apply -destroy`.

<details><summary>Ver respuesta</summary>

✅ **Verdadero**. `terraform destroy` es literalmente un alias de conveniencia de `apply -destroy`, aunque no acepta un plan file como argumento.
</details>

---

**3.5 (Opción múltiple)** ¿Qué símbolo del plan indica que Terraform va a destruir y volver a crear un recurso?

- ⬜ `~`
- ⬜ `+`
- ⬜ `-/+`
- ⬜ `-`

<details><summary>Ver respuesta</summary>

✅ **`-/+`** (destroy and then create replacement). `~` es update in-place, `+` es create, `-` es solo destroy.
</details>

---

**3.6 (V/F)** Un apply fallido a medio camino hace que Terraform revierta automáticamente (rollback) los cambios ya aplicados.

<details><summary>Ver respuesta</summary>

❌ **Falso**. Terraform **no hace rollback automático**. El state se actualiza con los cambios que sí se completaron, y hay que corregir el error y volver a aplicar.
</details>

---

**3.7 (Opción múltiple)** ¿Cuántos nodos procesa Terraform en paralelo por defecto al recorrer el resource graph?

- ⬜ 1 (secuencial)
- ⬜ 5
- ⬜ 10
- ⬜ Ilimitados

<details><summary>Ver respuesta</summary>

✅ **10**, configurable con `-parallelism`.
</details>

---

**3.8 (V/F)** `terraform fmt` valida que tu configuración no tenga errores de sintaxis, además de darle formato.

<details><summary>Ver respuesta</summary>

❌ **Falso**. `fmt` solo aplica el **estilo canónico** (espaciado, alineación) — no valida sintaxis ni semántica. Eso lo hace `validate`.
</details>

---

## Tema 4 — Terraform configuration

**4.1 (Opción múltiple)** ¿Qué diferencia principal hay entre un bloque `resource` y un bloque `data`?

- ⬜ `data` puede crear infraestructura si no existe; `resource` no
- ⬜ `resource` gestiona el ciclo de vida completo (create/update/destroy); `data` es solo lectura
- ⬜ No hay diferencia funcional real
- ⬜ `data` siempre se resuelve en el plan; `resource` siempre se difiere al apply

<details><summary>Ver respuesta</summary>

✅ **`resource` gestiona el ciclo de vida; `data` es read-only.**
</details>

---

**4.2 (V/F)** Un bloque `data` siempre se resuelve durante la fase de `plan`, nunca se difiere al `apply`.

<details><summary>Ver respuesta</summary>

❌ **Falso**. Si el `data` depende de valores que aún no se conocen (ej. un atributo de un resource que va a cambiar en ese mismo plan), Terraform **difiere su lectura al apply**.
</details>

---

**4.3 (Opción múltiple)** Dado `resource "aws_instance" "web" { count = 3 }`, ¿cómo referencias el id de **todas** las instancias en una lista?

- ⬜ `aws_instance.web.id`
- ⬜ `aws_instance.web[*].id`
- ⬜ `aws_instance.web["all"].id`
- ⬜ `each.value.id`

<details><summary>Ver respuesta</summary>

✅ **`aws_instance.web[*].id`** (splat expression). `each.value` es para `for_each`, no `count`.
</details>

---

**4.4 (V/F)** Los splat expressions (`[*]`) funcionan directamente sobre un recurso creado con `for_each`.

<details><summary>Ver respuesta</summary>

❌ **Falso**. `for_each` produce un **mapa**, no una lista, y el splat actúa sobre listas. Hay que usar `values(recurso)[*].attr` o un `for` expression.
</details>

---

**4.5 (Respuesta múltiple — elige 2)** ¿Qué tipos son **collection types** en Terraform (elementos todos del mismo tipo)?

- ⬜ `object`
- ⬜ `list`
- ⬜ `tuple`
- ⬜ `set`

<details><summary>Ver respuesta</summary>

✅ **`list`** y **`set`** son collection types.
❌ `object` y `tuple` son structural types (elementos de tipos distintos, definidos por un schema).
</details>

---

**4.6 (Opción múltiple)** ¿Qué hace `optional(string, "index.html")` dentro de un type constraint `object({...})`?

- ⬜ Hace el atributo obligatorio con valor `"index.html"`
- ⬜ Hace el atributo opcional; si se omite o se pasa `null`, usa `"index.html"` como valor por defecto
- ⬜ Convierte el atributo a tipo `any`
- ⬜ No tiene ningún efecto fuera de un `output`

<details><summary>Ver respuesta</summary>

✅ Atributo **opcional con default** — se aplica tanto si se omite como si se pasa `null` explícitamente.
</details>

---

**4.7 (V/F)** `can(expr)` devuelve el resultado de la expresión si tiene éxito, o el mensaje de error si falla.

<details><summary>Ver respuesta</summary>

❌ **Falso**. `can()` siempre devuelve un **booleano** (`true`/`false` según si hubo error o no). Para obtener el resultado real con fallback, se usa `try()`.
</details>

---

**4.8 (Opción múltiple)** ¿Cuál de estos 4 mecanismos de validación es el **único** que NO bloquea la operación (solo emite warning)?

- ⬜ `validation` en `variable`
- ⬜ `precondition`
- ⬜ `postcondition`
- ⬜ `check` block

<details><summary>Ver respuesta</summary>

✅ **`check` block** — corre al final de plan/apply, y si falla la `assert`, solo da warning y continúa.
</details>

---

**4.9 (V/F)** Marcar una variable como `sensitive = true` evita que su valor se guarde en el state file.

<details><summary>Ver respuesta</summary>

❌ **Falso** — es uno de los errores más comunes en el examen. `sensitive` solo oculta el valor en el output de CLI/UI; **sigue guardándose en el state en texto plano**. Para evitar que se guarde, hace falta `ephemeral = true`.
</details>

---

**4.10 (Respuesta múltiple — elige 2)** ¿En qué contextos se puede usar un valor `ephemeral`?

- ⬜ En un `output` del **root module**
- ⬜ En un `output` de un **child module** (con `ephemeral = true`)
- ⬜ En un write-only argument de un managed resource
- ⬜ En cualquier argumento de cualquier `resource`

<details><summary>Ver respuesta</summary>

✅ Output de **child module** con `ephemeral = true`.
✅ **Write-only arguments** (`_wo` / `_wo_version`).
❌ Nunca en un output del root module.
❌ No en cualquier argumento — solo en contextos ephemeral específicos (locals, variables ephemeral, bloque `ephemeral`, provider, provisioner/connection).
</details>

---

**4.11 (Opción múltiple)** ¿Qué versión mínima de Terraform requiere el uso de `precondition`/`postcondition`?

- ⬜ 0.12
- ⬜ 0.13
- ⬜ 1.2
- ⬜ 1.5

<details><summary>Ver respuesta</summary>

✅ **1.2+**. (Recordatorio: `validation` en variable → 0.13+; `check` blocks → 1.5+).
</details>

---

## Tema 5 — Terraform modules

**5.1 (V/F)** El argumento `version` en un bloque `module` funciona igual da igual si el `source` es local, de un registry, o de un repositorio Git directo.

<details><summary>Ver respuesta</summary>

❌ **Falso**. `version` solo aplica cuando `source` apunta a un **registry** (público o privado). Los módulos locales no lo soportan; en Git se usa el query param `ref` en su lugar.
</details>

---

**5.2 (Opción múltiple)** ¿Qué sintaxis usarías para referenciar un módulo del Terraform Registry público llamado "vpc" del namespace "terraform-aws-modules" para el provider aws?

- ⬜ `registry.terraform.io/terraform-aws-modules/vpc/aws`
- ⬜ `terraform-aws-modules/vpc/aws`
- ⬜ `aws/vpc/terraform-aws-modules`
- ⬜ Ambas A y B son correctas

<details><summary>Ver respuesta</summary>

✅ **Ambas A y B son correctas** — la sintaxis abreviada `<namespace>/<name>/<provider>` omite el hostname del registry público, pero la forma completa con hostname también es válida.
</details>

---

**5.3 (V/F)** Si defines un bloque `provider` dentro de un child module, es una buena práctica recomendada por HashiCorp.

<details><summary>Ver respuesta</summary>

❌ **Falso**. Se **recomienda fuertemente no** definir bloques `provider` en child modules — la configuración de provider debe vivir en el root module y heredarse.
</details>

---

**5.4 (Opción múltiple)** Un child module necesita usar una configuración de provider con alias (`aws.west`) pasada desde el root module. ¿Qué debe declarar el child module?

- ⬜ Nada especial, se hereda automáticamente
- ⬜ `configuration_aliases` en su bloque `required_providers`
- ⬜ Un bloque `provider "aws" { alias = "west" }` propio
- ⬜ `version = "west"` en su `required_providers`

<details><summary>Ver respuesta</summary>

✅ **`configuration_aliases`** dentro de `required_providers`, y el módulo padre debe pasarlo explícitamente con `providers = { aws.west = aws.west }`.
</details>

---

**5.5 (V/F)** Las variables (`variable` blocks) de un child module comparten el mismo namespace que las variables del root module que lo invoca.

<details><summary>Ver respuesta</summary>

❌ **Falso**. El scope de las variables es **local al módulo** que las declara — solo se conectan por los valores que el padre asigna explícitamente en el bloque `module`.
</details>

---

**5.6 (Respuesta múltiple — elige 2)** Según el patrón de "module composition" recomendado por HashiCorp, ¿qué se recomienda?

- ⬜ Anidar módulos varios niveles de profundidad para máxima reutilización
- ⬜ Mantener el árbol de módulos plano (un solo nivel de child modules)
- ⬜ Que los módulos reciban sus dependencias como input, en vez de crearlas internamente (dependency inversion)
- ⬜ Que cada módulo cree y gestione siempre sus propios recursos de red

<details><summary>Ver respuesta</summary>

✅ **Árbol plano** de módulos.
✅ **Dependency inversion** (recibir dependencias como argumentos).
❌ Anidar mucho va en contra de la recomendación.
❌ Ídem — se prefiere que reciban la red como argumento, no que la creen ellos mismos.
</details>

---

## Tema 6 — Terraform state management

**6.1 (Opción múltiple)** ¿Cuál es el backend por defecto de Terraform si no configuras ningún bloque `backend`?

- ⬜ `remote`
- ⬜ `s3`
- ⬜ `local`
- ⬜ `cloud`

<details><summary>Ver respuesta</summary>

✅ **`local`** — guarda el state como archivo en disco.
</details>

---

**6.2 (V/F)** Todos los backends de Terraform soportan state locking.

<details><summary>Ver respuesta</summary>

❌ **Falso**. El locking es **opcional** y depende de cada backend — hay que consultar la documentación específica de cada uno.
</details>

---

**6.3 (Opción múltiple)** ¿Qué comando usarías para desbloquear manualmente un state cuyo lock quedó atascado tras un fallo?

- ⬜ `terraform state rm`
- ⬜ `terraform force-unlock <LOCK_ID>`
- ⬜ `terraform state push -force`
- ⬜ `terraform untaint`

<details><summary>Ver respuesta</summary>

✅ **`terraform force-unlock <LOCK_ID>`** — requiere el lock ID único que Terraform muestra en el error de bloqueo.
</details>

---

**6.4 (V/F)** `terraform state mv` es el método que HashiCorp recomienda actualmente para migrar recursos entre dos state files distintos.

<details><summary>Ver respuesta</summary>

❌ **Falso**. Es un comando **legacy**. HashiCorp recomienda actualmente el patrón **`removed` + `import`** (requiere Terraform 1.7+) porque mantiene un registro histórico y es más seguro.
</details>

---

**6.5 (Opción múltiple)** Tienes un bloque `removed { from = aws_instance.example, lifecycle { destroy = false } }`. ¿Qué ocurre al aplicar?

- ⬜ Se elimina la instancia EC2 real y se saca del state
- ⬜ Se elimina del state, pero la instancia EC2 real sigue existiendo intacta
- ⬜ No ocurre nada hasta que se borre también el `resource` block
- ⬜ Da error porque `destroy = false` no es válido

<details><summary>Ver respuesta</summary>

✅ **Se elimina del state, sin destruir el objeto real** — `destroy = false` es justo para eso (traspasar gestión a otra herramienta/equipo).
</details>

---

**6.6 (V/F)** El bloque `moved { from, to }` provoca que Terraform destruya el recurso en la address antigua y lo recree en la nueva.

<details><summary>Ver respuesta</summary>

❌ **Falso**. `moved` renombra el objeto **dentro del state** antes de generar el plan — Terraform **no destruye ni recrea** nada, solo actualiza la referencia.
</details>

---

**6.7 (Opción múltiple)** ¿Qué diferencia hay entre "configuration drift" y "state drift"?

- ⬜ Son sinónimos exactos
- ⬜ Configuration drift invalida tu config (lo detecta drift detection); state drift no la invalida (se remedia con refresh-only)
- ⬜ State drift solo ocurre en HCP Terraform, nunca en local
- ⬜ Configuration drift se soluciona siempre con `terraform state rm`

<details><summary>Ver respuesta</summary>

✅ Configuration drift = cambios externos que **invalidan** la config (drift detection lo detecta). State drift = cambios externos que **no** la invalidan (se remedia con **refresh-only mode**).
</details>

---

## Tema 7 — Maintain infrastructure

**7.1 (V/F)** `terraform import` puede importar una colección completa de recursos (ej. una VPC entera con todos sus componentes) en una sola ejecución.

<details><summary>Ver respuesta</summary>

❌ **Falso**. `terraform import` solo puede importar **un recurso a la vez**. Un "complex import" trae recursos secundarios que hay que añadir manualmente a la config.
</details>

---

**7.2 (Opción múltiple)** Antes de ejecutar `terraform import aws_instance.example i-abcd1234`, ¿qué necesitas tener ya en tu configuración?

- ⬜ Nada, Terraform genera el `resource` block automáticamente siempre
- ⬜ Un bloque `resource "aws_instance" "example" {}` (puede estar vacío/incompleto)
- ⬜ Un bloque `data "aws_instance" "example" {}`
- ⬜ Un `output` que referencie `aws_instance.example`

<details><summary>Ver respuesta</summary>

✅ Un bloque **`resource`** ya escrito (aunque incompleto) — Terraform necesita saber a qué address vincular el objeto importado.
</details>

---

**7.3 (Respuesta múltiple — elige 2)** ¿Qué subcomandos de `terraform state` son **read-only** (no generan backup)?

- ⬜ `terraform state list`
- ⬜ `terraform state rm`
- ⬜ `terraform state show`
- ⬜ `terraform state mv`

<details><summary>Ver respuesta</summary>

✅ **`list`** y **`show`** son read-only.
❌ `rm` y `mv` **modifican** el state → siempre generan backup (no desactivable).
</details>

---

**7.4 (V/F)** Configurar `TF_LOG_PATH` es suficiente por sí solo para activar el logging detallado de Terraform.

<details><summary>Ver respuesta</summary>

❌ **Falso**. `TF_LOG_PATH` solo indica **dónde** persistir el log — hace falta además que `TF_LOG` esté seteado para que el logging esté activo.
</details>

---

**7.5 (Opción múltiple)** Ordena de más a menos verboso los niveles de `TF_LOG`.

- ⬜ ERROR > WARN > INFO > DEBUG > TRACE
- ⬜ TRACE > DEBUG > INFO > WARN > ERROR
- ⬜ DEBUG > TRACE > ERROR > WARN > INFO
- ⬜ INFO > DEBUG > TRACE > WARN > ERROR

<details><summary>Ver respuesta</summary>

✅ **TRACE > DEBUG > INFO > WARN > ERROR**.
</details>

---

## Tema 8 — HCP Terraform

**8.1 (V/F)** Los workspaces de HCP Terraform y los workspaces de Terraform CLI (`terraform workspace new`) son conceptualmente lo mismo.

<details><summary>Ver respuesta</summary>

❌ **Falso**. Los workspaces de **HCP Terraform** son obligatorios y son la unidad de RBAC (equivalen a un working directory completo con config+state+variables). Los workspaces de **Terraform CLI** son opcionales y solo aíslan states dentro del **mismo** directorio de trabajo.
</details>

---

**8.2 (Opción múltiple)** ¿Cuál de estos frameworks de policy enforcement soporta tanto workspaces como Stacks?

- ⬜ Sentinel
- ⬜ OPA
- ⬜ Terraform policy (HCL nativo)
- ⬜ Ninguno soporta Stacks todavía

<details><summary>Ver respuesta</summary>

✅ **Terraform policy** (framework nativo HCL, en beta) soporta workspaces y Stacks. Sentinel y OPA solo soportan workspaces.
</details>

---

**8.3 (V/F)** Un run trigger, por defecto, hace auto-apply del run que encola en el workspace destino.

<details><summary>Ver respuesta</summary>

❌ **Falso**. Los runs disparados por un run trigger **no auto-aplican** por defecto — hay que activar explícitamente el setting "Auto-apply run triggers" (independiente del auto-apply normal del workspace).
</details>

---

**8.4 (Opción múltiple)** ¿Qué data source se recomienda actualmente en vez de `terraform_remote_state` para leer outputs de otro workspace en HCP Terraform?

- ⬜ `tfe_workspace`
- ⬜ `tfe_outputs`
- ⬜ `terraform_state`
- ⬜ `hcp_outputs`

<details><summary>Ver respuesta</summary>

✅ **`tfe_outputs`** — más seguro, no requiere acceso completo al state para leer solo los outputs.
</details>

---

**8.5 (V/F)** Las variable sets se evalúan en workspaces configurados con Execution Mode "Local".

<details><summary>Ver respuesta</summary>

❌ **Falso**. HCP Terraform **no evalúa** variable sets en workspaces con execution mode **Local**.
</details>

---

**8.6 (Opción múltiple)** ¿Qué le pasa al valor de una variable marcada como `Sensitive` en HCP Terraform tras guardarla?

- ⬜ Se puede seguir leyendo desde la UI en cualquier momento
- ⬜ Se vuelve write-only: nadie (ni tú) puede volver a leer su valor por UI o API, solo sobreescribirlo
- ⬜ Se borra automáticamente a los 30 días
- ⬜ Deja de estar disponible para los runs

<details><summary>Ver respuesta</summary>

✅ **Write-only** — no se puede volver a leer, solo actualizar (o borrar y recrear para cambiar otros atributos).
</details>

---

**8.7 (V/F)** El comando `terraform import` se ejecuta de forma remota en HCP Terraform, igual que `plan` y `apply`.

<details><summary>Ver respuesta</summary>

❌ **Falso**. HCP Terraform **no soporta ejecución remota** para `terraform import` — corre siempre localmente; el workspace solo actúa como backend de state para ese comando. Por eso se recomienda usar bloques `import` en su lugar.
</details>

---

**8.8 (Opción múltiple)** ¿Qué versión mínima de Terraform CLI necesitas para usar el bloque `cloud {}` (integración CLI moderna con HCP Terraform)?

- ⬜ 0.15
- ⬜ 1.0
- ⬜ 1.1.0
- ⬜ 1.5.0

<details><summary>Ver respuesta</summary>

✅ **1.1.0+**. Versiones anteriores deben usar el backend `remote`.
</details>

---

**8.9 (V/F)** En el bloque `cloud`, puedes especificar `workspaces.name` y `workspaces.tags` al mismo tiempo para mayor flexibilidad.

<details><summary>Ver respuesta</summary>

❌ **Falso**. `name` y `tags` son **mutuamente excluyentes** dentro de `workspaces {}`.
</details>

---

*Fin del quiz teórico. Para preguntas de código/escenarios prácticos, ver [10-quiz-practico.md](10-quiz-practico.md).*
