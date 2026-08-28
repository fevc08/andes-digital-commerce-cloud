# Resumen Consolidado de Costos — Andes Digital Commerce (ADC)

## 1. Costo total por lección (modelo On-Demand)

| Lección | Componente | Costo mensual (USD) |
|---|---|---|
| 1 | Almacenamiento de objetos (S3) | 9.23 |
| 2 | Backup y recuperación (RDS + AWS Backup) | 185.48 |
| 3 | Modelo híbrido (VPN) | 36.95 |
| 4 | Escalabilidad de cómputo (Fargate) | 179.71 |
| 5 | Disponibilidad de red (ALB + NAT) | 65.50 |
| 6 | Disponibilidad de contenido (CloudFront) | 3.97 |
| 7 | Mensajería asíncrona (SQS + SNS) | 0.20 |
| **Total On-Demand** | | **481.04** |

## 2. Metodología de cálculo con compromiso (RI / Savings Plan)

**RDS (Reserved Instance):** la calculadora entrega un pago inicial
("Upfront") y un costo mensual recurrente por separado. Para comparar de
forma justa contra el On-Demand, el Upfront se amortiza en 12 meses y se
suma al costo recurrente. El cálculo es: costo mensual efectivo es igual al
costo mensual recurrente ($63.67) más el Upfront ($431.00) dividido en 12
meses ($35.92), lo que da un total de **$99.59 USD/mes**.

**Fargate (Compute Savings Plan):** la AWS Pricing Calculator **no soporta**
Compute Savings Plans para Fargate (confirmado directamente en el
formulario: *"Fargate Spot and Compute Savings Plans are currently not
supported on the AWS Pricing Calculator"*). Se calculó manualmente,
aplicando el 25% de descuento que AWS documenta como ejemplo oficial para
Fargate bajo Compute Savings Plans en su propia guía técnica
(*Understanding how Savings Plans apply to your usage*). Esta es una
limitación de la herramienta, no del servicio, en un despliegue real, la
tasa exacta se confirmaría en la consola de AWS al momento de la compra.

## 3. Impacto de la estrategia de compromiso (ADR-008)

| Componente | Costo On-Demand | Costo con RI/SP (1 año) | Ahorro mensual | Ahorro % |
|---|---|---|---|---|
| RDS PostgreSQL Multi-AZ (Reserved Instance, Partial Upfront) | 133.60 | 99.59 | 34.01 | 25.5% |
| Fargate — capacidad base (Compute Savings Plan, cálculo manual) | 144.16 | 108.12 | 36.04 | 25.0% |
| Fargate — capacidad CyberDay (sin cambio, On-Demand) | 35.55 | 35.55 | 0.00 | 0% |

## 4. Costo total optimizado

| Concepto | Monto mensual (USD) |
|---|---|
| Total On-Demand (línea base) | 481.04 |
| Ahorro por RDS Reserved Instance | -34.01 |
| Ahorro por Fargate Compute Savings Plan | -36.04 |
| **Total optimizado (con compromiso selectivo)** | **410.99** |
| **Ahorro total mensual** | **70.05** |
| **Ahorro total anual proyectado** | **840.60** |

## 5. Análisis de eficiencia

**El compromiso selectivo logra un ahorro del 14.6% sobre el total del
proyecto** ($70.05 de $481.04), concentrado exclusivamente en los dos
únicos componentes con patrón de uso verdaderamente estable: RDS Multi-AZ y
la capacidad base de Fargate. Ningún otro componente del proyecto se
benefició de comprometer capacidad, porque ninguno tiene ese perfil de uso
— y eso es evidencia, no coincidencia: cada lección fue diseñada desde el
principio con el patrón de tráfico real de ADC en mente (S3 con
Intelligent-Tiering para acceso impredecible, Fargate con Auto Scaling para
picos, CloudFront con pago por uso), así que para cuando llegamos a esta
lección, no quedaba "grasa" fácil de optimizar en el resto — la eficiencia
ya estaba incorporada en el diseño, no como un ajuste de último momento.

**Por qué no comprometimos la capacidad de CyberDay:** representa apenas
20 horas de uso al mes (~2.7% del tiempo total). Comprometerla a un
Savings Plan habría significado pagar por una reserva que el 97% del
tiempo no se usa, el peor escenario posible para este tipo de
instrumento financiero, que solo tiene sentido para carga *constante*.

**Composición del gasto optimizado:** con $410.99/mes, la infraestructura
de datos y cómputo (RDS + Fargate, ~$251/mes) sigue siendo, por lejos, el
componente dominante, coherente con que son los únicos recursos que
corren de forma continua. Los componentes de integración y distribución
(S3, CloudFront, SQS, SNS) juntos representan menos del 4% del gasto total,
validando que el diseño de esas capas (Lecciones 1, 6 y 7) ya estaba bien
optimizado por naturaleza, sin necesitar ningún compromiso financiero
adicional.