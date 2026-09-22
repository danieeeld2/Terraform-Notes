# 6. Terraform state management

| Subtema | Descripción | Cubierto |
|---|---|---|
| 6a | Describe the local backend | ✅ |
| 6b | Describe state locking | ✅ |
| 6c | Configure remote state using the backend block | ✅ |
| 6d | Manage resource drift and Terraform state | ✅ |

Fuentes: "Backend block configuration overview", "local" (backend), "State Storage and Locking", "State Locking", "State" (Terraform state overview), "Refactor Terraform state", "Refresh-Only Mode", "removed block reference", "moved block reference"

---

## 6a. Backend local

### Qué es un backend

- Terraform usa **state persistido** para trackear los recursos que gestiona. El `backend` define **dónde** se guardan esos datos de state.
- Dos opciones: integrar con **HCP Terraform** (gestiona el state automáticamente en los workspaces) o definir un bloque **`backend`** para guardar el state en un objeto remoto — así varias personas pueden acceder y colaborar.

### Definir un bloque `backend`

- ⚠️ **No** configures un `backend` si tu configuración usa workspaces de HCP Terraform/Terraform Enterprise (esas plataformas gestionan el state automáticamente). Si la config incluye un bloque **`cloud`**, **no puede** incluir además un `backend`.
- Se anida dentro del bloque `terraform`:

```hcl
terraform {
  backend "remote" {
    organization = "example_corp"
    workspaces {
      name = "my-app-prod"
    }
  }
}
```

**Limitaciones importantes:**

- Una config solo puede tener **un** bloque `backend`.
- Un bloque `backend` **no puede referenciar named values** (variables, locals, atributos de data sources).
- No se puede referenciar en otras partes de la config los valores declarados **dentro** de un `backend` block.

### Backend por defecto: `local`

- Terraform usa el backend **`local`** por defecto — guarda el state como **archivo local** en disco.
- Terraform trae varios **backend types built-in**; algunos funcionan como "discos remotos" para el state, otros además soportan **locking**. **No se pueden cargar backends adicionales como plugins**.
- El tipo se especifica como **label** del bloque: `backend "remote" { ... }`.

### El backend `local` en detalle

```hcl
terraform {
  backend "local" {
    path = "relative/path/to/terraform.tfstate"
  }
}
```

- **Kind**: Enhanced.
- Guarda el state en el filesystem local, lo **bloquea usando APIs del sistema**, y realiza las operaciones localmente.
- Configuration variables:
  - `path` (opcional): path al archivo `.tfstate`. Default: `terraform.tfstate` relativo al root module.
  - `workspace_dir` (opcional): path para workspaces no-default.
- Se puede leer con un data source `terraform_remote_state`, indicando `backend = "local"` y `config = { path = "..." }`.

**Argumentos de línea de comandos (legacy, no recomendados para sistemas nuevos):**

- `-state=FILENAME`: sobreescribe el nombre del archivo al **leer** el state previo.
- `-state-out=FILENAME`: sobreescribe el nombre al **escribir** un nuevo snapshot de state. Si usas `-state` sin `-state-out`, Terraform usa el mismo filename para ambos → sobreescribe el archivo de entrada.
- `-backup=FILENAME`: sobreescribe el archivo de **backup** que el backend local crea por defecto al escribir nuevo state. `-backup=-` desactiva la creación de backups.
- Estas opciones son **solo para el backend `local`** (o config sin backend explícito, que usa `local` por defecto) — no tienen efecto con otros backends.
- Predatan la introducción de **workspaces múltiples**: si usas las tres a la vez, el workspace seleccionado **no afecta** qué archivos usa Terraform.
- Recomendación actual: en vez de usar estas flags, **elegir un backend con soporte de remote state** y configurarlo en el root module.

---

### 💡 Puntos clave para el examen (6a)

