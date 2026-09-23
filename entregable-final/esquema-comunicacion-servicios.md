# Esquema de Comunicación entre Servicios - ADC

> Consolidado de todos los flujos de comunicación definidos en las 8
> lecciones del proyecto. Referencia cruzada a cada ADR para el detalle
> completo de cada decisión.

## Leyenda

| Símbolo | Significado |
|---|---|
| 🔵 Síncrono | El emisor espera una respuesta inmediata |
| 🟣 Asíncrono | El emisor continúa sin esperar; el mensaje se procesa después |
| 🔒 Cruza límite de seguridad | La comunicación atraviesa una frontera de red relevante (VPC, subred privada, enlace híbrido) |

## 1. Comunicación externa: Usuario ↔ Plataforma

| Origen | Destino | Mecanismo | Tipo | Referencia |
|---|---|---|---|---|
| Usuario final | CloudFront | HTTPS GET (contenido público del catálogo) | 🔵 Síncrono | ADR-0006 |
| Usuario final | Internet Gateway → ALB | HTTPS (checkout, API dinámica) | 🔵 Síncrono | ADR-0005 |
| CMS interno | Servicio de Catálogo | HTTPS (solicita URL firmada de subida) | 🔵 Síncrono | ADR-0001 |
| CMS interno | S3 (`adc-catalog-media`/`docs`) | PUT directo vía URL firmada | 🔵 Síncrono | ADR-0001 |

## 2. Comunicación interna: dentro de la VPC

| Origen | Destino | Mecanismo | Tipo | Referencia |
|---|---|---|---|---|
| ALB | Tareas Fargate | Distribución HTTP (Least Outstanding Requests) | 🔵 Síncrono | ADR-0005 |
| Tareas Fargate | RDS PostgreSQL | Consultas SQL (subred privada) | 🔵 Síncrono 🔒 | ADR-0002, ADR-0003 |
| Tareas Fargate | NAT Gateway → Internet Gateway | Tráfico saliente (imágenes de contenedor, proveedor de pagos) | 🔵 Síncrono 🔒 | ADR-0005 |
| CloudFront (cache miss) | S3 (origen, vía OAC) | HTTPS interno, sin exposición pública del bucket | 🔵 Síncrono 🔒 | ADR-0006 |
| RDS (Primaria) | RDS (Standby) | Replicación síncrona Multi-AZ | 🔵 Síncrono 🔒 | ADR-0002 |
| Tareas Fargate | CloudWatch | Métrica de CPU (cada 60s) | 🟣 Asíncrono | ADR-0004 |
| CloudWatch → Alarma | Auto Scaling (ECS) | Señal de escalado | 🟣 Asíncrono | ADR-0004 |

## 3. Comunicación híbrida: Cloud ↔ On-premise

| Origen | Destino | Mecanismo | Tipo | Referencia |
|---|---|---|---|---|
| Servicio de Checkout | SQS FIFO (`adc-pedidos-confirmados.fifo`) | Evento "pedido confirmado" | 🟣 Asíncrono | ADR-0007 |
| SQS FIFO | WMS (vía túnel VPN) | Consumo del pedido, dispara picking | 🟣 Asíncrono 🔒 | ADR-0003, ADR-0007 |
| SQS FIFO (tras 3 fallos) | Dead Letter Queue | Reintento agotado | 🟣 Asíncrono | ADR-0007 |
| WMS (vía túnel VPN) | SNS (`adc-stock-updates`) | Evento "actualización de stock" | 🟣 Asíncrono 🔒 | ADR-0003, ADR-0007 |
| SNS | SQS (`catalog-sync-queue`) → Servicio de Sincronización | Fan-out pub/sub | 🟣 Asíncrono | ADR-0007 |
| Servicio de Sincronización | Catálogo / RDS | Actualiza disponibilidad de producto | 🔵 Síncrono | ADR-0003 |

> **Lo que nunca cruza este límite híbrido:** datos de tarjetas/pagos
> (tokenizados en el proveedor externo y en RDS), datos contables del ERP,
> credenciales de Active Directory. Ver ADR-0003, sección "Qué cruza el
> enlace híbrido".

## 4. Comunicación de respaldo y monitoreo (automatizada)

| Origen | Destino | Mecanismo | Tipo | Referencia |
|---|---|---|---|---|
| RDS | AWS Backup | Snapshot programado (diario/semanal/mensual) | 🟣 Asíncrono | ADR-0002 |
| S3 (catálogo) | AWS Backup | Snapshot semanal | 🟣 Asíncrono | ADR-0002 |
| AWS Backup | Backup Vault (KMS) | Almacenamiento cifrado | 🟣 Asíncrono | ADR-0002 |
| CloudWatch (`AWS/Billing`) | AWS Budgets | Métrica `EstimatedCharges` (retraso 4-6h) | 🟣 Asíncrono | ADR-0008 |
| AWS Budgets / CloudWatch | SNS (notificación de costos) | Alerta de presupuesto | 🟣 Asíncrono | ADR-0008 |
| Recursos etiquetados | AWS Cost Explorer | Consulta filtrada por tags (`proyecto`, `leccion`, `componente`) | 🔵 Síncrono (bajo demanda) | ADR-0008 |

## 5. Observación transversal

De las 18 comunicaciones documentadas, **8 son asíncronas,** y no por
elección arbitraria: 6 de ellas cruzan el enlace híbrido o alimentan
mecanismos de respaldo/monitoreo, exactamente los casos donde una falla
temporal de disponibilidad **no debe** traducirse en pérdida de datos ni en
un usuario esperando una respuesta que depende de un sistema on-premise
fuera del control directo de la plataforma cloud. Las comunicaciones
síncronas, en cambio, se concentran en la ruta crítica de la experiencia de
usuario (ALB → Fargate → RDS), donde sí se necesita una respuesta
inmediata.