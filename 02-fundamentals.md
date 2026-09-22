# 2. Terraform fundamentals

| Subtema | Descripción | Cubierto |
|---|---|---|
| 2a | Install and version Terraform providers | ✅ |
| 2b | Describe how Terraform uses providers | ✅ |
| 2c | Write Terraform configuration using multiple providers | ✅ |
| 2d | Explain how Terraform uses and manages state | ✅ |

Fuentes: "Providers", "Provider Requirements", "Dependency Lock File", "provider block reference", "Purpose of Terraform State"

---

## 2a. Instalar y versionar providers

### Qué son los providers

- Terraform depende de plugins llamados **providers** para interactuar con proveedores cloud, SaaS y otras APIs.
- Cada provider añade un conjunto de **resource types** y/o **data sources** que Terraform puede gestionar. **Sin providers, Terraform no puede gestionar ningún tipo de infraestructura** — todo `resource` type está implementado por un provider.
- La mayoría de providers configuran una plataforma de infraestructura específica (cloud o self-hosted). Algunos ofrecen utilidades locales (ej. generar números aleatorios para nombres únicos — provider `random`).
- Los providers se **distribuyen por separado** de Terraform, cada uno con su propio ciclo de releases y versionado.
- El **Terraform Registry** (registry.terraform.io) es el directorio principal de providers públicos.

### Tiers de providers en el Registry

| Tier | Descripción | Namespace |
|---|---|---|
| **Official** | Mantenidos por HashiCorp | `hashicorp`, `IBM`, `ansible`... |
| **Partner Premier** | Empresas tecnológicas partner que cumplen requisitos premier | organización de terceros |
| **Partner** | Escritos/mantenidos por terceros contra sus propias APIs, participando en el HashiCorp Technology Partner Program | organización de terceros |
| **Community** | Publicados por mantenedores individuales o la comunidad | cuenta individual/org del mantenedor |
| **Archived** | Providers Official o Partner que ya no se mantienen | hashicorp o terceros |

### Declarar providers: `required_providers`

Cada **root module** debe declarar qué providers necesita, para que Terraform pueda instalarlos y usarlos. 3 pasos:

1. Definir **source**, **local name** y **version** en el bloque `required_providers` (anidado dentro del bloque `terraform`).
2. Añadir un bloque `provider` de nivel superior para configurarlo (auth, región, etc.).
3. Instalarlo (`terraform init`).

```hcl
terraform {
  required_providers {
    mycloud = {
      source  = "mycorp/mycloud"
      version = "~> 1.0"
    }
  }
}
```

- La sintaxis `name = { source, version }` existe desde **Terraform v0.13**.

### Local name vs Source address

Cada provider tiene **dos identificadores**:

- **Source address**: identificador global único, usado solo en `required_providers`. Formato: `[<HOSTNAME>/]<NAMESPACE>/<TYPE>`
  - `HOSTNAME` (opcional): host del registry (default `registry.terraform.io`).
  - `NAMESPACE`: organización que publica el provider.
  - `TYPE`: nombre corto de la plataforma (normalmente = local name preferido). Ej: `hashicorp/aws` → tipo `aws`.
  - Address "fully-qualified": `registry.terraform.io/hashicorp/random`; versión abreviada habitual: `hashicorp/random`.
  - Si se omite `source`, Terraform asume `registry.terraform.io/hashicorp/<LOCAL_NAME>` (compatibilidad hacia atrás — **no recomendado**, mejor ser explícito).
- **Local name**: identificador usado en el resto del módulo (fuera de `required_providers`). Es **específico del módulo** y debe ser único por módulo.
  - Se recomienda usar el **local name preferido** del provider (suele ser el prefijo de sus resource types, ej. `aws_instance` → local name `aws`). Así Terraform puede inferir el provider de un recurso sin necesitar el meta-argumento `provider`.
  - **Conflicto de local names**: si dos providers requeridos comparten el mismo type name, hay que asignar **nombres compuestos** (ej. `hashicorp-http`, `mycorp-http`) y especificar el meta-argumento `provider` en cada resource afectado.

### Version constraints

