# 1. Infrastructure as Code (IaC) con Terraform

| Subtema | Descripción | Cubierto |
|---|---|---|
| 1a | Explain what IaC is | ✅ |
| 1b | Describe the advantages of IaC patterns | ✅ |
| 1c | Explain how Terraform manages multi-cloud, hybrid cloud, and service-agnostic workflows | ✅ |

Fuentes: "What is Terraform?", "Why Terraform?", "Infrastructure as Code in a Private or Public Cloud"

---

## 1a. Qué es IaC / Qué es Terraform

- Terraform es una herramienta de **Infrastructure as Code (IaC)**: define infraestructura (cloud y on-prem) en **archivos de configuración legibles por humanos** que se pueden versionar, reutilizar y compartir.
- Con un workflow consistente se **provisiona y gestiona** infraestructura durante todo su ciclo de vida.
- Gestiona tanto componentes **low-level** (compute, storage, networking) como **high-level** (DNS, features de SaaS).

### Cómo funciona Terraform

- Terraform crea y gestiona recursos en plataformas cloud y otros servicios a través de sus **APIs**.
- Los **providers** son los que permiten a Terraform hablar con prácticamente cualquier plataforma/servicio que tenga una API accesible.
- Existen miles de providers públicos en el **Terraform Registry** (AWS, Azure, GCP, Kubernetes, Helm, GitHub, Splunk, DataDog, etc.), y también se pueden escribir providers propios.

### Core Terraform workflow (introducción — ver sección 3 para detalle)

Tres fases:

1. **Write**: defines los recursos en configuración (pueden abarcar múltiples providers/clouds). Ej: VMs en una VPC + security groups + load balancer.
2. **Plan**: Terraform genera un **execution plan** describiendo qué va a crear, actualizar o destruir, comparando el estado existente con la configuración.
3. **Apply**: tras aprobación, Terraform ejecuta las operaciones propuestas **en el orden correcto**, respetando dependencias entre recursos.
   - Ej: si cambias propiedades de una VPC y también el nº de VMs dentro, Terraform recreará primero la VPC antes de escalar las VMs.

### Por qué cambiar de la gestión manual a IaC

- Al mover infraestructura a cloud, muchos equipos siguen gestionándola **como si fuera hardware físico on-prem**: logueándose en la consola web o por CLI/GUI para aplicar cambios a mano. Esto **no es IaC**.
- IaC = infraestructura (CPU, memoria, disco, firewalls, etc.) **definida como código** en archivos de definición.
- Evolución: de scripts (difíciles de leer) → herramientas modernas con código **legible por humanos y máquinas**, que además:
  - simplifican el testing del código,
  - permiten aplicar y trackear cambios entre iteraciones,
  - permiten **reutilizar componentes** (módulos) entre proyectos.
- Muchas veces el cambio de gobernanza/herramientas solo llega **tras un fallo** (outage, DR fallido, etc.) — no por elección proactiva.

### IaC y el ciclo de vida de la infraestructura (Day 0 / Day 1)

- **Day 0**: código que **provisiona y configura** la infraestructura inicial (ej. crear una VPC).
- **Day 1 (→ Day N)**: configuraciones de OS y aplicación que se aplican **después** del build inicial (updates, patches, config de apps). Aquí suelen entrar herramientas como Chef, Ansible, Docker junto con Terraform.
- Si la infraestructura nunca cambia tras el build inicial, puede que no necesites herramientas para updates/cambios posteriores — pero normalmente sí se necesitan.
- IaC estandariza workflows **entre distintos proveedores** (VMware, AWS, Azure, GCP...) usando una **sintaxis común**.
- El código IaC puede organizarse en **múltiples archivos** según intención (ej. separar definición de variables de los bloques de ejecución), lo que facilita entender el impacto de un cambio.

Ejemplo de código Terraform (Day 0 — provisionar una VPC en AWS):

```hcl
resource "aws_vpc" "default" {
  cidr_block = "10.0.0.0/16"
}
```

