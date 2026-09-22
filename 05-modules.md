# 5. Terraform modules

| Subtema | Descripción | Cubierto |
|---|---|---|
| 5a | Explain how Terraform sources modules | ✅ |
| 5b | Describe variable scope within modules | ✅ |
| 5c | Use modules in configuration | ✅ |
| 5d | Manage module versions | ✅ |

Fuentes: "Find and use modules", "module block reference", "Module Composition"
(Nota: "output block reference" y "Manage values in modules" ya estaban cubiertos en [04-configuration.md](04-configuration.md) 4c — no se repiten aquí)

---

## 5a. Origen de los módulos (registry, local, git, etc.)

### Terraform Registry — encontrar módulos

- El **Terraform Registry** (registry.terraform.io) tiene un buscador: la query hace match contra nombre, provider y descripción del módulo.
- Se puede filtrar por **Partner modules** (revisados por HashiCorp para asegurar estabilidad/compatibilidad).
- Sintaxis para un módulo del registry público: **`<NAMESPACE>/<NAME>/<PROVIDER>`**, ej. `hashicorp/consul/aws`.
- Integración de registry añadida en **v0.10.6**; versionado completo desde **v0.11.0**.

```hcl
module "consul" {
  source  = "hashicorp/consul/aws"
  version = "0.1.0"
}
```

- `terraform init` descarga y cachea los módulos referenciados.

### Registry privado

- Sintaxis: **`<HOSTNAME>/<NAMESPACE>/<NAME>/<PROVIDER>`** (mismo formato que el público, con hostname delante).
- Ej: `app.terraform.io/example_corp/vpc/aws` (registry privado de HCP Terraform).
- Puede requerir credenciales configuradas según el registry usado.
- Soportado desde **v0.11.0**.

### El argumento `source` — tipos de origen soportados

`source` es **obligatorio**, debe ser un **literal string** (no admite expresiones ni interpolación), y hay que correr `terraform init` tras modificarlo. Puede referenciar variables/locals **constantes** (`const = true`).

| Tipo de origen | Sintaxis |
|---|---|
| **Local path** | `./<PATH>` o `../<PATH>` — módulo en disco. Un path absoluto (empieza por `/` o letra de unidad) se copia a la caché local; **no recomendado** (acopla la config a la estructura de filesystem de una máquina concreta) |
| **Terraform Registry (público)** | `<NAMESPACE>/<NAME>/<PROVIDER>` |
| **Registry privado (HCP Terraform)** | `app.terraform.io/<NAMESPACE>/<NAME>/<PROVIDER>` |
| **Registry privado (Terraform Enterprise)** | `<HOSTNAME>/<NAMESPACE>/<NAME>/<PROVIDER>` |
| **`localterraform.com`** | Hostname genérico que pide módulos a la instancia donde corre la plataforma (HCP Terraform/TFE) |
| **GitHub (HTTPS)** | `github.com/<ORG>/<MODULE-FOLDER>` |
| **GitHub (SSH)** | `git@github.com:<ORG>/<MODULE-FOLDER>` |
| **Git genérico** | `git::ssh://[user@]host[:port]/path`, `git::[user@]host/path` (scp-like), o `git::<protocol>://host[:port]/path` (http/https/ftp/ftps/git) |
| **Bitbucket** | `bitbucket.org/<PATH>` (es un host de Git, mismas reglas que Git) |
| **Mercurial** | prefijo **`hg::`** + URL de Mercurial (file/http/https/ssh/path) |
| **HTTP URL (vanity URL)** | URL http(s) — Terraform hace `GET` con `?terraform-get=1`, y lee la dirección real del módulo de la cabecera `X-Terraform-Get` o de un `<meta name="terraform-get">` en el HTML |
| **S3 bucket** | prefijo **`s3::`** + URL del objeto S3 (debe ser un archivo comprimido: `.zip`, `.tar.gz`, `.tar.bz2`, `.tar.xz`, etc.) |
| **GCS bucket** | prefijo **`gcs::`** + URL del objeto GCS (también archivo comprimido) |

