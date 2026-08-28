# ADR-008: Estrategia de administración y optimización de costos

**Estado:** Aceptado
**Fecha:** 2026-08-27
**Lección relacionada:** Lección 8 - Administración de costos en la nube

## Contexto

A lo largo de las Lecciones 1-7 se identificaron componentes con perfiles de
uso muy distintos:

| Componente | Patrón de uso | Lección |
|---|---|---|
| RDS PostgreSQL Multi-AZ | Continuo, 730 h/mes, sin variación estacional | ADR-002 |
| Fargate — capacidad base (4 tareas) | Continuo, 730 h/mes | ADR-004 |
| Fargate — capacidad CyberDay (36 tareas) | Estacional, ~20 h/mes, 4 veces al año | ADR-004 |
| NAT Gateway, ALB | Continuo, pero no son recursos de cómputo elegibles para RI/SP | ADR-005 |
| S3, CloudFront, SQS, SNS | Pago por uso, ya optimizado por diseño (Cap. 6-7) | ADR-001, 006, 007 |

Según el material de esta lección, **Reserved Instances (RI)** y **Savings
Plans (SP)** ofrecen descuentos significativos (hasta 72% y 66%
respectivamente) a cambio de un compromiso de uso, pero están
explícitamente recomendados para **"cargas estables y predecibles"**, no
para tráfico estacional.

## Decisión

Se aplica una estrategia de **compromiso selectivo**, no total:

**Reserved Instance (RDS):**
- RDS PostgreSQL Multi-AZ (ADR-002) se cubre con una **Reserved Instance a
  1 año**, pago parcial adelantado.
- Justificación: es el componente de mayor costo fijo del proyecto y su
  patrón de uso (24/7, sin variación) es exactamente el caso de uso que
  la tabla del material identifica para RI.

**Compute Savings Plan (Fargate, solo capacidad base):**
- Solo las **4 tareas de capacidad base** (ADR-004) se cubren con un
  Compute Savings Plan a 1 año.
- La **capacidad CyberDay (36 tareas adicionales, ~20h/mes)** permanece
  **On-Demand**, sin compromiso, comprometer capacidad que se usa <3% del
  tiempo total del año sería pagar por una reserva que rara vez se
  utiliza, anulando el propio ahorro buscado.

**Sin compromiso (quedan On-Demand / pago por uso):**
- NAT Gateway, ALB, no son elegibles para RI/SP.
- S3, CloudFront, SQS, SNS, AWS Backup, ya son pago por uso puro; el
  ahorro en estos componentes ya se logró por diseño (clasificación de
  contenido, cache-busting, colas correctas) en lecciones anteriores, no
  por compromiso financiero.

## Monitoreo de costos

- **AWS Budgets:** presupuesto mensual configurado sobre el total
  acumulado del proyecto, con alerta al superar el 100% y al proyectarse
  por encima del 110%.
- **CloudWatch + AWS/Billing:** alarma sobre la métrica `EstimatedCharges`
  (namespace `AWS/Billing`), con la salvedad documentada en el material de
  que esta métrica se actualiza cada 4-6 horas, no en tiempo real,
  adecuada para detección de tendencia, no para reacción inmediata.
- **AWS Cost Explorer + Cost Allocation Tags:** se define una convención de
  etiquetas (`proyecto:adc`, `leccion:01` a `leccion:08`) para poder filtrar
  el gasto por lección directamente en Cost Explorer, en vez de depender
  solo de las estimaciones manuales documentadas en cada `costos.md`.

## Pilares de AWS Well-Architected Framework

| Pilar | ¿Cómo lo aborda esta decisión? |
|---|---|
| **Cost Optimization** | Compromiso selectivo basado en el patrón de uso real de cada componente, no una política uniforme |
| **Operational Excellence** | Monitoreo automatizado (Budgets + CloudWatch) reemplaza la revisión manual periódica que el propio material identifica como más lenta y costosa a largo plazo |

## Alternativas consideradas

| Opción | Ventajas | Desventajas | ¿Por qué se descartó / eligió? |
|---|---|---|---|
| **Compromiso selectivo (RDS + Fargate base)** (elegida) | Ahorro real donde el patrón lo justifica, sin comprometer capacidad estacional | Requiere identificar correctamente qué es estable y qué no | Elegida: es exactamente el criterio que enseña el material |
| RI/SP sobre toda la capacidad de cómputo (incluyendo CyberDay) | Máximo descuento nominal | Se pagaría por capacidad reservada usada <3% del año | Descartada, contradice el propio objetivo de ahorro |
| Sin ningún compromiso (100% On-Demand) | Máxima flexibilidad, cero riesgo de sobre-comprometer | Deja sobre la mesa un descuento significativo en el componente de mayor costo fijo (RDS), sin ninguna razón de negocio para no comprometerlo | Descartada, es la opción más simple, pero no la más eficiente |
| Reserved Instances para Fargate (en vez de Savings Plans) | — | RI no cubre Fargate según la tabla del material (aplica a EC2/RDS/Redshift/ElastiCache); Fargate solo tiene descuento vía Savings Plans | Descartada por no ser técnicamente viable |

## Métricas de éxito

- **Presupuesto mensual objetivo:** definido sobre el total consolidado (ver `resumen-costos-total.md`)
- **Ahorro esperado:** se calcula en `docs/08-administracion-costos/resumen-costos-total.md`, comparando el costo On-Demand puro (Lecciones 1-7) contra el costo aplicando RI/SP selectivo

## Diseño ideal vs. restricción del AWS Academy Learner Lab

| Aspecto | Diseño ideal (producción) | Lo que se implementa/documenta en el Lab | Motivo de la brecha |
|---|---|---|---|
| Compra real de RI/Savings Plans | Compromiso activo de 1-3 años | Estrategia y cálculo de ahorro documentados, sin ejecutar la compra | Las sesiones del Learner Lab son de horas y las cuentas son efímeras — no aplica un compromiso de largo plazo |
| AWS Budgets con notificación real | Alerta a email corporativo de ADC | Configuración documentada | No hay email corporativo real para este proyecto académico |
| AWS Cost Explorer con historial | Tendencias sobre meses de uso real | Sin datos que mostrar (sesión de horas) | Se documenta el diseño de tagging para cuando exista uso real en producción |

## Costos estimados

- **Fuente:** AWS Pricing Calculator — *(el ahorro proyectado se calcula en `docs/08-administracion-costos/resumen-costos-total.md`)*
- **Resumen:** pendiente

## Consecuencias

**Positivas:**
- Ahorro real en el componente de mayor costo fijo del proyecto (RDS), sin sacrificar elasticidad donde importa (CyberDay)
- Monitoreo automatizado reduce la dependencia de revisión manual
- La convención de tags permite auditar el gasto por lección/componente en cualquier momento

**Negativas / riesgos:**
- Comprometerse a una RI de 1 año asume que el motor/tamaño de RDS no cambiará en ese período, si ADC migra a otro tipo de instancia antes de tiempo, pierde parte del beneficio (a diferencia de un Savings Plan, más flexible)
- La alarma de `EstimatedCharges` tiene un retraso de 4-6 horas, no sirve para detectar un pico de gasto en tiempo real durante CyberDay

## Referencias

- AWS Cost Explorer: https://aws.amazon.com/aws-cost-management/aws-cost-explorer/
- AWS Budgets: https://aws.amazon.com/aws-cost-management/aws-budgets/
- Reserved Instances vs Savings Plans: https://aws.amazon.com/savingsplans/