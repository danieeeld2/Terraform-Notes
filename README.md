# Terraform Associate (004) — Apuntes de estudio

Índice de objetivos del examen (Exam Content List 004) y estado de apuntes por tema.
Fuente: `HashiCorp.pdf` (Exam Content List - Terraform Associate 004).

Notas del examen 004 vs 003:
- 4 temas nuevos: 4f (depends_on / create_before_destroy), 4g (custom conditions), 4h (ephemeral values / write-only arguments), 8c (workspaces y projects en HCP Terraform)
- Examina sobre Terraform 1.12
- Incluye contenido de HCP Terraform

## Progreso

| # | Tema | Archivo | Estado |
|---|------|---------|--------|
| 1 | Infrastructure as Code (IaC) con Terraform | [01-iac.md](01-iac.md) | ✅ completo |
| 2 | Terraform fundamentals | [02-fundamentals.md](02-fundamentals.md) | ✅ completo |
| 3 | Core Terraform workflow | [03-workflow.md](03-workflow.md) | ✅ completo |
| 4 | Terraform configuration | [04-configuration.md](04-configuration.md) | 🟡 99% (falta fuente oficial de `create_before_destroy` y detalle del Vault provider) |
| 5 | Terraform modules | [05-modules.md](05-modules.md) | ✅ completo |
| 6 | Terraform state management | [06-state.md](06-state.md) | ✅ completo |
| 7 | Maintain infrastructure con Terraform | [07-maintain.md](07-maintain.md) | ✅ completo |
| 8 | HCP Terraform | [08-hcp-terraform.md](08-hcp-terraform.md) | ✅ completo |

## Repaso / autoevaluación

| Documento | Contenido |
|---|---|
| [09-quiz-teorico.md](09-quiz-teorico.md) | Preguntas V/F, opción múltiple y respuesta múltiple (formato oficial del examen), organizadas por los 8 temas. Respuestas en desplegables. |
| [10-quiz-practico.md](10-quiz-practico.md) | Escenarios con código HCL y situaciones "qué pasaría si..." — variables/tipos, resources y meta-argumentos, state, providers/módulos, workflow/CLI, sensitive/ephemeral, HCP Terraform. Respuestas en desplegables. |

## Cómo trabajamos

1. Pegas el contenido de un enlace de documentación (o el link).
2. Te lo resumo en apuntes dentro del `.md` de la sección correspondiente, en el bloque del subtema (1a, 1b, ...).
3. Marco el subtema como cubierto en la tabla de esa sección.
4. Cuando una sección esté completa, la marco como ✅ aquí.
