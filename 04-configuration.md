# 4. Terraform configuration

| Subtema | Descripción | Cubierto |
|---|---|---|
| 4a | Use and differentiate `resource` and `data` blocks | ✅ |
| 4b | Refer to resource attributes and create cross-resource references | ✅ |
| 4c | Use variables and outputs | ✅ |
| 4d | Understand and use complex types | ✅ |
| 4e | Write dynamic configuration using expressions and functions | ✅ |
| 4f | Define resource dependencies in configuration (`depends_on`, `create_before_destroy`) | 🟡 parcial |
| 4g | Validate configuration using custom conditions | ✅ |
| 4h | Best practices for sensitive data, secrets con Vault, ephemeral values, write-only arguments | ✅ |

_(Nota: 4f, 4g, 4h son temas nuevos en el examen 004)_

---

## 4a. `resource` vs `data` blocks

Fuentes: "Create and manage resources overview", "Query infrastructure data" (Data Sources)

### `resource` — visión general

- Un **resource** es cualquier objeto de infraestructura que quieres crear y gestionar con Terraform (red virtual, instancia de compute, un registro DNS...).
- Los tipos de resource disponibles dependen de los **providers instalados**.

**Workflow para crear/gestionar resources:**

1. **Escribir la config**: bloque `resource` con los argumentos necesarios. La mayoría son específicos del resource; también hay **meta-arguments** propios de Terraform (`count`, `for_each`, `depends_on`, `provider`, `lifecycle`...) que controlan cómo Terraform crea/gestiona el resource.
2. **Inicializar el workspace** (`terraform init`) — necesario al empezar un proyecto y cada vez que cambian providers/módulos.
3. **Aplicar la config** (`terraform apply`). Al aplicar, Terraform:
   - **Crea** resources de la config que aún no existen como objetos reales.
   - **Destruye** resources que están en el state pero ya no en la config.
   - **Actualiza in-place** resources cuyos argumentos cambiaron (si es posible).
   - **Destruye y recrea** resources cuyos argumentos cambiaron pero **no se pueden actualizar in-place** por limitaciones de la API remota.
   - **Actualiza el state file** para que config, infra real y state coincidan.
4. **Gestionar resources** con el tiempo: añadir/quitar de la config, refactorizar en módulos reutilizables, quitar del state sin destruir el objeto real (`terraform state rm` / bloque `removed`), o destruir de verdad (`terraform destroy` / eliminar de la config).

### `data` — Data Sources

- Muchos providers exponen **data sources**: permiten **leer** datos del provider (APIs, otros workspaces de Terraform, outputs de funciones...) **sin crear ni modificar** recursos.
- Se declaran con un bloque `data`, especificando tipo + label:

```hcl
data "aws_ami" "example" {
  most_recent = true
  owners      = ["self"]
  tags = {
    Name   = "app-server"
    Tested = "true"
  }
}
```

- Para referenciar el resultado: **`data.<TYPE>.<LABEL>.<ATTRIBUTE>`** (ej. `data.aws_ami.example.id`).
- Terraform **solo puede leer** (read-only) de un data source — nunca crea/modifica nada a través de `data`.
- El bloque `data` soporta expresiones y features dinámicas del lenguaje, y muchos de los **meta-arguments** built-in (`count`, `for_each`, `depends_on`, `provider`, `lifecycle` con pre/postcondition).

### Cuándo se consulta un data source: plan vs apply

- Terraform intenta consultar (query) los data sources durante la **fase de plan**, pero a veces **difiere la lectura a la fase de apply** — el plan output lo indica explícitamente cuando esto ocurre.
- Terraform difiere la lectura a `apply` cuando **al menos uno de los argumentos** no se puede predecir en la fase de plan, en escenarios como:
  - La config del `data` block depende **directamente** de un resource gestionado por Terraform que va a cambiar en el plan actual.
  - El `data` block tiene **custom conditions** (pre/postcondition) que dependen directa o indirectamente de un resource que va a cambiar.
  - Argumentos del `data` block se refieren a valores que deben **calcularse durante el apply**.
- Cuando el data source depende de otros objetos, los resultados de la query son **desconocidos (unknown)** durante `plan` → Terraform no puede provisionar recursos que referencien esos valores hasta el `apply`.

**Referencias a valores computados vs no computados:**

- Si los argumentos del `data` refieren a **valores computados** (que dependen de otro resource) → Terraform no puede leer el data source hasta que esos argumentos estén definidos → **difiere el refresh a apply**, y los atributos aparecen como `(known after apply)` / `computed` en el plan.
- Si los argumentos refieren a valores **no computados** → Terraform lee el data source y actualiza el state durante la **fase de refresh** (por defecto, Terraform hace refresh antes de crear el plan), así los datos están disponibles ya en el plan.

### Data sources "especializados" (solo-local)

Algunos data sources generan datos que **solo existen durante la operación de Terraform**, recalculándose en cada plan:

- `data "template_file"` (provider `template`): renderiza una plantilla.
- `data "local_file"` (provider `local`): lee archivos locales.
- `data "aws_iam_policy_document"` (provider `aws`): renderiza políticas IAM.

### `depends_on` en bloques `data`

- Terraform detecta dependencias automáticamente, incluso indirectas (ej. a través de un `local` value).
- Se puede añadir `depends_on` a un `data` block para forzar un orden concreto — hace que Terraform **difiera la query** del data source hasta después de que termine la operación de la dependencia especificada.
- **Recomendación**: usar `depends_on` en `data` blocks solo en Terraform **0.13+**. En 0.12 y anteriores, forzaba diferir la lectura a la fase de apply, con comportamiento potencialmente no deseado.

### Custom condition checks en `data`

- Se pueden añadir `precondition`/`postcondition` (dentro de `lifecycle`) para especificar asunciones/garantías sobre cómo opera el data source:

```hcl
data "aws_ami" "example" {
  id = var.aws_ami_id

  lifecycle {
    postcondition {
      condition     = self.tags["Component"] == "nomad-server"
      error_message = "tags[\"Component\"] must be \"nomad-server\"."
    }
  }
}
```

- Dan errores útiles y **tempranos**, con contexto, y ayudan a documentar la intención de la config para futuros mantenedores. (Detalle completo → 4g.)

### Múltiples instancias (`count` / `for_each` en `data`)

- Igual que en `resource`, se puede usar `count`/`for_each` para crear múltiples instancias de un data source.
- Se referencian con `data.<NAME>[<KEY>]`:
  - Con `count`: `<KEY>` es un número (empezando en `0` en la práctica actual del lenguaje — nota: la doc de esta página dice "starting at 1", pero en Terraform moderno el índice de `count` es base 0, como en `resource`).
  - Con `for_each`: `<KEY>` es la key de la colección. Ej: `data.azurerm_resource_group.rg["a_group"]`.

### Provider alternativo en `data`

- Igual que en `resource`, se puede indicar un provider con alias vía el meta-argumento `provider`:

```hcl
provider "aws" {
  region = "us-east-1"
}

provider "aws" {
  alias  = "uswest1"
  region = "us-west-1"
}

data "aws_ami" "web" {
  provider = aws.uswest1
  # ...
}
```

### Ejemplo completo (uso típico: `data` alimentando a `resource`)

```hcl
data "aws_ami" "web" {
  filter {
    name   = "state"
    values = ["available"]
  }
  filter {
    name   = "tag:Component"
    values = ["web"]
  }
  most_recent = true
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.web.id
  instance_type = "t1.micro"
}
```

