# Estimación de Costos: Lección 5 - Disponibilidad de Aplicaciones en la Red

**Fuente:** AWS Pricing Calculator
**Link de la estimación:** https://calculator.aws/#/estimate?id=f318505554209e32ff2dbdf2073eeefddd58c5d2
**Región asumida:** US East (N. Virginia) — `us-east-1`
**Fecha de estimación:** 2026-08-26

## Supuestos de diseño

| Componente | Configuración |
|---|---|
| Application Load Balancer | 1 ALB, ~15 nuevas conexiones/seg promedio, 60s duración, 50 KB/solicitud (~1.971 GB/mes procesados) |
| NAT Gateway | 1 (decisión de ADR-005), 730 h/mes, 10 GB/mes procesados |

**Nota sobre "Regional NAT Gateway":** la calculadora exige un mínimo de 1
en los campos de esta sección alternativa (modalidad de despliegue distinta
al NAT Gateway tradicional), pero su línea de total confirma $0.00 USD/mes
incluso cuando el desglose interno calcula $32.85 como referencia
ilustrativa. Se verificó que este monto no se suma al total general de la
estimación, el costo real de NAT Gateway proviene exclusivamente de la
sección "Network Address Translation (NAT) Gateway" tradicional ($33.30/mes).

## Resultado de la calculadora

| Concepto | Costo mensual estimado (USD) |
|---|---|
| Application Load Balancer | 32.20 |
| NAT Gateway | 33.30 |
| **Total Lección 5** | **65.50** |

*(Nota: estimación de diseño, no refleja el consumo real de créditos del AWS Academy Learner Lab.)*

## Lectura del resultado

El ALB y el NAT Gateway quedan prácticamente empatados en costo ($32.20 vs
$33.30), y juntos forman una capa de costo que, a diferencia de Fargate
(Lección 4), que escala en bloques según la demanda, se mantiene
relativamente **estable independientemente del tráfico real**: el NAT
Gateway cobra sobre todo por las 730 horas activas (el procesamiento de
datos es marginal), y el ALB tiene un componente fijo de LCU-hora que domina
frente al componente variable de bytes procesados. Es el mismo patrón que ya
identificamos con la VPN en la Lección 3: los componentes de **red** tienden
a ser costos fijos de mantener la arquitectura disponible, mientras que los
componentes de **cómputo** (Fargate) son los que realmente responden al
volumen de negocio.

**Costo total acumulado del proyecto hasta la Lección 5:** $411.37 (L1-L4) +
$65.50 (L5) = **$476.87 USD/mes**.