- Backend por defecto = **`local`**: state en un archivo en disco (`terraform.tfstate` por defecto), locking vía **APIs del sistema**.
- Solo **un** bloque `backend` por config; no puede usar variables/locals/data sources dentro de sí mismo.
- No se puede combinar bloque `backend` con bloque `cloud` en la misma config.
- `-state`/`-state-out`/`-backup` son legacy, solo aplican al backend `local`, no recomendados en sistemas nuevos.

---

## 6b. State locking

### Storage vs Locking — responsabilidades del backend

- Los backends son responsables de **guardar el state** y de proveer una **API de state locking** (locking es **opcional** — no todos los backends lo soportan).
- Aunque el state esté remoto, comandos como `terraform console`, `terraform state ...`, `terraform taint`, etc. siguen funcionando como si fuera local.

### State Storage

- El backend determina **dónde** se guarda el state. Ej: `local` → JSON en disco; `Consul` → dentro de Consul (ambos con locking, vía APIs del sistema o de Consul respectivamente).
- Con un backend **no-local**, Terraform **no persiste el state en disco** salvo en caso de **error no recuperable** al escribir en el backend (entonces lo escribe localmente para evitar pérdida de datos, y hay que empujarlo manualmente al backend remoto una vez resuelto el error).
- Beneficio clave: si el state tiene valores sensibles, un backend remoto permite usar Terraform **sin que ese state llegue nunca a persistirse en disco local**.

### Manual State Pull/Push

- **`terraform state pull`**: recupera el state remoto y lo manda a **stdout** (se puede guardar a archivo).
- **`terraform state push`**: escribe manualmente el state — **extremadamente peligroso**, sobreescribe el state remoto. Solo para fixups manuales.
- Protecciones al hacer push manual:
  - **Lineage distinto**: el "lineage" es un ID único asignado al crear el state. Si difiere, Terraform **no permite** el push (probablemente estás modificando un state distinto).
  - **Serial más alto**: cada state tiene un número "serial" monotónicamente creciente. Si el state destino tiene un serial más alto, Terraform **no permite** sobreescribirlo (indica que hubo cambios posteriores).
  - Ambas protecciones se pueden saltar con **`-force`** — se recomienda hacer backup con `terraform state pull` antes.

### State Locking

- Si el backend lo soporta, Terraform **bloquea el state** en **todas** las operaciones que puedan escribirlo — evita que otro proceso adquiera el lock y potencialmente corrompa el state.
- Ocurre **automáticamente**, sin mensaje visible (salvo que adquirir el lock tarde más de lo esperado, en cuyo caso Terraform muestra un status message).
- Si falla la adquisición del lock, Terraform **no continúa**.
- Se puede desactivar con **`-lock=false`** en la mayoría de comandos — **no recomendado**.
- No todos los backends soportan locking (consultar la doc de cada backend).

### Force Unlock

- **`terraform force-unlock`**: desbloquea manualmente el state si el unlock automático falló.
- ⚠️ Usar con mucho cuidado: si desbloqueas mientras **otra persona** tiene el lock legítimamente, puede provocar **múltiples escritores simultáneos**. Solo debería usarse para desbloquear **tu propio lock** tras un fallo del unlock automático.
- Requiere un **lock ID único** (Terraform lo muestra en el error cuando falla el unlock) — actúa como nonce, asegurando que el force-unlock apunta al lock correcto.

---

### 💡 Puntos clave para el examen (6b)

- Locking es **opcional**, depende del backend — no todos lo soportan.
- Con backend remoto, Terraform **no guarda el state en disco local** salvo en error no recuperable al escribir.
- `terraform state pull` (leer) / `terraform state push` (escribir, **peligroso**) para manipular el state remoto manualmente.
- Protecciones de `state push`: **lineage** distinto o **serial** más alto → bloquea el push (saltable con `-force`).
- `-lock=false` desactiva el locking para un comando — no recomendado.
- `terraform force-unlock <LOCK_ID>` para desbloquear manualmente — solo para tu propio lock atascado, nunca para "robar" el lock de otro.

---

## 6c. Remote state con `backend` block

### Backend arguments