Ejemplo de configuración Day 0 con un `provisioner` (instalar y arrancar un servidor web):

```hcl
provisioner "remote-exec" {
  inline = [
    "sudo apt-get -y update",
    "sudo apt-get -y install nginx",
    "sudo service nginx start"
  ]
}
```

Para Day 1 → Day N, se suele delegar en herramientas externas (ej. Chef):

```hcl
provider "chef" {
  server_url = "https://api.chef.io/organization/example"
  run_list   = ["recipe[example]"]
}
```

### IaC hace la infraestructura más fiable (reliable)

- IaC hace los cambios **idempotentes, consistentes, repetibles y predecibles**.
- Sin IaC: escalar infra a mano implica conectarse a cada máquina y ejecutar comandos manualmente → propenso a pasos saltados, variaciones entre servidores, rollbacks por errores humanos. Estas inconsistencias se **acumulan con el tiempo** y afectan performance/seguridad, sobre todo con equipos grandes.
- Con IaC: se puede **testear el código y revisar resultados** (plan) antes de aplicarlo al entorno real. Si el resultado no es el esperado, se itera sobre el código hasta que sí lo sea — el resultado se puede **predecir antes de aplicar** en producción.
- Al aplicarse mediante automatización, se garantiza **consistencia y repetibilidad** a escala.
- Al estar el código en un **VCS** (GitHub, GitLab, Bitbucket...), se puede revisar cómo evoluciona la infraestructura en el tiempo.
- **Idempotencia**: aplicar el mismo código varias veces produce el mismo resultado.

### IaC hace la infraestructura más manejable (manageable)

- Terraform permite **mutar infraestructura vía código** cuando es necesario (ej. añadir servidores a un entorno con load balancer para atender más carga): basta con revisar el código con cambios mínimos.
- Durante la ejecución, Terraform **examina el estado actual** de la infraestructura corriendo, determina las **diferencias** con el estado deseado (revisado) e indica los cambios necesarios.
- Tras aprobación, **solo se aplican los cambios necesarios**, dejando intacta la infraestructura válida existente.

## 1b. Ventajas de los patrones IaC (por qué Terraform)

Terraform permite definir y gestionar infraestructura de forma **consistente y repetible** mediante configuración legible y versionable. Ventajas clave:

- **Gestionar cualquier infraestructura**
  - Providers para casi cualquier plataforma/servicio (Terraform Registry), o se pueden escribir providers propios.
  - Enfoque **inmutable** de infraestructura → reduce la complejidad de actualizar/modificar servicios (en vez de mutar recursos existentes, se reemplazan).

- **Trackear la infraestructura**
  - Terraform genera un plan y pide **aprobación** antes de modificar infraestructura.
  - Mantiene un **state file** que actúa como fuente de verdad (source of truth) del entorno real.
  - Usa el state para calcular qué cambios aplicar para que la infraestructura real coincida con la configuración.

- **Automatizar cambios**
  - La configuración es **declarativa**: describe el estado final deseado, no los pasos para llegar a él.
  - No hace falta escribir instrucciones paso a paso — Terraform gestiona la lógica subyacente.
  - Terraform construye un **resource graph** para determinar dependencias entre recursos y puede crear/modificar recursos **no dependientes en paralelo** → provisión eficiente.

- **Estandarizar configuraciones**
  - Soporta **módulos**: componentes de configuración reutilizables que agrupan colecciones configurables de infraestructura.
  - Se pueden usar módulos públicos del Terraform Registry o escribir los propios.
  - Ahorran tiempo y fomentan buenas prácticas.

- **Colaborar**
  - Al estar la configuración en archivos, se puede versionar en un **VCS**.
  - **HCP Terraform** permite gestionar workflows de Terraform en equipo de forma eficiente: entorno consistente, acceso seguro a state compartido y secretos, RBAC, private registry para módulos y providers, etc.

## 1c. Multi-cloud, hybrid cloud y workflows agnósticos de servicio