- La combinación **type + label** de un `data` block debe ser **única** (igual que en `resource`).
- Los argumentos dentro de `filter`, y `most_recent`, son específicos del provider/data source (aquí `aws_ami` del provider AWS) — hay que consultar la documentación de cada provider.

---

### 💡 Puntos clave para el examen (4a)

- **`resource`** = crea/gestiona un objeto real (create, update in-place, destroy+recreate, destroy). **`data`** = solo **lectura** (read-only), nunca crea/modifica nada.
- `apply` con un `resource`: create / destroy (si ya no está en config) / update in-place / destroy+recreate (si el cambio no se puede aplicar in-place por la API) — y siempre actualiza el state.
- Referencias: `resource` → `<TYPE>.<NAME>.<ATTRIBUTE>`; `data` → **`data.<TYPE>.<NAME>.<ATTRIBUTE>`**.
- Un `data` block normalmente se resuelve en el **plan**, pero se **difiere al apply** si depende de valores que aún no se conocen (recursos que van a cambiar en ese mismo plan) → aparece como `(known after apply)`.
- `depends_on` en `data` fuerza orden explícito, pero **solo recomendado en 0.13+**.
- `data` soporta `count`, `for_each`, `provider` (con alias), y `precondition`/`postcondition` dentro de `lifecycle`, igual que `resource`.
- Existen data sources "solo-local" (`template_file`, `local_file`, `aws_iam_policy_document`) que no consultan infraestructura real, sino que generan/transforman datos localmente en cada operación.

## 4b. Resource addressing y referencias cruzadas

Fuentes: "Resource Address Reference", "References to Named Values"

### Resource Address (sintaxis)

Un **resource address** identifica cero o más instancias de recursos en la config. Formato:

```
[module path][resource spec]
```

**Module path** — dirige a un módulo dentro del árbol de módulos:

```
module.module_name[module index]
```

- `module`: keyword que indica un child module (no-root). Repetido = nesting (ej. `module.foo.module.bar`).
- `module_name`: nombre definido por el usuario.
- `[module index]` (opcional): índice para seleccionar una instancia de un `module` con múltiples instancias (`count`/`for_each`). Solo aplica desde **Terraform v0.13+** (antes un módulo no podía tener múltiples instancias).
- `module.foo` sin index → aplica a **todos** los recursos del módulo (o todas sus instancias, si el módulo tiene varias).
- Si se omite el module path → se refiere al **root module**.
- Ejemplo anidado: `module.foo[0].module.bar["a"]`.

**Resource spec** — dirige a una instancia de recurso concreta dentro del módulo seleccionado:

```
resource_type.resource_name[instance index]
```

- `resource_type`: tipo del recurso.
- `resource_name`: nombre definido por el usuario (el label del bloque `resource`).
- `[instance index]` (opcional): índice para seleccionar una instancia de un recurso con `count`/`for_each`.
- Desde **Terraform v0.12+**: un resource spec sin prefijo de module path solo hace match con recursos del **root module** (antes hacía match con cualquier módulo descendiente que tuviera el mismo type+name — comportamiento legacy).

**Índices:**

- `[N]`: índice numérico **base 0** para recursos con `count`. Omitir el índice cuando `count > 1` referencia **todas** las instancias.
- `["INDEX"]`: clave alfanumérica para recursos con `for_each`.

Ejemplos:

```hcl
resource "aws_instance" "web" {
  count = 4
}
```
- `aws_instance.web[3]` → solo la última instancia.
- `aws_instance.web` → las 4 instancias.

```hcl
resource "aws_instance" "web" {
  for_each = tomap({
    "terraform" = "value1"
    "resource"  = "value2"
    "indexing"  = "value3"
    "example"   = "value4"
  })
}
```
- `aws_instance.web["example"]` → solo esa instancia (resuelve a `"value4"`).

### References to Named Values (tipos de valores nombrados)

Terraform expone varios tipos de **named values** que puedes usar como expresiones o combinar con otras:

| Tipo | Sintaxis | Notas |
|---|---|---|
| **Resources** | `<RESOURCE_TYPE>.<NAME>` | Ver detalle abajo — cambia según `count`/`for_each` |
| **Input variables** | `var.<NAME>` | Terraform convierte automáticamente al `type` declarado |
| **Local values** | `local.<NAME>` | Pueden referenciar otros locals (sin ciclos) |
| **Child module outputs** | `module.<MODULE_NAME>.<OUTPUT_NAME>` | Objeto / mapa / lista según `count`/`for_each` del módulo |
| **Data sources** | `data.<DATA_TYPE>.<NAME>` | Igual que resources, con prefijo `data.` |
| **Filesystem/workspace info** | `path.module`, `path.root`, `path.cwd`, `terraform.workspace` | Ver detalle abajo |
| **Block-local values** | `count.index`, `each.key`/`each.value`, `self` | Solo dentro de contextos concretos |

⚠️ No son objetos reales: hay que usarlos **tal cual están escritos** (no se puede hacer `for` sobre `aws_instance` como si fuera la colección de todos los recursos de ese tipo).

**Resources (`<TYPE>.<NAME>`):**

- Sin `count`/`for_each` → la referencia es un **objeto** (atributos accesibles con notación de punto o corchetes).
- Con `count` → la referencia es una **lista de objetos** (una por instancia).
- Con `for_each` → la referencia es un **mapa de objetos**.
- Cualquier named value que no encaje con los otros patrones se interpreta como referencia a un **managed resource**.

**Input variables (`var.<NAME>`):**

- Si la variable tiene `type` declarado, Terraform **convierte automáticamente** el valor dado al type constraint — puedes asumir que `var.X` siempre cumple ese tipo.
- Si defines un `object` type con atributos concretos, **solo esos atributos** estarán disponibles, aunque el caller pase un objeto con más atributos.

**Local values (`local.<NAME>`):** pueden referenciar otros locals (incluso del mismo bloque `locals`), evitando dependencias circulares.

**Child module outputs (`module.<NAME>`):**

- Sin `count`/`for_each` en el bloque `module` → objeto con un atributo por cada output del child module. Acceso: `module.<NAME>.<OUTPUT_NAME>`.
- Con `for_each` → **mapa** de objetos (keys = las del `for_each`).
- Con `count` → **lista** de objetos.

**Data sources (`data.<TYPE>.<NAME>`):** igual que resources, con el prefijo `data.` (lista con `count`, mapa con `for_each`, objeto si ninguno).

**Filesystem / workspace info:**

- `path.module`: path del módulo donde está la expresión. **No recomendado en write operations** — módulos locales invocados varias veces comparten el mismo directorio fuente, pudiendo sobrescribirse y causar race conditions.
- `path.root`: path del root module de la config.
- `path.cwd`: path absoluto del directorio de trabajo **original**, antes de aplicar `-chdir`. Preferir `path.root`/`path.module` cuando sea posible.
- `terraform.workspace`: nombre del workspace actualmente seleccionado.
- ⚠️ Usar estos valores con cuidado — afectan la **portabilidad/reusabilidad** de un módulo (ej. usar `path.cwd` directamente en un argumento puede hacer que Terraform detecte cambios al aplicar desde otro directorio/máquina). Recomendación: usarlos solo en el **root module** (salvo `path.module`); en módulos compartidos, mejor exponer una **input variable** para que el caller decida el prefijo (pudiendo usar `terraform.workspace` si quiere):

```hcl
module "example" {
  name_prefix = "app-${terraform.workspace}"
}
```

**Block-local values** (solo dentro de contextos concretos):

- `count.index`: dentro de resources con `count`.
- `each.key` / `each.value`: dentro de resources con `for_each`.
- `self`: dentro de bloques `provisioner`/`connection`.
- No son input variables — son nombres temporales/locales al contexto del bloque.