**Autenticación (Git/GitHub/Bitbucket/Mercurial):**

- Terraform ejecuta `git clone` (o `hg clone`) usando la configuración/credenciales de Git ya presentes en el sistema.
- **SSH**: Terraform usa automáticamente tus claves SSH — método más común para automatización sin prompts.
- **HTTP/HTTPS**: hace falta configurar credenciales según la documentación de Git (credential storage).
- En **HCP Terraform**, solo se puede autenticar vía **claves SSH**.

**Query parameters en fuentes Git:**

- `ref`: branch, hash SHA-1 (completo o corto), o tag a clonar. Default: rama por defecto (`HEAD`).
- `depth`: shallow clone (profundidad de historial). Default `1`. Si se usa `depth`, `ref` se pasa como `--branch` al `git clone` → **debe ser un branch o tag con nombre**, no un commit ID crudo.

**S3**: busca credenciales AWS en este orden: (1) env vars `AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY`, (2) perfil default en `~/.aws/credentials`, (3) credenciales temporales del IAM instance profile si corre en EC2. Nota: buckets en `us-east-1` deben usar el hostname `s3.amazonaws.com` (no `s3-us-east-1.amazonaws.com`).

**GCS**: autentica vía Google Cloud SDK — `GOOGLE_OAUTH_ACCESS_TOKEN`, `GOOGLE_APPLICATION_CREDENTIALS` (path a service account key), credenciales por defecto en GCE, o `gcloud auth application-default login` en local.

### Subdirectorios dentro de un paquete (`//`)

- Si el código del módulo está en un **subdirectorio** dentro de un repo/archive (el "paquete"), se añade **`//`** al path para indicar que lo que sigue es un subdirectorio:

```hcl
module "consul" {
  source = "hashicorp/consul/aws//modules/consul-cluster"
}

module "vpc" {
  source = "git::https://example.com/network.git//modules/vpc?ref=v1.2.0"
}
```

- Los query params (como `ref`) van **después** del segmento de subdirectorio.
- Terraform extrae **todo el paquete** a disco local, pero lee el módulo desde el subdirectorio — por eso un módulo en un subdirectorio puede referenciar otro módulo del mismo paquete con un **path local**.

---

### 💡 Puntos clave para el examen (5a)

- `source` = obligatorio, **literal string**, sin expresiones (salvo variables/locals `const = true`); cambiarlo requiere `terraform init`.
- Sintaxis registry público: `<NAMESPACE>/<NAME>/<PROVIDER>` (sin hostname); privado: con hostname delante.
- Local paths (`./`, `../`) — evitar **paths absolutos** (acoplan la config a una máquina concreta).
- `//` marca el inicio de un **subdirectorio** dentro de un paquete Git/archive/registry.
- SSH es el método de auth recomendado para Git en automatización (incluido obligatorio en HCP Terraform).
- Prefijos especiales: `git::`, `hg::`, `s3::`, `gcs::`.

---

## 5b. Scope de variables en módulos

_(El detalle de `variable`/`output`/`locals` ya está cubierto a fondo en [04-configuration.md](04-configuration.md) 4c — aquí el foco es el **scope entre módulos padre/hijo**)_

### Cómo se comunican padre e hijo

- Un **child module** solo recibe los valores que el módulo padre le pasa explícitamente como **argumentos del bloque `module`** — no hereda automáticamente variables/locals del padre (salvo la configuración de **provider**, que sí se hereda implícitamente — ver [02-fundamentals.md](02-fundamentals.md) 2b/2c).
- Las **variables** (`variable` block) del child module definen su **interfaz de entrada** — solo son visibles/asignables **dentro de ese módulo**, y se les asigna valor desde el bloque `module` que lo invoca:

```hcl
module "consul" {
  source  = "./modules/consul-cluster"
  region  = var.region   # var.region es del módulo padre (root u otro)
}
```