- El argumento `version` en `required_providers` es **opcional**, pero se recomienda **siempre** especificarlo.
- Si se omite, Terraform acepta cualquier versión.
- Buenas prácticas:
  - Un **módulo reutilizable** (no root) debería declarar solo la **versión mínima**: `version = ">= 1.0"`.
  - El **módulo raíz** (donde corres `terraform apply`) debería fijar también un **máximo**, usando `~>` para evitar upgrades accidentales a versiones incompatibles. Ej: `~> 1.0.4` permite solo patch releases dentro del 1.0.x.
  - **No usar `~>`** (constraint de máximo) en módulos pensados para reutilizarse en muchas configuraciones — fuerza a actualizar muchos módulos a la vez.

### Built-in providers

- Terraform incluye **un provider integrado**: habilita el data source `terraform_remote_state`.
- Source address especial: `terraform.io/builtin/terraform`. No hace falta declararlo en `required_providers`.
- Existe un provider antiguo `hashicorp/terraform` (versión previa, **no compatible** con Terraform v0.11+, no usar).

### Instalación de providers

- **HCP Terraform / Terraform Enterprise**: instalan providers en cada run.
- **Terraform CLI**: instala/busca providers al ejecutar `terraform init`. Puede descargar del Terraform Registry o cargar desde un mirror/cache local.
- Si usas un working directory persistente, hay que **reinicializar** (`terraform init`) cada vez que cambien los providers de la configuración.
- Se puede habilitar un **plugin cache** (opción `plugin_cache_dir` en el archivo de configuración CLI) para ahorrar tiempo/ancho de banda.
- Para garantizar que Terraform instala siempre las mismas versiones, se usa el **dependency lock file** (ver abajo) — HCP Terraform, CLI y Enterprise lo respetan al instalar providers.

### Dependency Lock File (`.terraform.lock.hcl`)

- Rastrea las **decisiones de versión** que Terraform ha tomado para las dependencias de **providers** (no rastrea versiones de módulos remotos — para módulos, usa un version constraint exacto si quieres pinnearlos).
- Pertenece a la **configuración completa** (no a cada módulo por separado). Vive en el directorio de trabajo (junto a los `.tf` del root module).
- Nombre fijo: **`.terraform.lock.hcl`** (sufijo `.hcl`, no `.tf`, porque no es un archivo de configuración Terraform normal).
- Se crea/actualiza automáticamente en cada `terraform init`.
- **Se debe commitear al VCS** junto con la configuración, para poder revisar cambios de dependencias vía code review.

**Comportamiento de instalación:**

- Si un provider **no tiene selección previa** en el lock file → Terraform selecciona la versión más nueva que cumpla el constraint y la registra en el lock file.
- Si **ya tiene una selección registrada** → Terraform **siempre reinstala esa misma versión**, aunque haya una más nueva disponible.
- Para forzar a Terraform a considerar versiones más nuevas: `terraform init -upgrade`.
- Si `terraform init` modifica el lock file, lo indica explícitamente en el output.

**Verificación de checksums:**

- Terraform verifica que cada paquete instalado coincida con al menos uno de los checksums registrados previamente en el lock file (modelo **"trust on first use"**).
- Si no coincide ningún checksum → error (`Failed to install provider`).
- Dos esquemas de hash: `zh:` (zip hash, esquema legacy del protocolo de registry) y `h1:` (hash scheme 1, esquema preferido actual, calculado sobre el contenido del paquete, no sobre el `.zip`).
- Se pueden pre-poblar checksums para varias plataformas con `terraform providers lock -platform=...` (útil si se instala desde un mirror sin firma criptográfica, o para evitar tener que ir añadiendo hashes según se usan nuevas plataformas).

**Providers que dejan de ser necesarios:**

- Terraform determina la necesidad de un provider mirando **configuración + state**. Si eliminas la última dependencia de ambos, `terraform init` **elimina la entrada del lock file** para ese provider.
- (En Terraform ≤1.0, esto no se hacía automáticamente y podía dar error "missing or corrupted provider plugins" en versiones posteriores si el lock quedaba obsoleto.)

### Providers privados / in-house

- Si el provider no está en un registry hosteado por HashiCorp, puede requerir credenciales adicionales para las requests → se configuran vía archivo **`.netrc`** (por defecto en `HOME`, sobreescribible con la variable de entorno `NETRC`).
- Se pueden distribuir providers propios (in-house) mediante:
  - un **registry privado** propio (implementando el provider registry protocol), o
  - **filesystem mirrors**: colocar el plugin directamente en un directorio local con una estructura tipo `<hostname>/<namespace>/<type>/<version>/<platform>/terraform-provider-<type>[.exe]`.