### Referencias a atributos de un resource — sintaxis detallada

```hcl
resource "aws_instance" "example" {
  ami           = "ami-abc123"
  instance_type = "t2.micro"

  ebs_block_device {
    device_name = "sda2"
    volume_size = 16
  }
  ebs_block_device {
    device_name = "sda3"
    volume_size = 20
  }
}
```

- Argumento configurado: `aws_instance.example.ami`.
- Atributo exportado (computado): `aws_instance.example.id` (misma sintaxis que un argumento).
- Argumentos de **nested blocks repetidos** (`ebs_block_device`) → **splat expression**: `aws_instance.example.ebs_block_device[*].device_name` (lista de todos los `device_name`).
- Nested blocks con **key** (ej. `device "foo" { ... }`) → sintaxis de índice: `aws_instance.example.device["foo"].size`.
- Para un mapa de valores de nested blocks con key → `for` expression: `{for k, device in aws_instance.example.device : k => device.size}`.

**Con `count`** (el resource se convierte en **lista** de instancias):

- `aws_instance.example[*].id` → lista de todos los ids.
- `aws_instance.example[0].id` → id de la primera instancia.

**Con `for_each`** (el resource se convierte en **mapa** de instancias):

- `aws_instance.example["a"].id` → id de la instancia con key `"a"`.
- `[for value in aws_instance.example : value.id]` → lista de todos los ids (`for` expression).
- ⚠️ Los **splat expressions no aplican directamente** a recursos con `for_each` (actúan sobre listas) — usar `values(aws_instance.example)[*].id` para convertir el mapa en lista primero.

### Sensitive Resource Attributes

- Un provider puede marcar ciertos atributos como **sensitive** en su schema → Terraform muestra `(sensitive value)` en vez del valor real al renderizar un plan.
- Se comporta igual que una **input variable `sensitive = true`**: Terraform oculta el valor y cualquier valor **derivado** de él también se marca sensible (desde **Terraform v0.15**; antes solo se ocultaba el valor directo).
- Si usas un valor sensible de un resource attribute en un **output**, Terraform **exige** marcar ese output también como `sensitive = true`.
- ⚠️ Terraform **sigue guardando** los valores sensibles **en el state**, en texto plano — cualquiera con acceso al state puede verlos.

### Values Not Yet Known (`(known after apply)`)

- Durante `plan`, algunos atributos **no se pueden conocer todavía** (los decide el sistema remoto al crear el objeto, ej. un ID autogenerado) → Terraform usa un **placeholder de valor desconocido (unknown)**.
- El lenguaje maneja unknown values automáticamente en expresiones (ej. combinar un valor conocido con uno unknown → resultado unknown).

**Casos con efecto significativo:**

- **`count`** no puede ser unknown — debe evaluarse en la fase de plan para saber cuántas instancias crear.
- Si un `data` block usa valores unknown en su config → **no se puede leer en plan**, se difiere a apply, y su resultado también es unknown.
- Unknown asignado a un argumento de un bloque `module` → las referencias a esa input variable dentro del child module usan ese valor unknown.
- Unknown en el `value` de un `output` → las referencias a ese output en el módulo padre usan ese valor unknown.
- Terraform intenta validar tipos de unknown values cuando puede, pero un uso incorrecto puede no detectarse hasta el **apply**, haciendo que falle en ese momento.
- En el output de `terraform plan`, los unknown values se muestran como **`(known after apply)`**.

---

### 💡 Puntos clave para el examen (4b)

- Resource address: `[module path][resource_type.resource_name[index]]`. Sin module path → root module (desde v0.12+, ya no hace match "en cualquier módulo").
- Índices: `[N]` (base 0, para `count`) vs `["KEY"]` (para `for_each`).
- `resource`/`data` sin `count`/`for_each` → objeto; con `count` → **lista**; con `for_each` → **mapa**.
- Splat (`[*]`) funciona sobre **listas** (`count`) — con `for_each` (mapa) hay que usar `values(...)[*]` o un `for` expression.
- `var.<NAME>` se auto-convierte al `type` declarado; solo expone los atributos definidos en el type constraint (si es `object`).
- `path.cwd`/`path.module`/`terraform.workspace`: cuidado con portabilidad — usar `path.module` libremente, el resto mejor solo en root module.
- Atributos `sensitive` de un provider: ocultos en plan/apply, valores derivados también se ocultan (desde v0.15) — pero **quedan en claro en el state**. Exportarlos en un `output` obliga a marcar ese output `sensitive`.
- `(known after apply)` = unknown value en plan; `count` nunca puede ser unknown.

## 4c. Variables y outputs

Fuentes: "Manage values in modules", "output block reference", "variable block reference"

### Visión general: cómo se comunican los módulos

- **Variables** (input): parametrizan el módulo — permiten que otros pasen valores custom en runtime.
- **Outputs**: exponen datos de un módulo hacia afuera (CLI, HCP Terraform, otras configs vía `terraform_remote_state`, o al módulo padre).
- **Locals**: definen y reutilizan expresiones **dentro** del módulo (no son ni input ni output).

```hcl
variable "instance_type" {
  type        = string
  description = "EC2 instance type for the web server"
  default     = "t2.micro"
}

resource "aws_instance" "web" {
  instance_type = var.instance_type
}
```

```hcl
locals {
  app_name = "${var.project_name}-${var.environment}"
}

resource "aws_instance" "example" {
  tags = { Name = local.app_name }
}
```

```hcl
output "instance_ip" {
  description = "Private IP address of the EC2 instance"
  value       = aws_instance.web.private_ip
}
```

### `variable` block — argumentos completos

```hcl
variable "<LABEL>" {
  type        = <TYPE>
  default     = <DEFAULT_VALUE>
  description = "<DESCRIPTION>"
  sensitive   = <true|false>
  nullable    = <true|false>
  ephemeral   = <true|false>
  const       = <true|false>
  deprecated  = "<STRING>"

  validation {
    condition     = <EXPRESSION>
    error_message = "<ERROR_MESSAGE>"
  }
}
```

El **label** debe ser único por módulo, y no puede ser un nombre reservado: `source`, `version`, `providers`, `count`, `for_each`, `lifecycle`, `depends_on`, `locals`.

| Argumento | Descripción | Tipo | Requerido |
|---|---|---|---|
| `type` | Type constraint del valor de la variable | Type constraint | Opcional (default: acepta cualquier tipo) |
| `default` | Valor por defecto — sin él, la variable es **obligatoria** | Expression | Opcional |
| `description` | Documentación desde el punto de vista del **consumidor** del módulo | String | Opcional |
| `validation` | Regla adicional que debe cumplir el valor (además del `type`) | Block | Opcional |
| `sensitive` | Oculta el valor en output de CLI | Boolean | Opcional (default `false`) |
| `nullable` | Permite asignar `null` a la variable | Boolean | Opcional (default `true`) |
| `ephemeral` | Evita guardar el valor en state/plan files | Boolean | Opcional (default `false`) — desde **v1.10** |
| `const` | Permite usar la variable en operaciones tempranas (ej. `init`) | Boolean | Opcional (default `false`) |
| `deprecated` | Mensaje de deprecación | String | Opcional — desde **v1.15** |

**`type`**: sin constraint → acepta cualquier tipo. Ayuda a documentar y da mejores mensajes de error si el consumidor pasa un tipo inválido.

**`default`**: si se define `type` **y** `default`, el default debe convertir al tipo especificado. El `default` requiere un **valor literal** — no puede referenciar otros objetos de la config.

