# Informe Técnico: Lección 8 - Administración de Costos en la Nube

## 1. Objetivo

Documentar la configuración concreta de la estrategia de optimización y
monitoreo de costos definida en
**[ADR-008](../../adr/008-estrategia-costos.md)**.

## 2. Convención de Cost Allocation Tags

Para poder filtrar el gasto por lección directamente en AWS Cost Explorer,
todo recurso creado en este proyecto debe llevar estas etiquetas:

| Tag | Valores posibles | Ejemplo |
|---|---|---|
| `proyecto` | `adc` | `adc` |
| `leccion` | `01` a `08` | `04` (para recursos de Fargate) |
| `componente` | según el recurso | `computo`, `almacenamiento`, `red`, `mensajeria` |
| `ambiente` | `lab`, `produccion` | `lab` (mientras el proyecto viva en AWS Academy) |

Ejemplo aplicado a la instancia RDS: `proyecto=adc`, `leccion=02`,
`componente=basedatos`, `ambiente=lab`.

**Por qué esto importa más allá de "orden":** sin tags, Cost Explorer
muestra el gasto agregado por *servicio* (ej. "todo lo de RDS"), pero no
por *lección* o *propósito de negocio*. Con tags, se puede responder
directamente en la consola preguntas como "¿cuánto cuesta la capa de
mensajería completa?" sin tener que sumar manualmente como hicimos en cada
`costos.md`.

## 3. Configuración de AWS Budgets

| Parámetro | Valor |
|---|---|
| Tipo de presupuesto | Costo mensual |
| Monto | Basado en el total consolidado, ver `resumen-costos-total.md` |
| Alerta 1 | 100% del presupuesto alcanzado (real) |
| Alerta 2 | 110% del presupuesto **proyectado** (forecast), permite reaccionar antes de que el gasto real llegue a superarlo |
| Notificación | Email al responsable del proyecto |

La alerta por **proyección** (no solo por gasto real) es la que más valor
aporta: AWS Budgets estima, según la tendencia del mes en curso, si vas a
terminar por sobre el presupuesto, avisando días antes de que eso
realmente ocurra, en vez de notificar cuando el daño ya está hecho.

## 4. Alarma de CloudWatch sobre `EstimatedCharges`

| Parámetro | Valor |
|---|---|
| Namespace | `AWS/Billing` |
| Métrica | `EstimatedCharges` |
| Umbral | Mismo monto que el presupuesto mensual de Budgets |
| Periodo de evaluación | 6 horas (coherente con el retraso natural de la métrica, documentado en ADR-008) |
| Acción | Notificación SNS *(el mismo servicio que ya diseñamos en ADR-007, se reutiliza como canal de notificación, en vez de crear uno nuevo solo para esto)* |

**Nota importante, tomada directamente del material:** CloudWatch no
provee métricas financieras nativas, `EstimatedCharges` es una excepción
puntual vía integración con AWS Billing, y no reemplaza a Cost Explorer o
Budgets para un análisis serio. Se usa acá como una segunda capa de alerta,
no como la herramienta principal.

## 5. Aplicación de Reserved Instance y Savings Plan

| Compromiso | Recurso | Término | Modalidad de pago |
|---|---|---|---|
| Reserved Instance | RDS PostgreSQL (`db.t3.medium`, Multi-AZ) | 1 año | Parcial adelantado |
| Compute Savings Plan | Fargate — 4 tareas de capacidad base únicamente | 1 año | Sin adelanto |

La capacidad CyberDay de Fargate (36 tareas adicionales) **no** se incluye
en el Savings Plan, permanece On-Demand, tal como se justificó en
ADR-008.

## 6. Relación con otras lecciones

Este informe no introduce arquitectura nueva, es la capa de gobernanza
financiera que envuelve todo lo construido en las Lecciones 1-7. El detalle
numérico del ahorro proyectado se documenta en
`docs/08-administracion-costos/resumen-costos-total.md`, que también sirve
de base para el documento consolidado final del proyecto.