- Los argumentos del cuerpo del `backend` block son **específicos de cada tipo** — definen dónde y cómo guarda el state.
- Algunos backends permiten pasar **credenciales** como argumentos — **no se recomienda** hardcodearlas; mejor dejarlas sin definir y proveerlas vía archivos de credenciales o **variables de entorno** convencionales del sistema destino.

### Credenciales y datos sensibles

- ⚠️ Recomendación: usar **variables de entorno** para credenciales y datos sensibles. Si usas `-backend-config` o las hardcodeas en la config, Terraform las incluye **en texto plano** tanto en el directorio `.terraform` como en los **plan files**.
- Terraform escribe la configuración del backend en texto plano en:
  - **`.terraform/terraform.tfstate`**: contiene la config del backend del working directory actual.
  - **Plan files**: capturan esa misma info en el momento de crear el plan (garantiza aplicar el plan al set correcto de infraestructura).
- Al aplicar un plan guardado previamente, Terraform usa la config de backend **guardada en ese archivo**, no la actual — si esa config tenía credenciales de vida corta, pueden **expirar** antes de terminar el apply. Usar variables de entorno si necesitas valores distintos entre `plan` y `apply`.

### Inicializar el backend

- Al cambiar la config de un backend, hay que volver a correr **`terraform init`** para validar/configurar el backend antes de poder hacer plan/apply/operaciones de state.
- Tras `init`, Terraform crea un directorio **`.terraform/`** local con la config de backend más reciente (incluidos parámetros de auth) — **no debe subirse a Git** (puede contener credenciales).
- La config del backend local (`.terraform/`) es **distinta y separada** del `terraform.tfstate` que contiene los datos de state reales — ese vive en el backend remoto.
- Al cambiar de backend, Terraform ofrece **migrar el state** al nuevo backend. **Recomendación fuerte**: hacer backup manual del `terraform.tfstate` antes de migrar.

### Partial configuration

- No hace falta especificar **todos** los argumentos requeridos del backend en la config — se puede dejar una **partial configuration**, completando el resto durante `terraform init`. Útil cuando algunos valores los provee un script de automatización.
- 3 formas de completar la config parcial:
  1. **Archivo**: `terraform init -backend-config=PATH`. El bloque `backend` en la config debe tener las keys con valores **vacíos**; el archivo de config (convención de nombre: `*.backendname.tfbackend`, ej. `config.consul.tfbackend`) las rellena.
  2. **Pares clave/valor en línea de comandos**: `-backend-config="KEY=VALUE"` (repetible). No recomendado para secretos (queda en el historial del shell).
  3. **Interactivamente**: Terraform pregunta los valores requeridos (no pregunta por los opcionales), salvo que el input interactivo esté deshabilitado.
- Si hay settings en varios sitios, se **mergean**: las opciones de línea de comandos sobreescriben la config principal, y entre varias `-backend-config` procesadas en orden, las últimas sobreescriben a las anteriores.
- La config final mergeada se guarda en `.terraform/` (que debe ignorarse en VCS) — así se puede omitir info sensible del control de versiones, aunque queda en texto plano en disco local.
- Con partial config, como mínimo hace falta un bloque `backend` (vacío) en la config raíz para indicar el **tipo**: `backend "consul" {}`.

### Cambiar / eliminar la configuración del backend

- Se puede cambiar la config del backend (o incluso el **tipo**, ej. de `consul` a `s3`) en cualquier momento.
- Terraform detecta el cambio automáticamente y pide **reinicialización** (`terraform init`), ofreciendo **migrar el state existente** al nuevo backend.
- Con **múltiples workspaces**, Terraform puede copiar todos los workspaces al destino (pregunta si es lo que quieres).
- Si solo estás re-configurando el **mismo** backend, Terraform también pregunta si quieres migrar — puedes responder "no".
- Para **eliminar** un backend: quitar el bloque `backend` de la config y reinicializar — Terraform ofrece migrar el state al backend **`local`** por defecto.

---

