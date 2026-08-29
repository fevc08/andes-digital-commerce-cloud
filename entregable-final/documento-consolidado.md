# Documento Consolidado: Arquitectura Cloud de Andes Digital Commerce (ADC)

> Proyecto ABP · Módulo 5 · Arquitecturas Cloud Básicas · SOFOFA

## 1. Resumen ejecutivo

Andes Digital Commerce (ADC) es un retailer de e-commerce y logística con
operación en Chile, Perú y Colombia, cuya plataforma debe absorber picos de
tráfico de 10-20x durante eventos CyberDay, mientras su ERP y WMS
permanecen on-premise por restricciones contractuales y de hardware.

Este documento consolida el diseño de arquitectura cloud desarrollado a lo
largo de 8 lecciones, cada una resuelta con su propio ADR, informe técnico,
estimación de costos y diagrama, disponibles en `adr/` y `docs/`. El
resultado es una arquitectura **híbrida**, **multi-AZ**, con **escalado
automático** y una estrategia de costos que reduce el gasto mensual de
**$481.04 a $410.99 USD** (-14.6%) mediante compromiso financiero selectivo.

## 2. Contexto de negocio

El detalle completo está en
[`docs/00-contexto-negocio/contexto-y-requerimientos.md`](../docs/00-contexto-negocio/contexto-y-requerimientos.md).
En resumen: ADC clasifica sus datos en función de su criticidad real
(catálogo con SLA 99.9%/RTO≤4h/RPO≤24h vs. datos transaccionales con SLA
99.95% durante CyberDay/RTO≤1h/RPO≤15min), y esa clasificación, no una
plantilla genérica, es la que sostiene cada decisión posterior.

## 3. Arquitectura por capas

### 3.1 Almacenamiento de objetos ([ADR-001](../adr/001-tipo-almacenamiento-objetos.md))
Catálogo en dos buckets S3 diferenciados por patrón de acceso: S3
Intelligent-Tiering para medios (acceso impredecible por campañas) y S3
Standard para documentos. Acceso desacoplado del backend mediante URLs
firmadas para subida, refinado en la Lección 6 para lectura pública.

### 3.2 Backup y recuperación ([ADR-002](../adr/002-estrategia-backup-recuperacion.md))
AWS Backup centralizado con políticas diferenciadas: retención agresiva
(12 meses, con transición a frío) para datos transaccionales por
exigencia de auditoría financiera; retención simple (90 días) para el
catálogo, evitando Cross-Region Replication innecesaria dado que es
re-sincronizable.

### 3.3 Modelo de nube híbrido ([ADR-003](../adr/003-modelo-nube-publica-privada-hibrida.md))
Aplicando el árbol de decisión del material: el ERP/WMS se queda
on-premise (legacy no migrable en este horizonte), la plataforma
e-commerce vive 100% en AWS. Conectados vía Site-to-Site VPN, con solo dos
eventos de negocio cruzando el enlace, nunca datos financieros o de
identidad corporativa.

### 3.4 Escalabilidad de cómputo ([ADR-004](../adr/004-auto-scaling-computo.md))
ECS + Fargate, elegido sobre EC2 y Lambda por el patrón de tráfico
"continuo y variable" de ADC, arranque en segundos crítico durante el
inicio súbito de un pico CyberDay. Target tracking sobre CPU, 2-20 tareas
por AZ.

### 3.5 Disponibilidad de red ([ADR-005](../adr/005-balanceo-carga-multi-az.md))
Application Load Balancer (Capa 7, Least Outstanding Requests) + NAT
Gateway, completando el diseño de red iniciado en ADR-003. Trade-off de
NAT Gateway único (vs. 2 ideales) documentado explícitamente como
restricción del Lab.

### 3.6 Disponibilidad de contenido ([ADR-006](../adr/006-cdn-distribucion-contenido.md))
CloudFront + Origin Access Control, tras clasificar el catálogo como
contenido público, decisión que revisó y mejoró el diseño original de
ADR-001, reemplazando URLs firmadas de lectura por acceso público
cacheable, reduciendo latencia hacia los 3 países de operación.

### 3.7 Mensajería asíncrona ([ADR-007](../adr/007-mensajeria-asincrona-colas.md))
Dos mecanismos, cada uno según la naturaleza del evento: SQS FIFO (cola de
trabajo) para "pedido confirmado", exactamente un consumidor, sin
duplicados; SNS→SQS (pub/sub) para "actualización de stock", abierto a
futuros suscriptores sin modificar al productor.

### 3.8 Administración de costos ([ADR-008](../adr/008-estrategia-costos.md))
Compromiso financiero selectivo: Reserved Instance para RDS y Compute
Savings Plan solo para la capacidad base de Fargate, nunca para la
capacidad estacional de CyberDay. Monitoreo vía AWS Budgets + CloudWatch +
Cost Explorer con tags.

## 4. Diagrama de arquitectura integrado

*(Ver [`diagramas/exportados/entregable-final.png`](../diagramas/exportados/entregable-final.png), diagrama único que consolida las 8 capas anteriores, construido en el
siguiente paso de este cierre de proyecto.)*

## 5. Estimación total de costos y análisis de eficiencia

| | Monto mensual (USD) |
|---|---|
| Total On-Demand (línea base, sin compromiso) | 481.04 |
| Total optimizado (RI + Savings Plan selectivo) | **410.99** |
| Ahorro mensual | 70.05 (14.6%) |
| Ahorro anual proyectado | 840.60 |

Detalle completo, metodología de cálculo y análisis por componente en
[`docs/08-administracion-costos/resumen-costos-total.md`](../docs/08-administracion-costos/resumen-costos-total.md).

## 6. Principios transversales aplicados

**Well-Architected Framework:** cada ADR referencia explícitamente los
pilares que aborda. En conjunto, el proyecto prioriza **Cost Optimization**
(diseño consciente del costo desde la Lección 1, no como ajuste final) y
**Reliability** (Multi-AZ en las tres capas: datos, cómputo y red, ver
Lección 5, sección 7 de su informe técnico).

**Diseño ideal vs. restricción del Lab:** cada ADR documenta
explícitamente dónde el diseño de producción difiere de lo desplegable en
AWS Academy Learner Lab (NAT Gateway único, sin dominio real, sin
KMS personalizado, sin compra real de RI/SP), una brecha reconocida y
justificada, no un descuido.

**Trazabilidad de costos:** cada cifra en este proyecto proviene de AWS
Pricing Calculator, verificada con "Show calculations" cuando el resultado
no era intuitivo (se detectaron y corrigieron errores de hasta 30x en el
camino, documentados en cada `costos.md` correspondiente).

## 7. Reflexión de portafolio

Este proyecto extiende el trabajo de "Infraestructura Viva" (Módulo 4),
pasando de una migración inicial a una arquitectura **conectada**, cada
lección no resolvió un problema aislado, sino una pieza de un sistema
donde las decisiones se heredan y se refinan entre sí (el versionado de
S3 sostiene el backup del catálogo; el modelo híbrido explica por qué se
necesita mensajería asíncrona; la clasificación de contenido de la
Lección 6 corrige una decisión de la Lección 1 con mejor criterio). Esa
capacidad de revisar y mejorar decisiones anteriores con nueva
información es, en la práctica, tan importante como tomarlas bien la
primera vez.