- Terraform es **agnóstico de plataforma**: gracias a los providers, puede gestionar recursos de múltiples nubes (AWS, Azure, GCP...), on-prem, y SaaS **con el mismo workflow** (write → plan → apply).
- Al escribir configuración con **múltiples providers** en un mismo proyecto, se pueden orquestar recursos que viven en distintas nubes o en cloud + on-prem (hybrid cloud) de forma unificada.
- Esto permite un workflow **consistente y repetible** independientemente de dónde viva el recurso — la lógica de "escribir configuración declarativa, planificar, aplicar" es la misma sin importar el proveedor.

### Multi-Cloud Deployment (Terraform Use Cases)

- Provisionar infraestructura en **múltiples clouds** aumenta la tolerancia a fallos (recuperación más elegante ante un outage de un proveedor).
- Contrapartida: los despliegues multi-cloud añaden **complejidad**, porque cada proveedor tiene sus propias interfaces, herramientas y workflows.
- Terraform permite usar el **mismo workflow** para gestionar múltiples providers y manejar **dependencias cross-cloud**, simplificando la gestión/orquestación de infraestructuras multi-cloud a gran escala.
- Ejemplo típico: provisionar clusters Kubernetes en Azure y AWS a la vez, configurar federación entre ellos (ej. con Consul mesh gateways) y desplegar microservicios en ambos.

### Otros casos de uso destacados de Terraform (contexto — no todos entran en 1c pero dan contexto de "qué resuelve Terraform")

- **Application infrastructure (deploy, scale, monitor)**: gestiona arquitecturas n-tier (ej. web servers → DB tier → API → caching → routing mesh) y **maneja automáticamente las dependencias entre tiers** (ej. despliega la capa de BD antes que los web servers que dependen de ella).
- **Self-service clusters**: equipos de producto pueden auto-gestionar su infraestructura usando **módulos** que codifican los estándares de la organización, reduciendo tickets repetitivos al equipo central de operaciones. HCP Terraform puede integrarse con sistemas de ticketing (ej. ServiceNow).
- **Policy compliance and management**: en vez de revisiones manuales por ticket (cuello de botella), se usa **Sentinel** (policy-as-code, disponible en Terraform Enterprise / HCP Terraform) para forzar automáticamente políticas de compliance/gobernanza antes de aplicar cambios. También existe **cost estimation** para limitar costes asociados a cambios de infra.
- **PaaS application setup**: codificar el setup de apps en PaaS (ej. Heroku) junto con add-ons externos (DNS, CDN...) de forma rápida y consistente, sin usar la interfaz web.
- **Software Defined Networking (SDN)**: Terraform puede configurar automáticamente la red según las necesidades de las apps. Ej: **Consul-Terraform-Sync (Network Infrastructure Automation / NIA)** genera configuración Terraform automáticamente cuando un servicio se registra en Consul, pasando de un workflow basado en tickets a uno automatizado.
- **Kubernetes**: Terraform puede tanto **desplegar un cluster K8s** como **gestionar sus recursos** (pods, deployments, services...). El **HCP Terraform Operator** permite gestionar infra cloud/on-prem vía un Custom Resource Definition (CRD) de Kubernetes junto con HCP Terraform.
- **Parallel environments**: crear y destruir rápidamente entornos de dev/test/QA/producción — más coste-eficiente que mantenerlos indefinidamente.
- **Software demos**: provisionar y bootstrapear demos en distintos cloud providers para que usuarios finales prueben el software en su propia infraestructura.

---

### 💡 Puntos clave para el examen

- IaC = infraestructura definida en archivos versionables, no gestión manual/clicks en consola.
- Workflow core: **Write → Plan → Apply**.
- Terraform habla con APIs vía **providers**.
- Config es **declarativa** (describe el "qué", no el "cómo").
- **State file** = fuente de verdad de la infraestructura real gestionada por Terraform.
- **Resource graph** permite paralelizar recursos no dependientes.
- **Módulos** = reutilización y estandarización de configuración.
- Enfoque **inmutable**: se recrean recursos en vez de mutarlos in-place cuando es necesario.