### 💡 Puntos clave para el examen (6c)

- El `backend` block puede tener una **partial configuration**: archivo (`-backend-config=PATH`), pares clave/valor (`-backend-config="K=V"`), o interactivo — se mergean con prioridad a las opciones de CLI (y entre estas, la última gana).
- Con partial config, el tipo de backend **siempre** debe estar en la config raíz (aunque sea `backend "consul" {}` vacío).
- Cambiar de backend (o de tipo de backend) → requiere `terraform init` → Terraform **ofrece migrar el state** automáticamente.
- **No hardcodear credenciales** en el `backend` block ni pasarlas por `-backend-config` en texto plano — usar variables de entorno (quedan en `.terraform/terraform.tfstate` y en los plan files en claro si no).
- `.terraform/` (con la config de backend resuelta) **nunca** va a Git.

---

## 6d. Drift, refactor de state, refresh-only, moved/removed blocks

### Qué es y para qué sirve el state (repaso ampliado — ver también [02-fundamentals.md](02-fundamentals.md) 2d)

- Terraform usa el state del workspace para **mapear** recursos reales a la config, trackear **metadata**, y mejorar el **rendimiento** en infra grande.
- Antes de cualquier operación, Terraform hace un **refresh** para actualizar el state con la infra real.
- Propósito primario: guardar los **bindings** entre objetos de un sistema remoto y las resource instances declaradas en la config.

### Guardar el state

- Por defecto: `terraform.tfstate` local + backup en `terraform.tfstate.backup`. Sin config adicional, pero limita la colaboración y arriesga perder el state si se pierde el archivo local.
- Recomendado: HCP Terraform o un **backend remoto**.
- ⚠️ Evitar guardar el state en un VCS u otro storage **sin locking ni control de acceso seguro** — riesgo de pérdida de datos o exposición de secretos.

### Inspección y modificación

- El state se guarda como **JSON** — **no editar el archivo directamente**. Usar el comando **`terraform state`** (subcomandos) para modificaciones básicas vía CLI.
- El output del CLI está pensado para ser friendly con herramientas Unix (`grep`, `awk`...), aislando al usuario de cambios de formato internos.
- Terraform espera un mapeo **uno a uno** entre resource instances configuradas y objetos remotos — normalmente garantizado porque es el propio Terraform quien crea/destruye y registra esos bindings.
- Si añades/quitas bindings por otros medios (`terraform import`, `terraform state rm`), **tú** eres responsable de mantener esa regla 1:1 (ej. borrar manualmente un objeto "olvidado", o reimportarlo a otra resource instance).

### Formato del state y salidas JSON pensadas para software externo

- El formato JSON del state puede **cambiar entre versiones** de Terraform — si construyes software que lo parsea/modifica directamente, requiere mantenimiento continuo.
- Alternativas **estables** pensadas para integración externa:
  - **`terraform output -json`**: valores de output del último state snapshot.
  - **`terraform show -json`**: inspecciona el último state snapshot completo, o un plan file guardado (incluye copia del state previo al plan).
- Patrón típico en automatización: correr estos comandos justo tras un `apply` exitoso y guardar el resultado como artefacto.

### Refactorizar el state (dividir configuraciones)

**Cuándo conviene refactorizar:**

- **Applies largos**: config monolítica que ha crecido demasiado → plan/apply lentos, riesgo de cambios no intencionados.
- **Ciclos de vida distintos**: separar recursos que cambian frecuentemente de los que cambian poco, reduce el "blast radius".
- **Cambios de ownership**: equipos que dividen responsabilidad de partes de la arquitectura.
- **Oportunidad de módulo reutilizable**: una subsección de config que se repite en varios sitios → convertir en módulo.

**Cómo agrupar recursos al planificar el refactor:**

- **Volatilidad / rate of change**: separar infra que cambia mucho de la que es estable (ej. compute vs networking).
- **Stateful vs stateless**: gestionar recursos con estado (BDs) por separado de los stateless → limita el blast radius de operaciones que recrean recursos, protege contra pérdida accidental de datos.
- **Acceso y responsabilidad de equipo**: dividir workspaces por equipo/ownership.