Dentro de `./modules/consul-cluster`, esa variable se declara y usa como `var.region` — **scope local a ese módulo**, sin relación directa con `var.region` del padre salvo por el valor pasado explícitamente.

- Los **outputs** del child module son lo único que "sube" hacia el padre, y se acceden como **`module.<LABEL>.<OUTPUT_NAME>`**.
- Los **locals** de un módulo son estrictamente internos a ese módulo — nunca visibles desde fuera (ni padre ni hermanos).

### Label del bloque `module`

- El `<LABEL>` en `module "<LABEL>" {...}` es un **nombre local** para esa instancia del módulo dentro del módulo que lo llama. Con outputs expuestos, se referencian como `module.<LABEL>.<output>`.
- Se puede usar el **mismo `source`** en varios bloques `module`, siempre que cada uno tenga un **label único** — así se instancia el mismo módulo varias veces con configuraciones distintas.

---

### 💡 Puntos clave para el examen (5b)

- El scope de una `variable` es **local al módulo** que la declara — el padre la asigna vía el bloque `module`, pero no comparte namespace con las variables del padre.
- Solo los **outputs** de un child module son visibles desde el padre (`module.<LABEL>.<OUTPUT>`); los `locals` nunca se exponen fuera del módulo.
- La **configuración de provider** es la excepción: se hereda implícitamente del padre a los child modules (a menos que se use `configuration_aliases` para pasar una con alias).
- Mismo `source` + labels distintos = múltiples instancias independientes del mismo módulo.

---

## 5c. Uso de módulos en configuración

Fuente: "module block reference" (meta-arguments `count`/`for_each`/`providers`/`depends_on`), "Module Composition"

### `module` block — argumentos completos

```hcl
module "<LABEL>" {
  <module-specific-inputs>
  source                      = "<location>"
  version                     = "<constraint>"   # solo para módulos de un registry
  count                       = <number>          # mutuamente excluyente con for_each
  for_each                    = { <KEY> = <VALUE> } # o un set de strings
  providers = {
    "<provider-en-child>" = "<provider-en-padre>"
  }
  depends_on                  = [ <resource.address> ]
  ignore_nested_deprecations  = <true|false>
}
```

- **Module-specific inputs**: los define el desarrollador del módulo (sus `variable` blocks) — algunos pueden ser obligatorios.
- **`count`**: crea múltiples instancias **idénticas o casi idénticas** del módulo. Meta-argumento (igual que en `resource`).
- **`for_each`**: crea instancias con **configuración variable** a partir de un map o set de strings — más apropiado quee `count` cuando las instancias difieren entre sí.
- **`providers`**: pasa una configuración de provider **alternativa** (con alias) al child module — ver [02-fundamentals.md](02-fundamentals.md) 2c.
- **`depends_on`**: fuerza que Terraform complete **todas** las operaciones del resource upstream antes de operar sobre el módulo.
- **`ignore_nested_deprecations`** (bool, default `false`, desde **v1.15**): si `true`, suprime warnings de deprecación del module call y de módulos anidados dentro de él.

### Ejemplo con `count`

```hcl
locals {
  instance_names = ["example-instance-1", "example-instance-2", "example-instance-3"]
}

module "ec2_instance" {
  source  = "terraform-aws-modules/ec2-instance/aws"
  version = "6.0.2"
  count   = length(local.instance_names)

  name          = local.instance_names[count.index]
  instance_type = "t2.micro"

  depends_on = [aws_s3_bucket.example]
}
```

### Ejemplo con `providers` (alias por recurso dentro del módulo)

```hcl
provider "aws" {
  alias  = "usw1"
  region = "us-west-1"
}
provider "aws" {
  alias  = "usw2"
  region = "us-west-2"
}

module "tunnel" {
  source = "./tunnel"
  providers = {
    aws.src = aws.usw1
    aws.dst = aws.usw2
  }
}
```

