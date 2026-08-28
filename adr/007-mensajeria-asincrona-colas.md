# ADR-007: Arquitectura de mensajería asíncrona

**Estado:** Aceptado
**Fecha:** 2026-08-27
**Lección relacionada:** Lección 7 - Arquitecturas básicas orientadas a mensajes

## Contexto

Desde ADR-003 quedaron definidos dos eventos de negocio que cruzan el
enlace híbrido entre la VPC de ADC y el datacenter on-premise:

1. **Cloud → On-premise:** "Pedido confirmado", dispara el proceso físico
   de picking en el WMS.
2. **On-premise → Cloud:** "Actualización de stock", mantiene el catálogo
   y el checkout con disponibilidad real.

En ADR-003 se estableció que estos eventos viajan de forma asíncrona por el
túnel VPN, pero el **mecanismo interno de mensajería** quedó pendiente para
esta lección. La necesidad es real y no teórica: la VPN tiene latencia
variable (documentado en ADR-003) y el WMS puede tener ventanas de
mantenimiento, un mecanismo síncrono perdería el evento si el otro extremo
no está disponible en ese instante exacto.

## Decisión

Se implementan **dos mecanismos de mensajería distintos**, elegidos según
la naturaleza de cada evento:

**"Pedido confirmado" → Amazon SQS (cola FIFO)**
- Patrón: **Cola de trabajo,** un mensaje, un solo consumidor.
- Cola FIFO (no Standard), con `MessageGroupId = order_id`, para garantizar
  orden por pedido y evitar procesamiento duplicado (deduplicación nativa
  en ventana de 5 minutos).
- Dead Letter Queue (DLQ) tras 3 intentos fallidos de procesamiento.

**"Actualización de stock" → Amazon SNS (topic) + SQS (suscriptor)**
- Patrón: **Publicador-suscriptor**.
- Topic SNS `adc-stock-updates`, con una cola SQS `catalog-sync-queue`
  suscrita (consumida por el Servicio de Sincronización de Inventario,
  ADR-003).
- Diseño abierto a futuros suscriptores (ej. una alerta de bajo stock)
  sin modificar el productor (WMS) ni el suscriptor actual.

## Pilares de AWS Well-Architected Framework

| Pilar | ¿Cómo lo aborda esta decisión? |
|---|---|
| **Reliability** | La DLQ evita pérdida silenciosa de mensajes tras fallos repetidos; SQS FIFO garantiza que un pedido no se procese duplicado ni fuera de orden |
| **Operational Excellence** | El patrón pub/sub permite agregar nuevos consumidores de "actualización de stock" sin coordinar cambios con el WMS (productor) |
| **Performance Efficiency** | Desacopla el tiempo de respuesta del WMS del tiempo de respuesta de la plataforma cloud, ninguno bloquea al otro |
| **Cost Optimization** | Pago por mensaje, sin infraestructura de integración corriendo de forma continua/ociosa |

## Alternativas consideradas

| Opción | Ventajas | Desventajas | ¿Por qué se descartó / eligió? |
|---|---|---|---|
| **SQS FIFO (pedido) + SNS→SQS (stock)** (elegida) | Cada evento usa el patrón que corresponde a su necesidad real | Dos mecanismos distintos que mantener, en vez de uno solo | Elegida: la necesidad de "exactamente un consumidor" vs. "potencialmente varios" es real, no cosmética |
| SQS Standard para ambos eventos | Más simple, un solo tipo de recurso | Sin garantía de orden ni deduplicación — riesgo de picking duplicado en un evento que dispara una acción física irreversible | Descartada para "pedido confirmado" por el riesgo operativo que implica |
| SNS pub/sub para ambos eventos (incluyendo pedido confirmado) | Máxima flexibilidad de suscriptores | "Pedido confirmado" debe procesarse una sola vez; pub/sub no fue diseñado para esa garantía | Descartada, no calza con el requisito de exactamente-un-consumidor |
| Comunicación síncrona directa (API REST vía VPN) | Respuesta inmediata, más simple de depurar | Pierde el evento si el otro extremo no está disponible en ese instante, contradice la razón original por la que se eligió un modelo híbrido asíncrono en ADR-003 | Descartada |

## Métricas de éxito

- **SLA/SLO:** heredado del contexto de negocio
- **Nueva métrica de este ADR — Antigüedad máxima de mensaje en cola:** ≤ 5 minutos (alarma de CloudWatch sobre `ApproximateAgeOfOldestMessage`), como indicador de que el consumidor no se está quedando atrás
- **Reintentos antes de DLQ:** 3 intentos

## Diseño ideal vs. restricción del AWS Academy Learner Lab

| Aspecto | Diseño ideal (producción) | Lo que se implementa/documenta en el Lab | Motivo de la brecha |
|---|---|---|---|
| Consumidor en el WMS real | Un listener corriendo en el datacenter de ADC, consumiendo la cola SQS a través del túnel VPN | Se documenta el mecanismo y se valida el encolado/desencolado manual desde la consola de SQS | El WMS de ADC es ficticio — no existe un endpoint real on-premise contra el cual integrar (misma naturaleza de brecha que el túnel VPN en ADR-003) |
| Cifrado de mensajes | SSE con clave KMS gestionada por el cliente | Cifrado gestionado por SQS/SNS por defecto (SSE-SQS) | El rol `LabRole` no tiene permisos para administrar claves KMS propias (misma brecha que ADR-001 y ADR-002) |

## Costos estimados

- **Fuente:** AWS Pricing Calculator - https://calculator.aws/#/estimate?id=8d4d2770908ca00bb9002192178c0a566c2faf41
- **Resultado:** $0.20 USD/mes ($0.11 SQS FIFO+Standard + $0.09 SNS)
- **Detalle completo:** ver `docs/07-mensajeria-asincrona/costos.md`

## Consecuencias

**Positivas:**
- Cada evento usa el mecanismo que corresponde a su necesidad real de consumo, no uno genérico
- La arquitectura queda preparada para agregar nuevos consumidores de eventos de stock sin tocar el productor
- La DLQ da visibilidad operativa ante fallos, en vez de pérdida silenciosa de mensajes

**Negativas / riesgos:**
- Mantener dos mecanismos (SQS FIFO y SNS+SQS) suma una pieza más de complejidad operativa que un único mecanismo genérico
- Sin un consumidor real validado en el Lab, el flujo de "pedido confirmado" no se prueba de punta a punta dentro de este proyecto

## Referencias

- Amazon SQS: https://aws.amazon.com/sqs/
- Amazon SNS: https://aws.amazon.com/sns/
- AWS Well-Architected Framework