**Identificar dependencias antes de migrar:**

- Si migras un recurso a otro state, los recursos que dependían de él en el state original pueden dejar de poder referenciarlo.
- Recomendación: usar **referencias dinámicas** en vez de hardcodear información. Opciones:
  - **Data source específico del provider** (ej. `aws_vpc`) para consultar el proveedor cloud.
  - **`tfe_outputs`** (HCP Terraform/TFE) para leer outputs de otro workspace.
  - **`terraform_remote_state`** para otros backends remotos o local. (En HCP Terraform, hay que autorizar explícitamente qué workspaces pueden leer el state de otro — remote state sharing).
- **`terraform graph`** ayuda a visualizar relaciones/dependencias entre recursos.

### Migrar recursos entre state files — dos enfoques

**Recursos stateless**: mejor **recrearlos** en la nueva config si no implica downtime/coste extra.

**Recursos stateful** (BDs, object stores): más complejo — normalmente hay que **mover** entre state files. Dos métodos:

**1. Remove and import** (recomendado, requiere **Terraform 1.7+**) — usa bloques `removed` + `import`, mantiene un registro histórico de la config:

Config origen — pasos:
1. Backup: `terraform state pull > terraform.tfstate.backup`.
2. Revisar qué atributo necesita el provider para importar ese resource type (ej. `id` para `aws_instance`): `terraform state show aws_instance.example`.
3. Sustituir el `resource` block por un **`removed`** block:
   ```hcl
   removed {
     from = aws_instance.example
     lifecycle {
       destroy = false
     }
   }
   ```
4. `terraform plan` → confirma que Terraform **no destruirá** el recurso.
5. `terraform apply` → elimina el resource del state (sin tocar la infra real).

Config destino — pasos:
1. Añadir el `resource` block correspondiente.
2. Añadir un bloque **`import`**:
   ```hcl
   resource "aws_instance" "example" {
     instance_type = "t3.micro"
     ami           = data.aws_ami.example.id
   }

   import {
     id = "i-07b510cff5f79af00"
     to = aws_instance.example
   }
   ```
3. `terraform plan` → confirma que se **importará** correctamente.
4. `terraform apply` → completa el import (`Plan: 1 to import, 0 to add, 0 to change, 0 to destroy.`).

Tras completar, los bloques `removed`/`import` se pueden eliminar o dejar como registro histórico.

**2. `terraform state mv`** (legacy, requiere CLI **1.0+**) — mueve recursos directamente entre state files:

- Con backend **local**: mover directamente entre dos archivos de state.
- Con backend **remoto**: primero `terraform state pull` en origen y destino (guardar a archivos locales), luego:
  ```bash
  terraform state mv -state source/source.tfstate -state-out destination/destination.tfstate aws_instance.example aws_instance.example
  ```
- Después, `terraform state push` en ambos directorios (origen y destino) para subir los archivos modificados al backend remoto.
- Luego actualizar ambas configs (quitar el resource del origen, añadirlo al destino) y verificar con `terraform plan` (0 cambios) en ambos lados antes de mergear.
- Terraform recomienda usar **remove+import** para migraciones nuevas — `state mv` tiene más riesgo de corromper el state remoto pese a sus safety checks.

### `removed` block — referencia completa

Elimina un resource del **state** sin tocar la infraestructura real.

```hcl
removed {
  from = <resource.address>
  lifecycle {
    destroy = <true|false>
  }
  connection { ... }        # opcional, para provisioners
  provisioner "<TYPE>" {
    when = destroy
    ...
  }
}
```