### Module Composition — filosofía de diseño

- Con un solo root module, los recursos son un **conjunto plano** relacionado por expresiones.
- Al introducir `module` blocks, la config se vuelve **jerárquica**. Recomendación fuerte: mantener el árbol de módulos **plano** (un solo nivel de child modules), conectándolos con expresiones — a esto se le llama **module composition**: ensamblar módulos "componibles" en vez de anidar módulos dentro de módulos.

```hcl
module "network" {
  source           = "./modules/aws-network"
  base_cidr_block  = "10.0.0.0/8"
}

module "consul_cluster" {
  source     = "./modules/aws-consul-cluster"
  vpc_id     = module.network.vpc_id
  subnet_ids = module.network.subnet_ids
}
```

- El módulo **recibe sus dependencias** del root module (en vez de crear/gestionar sus propias copias) → el root module puede conectar los mismos módulos de formas distintas para producir resultados distintos.

### Dependency Inversion

- En vez de que un módulo (ej. `consul_cluster`) **cree su propia red**, se prefiere que la **reciba como argumento** (VPC ID, subnet IDs) — así el módulo no sabe ni le importa **cómo** se obtuvieron esos valores.
- Esto facilita refactors futuros: se puede pasar de crear la red inline a leerla vía **data sources**, sin tocar el módulo consumidor:

```hcl
data "aws_vpc" "main" { tags = { Environment = "production" } }
data "aws_subnet_ids" "main" { vpc_id = data.aws_vpc.main.id }

module "consul_cluster" {
  source     = "./modules/aws-consul-cluster"
  vpc_id     = data.aws_vpc.main.id
  subnet_ids = data.aws_subnet_ids.main.ids
}
```

### Creación condicional de objetos

- Cuando un mismo módulo se usa en varios entornos, y en algunos ya existe el objeto necesario mientras en otros hay que crearlo: **no** metas lógica de detección dentro del módulo — aplica **dependency inversion**: el módulo acepta el objeto vía una **input variable** tipada como `object({...})` con solo los atributos que necesita.
- El **caller** decide si lo crea (`resource`) o lo lee (`data`), y se lo pasa al módulo de la misma forma en ambos casos:

```hcl
variable "ami" {
  type = object({ id = string, architecture = string })
}

# Caller A: crea el AMI
resource "aws_ami_copy" "example" { ... }
module "example" { source = "./modules/example"; ami = aws_ami_copy.example }

# Caller B: el AMI ya existe
data "aws_ami" "example" { ... }
module "example" { source = "./modules/example"; ami = data.aws_ami.example }
```

- Consistente con el estilo **declarativo** de Terraform: describes qué existe ya y qué debe gestionar Terraform, en vez de meter condicionales complejos dentro del módulo.

### Assumptions and Guarantees

- **Assumption**: condición que debe ser cierta para que la config de un resource sea usable (ej. "la AMI debe ser x86_64").
- **Guarantee**: característica/comportamiento que el resto de la config puede **dar por hecho** (ej. "la instancia EC2 tendrá un registro DNS privado").
- Se recomienda **validar la configuración** (ver 4g: `precondition`/`postcondition`) para capturar y testear assumptions/guarantees explícitamente — ayuda a mantenedores futuros y da errores más tempranos y en contexto.

```hcl
output "api_base_url" {
  value = "https://${aws_instance.example.private_dns}:8433/"
  precondition {
    condition     = data.aws_ebs_volume.example.encrypted
    error_message = "The server's root volume is not encrypted."
  }
}
```

### Multi-cloud Abstractions

- Terraform **no** intenta abstraer servicios similares de distintos vendors (evita el enfoque "mínimo común denominador"). Pero se pueden crear **abstracciones propias ligeras** mediante composición de módulos, cuando varios vendors implementan un concepto/protocolo/estándar común (ej. DNS).
- Patrón: definir un **tipo de objeto** Terraform que represente el concepto común (ej. "recordset" DNS), usarlo como tipo de variable de entrada, e implementar un módulo específico por vendor que lo consuma. Cambiar de proveedor = sustituir solo ese módulo de implementación, sin tocar el resto de la config.