- Todo provider necesita un source address con un hostname (aunque no resuelva en DNS realmente) para usarse como placeholder.

---

## 2b. Cómo Terraform usa los providers (`provider` block)

### Arquitectura: Terraform Core vs Terraform Plugins

Terraform tiene una **arquitectura basada en plugins**, dividida en dos partes:

- **Terraform Core**: binario compilado estáticamente (Go), es el CLI `terraform` — el entrypoint de todo el mundo. Responsabilidades principales:
  - Leer e interpolar los archivos de configuración y módulos (IaC).
  - Gestión del **resource state**.
  - Construcción del **Resource Graph**.
  - Ejecución del plan.
  - Comunicación con los plugins vía **RPC** (remote procedure calls).
- **Terraform Plugins**: binarios ejecutables (Go) invocados por Terraform Core vía RPC, cada uno implementando un servicio concreto (ej. AWS) o un provisioner (ej. bash). Se ejecutan como **proceso separado**. Todos los **Providers** y **Provisioners** son plugins.
  - **Provider plugins**: inicializan librerías para llamar a la API, gestionan **autenticación** con el proveedor de infraestructura, definen los **managed resources** y **data sources** que mapean a servicios concretos, y definen funciones que simplifican lógica en la configuración.
  - **Provisioner plugins**: ejecutan comandos/scripts en un resource tras su creación o en su destrucción. Terraform trae varios provisioners **built-in**, mientras que los providers se **descubren dinámicamente**.

Terraform Core ofrece un framework de alto nivel que abstrae el descubrimiento de plugins y la comunicación RPC, para que los desarrolladores de plugins no tengan que gestionarlo.

### Discovery (descubrimiento de plugins) — qué pasa en `terraform init`

Cuando ejecutas `terraform init`, Terraform:

1. Lee la configuración del working directory para determinar qué plugins necesita.
2. Busca plugins ya instalados en varias ubicaciones.
3. Descarga plugins adicionales si hace falta.
4. Decide qué **versión** de cada plugin usar.
5. Escribe el **lock file**, para usar las mismas versiones hasta el próximo `terraform init`.

**Selección de versión de plugin** (comparando lo instalado contra los version constraints de la configuración):

- Si hay versiones aceptables **ya instaladas** → usa la **más nueva instalada** que cumpla el constraint (aunque el Registry tenga una más nueva disponible).
- Si **no hay versión instalada** aceptable y el provider es de los distribuidos por HashiCorp → lo **descarga** del Terraform Registry y lo guarda en `.terraform/providers/`.
- Si **no hay versión instalada** y el provider **no está** en el Terraform Registry → la inicialización **falla**, hay que instalarlo manualmente.

**Upgrade de plugins**: con `terraform init -upgrade`, Terraform vuelve a comprobar el Registry en busca de versiones más nuevas aceptables y las descarga si existen. Esto solo aplica a providers cuya única versión aceptable esté en el directorio de descargas automáticas (`.terraform/providers/`) — si ya hay una versión aceptable instalada en otro sitio, `-upgrade` no la sobreescribe.

### Qué hace el bloque `provider`

- El bloque `provider` **configura** un provider ya declarado en `required_providers` (auth, región, endpoints, etc.) — es la instancia configurada de un provider.
- Se define en el **root module** de la configuración. Los **child modules reciben su configuración de provider del módulo padre** — se recomienda **no** definir bloques `provider` dentro de child modules.
- Si no defines explícitamente un bloque `provider`, Terraform asume una **configuración vacía por defecto**. Si el provider requiere argumentos obligatorios, Terraform lanzará un error al no poder crear esa configuración.

```hcl
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 4.0"
    }
  }
}

provider "google" {
  project = "acme-app"
  region  = "us-central1"
}
```

### Argumentos del bloque `provider`

