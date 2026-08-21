# Estimación de Costos: Lección 3 - Modelo de Nube Híbrido

**Fuente:** AWS Pricing Calculator
**Link de la estimación:** https://calculator.aws/#/estimate?id=d8903dcc89494a5193874bf709a07f0f3d03a2f4
**Región asumida:** US East (N. Virginia) - `us-east-1`
**Fecha de estimación:** 2026-08-20

## Supuestos de diseño

| Componente | Configuración |
|---|---|
| AWS Site-to-Site VPN | 1 conexión, 730 h/mes (continua) |
| Data Transfer | 5 GB/mes entrante (Internet) + 5 GB/mes saliente (Internet) |

**Nota de alcance:** esta estimación cubre únicamente el enlace híbrido
(VPN). El NAT Gateway y el Application Load Balancer, aunque forman parte
de la misma VPC, se presupuestan en `docs/05-disponibilidad-red/costos.md`
(Lección 5).

## Resultado de la calculadora

| Concepto | Costo mensual estimado (USD) |
|---|---|
| VPN Connection (1 conexión, 730 h/mes) | 36.50 |
| Data Transfer (5 GB in + 5 GB out) | 0.45 |
| **Total Lección 3** | **36.95** |

*(Nota: estimación de diseño. El túnel VPN completo no es desplegable en el
Lab según lo documentado en ADR-003, ya que no existe un datacenter físico
real de ADC contra el cual conectar, esta estimación representa el costo
del lado AWS de la conexión.)*

## Lectura del resultado

El **99% del costo de esta lección ($36.50 de $36.95) proviene del cargo
fijo por hora de conexión**, no de la transferencia de datos. Esto confirma
lo anticipado: a diferencia de S3 (Lección 1) o RDS (Lección 2), el enlace
híbrido **no se beneficia del modelo de pago por uso**, cuesta prácticamente
lo mismo si ADC transmite 5 GB o 50 GB al mes, porque el cargo por hora de
conexión activa domina el total. Esto es relevante para la Lección 8: es un
costo fijo de mantener el modelo híbrido, independiente del volumen de
negocio, y solo se elimina si en el futuro se completa la migración del
ERP/WMS o se reemplaza por Direct Connect (que tiene una estructura de
costo distinta, con cargo por puerto dedicado).

**Costo total acumulado del proyecto hasta la Lección 3:** $9.23 (L1) +
$185.48 (L2) + $36.95 (L3) = **$231.66 USD/mes**.

**Nota sobre Client VPN:** la calculadora exige un mínimo de 22 días
hábiles en el campo "Working days per month" del bloque Client VPN, incluso
cuando ese servicio no se utiliza. Como "Number of subnet associations" y
"Number of active connections" se dejaron en 0, este campo no genera costo
(0 × 22 días × tarifa = 0), el total de la estimación proviene
exclusivamente de Site-to-Site VPN y Data Transfer, verificado en el
desglose de la calculadora.