**`description`**: escribirla desde la perspectiva del **consumidor** del módulo (para comentarios internos de mantenimiento, usar comentarios normales, no `description`).

**`validation`** _(detalle completo en 4g)_: bloque con `condition` (debe evaluar a `true`) y `error_message`. Se evalúa **al crear el plan**; si falla, Terraform lanza error y para la operación.

**`sensitive`**: oculta el valor en logs de plan/apply (`(sensitive value)`). Cualquier expresión que use una variable sensible **se vuelve sensible automáticamente**. Terraform **sigue guardando el valor en el state** en claro.

**`nullable`**: si `false`, la variable **no puede ser `null`**. Si `true` (default) y tiene `default`, se puede pasar explícitamente `null` para sobreescribir el default. En tipos de colección/estructura, se puede usar `null` en elementos anidados aunque la colección en sí no sea `null`.

**`ephemeral`** _(detalle completo en 4h)_: valor disponible durante el runtime, pero **omitido del state y plan files** — pensado para tokens/credenciales de vida corta. Contextos que pueden setear/referenciar variables ephemeral: otro `output` ephemeral, otra `variable` ephemeral, un **write-only argument**, un bloque de **ephemeral resource**, el bloque `provider`, o `provisioner`/`connection`. Si una expresión referencia una variable ephemeral, **esa expresión también se vuelve ephemeral** implícitamente.

**`const`**: si `true`, la variable solo puede tener un valor **constante y conocido** (no puede depender de resultados dinámicos de un plan) — se puede usar en los argumentos `source`/`version` de un bloque `module` (que se evalúan antes del plan).

**`deprecated`**: mensaje que Terraform muestra cuando el módulo **consumidor** setea esa variable (o el caller de un child module le pasa un valor). No se muestra dentro del propio módulo que la define.

### `output` block — argumentos completos

```hcl
output "<LABEL>" {
  type        = <TYPE>
  value       = <EXPRESSION>
  description = "<STRING>"
  sensitive   = <true|false>
  ephemeral   = <true|false>
  depends_on  = [<REFERENCE>]
  deprecated  = "<STRING>"

  precondition {
    condition     = <EXPRESSION>
    error_message = "<STRING>"
  }
}
```

4 propósitos principales de `output`:

1. Los **child modules** exponen atributos de resource a su módulo padre.
2. Los **root modules** muestran valores en el output de la CLI.
3. Otras configs con remote state pueden leer outputs del root module vía el data source **`terraform_remote_state`**.
4. Pasar info de una operación de Terraform a una herramienta de automatización.

| Argumento | Descripción | Tipo | Requerido |
|---|---|---|---|
| `type` | Type constraint del valor del output | Type constraint | Opcional |
| `value` | El valor que devuelve el output | Expression | **Requerido** |
| `description` | Descripción del propósito del output | String | Opcional |
| `sensitive` | Oculta el valor en output de CLI | Boolean | Opcional (default `false`) |
| `ephemeral` | Evita guardar el valor en state | Boolean | Opcional (default `false`) — desde **v1.10**, **solo en child modules** (no en root) |
| `depends_on` | Dependencias explícitas para este output | List | Opcional |
| `deprecated` | Mensaje de deprecación — **solo en child modules** | String | Opcional — desde **v1.15** |
| `precondition` | Condición a validar antes de calcular/guardar el output | Block | Opcional |

**`value`**: obligatorio; Terraform evalúa la expresión y **guarda el resultado en el state**.

**`sensitive`**: igual que en `variable` — Terraform muestra `(sensitive value)` en plan/apply. Si usas `terraform output -json` o `-raw`, Terraform **sí** muestra el valor sensible en texto plano. El valor **queda en el state en claro** igualmente. Si un output usa un valor sensible de un **resource attribute**, Terraform **exige** marcar ese output como `sensitive` también.

**`ephemeral`** _(detalle en 4h)_: solo en **child modules** (no en root module). Si `true`: el `value` del output debe venir de un **contexto ephemeral**, y solo se puede referenciar desde otros contextos ephemeral (otro output ephemeral, write-only argument, variable ephemeral, ephemeral resource, bloque `provider`, `provisioner`/`connection`).

**`depends_on`**: meta-argumento para forzar dependencia explícita — Terraform completa todas las operaciones del recurso upstream antes de calcular el output. Se recomienda comentar **por qué** hace falta, ya que normalmente `output` no necesita dependencias explícitas (Terraform ya las infiere de `value`).

**`deprecated`**: solo en child modules; Terraform muestra el mensaje al **consumidor** del módulo, no dentro del propio módulo. Se puede suprimir con `ignore_nested_deprecations` en el bloque `module` que llama.

**`precondition`** _(detalle completo en 4g)_: `condition` + `error_message`, igual que en `variable` `validation` — se evalúa al crear/aplicar el plan; si `condition` es `false`, error y se detiene la operación.

### Ejemplos destacados

```hcl
# Output accediendo a un child module
output "website_url" {
  value       = "https://${module.web_server.instance_ip_addr}"
  description = "The URL of the web server, starting with https://."
}

# Output sensible
output "db_password" {
  value       = aws_db_instance.db.password
  sensitive   = true
}

# Variable con validation
variable "image_id" {
  type = string
  validation {
    condition     = length(var.image_id) > 4 && substr(var.image_id, 0, 4) == "ami-"
    error_message = "The image_id value must be a valid AMI ID, starting with \"ami-\"."
  }
}

# Variables ephemeral (credenciales de corta vida, no van al state)
variable "access_key" {
  type      = string
  ephemeral = true
}
provider "aws" {
  access_key = var.access_key
}
```

---

### 💡 Puntos clave para el examen (4c)

- `variable` = input del módulo; `output` = valor expuesto hacia afuera; `local` = expresión reutilizable **interna**, ni input ni output.
- `variable` sin `default` → **obligatoria**; nombres reservados prohibidos como label (`count`, `for_each`, `lifecycle`, `depends_on`, `source`, `version`, `providers`, `locals`).
- `output.value` es **obligatorio**; se guarda siempre en el state (salvo `ephemeral`).
- `sensitive` en variable/output: oculta en CLI plan/apply, **pero se guarda en el state en claro** — `terraform output -json`/`-raw` sí muestra el valor real.
- `ephemeral` (v1.10+): valor no se persiste en state/plan; en **output solo vale en child modules**, nunca en root module.
- `nullable` (variable, default `true`): controla si se puede pasar `null` explícitamente.
- `const` (variable): permite usarla en `source`/`version` de un `module` block (operaciones tempranas, antes del plan).
- `validation` (variable) / `precondition` (output): mismo patrón `condition` + `error_message`, evaluados al generar el plan.
- `deprecated` (v1.15+): solo visible para el **consumidor** del módulo, no dentro del módulo que la define; solo aplica a variables/outputs de **child modules**.
- `depends_on` en `output` es un meta-argumento explícito — raramente necesario (Terraform ya infiere dependencias de `value`).

## 4d. Tipos complejos

Fuente: "Complex Types"

Un **complex type** agrupa varios valores en un único valor. Dos categorías:

- **Collection types**: agrupan valores **similares** (todos del mismo tipo).
- **Structural types**: agrupan valores potencialmente **distintos** entre sí, según un schema.

### Collection types

| Tipo | Descripción |
|---|---|
| `list(...)` | Secuencia de valores identificados por números consecutivos empezando en 0. `list` a secas = shorthand de `list(any)` (legacy, mejor usar la forma completa) |
| `map(...)` | Colección de valores identificados por una etiqueta string. `map` a secas = shorthand de `map(any)` |
| `set(...)` | Colección de valores **únicos**, sin identificador secundario ni orden |

