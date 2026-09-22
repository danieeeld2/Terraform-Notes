# 7. Maintain infrastructure con Terraform

| Subtema | Descripción | Cubierto |
|---|---|---|
| 7a | Import existing infrastructure into your Terraform workspace | ✅ |
| 7b | Use the CLI to inspect state | ✅ |
| 7c | Describe when and how to use verbose logging | ✅ |

Fuentes: "terraform import command reference", "Import existing resources", "terraform state commands", "Enable Terraform logs"

---

## 7a. `terraform import` / import blocks

### Qué hace

- Importa infraestructura **existente** al state de Terraform, para poder gestionarla como código.
- Solo puede importar **un recurso a la vez** — no puede importar una colección entera (ej. una VPC completa con todos sus componentes) de golpe.
- Uso: `terraform import [options] ADDRESS ID`
  - **`ADDRESS`**: un [resource address](https://developer.hashicorp.com/terraform/cli/state/resource-addressing) válido — puede apuntar a la raíz del state o **dentro de un módulo**.
  - **`ID`**: depende del **tipo de recurso** (ej. AWS EC2 → instance ID `i-abcd1234`; AWS Route53 zone → zone ID `Z12ABC4UGMOZ2N`; Route53 a veces usa el propio nombre de dominio). Consultar la doc del provider para el formato correcto.

### Workflow paso a paso

1. **Añadir el resource a la config** — escribir un bloque `resource` para el recurso que quieres importar (con un nombre único). No hace falta rellenar todos los argumentos — se puede completar **después** de importar.
   ```hcl
   resource "aws_instance" "example" {
     # ...
   }
   ```
2. **Ejecutar `terraform import`**:
   ```bash
   terraform import aws_instance.example i-abcd1234
   ```
   Terraform localiza el objeto real, adjunta su config/atributos actuales (según la API del provider) a la resource address dada, y **guarda el mapeo en el state**.
3. Tras importar, correr **`terraform plan`** para ver cómo difiere la config actual del recurso importado, y ajustar la config para alinearla con el estado real (o el deseado).

### ⚠️ Regla del mapeo 1:1 (repaso de 2d/6d)

- Terraform espera que cada objeto remoto esté **vinculado a una única resource address**. Si importas el **mismo objeto varias veces** (a distintas addresses), Terraform puede comportarse de forma inesperada.

### Import a módulos, `count`, `for_each`

```bash
# a un módulo
terraform import module.foo.aws_instance.bar i-abcd1234

# a una instancia con count
terraform import 'aws_instance.baz[0]' i-abcd1234

# a una instancia con for_each (Linux/Mac/UNIX)
terraform import 'aws_instance.baz["example"]' i-abcd1234
```

### Flags principales (todos opcionales)

- `-config=PATH`: directorio con la config que configura el provider para el import (default: working directory). Si no hay archivos de config ahí, hay que configurar el provider vía input manual o env vars.
- `-input=true`: pedir input para la config del provider.
- `-lock=false` / `-lock-timeout=0s`: igual que en otros comandos.
- `-no-color`.
- `-parallelism=n` (default 10).
- `-provider=provider` (**deprecated**): sobreescribe el provider a usar — por defecto usa el especificado en la config del resource target (lo recomendado en la mayoría de casos).
- `-var 'foo=bar'` / `-var-file=foo`: fijar variables (interpretadas como expresiones literales del lenguaje Terraform).
- Solo con **HCP Terraform CLI integration** / backend `remote`: `-ignore-remote-version`.
- Solo con backend **`local`**: legacy `-state`, `-state-out`, `-backup`.

### Configuración del provider durante el import

- Terraform intenta cargar los archivos de config que configuran el provider usado. Si no hay config presente (o no para ese provider concreto), **pregunta credenciales** interactivamente, o se pueden pasar por variables de entorno.
- **Limitación**: la config del provider usada para el import **no puede depender de inputs no-variable** — ej. no puede depender de un **data source**.

### Complex imports

- Un import "simple" = un solo recurso al state.
- Un import puede ser **"complejo"**: importa un recurso principal **y además recursos secundarios relacionados** (ej. un `aws_network_acl` trae consigo un `aws_network_acl_rule` por cada regla).
- Esos recursos secundarios **no existen aún en la config** — hay que consultar el output del import y **crear manualmente** un bloque `resource` para cada uno. Si no se hace, Terraform planificará **destruirlos** en el siguiente run (porque están en el state pero no en la config).
- Para renombrar/mover recursos importados: usar los comandos de **state management** (`terraform state mv`, etc. — ver 7b).

### Nota: bloques `import` en configuración (alternativa moderna a `terraform import`)

- En vez de importar manualmente vía CLI, se puede añadir un **bloque `import`** a la configuración, de forma que Terraform importe los recursos automáticamente al correr `terraform apply` — permite **automatizar imports** en pipelines CI/CD (ver también 6d: remove+import, y `-generate-config-out`).

---

### 💡 Puntos clave para el examen (7a)

- `terraform import ADDRESS ID` — importa **un solo recurso** al state; el `ID` depende del tipo de recurso/provider.
- Antes de importar, el `resource` block debe existir (aunque esté vacío/incompleto) en la config.
- Tras importar, correr `plan` para reconciliar config vs recurso real.
- Import "complejo": recursos secundarios asociados que hay que añadir **manualmente** a la config o Terraform los planificará para destruir.
- Regla 1:1: nunca importar el mismo objeto remoto a más de una resource address.
- La config del provider para import **no puede depender de data sources**.
- Bloques `import` en la config = forma moderna/automatizable, alternativa a `terraform import` como comando CLI manual.

---

## 7b. `terraform state` (inspección de state vía CLI)

### Introducción

- Los subcomandos `terraform state` permiten **modificar el state** de forma segura, en vez de editar el archivo directamente.
- Uso: `terraform state <subcommand> [options] [args]`

### Subcomandos principales

| Subcomando | Función |
|---|---|
| `terraform state list` | Lista los resource addresses en el state |
| `terraform state show` | Muestra los atributos de un resource concreto del state |
| `terraform state mv` | Mueve/renombra un item dentro del state (o entre state files) |
| `terraform state rm` | Elimina un item del state sin destruir la infra real |
| `terraform state pull` | Descarga el state remoto y lo imprime en stdout |
| `terraform state push` | Sube un state local para sobreescribir el remoto (peligroso) |
| `terraform state replace-provider` | Cambia el provider asociado a los resources del state |

### Remote state

- Los subcomandos de `state` funcionan igual con state **remoto** que con local — cada lectura/escritura implica un **roundtrip de red completo** (más lento que local), pero el uso del CLI es idéntico.

### Backups

- **Todos** los subcomandos que **modifican** el state escriben un archivo de **backup** (path configurable con `-backup`).
- Los subcomandos **read-only** (como `list`) **no** escriben backup (no modifican nada).
- ⚠️ Los backups de comandos que modifican el state **no se pueden desactivar** — por la sensibilidad del archivo de state, Terraform fuerza siempre ese backup. Si no los quieres conservar, hay que borrarlos manualmente.

### Diseño command-line friendly

- El output y la estructura de estos subcomandos están pensados para funcionar bien con herramientas Unix (`grep`, `awk`...) — se recomienda **encadenarlos (pipe)** con otras herramientas de línea de comandos para filtrado/modificación avanzada.

---

### 💡 Puntos clave para el examen (7b)

- `terraform state <subcommand>` es la forma **segura** de inspeccionar/modificar el state — nunca editar el JSON a mano.
- Funciona igual en local y remoto, solo cambia la latencia (roundtrip de red).
- Comandos que **modifican** el state → **siempre** generan backup (no desactivable); comandos **read-only** (`list`) → no generan backup.
- Pensado para integrarse con herramientas Unix vía pipes.

---

## 7c. Logging verbose (`TF_LOG`)

### Activar logs detallados

- Se activan con la variable de entorno **`TF_LOG`**, seteada a cualquier valor — los logs detallados aparecen en **stderr**.
- Niveles de verbosidad (de más a menos detallado): **`TRACE` > `DEBUG` > `INFO` > `WARN` > `ERROR`**.
- **`TF_LOG=JSON`**: saca logs a nivel `TRACE` o superior, en **encoding JSON parseable**.
  - ⚠️ El encoding JSON de los logs **no es una interfaz estable** — puede cambiar sin aviso. Pensado para dar soporte a tooling específico (esa es la única forma soportada de consumir logs JSON).

### Logging separado: core vs provider

- **`TF_LOG_CORE`**: activa logging solo para **Terraform Core**.
- **`TF_LOG_PROVIDER`**: activa logging solo para los **provider plugins**.
- Ambas aceptan los mismos niveles que `TF_LOG`, pero activan solo un **subconjunto** de los logs.

### Persistir el output

- **`TF_LOG_PATH`**: fuerza que el log se **anexe siempre** a un archivo concreto cuando el logging está activo.
- ⚠️ Aunque se setee `TF_LOG_PATH`, **también hace falta `TF_LOG`** para que se active el logging — `TF_LOG_PATH` por sí solo no activa nada.

### Uso recomendado

- Si encuentras un bug en Terraform, se recomienda incluir el log detallado (ej. subiéndolo a un servicio tipo `gist`) al reportarlo.

---

### 💡 Puntos clave para el examen (7c)

- `TF_LOG` = variable de entorno que activa logs verbose; niveles **TRACE > DEBUG > INFO > WARN > ERROR**.
- `TF_LOG=JSON` → logs en JSON a nivel TRACE+ (formato **no estable**, sujeto a cambios).
- `TF_LOG_CORE` / `TF_LOG_PROVIDER` = logging granular (solo Core o solo providers), mismos niveles que `TF_LOG`.
- `TF_LOG_PATH` = dónde persistir el log — **requiere `TF_LOG` activo**, no funciona solo.