- **Argumentos específicos del provider**: definidos por el propio provider (ver su documentación). Solo se pueden usar expresiones que Terraform conoce **antes de aplicar** (input variables, valores literales) — **no** se pueden referenciar atributos computados de otros recursos (ej. `google.web.public_ip`).
- Muchos providers soportan **variables de entorno** u otras fuentes alternativas de configuración, para mantener credenciales fuera del código versionado.
- **`alias`** (string, opcional): identificador único para tener **múltiples configuraciones del mismo provider** (ver 2c — configuraciones alternativas / multi-región).
- **`version`** (string, **deprecated**): constraint de versión a nivel de bloque `provider` — ya no se usa; se declara en `required_providers`.

### Cómo Terraform relaciona resources con providers

- Si un resource no especifica el meta-argumento `provider`, Terraform interpreta el **primer segmento del nombre del resource type** como local provider name (ej. `aws_instance` → provider local name `aws`).
- Si hay **múltiples alias** para un provider, el bloque `provider` **sin `alias`** es la configuración **default**. Si **todos** los bloques usan alias, Terraform crea una configuración default **vacía implícita** — cualquier resource sin `provider` explícito usará esa config vacía (y fallará si el provider requiere argumentos obligatorios).

### Herencia de providers en módulos

- Cuando defines un provider en el módulo raíz, Terraform lo **pasa implícitamente** a los child modules para que todos usen la misma configuración.
- Los child modules **no heredan** los requisitos de `source`/`version` del provider — hay que declararlos explícitamente también en el child module (mismo `required_providers`).
- Para usar una configuración **con alias** dentro de un child module, el child module debe declarar esa expectativa con **`configuration_aliases`** dentro de su `required_providers`, y el módulo padre debe pasarla explícitamente vía el argumento `providers = { aws.west = aws.west }` en el bloque `module`.

---

### 💡 Puntos clave para el examen (2a/2b)

- Providers = plugins que dan a Terraform los `resource`/`data` types; sin ellos, Terraform no gestiona nada.
- 3 pasos para usar un provider: declarar en `required_providers` → configurar con `provider {}` → `terraform init`.
- **Source address** (`hostname/namespace/type`, global) vs **local name** (por módulo, el que usas en el código).
- Version constraints: `>=` para módulos reutilizables, `~>` para el root module (evita upgrades accidentales).
- `.terraform.lock.hcl`: fija versiones exactas de **providers** (no de módulos remotos); se genera/actualiza con `terraform init` (o `-upgrade`); **se commitea**.
- El lock file usa checksums (`h1:` y `zh:`) con modelo "trust on first use".
- El bloque `provider` va en el **root module**; los child modules **heredan** la configuración de provider automáticamente, pero no el `source`/`version`.
- `alias` permite múltiples configuraciones del mismo provider (ej. multi-región) — el bloque sin alias es el default.
- `version` dentro de `provider {}` está **deprecated** → usar `required_providers`.

---

## 2c. Configuración con múltiples providers

_(Contenido ya cubierto en detalle en 2b → "Argumentos del bloque `provider`" y "Herencia de providers en módulos", ya que viene de la misma página "provider block reference". Resumen orientado a este subtema:)_

Hay dos formas de usar **múltiples providers** en una configuración:

1. **Providers distintos** (ej. `aws` + `google` + `random` a la vez): basta con declarar cada uno en `required_providers` y su bloque `provider` correspondiente — Terraform los gestiona en paralelo dentro del mismo `resource graph`.

2. **Múltiples configuraciones del mismo provider** (ej. AWS en dos regiones distintas), usando **`alias`**:

```hcl
provider "aws" {
  region = "us-east-1"          # configuración default (sin alias)
}

provider "aws" {
  alias  = "west"
  region = "us-west-2"          # configuración alternativa
}

resource "aws_instance" "foo" {
  provider = aws.west           # usa la config alternativa
}

resource "aws_instance" "bar" {
  # usa la config default (us-east-1) al no especificar provider
}
```

- Para referenciar un alias: `<PROVIDER_NAME>.<ALIAS>` en el meta-argumento `provider` de bloques `resource`, `data` o `module`.
- Si **todos** los bloques `provider` de un provider usan alias (ninguno es el default), Terraform crea una configuración default **vacía implícita** — cualquier resource que no especifique `provider` la usará (y fallará si el provider necesita argumentos obligatorios).
- Para pasar un alias a un **child module**, el módulo hijo debe declarar `configuration_aliases` en su `required_providers`, y el módulo padre debe pasarlo explícitamente con el argumento `providers = { aws.west = aws.west }` en el bloque `module`.