- Todos los elementos de una colección deben ser del **mismo tipo** (el "element type", argumento del constructor). `list(string)` ≠ `list(number)`.
- Sintaxis de mapas: `{}`, `:` o `=` como delimitador clave-valor. `{ "foo": "bar" }` ≡ `{ foo = "bar" }`. Las claves van entre comillas si empiezan por número, tienen espacios o caracteres especiales.
- `terraform fmt` ignora los `:` en mapas, pero alinea verticalmente los `=`.

### Structural types

| Tipo | Descripción |
|---|---|
| `object(...)` | Colección de atributos **con nombre**, cada uno con su propio tipo. Schema: `{ <KEY> = <TYPE>, ... }` |
| `tuple(...)` | Secuencia de elementos identificados por posición (0, 1, 2...), cada uno con su propio tipo. Schema: `[<TYPE>, <TYPE>, ...]` |

- `object`: el valor debe contener **todos** los keys del schema, con el tipo correcto. Si el valor tiene **atributos extra**, siguen siendo válidos, pero se **descartan** en la conversión de tipo (ej. un `aws_vpc.example_vpc` con más atributos igual encaja en `object({ id=string, cidr_block=string })`, descartando el resto).
- `tuple`: el valor debe tener **exactamente** el mismo nº de elementos, cada uno del tipo correspondiente a su posición.

Ejemplo:
```hcl
object({ name = string, age = number })  # matches { name = "John", age = 52 }
tuple([string, number, bool])            # matches ["a", 15, true]
```

### Conversión de complex types

Terraform convierte automáticamente entre tipos "similares" cuando es posible:

- **Objects y maps son similares**: un map (o un object más grande) se puede convertir a `object` si tiene **al menos** las keys requeridas — atributos extra se descartan (map → object → map puede perder datos).
- **Tuples y lists son similares**: una list solo se convierte a `tuple` si tiene **exactamente** el nº de elementos requerido.
- **Sets** son "casi similares" a tuples/lists:
  - list/tuple → set: se descartan duplicados y se **pierde el orden**.
  - set → list/tuple: orden **arbitrario** (si los elementos son strings, orden lexicográfico; para otros tipos, sin garantía de orden).
- También se convierten recursivamente los **elementos** dentro del complex type (según reglas de conversión de tipos primitivos).

Ejemplo: si un argumento requiere `list(string)` y se pasa la tupla `["a", 15, true]`, Terraform la transforma a `["a", "15", "true"]`. Si la conversión es imposible (ej. pasar un `tuple` donde se requiere `string`), Terraform lanza un **type mismatch error**.

### El tipo `any` (dynamic type)

⚠️ `any` **rara vez** es el type constraint correcto — no usarlo solo para "evitar especificar un tipo". Es un placeholder para un tipo **aún por decidir**; Terraform intenta encontrar un único tipo real que lo sustituya.

- Uso apropiado: cuando el valor se pasa **directamente a otro sistema sin inspeccionar su contenido** (ej. `jsonencode(var.settings)` con `variable "settings" { type = any }`).
- Uso incorrecto: si el módulo accede a elementos/atributos del valor, o espera que sea string/number/etc.

**`any` en colecciones** (`list(any)`, `map(any)`, `set(any)`): Terraform busca **un único tipo de elemento** que sirva para todos los elementos dados.
- `["a", "b", "c"]` (tuple de strings) con `list(any)` → resuelve a `list(string)`.
- `["a", 1, "b"]` con `list(any)` → resuelve a `list(string)` (conversión de tipos primitivos → `["a", "1", "b"]`).
- `["a", [], "b"]` con `list(any)` → **error**: no hay un tipo único al que convertir un string y una tupla vacía.

### Optional Object Type Attributes

Se puede marcar un atributo de un `object` como **opcional** con el modificador `optional(...)`, evitando el error por defecto de "falta el atributo":

```hcl
variable "with_optional_attribute" {
  type = object({
    a = string                # requerido
    b = optional(string)      # opcional, default null
    c = optional(number, 127) # opcional, con valor por defecto 127
  })
}
```

- `optional(TYPE)`: tipo requerido; si no se pasa, valor por defecto = `null` del tipo correspondiente.
- `optional(TYPE, DEFAULT)`: segundo argumento opcional = valor por defecto (debe ser compatible con `TYPE`).
- Un atributo opcional **con default no-null** nunca será `null` dentro del módulo receptor — Terraform sustituye el default tanto si el caller **omite** el atributo como si lo pasa explícitamente como `null`.
- Los defaults se aplican **top-down** en estructuras anidadas: primero el default del propio `optional`, luego los defaults anidados dentro de ese valor.
- Se puede usar un operador condicional con `null` como una de las ramas, para **dejar dinámicamente sin definir** un atributo opcional (y así heredar su default):
  ```hcl
  website = {
    error_document = var.legacy_filenames ? "ERROR.HTM" : null
  }
  ```

---

### 💡 Puntos clave para el examen (4d)

- **Collection** (list/map/set) = elementos del **mismo tipo**. **Structural** (object/tuple) = elementos de **tipos distintos**, definidos por un schema.
- `list`/`map` a secas = shorthand legacy de `list(any)`/`map(any)` — usar forma completa en código nuevo.
- Conversión automática: map↔object (por keys, puede perder atributos extra), list↔tuple (mismo nº elementos), set↔list/tuple (pierde orden/duplicados).
- `any`: placeholder de tipo, **casi nunca correcto** salvo paso directo sin inspección (ej. `jsonencode`).
- `optional(TYPE, DEFAULT)` en `object`: permite atributos opcionales; con default no-null, **garantiza no-null** dentro del módulo (sustituye tanto ausencia como `null` explícito).

---

## 4e. Expresiones y funciones built-in

Fuente: "Built-in Functions"

### Sintaxis de llamada a función

```hcl
max(5, 12, 9)
```

Nombre de función + argumentos separados por comas entre paréntesis. Terraform **no permite definir funciones propias** en el lenguaje de configuración — pero un **provider** puede exponer funciones propias (**provider-defined functions**), invocadas como `provider::<local_name>::<function>(...)`:

```hcl
provider::terraform::encode_tfvars({
  example = "Hello!"
})
```

### Experimentar con funciones: `terraform console`

```
$ terraform console
> max(5, 12, 9)
12
```

Consola interactiva para probar expresiones y funciones sin tocar infraestructura real.

### Categorías de funciones built-in (con ejemplos representativos — no hace falta memorizar todas, sino saber qué categoría resuelve qué problema)

**Numéricas**: `ceil`, `floor`, `log`, `max`, `min`, `parseint`, `pow`, `signum`.

**String**: `chomp`, `endswith`, `startswith`, `format`, `formatlist`, `indent`, `join`, `lower`/`upper`, `regex`, `regexall`, `replace`, `split`, `strcontains`, `strrev`, `substr`, `templatestring`, `title`, `trim`/`trimprefix`/`trimsuffix`/`trimspace`.

**Collection**: `alltrue`, `anytrue`, `chunklist`, `coalesce` (primer valor no-null/no-vacío), `coalescelist`, `compact` (quita null/"" de una list), `concat`, `contains`, `distinct` (quita duplicados), `element`, `flatten`, `index`, `keys`, `length`, `lookup` (valor de un map por key), `matchkeys`, `merge` (combina maps/objects), `one` (único elemento, o error si hay más de uno), `range`, `reverse`, `setintersection`/`setproduct`/`setsubtract`/`setunion`, `slice`, `sort`, `sum`, `transpose`, `values`, `zipmap`.
- ⚠️ `list` y `map` como **funciones constructoras** están **deprecadas desde 0.12** → usar `tolist`/`tomap`.

