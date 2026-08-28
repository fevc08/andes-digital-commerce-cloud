# Estimación de Costos: Lección 7 - Mensajería Asíncrona

**Fuente:** AWS Pricing Calculator
**Link de la estimación:** https://calculator.aws/#/estimate?id=8d4d2770908ca00bb9002192178c0a566c2faf41
**Región asumida:** US East (N. Virginia) - `us-east-1`
**Fecha de estimación:** 2026-08-27

## Supuestos de diseño

| Componente | Configuración |
|---|---|
| SQS FIFO (`adc-pedidos-confirmados`) | 5.000 pedidos/mes × 3 solicitudes = 15.000 solicitudes/mes (0.015 millones) |
| SQS Standard (`catalog-sync-queue`) | 10.000 eventos/mes × 2 solicitudes = 20.000 solicitudes/mes (0.02 millones) |
| SNS (`adc-stock-updates`, Standard topic) | 10.000 publicaciones/mes, 10.000 entregas a SQS |
| Transferencia de datos | 1 GB/mes entrante + 1 GB/mes saliente, en ambos servicios |

## Notas metodológicas

**Sobre la unidad de solicitudes en SQS:** los campos "Standard queue
requests" y "FIFO queue requests" están expresados en **millones** de
solicitudes. Ingresar el volumen real (20.000 y 15.000) sin convertir habría
inflado el costo por un factor de un millón (~$15.500/mes vs. el valor real
de fracciones de centavo), corregido a 0.02 y 0.015 millones
respectivamente.

**Sobre "Message Data Protection" en SNS:** ADC no utiliza escaneo de
mensajes ni auditoría (funcionalidades no evaluadas en ADR-007). La
calculadora exige un mínimo técnico de 0.0000095367432 GB/mes (~10 KB) en
ambos campos, el costo resultante es despreciable y no representa una
funcionalidad activa del diseño.

## Resultado de la calculadora

| Concepto | Costo mensual estimado (USD) |
|---|---|
| Amazon SQS (FIFO + Standard) | 0.11 |
| Amazon SNS (Standard topic) | 0.09 |
| **Total Lección 7** | **0.20** |

*(Nota: estimación de diseño, no refleja el consumo real de créditos del AWS Academy Learner Lab.)*

## Lectura del resultado

El costo de la mensajería en sí (solicitudes SQS + SNS) es prácticamente
nulo — el volumen de ADC está muy por debajo del millón de solicitudes
gratuitas que cada servicio ofrece de forma permanente. Lo poco que sí
aparece como costo ($0.20 en total) proviene casi enteramente de la
transferencia de datos, no del mecanismo de mensajería en sí. Esto confirma
un patrón que se repite en la Lección 6 (CloudFront) y ahora acá: los
servicios de integración y distribución de AWS están diseñados con un free
tier generoso pensado para volúmenes moderados como el de ADC, mientras que
los costos reales del proyecto se concentran en los recursos que corren de
forma continua (RDS Multi-AZ, NAT Gateway, ALB), el hallazgo central que
se va a formalizar en el análisis de la Lección 8.

**Costo total acumulado del proyecto hasta la Lección 7:** $480.84 (L1-L6)
+ $0.20 (L7) = **$481.04 USD/mes**.