---

## 2d. Cómo Terraform usa y gestiona el state

_(Purpose of Terraform State)_

El **state** es un requisito necesario para que Terraform funcione — no es opcional ni un "extra". Terraform necesita algún tipo de base de datos que mapee la configuración con el mundo real. Razones:

### 1. Mapeo con el mundo real (mapping to the real world)

- Cuando tienes `resource "aws_instance" "foo" {...}`, Terraform usa el **state** para saber que ese bloque de configuración representa un objeto real con, por ejemplo, instance ID `i-abcd1234`.
- Prototipos tempranos de Terraform intentaron usar algo como **tags de AWS** en vez de un state propio, pero fallaba: no todos los recursos/providers soportan tags. Por eso Terraform usa su **propia estructura de state**.
- Terraform espera que **cada objeto remoto esté vinculado a una única resource instance** en la configuración. Si un objeto remoto queda vinculado a múltiples resource instances, el mapeo se vuelve ambiguo y Terraform puede comportarse de forma inesperada. Terraform garantiza este mapeo 1:1 cuando **él mismo crea** los objetos; al **importar** objetos creados fuera de Terraform, hay que asegurarse de importar cada objeto distinto a una única resource instance.

### 2. Metadata

- Además del mapeo, Terraform trackea metadata como las **dependencias entre recursos**.
- Normalmente Terraform deduce el orden de dependencias **de la configuración**. Pero si **eliminas un resource de la configuración**, Terraform ya no tiene esa info en el código — por eso guarda una **copia de las dependencias más recientes en el state**, lo que le permite determinar el orden correcto de destrucción incluso cuando el recurso ya no está en la configuración.
- Alternativa descartada: usar una jerarquía de orden fija entre tipos de recursos (ej. "los servidores se borran antes que sus subnets") — la complejidad **explota** porque Terraform tendría que entender el orden de cada tipo de recurso de cada provider, y entre providers.
- El state también guarda otra metadata, como un puntero a la **configuración de provider** usada más recientemente con un recurso (relevante cuando hay múltiples providers con alias).

### 3. Performance

- Terraform guarda en el state una **caché de los valores de atributos** de todos los recursos — la parte más "opcional" del state, pensada solo como mejora de rendimiento.
- En `terraform plan`, Terraform necesita conocer el estado actual de los recursos para calcular los cambios necesarios.
- Para infraestructuras **pequeñas**, Terraform puede consultar a los providers y sincronizar (refresh) todos los recursos en cada plan/apply (comportamiento por defecto).
- Para infraestructuras **grandes**, consultar cada recurso es demasiado lento (APIs sin queries batch, latencia de cientos de ms por recurso, rate limiting). En estos casos se usa mucho `-refresh=false` y `-target` para evitar refrescar todo — ahí el **state cacheado se trata como la fuente de verdad**.

### 4. Sincronización (syncing) — state local vs remoto

- Por defecto, Terraform guarda el state en un **archivo en el directorio de trabajo local**. Vale para empezar, pero en equipo **todo el mundo necesita trabajar con el mismo state** para que las operaciones se apliquen sobre los mismos objetos remotos.
- Solución recomendada: **remote state**. Con un backend de state completo, Terraform puede usar **locking remoto** para evitar que dos o más personas ejecuten Terraform a la vez sobre el mismo state, garantizando que cada run empieza con el state más actualizado.

---

### 💡 Puntos clave para el examen (2c/2d)

- `alias` en `provider {}` = múltiples configuraciones del mismo provider (ej. multi-región); se referencia como `provider = <name>.<alias>` en resource/data/module.
- El bloque `provider` **sin alias** = configuración default; si todos tienen alias, la default es una config vacía implícita.
- Child modules necesitan `configuration_aliases` en `required_providers` + el argumento `providers = {...}` en el bloque `module` para recibir un alias del padre.
- El **state es obligatorio** en Terraform: mapea config ↔ recursos reales, guarda metadata de dependencias (crucial al borrar recursos de la config), y cachea atributos por performance.
- Terraform garantiza mapeo **1:1** entre resource instance y objeto remoto.
- State remoto + **locking** es la solución estándar para trabajar en equipo sin pisarse.
- `-refresh=false` y `-target` existen para infraestructuras grandes donde refrescar todo el state es demasiado lento.