**Encoding**: `base64decode`/`base64encode`, `base64gzip`, `csvdecode`, `jsondecode`/`jsonencode`, `textdecodebase64`/`textencodebase64`, `urlencode`, `yamldecode`/`yamlencode`.

**Filesystem**: `abspath`, `dirname`, `basename`, `pathexpand` (expande `~`), `file` (lee contenido como string), `fileexists`, `fileset`, `filebase64`, `templatefile` (renderiza un archivo como template con variables).

**Date/time**: `formatdate`, `plantimestamp` (timestamp UTC en el momento del **plan**), `timeadd`, `timecmp`, `timestamp` (timestamp UTC actual — ¡ojo, cambia en cada run!).

**Hash/crypto**: `base64sha256`/`base64sha512`, `bcrypt`, `filebase64sha256`/`filebase64sha512`, `filemd5`, `filesha1`/`filesha256`/`filesha512`, `md5`, `rsadecrypt`, `sha1`/`sha256`/`sha512`, `uuid`, `uuidv5`.

**IP network**: `cidrhost`, `cidrnetmask`, `cidrsubnet`, `cidrsubnets`.

**Type conversion**: `can` (evalúa una expresión y devuelve `true`/`false` según si dio error), `try` (evalúa varias expresiones en orden, devuelve la primera sin error), `tobool`/`tolist`/`tomap`/`tonumber`/`toset`/`tostring`, `type` (devuelve el tipo de un valor), `sensitive`/`nonsensitive` (marca/desmarca un valor como sensible), `issensitive`, `ephemeralasnull` (convierte un valor ephemeral en `null`).

**Terraform-specific (provider `terraform`)**: `provider::terraform::encode_tfvars`, `provider::terraform::decode_tfvars`, `provider::terraform::encode_expr`.

### Soporte por tipo de archivo de configuración

- Los archivos `.tf` estándar soportan **todas** las funciones.
- Otros tipos de configuración (ej. **Terraform Stacks**: `.tfcomponent.hcl`, `.tfdeploy.hcl`) solo soportan un **subconjunto** de funciones — algunas funciones (ej. `signum`, `replace`, `index`, `lookup`, `length`, `sum`, muchas de hash/crypto, filesystem, date/time salvo `formatdate`/`timeadd`) son **solo `.hcl`** (configuración estándar), no disponibles en Stacks.

---

### 💡 Puntos clave para el examen (4e)

- No se pueden definir funciones propias en HCL — solo usar las built-in o las expuestas por un **provider** (`provider::<name>::<function>`).
- `terraform console` = forma de experimentar con expresiones/funciones sin aplicar cambios.
- `can(expr)` → bool (¿dio error o no?); `try(expr1, expr2, ...)` → devuelve el primer resultado sin error.
- `coalesce` (primer no-null/no-vacío) vs `coalescelist` (primera lista no vacía).
- `lookup(map, key, default)` para leer de un map con fallback; `merge()` para combinar maps/objects.
- `list()`/`map()` como funciones están **deprecadas** → `tolist()`/`tomap()`.
- `timestamp()` cambia en cada ejecución (usar con cuidado, puede generar diffs infinitos); `plantimestamp()` es estable durante un mismo plan.
- `sensitive()`/`nonsensitive()`/`issensitive()` para gestionar el marcado de sensibilidad de un valor programáticamente.

---

## 4f. Dependencias: `depends_on` y `create_before_destroy`

Fuente oficial en el exam content list: **"Resource Graph"** (= la página "Dependency Graph", ya documentada al completo en [03-workflow.md](03-workflow.md) → sección 3d — no la repito aquí para no duplicar).

### `depends_on` — resumen (detalle completo en 3d)

- Terraform infiere dependencias automáticamente a partir de **interpolaciones** (referencias a atributos de otro resource dentro de la config) → esto genera la mayoría de los edges del grafo de dependencias.
- `depends_on` es un **meta-argumento explícito** para casos donde una dependencia **no es visible** en las referencias de la config (ej. un resource que depende de un efecto secundario de otro, sin usar ninguno de sus atributos directamente).
- En la construcción del grafo (ver 3d, paso 3), las dependencias explícitas de `depends_on` se procesan **antes** que las interpoladas, creando edges entre resources.
- Uso típico: `depends_on = [aws_iam_role_policy.example]` dentro de un `resource`, `data`, o `module`.
- Regla general: usar `depends_on` **solo cuando haga falta** — abusar de él genera grafos más rígidos/lentos (menos paralelismo) y puede ocultar relaciones que deberían expresarse como referencias reales.

### `create_before_destroy` — ⚠️ no venía en la documentación pegada

_(Esto es conocimiento general de Terraform, no proviene de la doc pegada en esta sesión — para el examen te recomiendo buscar y pegar la página oficial "Resource Behavior" o "Meta-Arguments: lifecycle" de developer.hashicorp.com para completar apuntes 100% basados en fuente.)_

- Es un argumento dentro del bloque **`lifecycle`** de un `resource`:

```hcl
resource "aws_instance" "example" {
  # ...
  lifecycle {
    create_before_destroy = true
  }
}
```

- Comportamiento **por defecto** de Terraform al reemplazar un resource (cambio que fuerza recreate): **destroy** primero el objeto viejo, **luego create** el nuevo.
- Con `create_before_destroy = true`, Terraform invierte el orden: **crea primero** el reemplazo, y **solo si tiene éxito**, destruye el objeto antiguo. Útil para evitar downtime (ej. un load balancer o un recurso referenciado por otros que no puede desaparecer aunque sea un instante).
- Implicaciones a tener en cuenta:
  - Puede requerir que el nombre/identificador del recurso **no colisione** con el antiguo mientras coexisten ambos (algunos recursos tienen nombres únicos y esto puede fallar si no se usa algo como `name_prefix` en vez de `name` fijo).
  - Otros recursos que dependen del que se recrea pueden necesitar también `create_before_destroy` para evitar quedarse referenciando temporalmente un recurso que va a desaparecer.
  - Interactúa con el grafo de dependencias visto en 3d: en el paso de "split destroy/create" (paso 8), Terraform separa el nodo en dos (uno de destroy, otro de create) precisamente porque el orden puede diferir — `create_before_destroy` es lo que determina **cuál de los dos va primero**.

---

### 💡 Puntos clave para el examen (4f)

- `depends_on`: fuerza una dependencia explícita **no** detectable por referencias normales en la config; usarlo con moderación.
- Terraform infiere dependencias automáticamente casi siempre — `depends_on` es la excepción, no la norma.
- `create_before_destroy` (dentro de `lifecycle`): invierte el orden por defecto (destroy→create) a **create→destroy**, para minimizar downtime en reemplazos.
- Ambos mecanismos afectan directamente cómo Terraform construye y recorre el **resource graph** (ver 3d): `depends_on` añade edges explícitos; `create_before_destroy` decide el orden entre el nodo "destroy" y el nodo "create" cuando un resource se divide en dos por un replace.
- ⚠️ Repasa la página oficial "Resource Behavior" / "lifecycle meta-argument" en la doc de Terraform si quieres el detalle 100% oficial de `create_before_destroy` (no estaba en el material pegado hasta ahora).

## 4g. Custom conditions (`validation`, `precondition`, `postcondition`, `check`)

Fuente: "Validate your configuration"