- **`from`** (string, **requerido**): address del resource a eliminar del state.
- **`lifecycle.destroy`** (bool): por defecto Terraform **elimina del state y destruye** el recurso real. `destroy = false` → elimina **solo del state**, sin destruir la infra (útil para transferir la gestión a otra herramienta/equipo).
- **`connection`** / **`provisioner`**: solo se soportan **destroy-time provisioners** dentro de `removed` (requieren `when = destroy`), para ejecutar acciones al eliminar.
- Se puede declarar en cualquier parte de la config — buena práctica: estandarizar la ubicación (ej. mismo archivo donde estaba el `resource` original).

### `moved` block — referencia completa

Cambia **programáticamente** la dirección de un resource sin destruirlo/recrearlo (renombrado o movido a un módulo distinto).

```hcl
moved {
  from = aws_instance.a
  to   = aws_instance.b
}
```

- **`from`** / **`to`** (string, ambos **requeridos**): direcciones anterior y nueva. La sintaxis permite seleccionar módulos, resources, y resources dentro de child modules.
- Antes de generar el plan, Terraform comprueba si existe un objeto en el state en la address de `from` → si existe, lo **renombra** a `to` y genera el plan como si siempre hubiera estado ahí. Resultado: Terraform **no destruye** el recurso durante ese run.
- Uso típico: renombrar un resource, o mover un resource dentro/fuera de un módulo, **sin recrearlo**.

### Refresh-Only Mode y otras opciones de plan/apply (repaso — detalle base en [03-workflow.md](03-workflow.md) 3d)

- **Refresh-only mode**: `terraform plan -refresh-only` / `apply -refresh-only` (CLI), o "Refresh state" como run type en la UI de HCP Terraform. Actualiza el state para que coincida con cambios hechos **fuera** de Terraform (drift) — **no** provoca más cambios en los objetos remotos. Requiere CLI **v0.15.4+**.
- **`-refresh=false`**: salta el refresh automático del state antes de comprobar cambios de config (modo normal de planning).
- **`-replace=ADDRESS`**: reemplaza el objeto en la address dada (requiere CLI **v0.15.2+**).
- **`-target=ADDRESS`** (Targeted plan/apply): solo para circunstancias excepcionales. En HCP Terraform, un plan targeted tiene limitaciones: **Sentinel** solo ve el subconjunto de recursos seleccionados (puede fallar una policy que dependa de un recurso excluido), y **Cost Estimation se desactiva** (para no dar una estimación de coste engañosa por recursos excluidos). Se puede restringir el uso de `-target` en un workspace vía policy Sentinel (`tfrun.target_addrs`).
- **Generating Configuration** (con bloques `import`): Terraform puede **autogenerar configuración** en el plan para recursos importados que no tengan ya un `resource` block. Requiere CLI **v1.5.0+**. CLI: `terraform plan -generate-config-out=generated.tf`. Tras generarla, hay que **revisarla, incorporarla a la config** (commitear) y volver a planificar — **no se puede aplicar directamente** un plan con config autogenerada (da error).

---

### 💡 Puntos clave para el examen (6d)

- El state garantiza mapeo **1:1** entre resource instances y objetos remotos — al usar `import`/`state rm` fuera del flujo normal, esa garantía pasa a ser responsabilidad tuya.
- Formato del state = JSON, **cambia entre versiones** → para integraciones externas usar `terraform output -json` / `terraform show -json`, no parsear el state directamente.
- Migrar recursos entre state files: **`removed` + `import`** (recomendado, Terraform 1.7+, mantiene historial) vs **`terraform state mv`** (legacy, 1.0+, más riesgoso).
- `removed { lifecycle { destroy = false } }` = sacar del state **sin destruir** la infra real (default es `destroy = true`).
- `moved { from, to }` = renombrar/mover un resource **sin destruir/recrear** — Terraform lo detecta antes de generar el plan.
- **Refresh-only mode** = sincroniza state con drift externo, sin tocar infraestructura real.
- `-target` en HCP Terraform: **desactiva Cost Estimation** y limita el alcance de las policy checks de Sentinel al subconjunto targeted.
- Autogenerar config desde `import` blocks (`-generate-config-out`) requiere **revisión manual y un plan adicional** antes de poder aplicar — nunca se aplica directamente.
