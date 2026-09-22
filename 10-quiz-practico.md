# Quiz práctico / escenarios — Terraform Associate 004

Preguntas basadas en fragmentos de código HCL, salidas de CLI y situaciones "qué pasaría si...", que es el estilo donde más suele fallar quien solo ha estudiado teoría. Respuestas en desplegables.

Índice: [Variables y expresiones](#bloque-a--variables-expresiones-y-tipos) · [Resources y meta-argumentos](#bloque-b--resources-count-for_each-y-meta-argumentos) · [State](#bloque-c--state) · [Providers y módulos](#bloque-d--providers-y-módulos) · [Workflow y CLI](#bloque-e--workflow-y-cli) · [Sensitive/ephemeral](#bloque-f--secrets-sensitive-y-ephemeral) · [HCP Terraform](#bloque-g--hcp-terraform)

---

## Bloque A — Variables, expresiones y tipos

**A.1** Dado:

```hcl
variable "vpc_cidrs" {
  type = map(string)
  default = {
    us-east-1 = "10.0.0.0/16"
    us-west-2 = "10.3.0.0/16"
  }
}

resource "aws_vpc" "shared" {
  cidr_block = ______________
}
```

¿Qué expresión rellena correctamente el hueco para usar el CIDR de `us-east-1`?

<details><summary>Ver respuesta</summary>

```hcl
cidr_block = var.vpc_cidrs["us-east-1"]
```

`var.vpc_cidrs.us-east-1` también funcionaría sintácticamente si la key fuera un identificador válido, pero como contiene guiones, hace falta la notación de corchetes con comillas.
</details>

---

**A.2** ¿Qué imprime `terraform console` para esta expresión?

```hcl
> ["a", 1, "b"]
```
si se le fuerza el type constraint `list(any)` en una variable que recibe ese valor.

<details><summary>Ver respuesta</summary>

`["a", "1", "b"]` — Terraform busca un único tipo de elemento válido para todos (`string`), y convierte el `1` numérico a `"1"` string mediante las reglas de conversión de tipos primitivos.
</details>

---

**A.3** Tienes este `variable` block:

```hcl
variable "with_optional_attribute" {
  type = object({
    a = string
    b = optional(string)
    c = optional(number, 127)
  })
}
```

Un caller pasa `{ a = "x" }` (sin `b` ni `c`). ¿Qué valores tendrán `b` y `c` dentro del módulo?

<details><summary>Ver respuesta</summary>

`b = null` (optional sin default explícito → default `null`), `c = 127` (optional con default explícito).
</details>

---

**A.4** ¿Qué diferencia hay en el resultado entre `can(expr)` y `try(expr, "fallback")` si `expr` lanza un error?

<details><summary>Ver respuesta</summary>

`can(expr)` devuelve `false` (booleano). `try(expr, "fallback")` devuelve el string `"fallback"` (el resultado de la primera expresión sin error de la lista de argumentos).
</details>

---

**A.5** Tienes:

```hcl
locals {
  environment = "prod"
}

output "name_tag" {
  value = "app-${local.environment}-${terraform.workspace}"
}
```

Si ejecutas esto en dos workspaces CLI distintos (`default` y `staging`), ¿el valor de `name_tag` será el mismo en ambos?

<details><summary>Ver respuesta</summary>

**No**. `terraform.workspace` cambia según el workspace CLI activo (`default` vs `staging`), así que el output difiere aunque `local.environment` sea fijo.
</details>

---

## Bloque B — Resources, `count`, `for_each` y meta-argumentos

**B.1** Dado:

```hcl
resource "aws_instance" "web" {
  for_each = toset(["a", "b", "c"])
  ami      = "ami-123"
}
```

¿Cómo referencias el id de la instancia con key `"b"`?

<details><summary>Ver respuesta</summary>

`aws_instance.web["b"].id`
</details>

---

**B.2** ¿Qué error obtendrías si intentas hacer `aws_instance.web[*].id` sobre el resource del ejercicio B.1?

<details><summary>Ver respuesta</summary>

Error — los splat expressions requieren un valor tipo **lista**, y un resource con `for_each` es un **mapa**. Hay que usar `values(aws_instance.web)[*].id` o un `for` expression: `[for k, v in aws_instance.web : v.id]`.
</details>

---

**B.3** Tienes un resource con `count = 3` en tu configuración y aplicas. Luego cambias a `count = 1`. ¿Qué instancias destruye Terraform?

<details><summary>Ver respuesta</summary>

Destruye las instancias en los **índices más altos** (`[1]` y `[2]`), conservando `[0]`. Por esto `count` es frágil frente a `for_each` cuando el orden de la lista puede cambiar — con `for_each` cada instancia está atada a su key, no a una posición.
</details>

---

**B.4** ¿Qué diferencia hay entre este bloque en un `data` y en un `resource`?

```hcl
lifecycle {
  postcondition {
    condition     = self.tags["Component"] == "nomad-server"
    error_message = "..."
  }
}
```

<details><summary>Ver respuesta</summary>

Ninguna diferencia funcional relevante — ambos soportan `precondition`/`postcondition` dentro de `lifecycle`, y en ambos `self` referencia al propio objeto (resource o data source). La diferencia es **cuándo** se evalúa: en un `resource`, tras plan+apply; en un `data`, tras la lectura del data source.
</details>

---

**B.5** ¿Qué crees que ocurre si defines `depends_on` en un `output` que apunta a un resource, pero ese resource no se referencia en ningún argumento de `value` del output?

<details><summary>Ver respuesta</summary>

Es válido — `depends_on` fuerza una dependencia **explícita** aunque no exista una dependencia implícita por referencia. Es poco común, pero legítimo (ej. asegurar que un security group rule existe antes de exponer una IP, aunque el output no referencie el security group directamente). Se recomienda comentar el motivo.
</details>

---

## Bloque C — State

**C.1** Ejecutas `terraform state rm aws_instance.example`. ¿Qué le pasa a la instancia EC2 real en AWS?

<details><summary>Ver respuesta</summary>

**Nada** — sigue existiendo intacta en AWS. `terraform state rm` solo elimina el **binding en el state**, sin tocar la infraestructura real. Terraform "olvida" ese recurso.
</details>

---

**C.2** Tras el C.1, ¿qué mostraría `terraform plan` si el `resource` block sigue en tu `.tf`?

<details><summary>Ver respuesta</summary>

Un plan de **creación** (`+`) — Terraform ya no tiene el binding en el state, así que interpreta que el recurso descrito en la config **no existe** y planea crearlo (lo cual probablemente fallaría por conflicto de nombre/ID en el cloud real, salvo que lo reimportes).
</details>

---

**C.3** Dos compañeros ejecutan `terraform apply` casi a la vez sobre el mismo backend remoto con locking soportado. ¿Qué pasa?

<details><summary>Ver respuesta</summary>

El **segundo** en llegar no puede adquirir el lock del state y su operación **falla** (o espera, según `-lock-timeout`) hasta que el primero libere el lock. Esto evita corrupción del state por escrituras concurrentes.
</details>

---

**C.4** Tienes state en local (`terraform.tfstate`) con valores sensibles dentro. ¿Es seguro subir ese archivo a un repo Git privado de tu organización?

<details><summary>Ver respuesta</summary>

**No se recomienda**. El state guarda valores sensibles **en texto plano**, y los repos Git no ofrecen locking ni control de acceso granular sobre el archivo — cualquiera con acceso al repo (y a su historial, aunque luego se borre) puede leer los secretos. Se recomienda backend remoto con cifrado at-rest y access control.
</details>

---

**C.5** Quieres mover un recurso `aws_db_instance.main` desde el state de la config A al state de la config B, manteniendo un registro de por qué se movió. ¿Qué enfoque usarías (Terraform 1.7+)?

<details><summary>Ver respuesta</summary>

`removed` block en la config A (con `lifecycle { destroy = false }`) + `import` block en la config B — el enfoque recomendado actualmente, en vez del legacy `terraform state mv`.
</details>

---

**C.6** Ejecutas `terraform plan -refresh=false`. Un compañero cambió manualmente un tag en la consola de AWS ayer. ¿Ese cambio aparecerá en tu plan?

<details><summary>Ver respuesta</summary>

**No** — `-refresh=false` hace que Terraform **no sincronice** el state con la infraestructura real antes de calcular el plan, así que ignora ese cambio externo (drift) y compara la config contra el state **desactualizado**.
</details>

---

## Bloque D — Providers y módulos

**D.1** Dos providers distintos (`hashicorp/http` y `mycorp/http`) tienen el mismo "type" preferido (`http`). ¿Cómo los declaras ambos en el mismo módulo?

<details><summary>Ver respuesta</summary>

```hcl
terraform {
  required_providers {
    hashicorp-http = { source = "hashicorp/http", version = "~> 2.0" }
    mycorp-http    = { source = "mycorp/http",    version = "~> 1.0" }
  }
}
```

Con **local names compuestos** (namespace + type), y especificando el meta-argumento `provider = hashicorp-http` (o `mycorp-http`) en cada resource/data afectado, ya que Terraform no puede inferirlo del nombre del resource type.
</details>

---

**D.2** Tu root module define:

```hcl
provider "aws" {
  region = "us-west-2"
}

module "vpc" {
  source = "./modules/vpc"
}
```

`./modules/vpc/main.tf` tiene `required_providers { aws = { source = "hashicorp/aws", version = "~> 5.0" } }` pero **no** tiene un bloque `provider "aws" {}`. ¿Funciona igualmente?

<details><summary>Ver respuesta</summary>

**Sí**. La configuración de provider se **hereda implícitamente** del root module a los child modules — el child module solo necesita declarar el `required_providers` (source/version), no repetir el bloque `provider` con la configuración.
</details>

---

**D.3** Añades una nueva versión `~> 2.0` a un módulo que antes usaba `~> 1.0`, sin ejecutar ningún comando adicional. ¿Se aplicará la nueva versión en el próximo `apply`?

<details><summary>Ver respuesta</summary>

**No** — hace falta ejecutar `terraform init` (posiblemente con `-upgrade` según el caso) tras cambiar `version` o `source`, para que Terraform descargue y resuelva la nueva versión antes de poder planificar/aplicar.
</details>

---

**D.4** Tienes un módulo `consul_cluster` que internamente crea su propia VPC con `resource "aws_vpc" "internal"`. Otro compañero necesita desplegar el cluster dentro de una VPC ya existente compartida. ¿Qué patrón de diseño de módulos resolvería esto mejor a futuro?

<details><summary>Ver respuesta</summary>

**Dependency inversion**: el módulo debería **aceptar el `vpc_id`/`subnet_ids` como input variables**, en vez de crear su propia red internamente. Así el caller decide si la crea con un `resource` o la lee con un `data source`, sin tener que modificar el módulo.
</details>

---

## Bloque E — Workflow y CLI

**E.1** Ejecutas `terraform plan -out=tfplan` y luego, 2 horas después, `terraform apply tfplan`. Entre medias, alguien cambió manualmente un recurso en la consola cloud. ¿El apply detecta ese cambio?

<details><summary>Ver respuesta</summary>

**No** — un plan guardado (`-out`) es un snapshot congelado; `apply` con un archivo de plan **no vuelve a evaluar** cambios externos, ejecuta exactamente las acciones planificadas en ese momento. Es una de las razones por las que los plan files pueden quedar desactualizados si pasa mucho tiempo.
</details>

---

**E.2** ¿Qué exit code devolvería `terraform plan -detailed-exitcode` si no hay cambios pendientes?

<details><summary>Ver respuesta</summary>

**`0`**. (`1` = error, `2` = hay cambios pendientes). Muy usado en pipelines de CI para decidir si hace falta una revisión/apply.
</details>

---

**E.3** Corres `terraform plan -target=aws_instance.web`. ¿Se ignoran completamente los recursos de los que depende `aws_instance.web`?

<details><summary>Ver respuesta</summary>

**No** — Terraform **extiende automáticamente** la selección a todo lo que `aws_instance.web` dependa (directa o indirectamente), aunque no lo hayas targeteado explícitamente. Solo excluye lo que **no** es una dependencia de los recursos targeteados.
</details>

---

**E.4** ¿Qué diferencia hay entre `terraform plan -refresh-only` y `terraform plan -refresh=false`?

<details><summary>Ver respuesta</summary>

`-refresh-only` genera un plan cuyo **único** objetivo es actualizar el state para reflejar cambios hechos fuera de Terraform (no cambia infra real). `-refresh=false` es lo contrario: **salta** el refresh normal antes de calcular un plan de cambios de config — puede dar un plan incompleto si hubo drift.
</details>

---

**E.5** Tu pipeline de CI ejecuta `terraform plan -input=false -no-color` de forma no interactiva. Una variable requerida no tiene valor asignado por ningún medio (`-var`, `.tfvars`, env var). ¿Qué pasa?

<details><summary>Ver respuesta</summary>

**Falla** — con `-input=false`, Terraform no puede preguntar interactivamente por el valor, así que la operación termina en error en vez de quedarse esperando input.
</details>

---

## Bloque F — Secrets, sensitive y ephemeral

**F.1** Tienes:

```hcl
variable "db_password" {
  type      = string
  sensitive = true
}

resource "aws_db_instance" "main" {
  password = var.db_password
}
```

¿El valor de `var.db_password` aparece en `terraform.tfstate` tras el apply?

<details><summary>Ver respuesta</summary>

**Sí, en texto plano**. `sensitive = true` solo oculta el valor en el output de CLI/UI (`(sensitive value)`) — el state sigue guardándolo sin cifrar.
</details>

---

**F.2** Ahora cambias a:

```hcl
variable "db_password" {
  type      = string
  sensitive = true
  ephemeral = true
}
```

¿Puedes usar `var.db_password` directamente como `password` en el `resource "aws_db_instance"` de arriba?

<details><summary>Ver respuesta</summary>

**No directamente** — un valor `ephemeral` solo se puede referenciar en contextos ephemeral (locals, otras variables/outputs ephemeral, bloque `provider`, `provisioner`/`connection`, o un **write-only argument**). Un argumento normal como `password` no es un contexto ephemeral; haría falta usar el write-only equivalente (`password_wo` + `password_wo_version`) si el provider lo ofrece.
</details>

---

**F.3** ¿Qué comando de CLI revelaría el valor real de un output marcado como `sensitive`, aunque en el apply normal se muestre como `(sensitive value)`?

<details><summary>Ver respuesta</summary>

`terraform output -json` o `terraform output -raw <name>` — ambos muestran el valor sensible en texto plano.
</details>

---

**F.4** Defines un `ephemeral "random_password" "db" { length = 16 }` y lo usas como `password_wo = ephemeral.random_password.db.result`. Necesitas recuperar esa password después, en un futuro `plan`. ¿Puedes leerla del state?

<details><summary>Ver respuesta</summary>

**No** — ni el bloque `ephemeral` ni los write-only arguments persisten en state/plan. Si necesitas conservar el valor generado, hay que **capturarlo explícitamente** en otro recurso (ej. guardarlo en Secrets Manager/Vault dentro del mismo apply).
</details>

---

## Bloque G — HCP Terraform

**G.1** Un workspace de HCP Terraform tiene Execution Mode = **Local**. ¿HCP Terraform ejecuta el `plan`/`apply` en sus propias VMs?

<details><summary>Ver respuesta</summary>

**No** — en modo Local, toda la ejecución ocurre en tus propias máquinas/CI; el workspace **solo almacena y sincroniza el state** (actúa como backend remoto puro). Features como Sentinel, cost estimation y notificaciones dejan de estar disponibles.
</details>

---

**G.2** Tienes un run trigger configurado desde el workspace `networking` hacia el workspace `app`. Alguien hace un `apply` **fallido** en `networking`. ¿Se dispara un nuevo run en `app`?

<details><summary>Ver respuesta</summary>

**No** — los run triggers solo se disparan en apply **exitoso** de la source workspace.
</details>

---

**G.3** Configuras una variable set "Priority" a nivel organización con `region = "us-east-1"`. Un workspace concreto también tiene una variable local `region = "us-west-2"` puesta directamente. ¿Cuál gana en los runs de ese workspace?

<details><summary>Ver respuesta</summary>

**`us-east-1`** (la del priority variable set) — los priority variable sets **sobreescriben** variables con la misma key definidas en scopes más específicos, incluidas las puestas directamente en el workspace o por `.tfvars`/CLI.
</details>

---

**G.4** Quieres importar un recurso existente a un workspace de HCP Terraform que usa Execution Mode = Remote. ¿Puedes hacerlo con `terraform import` corriendo remotamente en HCP Terraform?

<details><summary>Ver respuesta</summary>

**No** — `terraform import` **siempre corre localmente**, nunca de forma remota en HCP Terraform, sea cual sea el execution mode del workspace. Alternativa recomendada: bloques `import` en la config (esos sí se aplican en un run normal, remoto o local).
</details>

---

**G.5** Migras tu configuración de `backend "remote" { workspaces { prefix = "app-" } }` a un bloque `cloud`. ¿Puedes seguir usando `prefix` dentro de `cloud { workspaces { ... } }`?

<details><summary>Ver respuesta</summary>

**No** — el bloque `cloud` **no soporta `prefix`**, solo `name` o `tags`. Hay que convertir el prefix a un esquema de tags equivalente, y tras la migración referenciar los workspaces por su **nombre completo** (ej. `terraform workspace select app-prod` en vez de `prod`).
</details>

---

*Documento hermano: [09-quiz-teorico.md](09-quiz-teorico.md) para preguntas tipo examen oficial (V/F, opción múltiple, respuesta múltiple).*