### Para qué sirve la validación

- Verificar que las input variables cumplen requisitos concretos.
- Evitar que outputs incorrectos se escriban en el state.
- Asegurar que resources/data sources quedan bien configurados tras el apply.
- Verificar el comportamiento **general** de la infraestructura.
- Documentar asunciones sobre la infraestructura (ayuda a mantenedores futuros).
- Con HCP Terraform: verificación **continua** de la infraestructura.

Cuando falla una validación, Terraform da contexto útil en el error. Cada tipo de validación se evalúa en un momento distinto del ciclo de ejecución, y puede **bloquear** la operación o solo avisar con un **warning**.

**Requisitos de versión:**

| Feature | Versión mínima |
|---|---|
| Input variable `validation` | Terraform 0.13+ |
| `precondition` / `postcondition` | Terraform 1.2+ |
| `check` blocks | Terraform 1.5+ |

### 4 tipos de validación — resumen

| Tipo | Dónde | Cuándo se evalúa | ¿Bloquea? |
|---|---|---|---|
| **Input variable validation** | dentro de `variable` | al crear el plan | Sí |
| **Precondition** | `lifecycle` de `resource`/`data`/`output` | antes de crear/leer el resource/data/output | Sí |
| **Postcondition** | `lifecycle` de `resource`/`data` | después de plan+apply (o de leer un data source) | Sí |
| **`check` block** | bloque independiente `check {}` | al final de plan/apply | **No** — solo warning |

### Input variable validation

```hcl
variable "image_id" {
  type = string
  validation {
    condition     = length(var.image_id) > 4 && substr(var.image_id, 0, 4) == "ami-"
    error_message = "The image_id value must be a valid AMI id, starting with \"ami-\"."
  }
}
```

- Si `condition` es `false` → Terraform da error con `error_message` y **detiene la operación**.
- Útil para: formato de valores, rangos aceptables, o **imponer convenciones de la organización** (ej. naming conventions) — evita depender solo de que la API del provider dé error.

### Preconditions

- Se evalúan **al crear el plan**, y tienen **precedencia** sobre errores de argumentos que lance el propio provider en resources/data/outputs mal configurados.
- En `resource`:
  ```hcl
  resource "aws_instance" "example" {
    ami = data.aws_ami.example.id
    lifecycle {
      precondition {
        condition     = data.aws_ami.example.architecture == "x86_64"
        error_message = "The selected AMI must be for the x86_64 architecture."
      }
    }
  }
  ```
- También en `output`:
  ```hcl
  output "instance_public_ip" {
    value = aws_instance.web.public_ip
    precondition {
      condition     = length([for rule in aws_security_group.web.ingress : rule if rule.to_port == 80 || rule.to_port == 443]) > 0
      error_message = "Security group must allow HTTP (port 80) or HTTPS (port 443) traffic."
    }
  }
  ```
- Si falla → error con `error_message`, se detiene la operación.

### Postconditions

- Se evalúan **después** de planificar/aplicar cambios a un resource, o después de **leer** un data source.
- Dentro de `lifecycle`, usan `self` para referenciar el propio resource/data:
  ```hcl
  data "aws_ami" "example" {
    # ...
    lifecycle {
      postcondition {
        condition     = self.tags["Component"] == "nomad-server"
        error_message = "tags[\"Component\"] must be \"nomad-server\"."
      }
    }
  }
  ```
- Ayudan a evitar **cambios en cascada** a otros recursos dependientes, detectando errores justo tras crear/leer el objeto.
- Sirven como "guardarraíles estáticos". Para verificar infra dinámicamente contra condiciones externas/cambiantes, mejor usar **`check` blocks** (corren después de los postconditions, como paso final).

### Precondition vs Postcondition — cómo elegir

- **Precondition** = verificar **asunciones** antes de crear el bloque objetivo (ej. "esta AMI debe ser x86_64" antes de crear la instancia). Ayuda a futuros mantenedores a entender qué valores debería aceptar un resource/output/data.
- **Postcondition** = verificar **garantías** tras crear el resource / leer el data source (ej. "la instancia debe tener DNS privado asignado"). Ayuda a entender qué comportamientos hay que preservar al cambiar la config.
- Consideraciones:
  - Si un resource tiene muchas dependencias, puede ser más pragmático poner **un postcondition** en ese resource en vez de preconditions en cada dependencia.
  - Si precondition y postcondition viven en **módulos distintos**, puede convenir tener ambos — cada módulo se verifica a sí mismo según evoluciona independientemente.

### `check` blocks

- Se ejecutan como **último paso** de un `plan` o `apply`, tras planificar/provisionar la infra.
- Si la `assert` falla → Terraform da un **warning** y **continúa** la operación (a diferencia de `validation`/`precondition`/`postcondition`, que bloquean).
- Usos: validar resources/data/variables/outputs, validar el comportamiento **global** de la infra, verificar sin bloquear operaciones, y habilitar **continuous validation** en HCP Terraform.

```hcl
check "health_check" {
  data "http" "terraform_io" {
    url = "https://www.terraform.io"
  }

  assert {
    condition     = data.http.terraform_io.status_code == 200
    error_message = "${data.http.terraform_io.url} returned an unhealthy status code"
  }
}
```

- **Continuous validation en HCP Terraform**: con health checks habilitados en un workspace, HCP Terraform revalida periódicamente `check` blocks, preconditions y postconditions (ej. monitorizar la validez de un certificado de API gateway), alertando cuando falla algo — sin esperar al siguiente `apply`.

### Orden de validación (order of validation)

1. **Input variable validations** — inmediatamente, **antes** de generar el plan.
2. **Preconditions** — después de generar el plan, **antes** de crear el resource/data/output.
3. **Postconditions** — después de planificar y aplicar cambios.
4. **`check` blocks** — al final de plan/apply, y en cada health assessment de HCP Terraform.

- El orden exacto de checks/pre/postconditions puede depender de si Terraform conoce el valor de la condición **antes o después del apply**:
  - Si el valor está disponible antes de aplicar → se valida en la fase de **plan**.
  - Si el valor solo se conoce tras aplicar (ej. un ID que asigna AWS al arrancar una instancia) → se difiere la validación a la fase de **apply**.
- Durante el apply: una **precondition fallida** impide implementar las acciones planificadas para ese resource/data/output. Una **postcondition fallida** detiene el procesamiento y evita acciones downstream que dependan de ese resource/data — pero **no deshace** acciones ya realizadas.

### Error messages

- `validation`, `precondition`, `postcondition` y `check` **requieren** el argumento `error_message`.
- `error_message` acepta cualquier expresión que evalúe a **string** (literal, heredoc, template). Se puede usar `format()` para convertir `null`/`list`/`map` en un string formateado.
- Se recomienda escribir mensajes como **frases completas**, en un estilo similar a los propios mensajes de error de Terraform.

---

### 💡 Puntos clave para el examen (4g)

- 4 mecanismos: **`validation`** (en `variable`, antes del plan), **`precondition`** (en `lifecycle`, tras generar plan/antes de crear), **`postcondition`** (en `lifecycle`, tras plan+apply o tras leer un `data`), **`check`** (bloque independiente, al final — **solo warning, no bloquea**).
- Versiones mínimas: `validation` (0.13+), pre/postcondition (1.2+), `check` (1.5+).
- `postcondition` usa **`self`** para referenciar el propio resource/data.
- `precondition` tiene **precedencia** sobre errores de argumento del propio provider.
- `check` es el **único** de los 4 que **no bloquea** la operación — solo emite warning. Es el que habilita **continuous validation** en HCP Terraform.
- Orden: variable validation → precondition → apply → postcondition → check.
- Todos requieren `error_message` (expresión que evalúe a string).