```hcl
variable "recordsets" {
  type = list(object({
    name    = string
    type    = string
    ttl     = number
    records = list(string)
  }))
}
```

### Data-only Modules

- Módulos que **no contienen `resource` blocks**, solo **data sources** — encapsulan **cómo** se obtiene cierta información compartida (ej. una red compartida entre subsistemas), sin gestionar infraestructura nueva.
- Ventaja: el origen de esos datos puede cambiar con el tiempo (API directa, Consul, `terraform_remote_state`...) sin tener que actualizar cada configuración que depende de ellos — y si un data-only module expone los mismos outputs que un módulo de gestión equivalente, se pueden intercambiar fácilmente en refactors.

```hcl
module "network" {
  source      = "./modules/join-network-aws"
  environment = "production"
}

module "k8s_cluster" {
  source     = "./modules/aws-k8s-cluster"
  subnet_ids = module.network.aws_subnet_ids
}
```

---

### 💡 Puntos clave para el examen (5c)

- `count` (instancias idénticas/similares) vs `for_each` (instancias con config variable, por key) — mutuamente excluyentes, igual que en `resource`.
- `providers` en `module` pasa configuraciones con **alias** al child module (requiere `configuration_aliases` dentro del módulo).
- `ignore_nested_deprecations` (v1.15+): suprime warnings de deprecación de módulos anidados.
- **Module composition**: preferir árbol **plano** de módulos conectados por expresiones, en vez de anidar módulos dentro de módulos.
- **Dependency inversion**: los módulos reciben sus dependencias como input (no las crean internamente) → más flexibles ante refactors.
- **Creación condicional**: el módulo acepta el objeto vía variable tipada; el **caller** decide si lo crea o lo lee con `data`.
- **Data-only modules**: solo `data` blocks, para encapsular el "cómo" se obtiene info compartida.

---

## 5d. Versionado de módulos

Fuente: "module block reference" — argumento `version`

### El argumento `version`

```hcl
module "consul" {
  source  = "hashicorp/consul/aws"
  version = ">= 0.10.0"
}
```

- Solo aplica cuando `source` apunta a un módulo de un **registry** (público o privado) — **no aplica** a módulos locales (comparten versión con su caller automáticamente, al vivir en el mismo repo) ni a otras fuentes como Git/S3/HTTP directamente (ahí el versionado se gestiona con `ref` en la URL, no con `version`).
- Acepta un **version constraint string** (igual sintaxis que en providers: `>=`, `~>`, `=`, etc. — ver [02-fundamentals.md](02-fundamentals.md) 2a).
- Terraform usa la **versión instalada más nueva** que cumpla el constraint; si no hay ninguna instalada que cumpla, **descarga** la más nueva que sí cumpla.
- Se recomienda **siempre** fijar un constraint explícito, para evitar cambios inesperados.
- Cambiar `version` requiere volver a correr `terraform init`.
- El valor de `version` puede referenciar variables/locals **constantes** (`const = true`), igual que `source`.
- Módulos siguen **semantic versioning** (semver): `MAJOR.MINOR.PATCH`.

### Ejemplo de constraint

```hcl
module "consul" {
  source  = "hashicorp/consul/aws"
  version = ">= 0.10.0"
  servers = 3
}
```

---

### 💡 Puntos clave para el examen (5d)

- `version` solo funciona con `source` de un **registry** (público o privado) — módulos locales no lo soportan.
- Mismo formato de version constraints que los providers (`>=`, `~>`, etc.).
- Sin `version` → Terraform toma la **última versión disponible** — riesgo de breaking changes; se recomienda **siempre** poner constraint.
- Cambiar `version` → hace falta `terraform init` (opcionalmente `-upgrade` si la versión ya estaba resuelta a otra).