---

## 4h. Datos sensibles, Vault, ephemeral values y write-only arguments

Fuente: "Manage sensitive data in your configuration"

### El problema

- Terraform necesita a veces datos sensibles (credenciales cloud, tokens de API, secretos) para provisionar infra.
- Si pones esos valores directamente en la config, Terraform los **guarda en el state y en los plan files**.
- En local, el state es un **archivo en texto plano** — trátalo como dato sensible (excluir de Git, seguir buenas prácticas de seguridad de state).
- Con **remote state**, Terraform solo mantiene el state en memoria mientras lo usa activamente; se puede cifrar at-rest según el backend (ej. **HCP Terraform cifra el state at-rest automáticamente** y lo protege con TLS in-transit).

**Requisitos de versión:**

| Feature | Versión mínima |
|---|---|
| `sensitive` en `variable`/`output` | Terraform 0.15+ |
| `ephemeral` en variables/child module outputs, o bloque `ephemeral` | Terraform 1.10+ |
| Write-only arguments en managed resources | Terraform 1.11+ |

### Dos preguntas para decidir el enfoque

1. ¿Quiero **ocultar** el valor del output de CLI/UI? → `sensitive`.
2. ¿Quiero que Terraform **no lo guarde en absoluto** (ni state ni plan)? → `ephemeral`.

Se pueden combinar `sensitive` + `ephemeral` para ambos beneficios a la vez.

### `sensitive` — ocultar sin dejar de guardar

```hcl
variable "database_password" {
  type      = string
  sensitive = true
}

output "connection_string" {
  value     = "postgresql://${var.db_username}:${var.database_password}@..."
  sensitive = true
}
```

- Terraform **redacta** (`(sensitive value)`) el valor en logs de CLI y en la UI de HCP Terraform.
- Cualquier **expresión que referencie** una variable/output sensible se vuelve sensible **automáticamente** (ej. el argumento `password` de un resource que usa `var.database_password` se redacta también en el plan).
- ⚠️ Terraform **sigue guardando** estos valores **en el state y en el plan file**, en claro — cualquiera con acceso a esos archivos los puede ver.
- ⚠️ `terraform output -json` o `-raw` **sí muestran** el valor real en texto plano.

### `ephemeral` — no guardar en absoluto

- Los **valores ephemeral** están disponibles **en tiempo de ejecución**, pero Terraform los **omite completamente** de state y plan files.
- Como no se guardan, si quieres preservar un valor generado (ej. una password random), tienes que **capturarlo** en otro resource/output tú mismo.
- 3 formas de definir valores ephemeral:
  1. El **argumento `ephemeral`** en `variable` o en `output` de **child modules**.
  2. El **bloque `ephemeral`** (declara un **ephemeral resource** temporal).
  3. Un **write-only argument** en un managed resource.

**Dónde se pueden referenciar valores ephemeral** (contextos restringidos):

- Bloque `locals`.
- `variable` con argumento `ephemeral`.
- `output` de **child module** con argumento `ephemeral` (⚠️ **no permitido en el root module**).
- Write-only argument de un managed resource.
- Bloque `ephemeral`.
- Bloque `provider` (para configurar el provider).
- Bloques `provisioner` / `connection`.

**Ejemplo — variable ephemeral usada en un provider:**

```hcl
variable "api_token" {
  type      = string
  sensitive = true
  ephemeral = true
}

provider "example" {
  api_token = var.api_token
}
```

**Ejemplo — output ephemeral en child module (pasar credenciales sin persistir):**

```hcl
output "session_token" {
  value     = ephemeral.auth_provider.main.token
  ephemeral = true
  sensitive = true
}
```

### Bloque `ephemeral` (ephemeral resources)

- Declara un **recurso temporal** que solo existe durante la operación actual de Terraform — no se guarda en state ni plan.
- Ideal para datos sensibles/temporales que no quieres persistir (passwords temporales, conexiones a otros sistemas).
- Cada provider define sus propios ephemeral resources disponibles (consultar el Terraform Registry).

```hcl
ephemeral "random_password" "db_password" {
  length           = 16
  override_special = "!#$%&*()-_=+[]{}<>:?"
}
```

- Solo se puede referenciar en **otros contextos ephemeral**, como un write-only argument:

```hcl
resource "aws_db_instance" "example" {
  # ...
  password_wo         = ephemeral.random_password.db_password.result
  password_wo_version = 1
}
```

- Ni el bloque `ephemeral` ni los write-only arguments persisten fuera del run actual — el valor de `ephemeral.random_password.db_password.result` queda **totalmente omitido** de state/plan.

### Write-only arguments

- Permiten pasar valores temporales **de forma segura** a un managed resource durante una operación, **sin persistirlos** en state/plan.
- Cada provider define qué argumentos son write-only. Convención: terminan en **`_wo`**, con un argumento hermano **`_wo_version`**:

```hcl
resource "aws_db_instance" "main" {
  # ...
  password_wo         = ephemeral.random_password.db_password.result
  password_wo_version = 1
}
```

- Durante la operación, el provider usa el valor de `password_wo` para crear/actualizar el recurso, y luego Terraform **lo descarta** sin guardarlo.
- El argumento `_wo_version` sirve para indicarle a Terraform que el valor **ha cambiado** (forzar una actualización del write-only argument incrementando la versión).

### State security best practices

Si guardas valores sensibles en el state (con `sensitive` pero sin `ephemeral`), medidas recomendadas:

- **State remoto** (no local).
- **Cifrado at-rest**.
- **Access controls** para limitar quién accede al state.
- **Audit logs** para rastrear accesos al state en el tiempo.

Backends con soporte de cifrado at-rest:

- **HCP Terraform**: cifra at-rest automáticamente, permite **traer tu propia clave** de cifrado (BYOK), y protege con TLS in-transit.
- **Backend S3**: cifra at-rest si activas la opción `encrypt`, protege con TLS in-transit.
- **Backend GCS**: soporta claves de cifrado **customer-supplied** o **customer-managed** (Cloud KMS).

### Nota sobre Vault (secrets management)

- Para gestión de secretos más avanzada (rotación, leasing dinámico de credenciales, etc.), Terraform se integra con **HashiCorp Vault** mediante el **Vault provider**, permitiendo inyectar secretos gestionados por Vault en la configuración en vez de tenerlos hardcodeados o como variables sensibles estáticas. _(Sin documentación específica pegada aún sobre el Vault provider — pendiente si quieres ampliar)_

---

### 💡 Puntos clave para el examen (4h)

- **`sensitive`** = oculta en CLI/UI, pero **se guarda en state/plan en claro** (y `-json`/`-raw` lo revelan).
- **`ephemeral`** = **no se guarda nunca** en state/plan; disponible solo durante el run. Variables y outputs de **child module** (nunca root module para outputs).
- Combinar `sensitive = true` + `ephemeral = true` en una variable = oculto en UI **y** no persistido.
- **Bloque `ephemeral "TYPE" "NAME" {}`** = ephemeral **resource**, temporal, solo referenciable en contextos ephemeral.
- **Write-only arguments** (`_wo` + `_wo_version`) = forma de pasar secretos a un `resource` managed sin persistirlos — el provider los usa y Terraform los descarta.
- Versiones: `sensitive` (0.15+), `ephemeral` (1.10+), write-only arguments (1.11+).
- Mejor práctica si algo sí queda en el state: **remote state + cifrado at-rest + access controls + audit logs**.
