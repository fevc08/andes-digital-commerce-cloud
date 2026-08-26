# ADR-004: Estrategia de escalabilidad de servicios de cómputo

**Estado:** Aceptado
**Fecha:** 2026-08-21
**Lección relacionada:** Lección 4 - Escalabilidad de servicios de cómputo

## Contexto

La plataforma de e-commerce de ADC (catálogo, checkout, procesamiento de
pedidos) debe atender tráfico **continuo durante todo el año**, con picos
de **10-20x** durante eventos CyberDay que duran entre 48 y 72 horas
(definido en el contexto de negocio).

Este patrón descarta dos extremos:
- **Capacidad fija sobredimensionada** (aprovisionar para el peak todo el
  año): genera sobreaprovisionamiento, servidores encendidos sin tráfico
  real la mayor parte del año, el "gasto invisible" que identifica el
  material de la lección.
- **Capacidad fija subdimensionada** (aprovisionar para el tráfico normal):
  genera subaprovisionamiento durante CyberDay (timeouts, carritos
  abandonados, pérdida de venta directa).

La solución debe **seguir la demanda real**, agregando y quitando capacidad
de forma automática, con un tiempo de reacción lo suficientemente rápido
para que el escalado no llegue tarde al pico.

## Decisión

Se implementa la plataforma de e-commerce sobre **Amazon ECS con AWS
Fargate** (contenedores sin servidor), con un servicio de Auto Scaling
basado en **target tracking** (CPU y conteo de solicitudes por tarea).

**Justificación de la elección del motor de cómputo** (unidad de escala +
patrón de tráfico, tal como exige el material):

| Motor | Unidad de escala | Tiempo de arranque | ¿Calza con el patrón de ADC? |
|---|---|---|---|
| EC2 + Auto Scaling | Instancia completa | 1-3 minutos | No, demasiado lento para el inicio súbito de un pico de CyberDay |
| **ECS + Fargate (elegido)** | Tarea/contenedor | Segundos | **Sí,** tráfico continuo y variable, exactamente el caso de uso descrito en el material |
| AWS Lambda | Invocación de función | Milisegundos | No como motor principal, ideal para eventos esporádicos/batch, no para servir una aplicación web con lógica de sesión sostenida (se reserva como candidato para la mensajería asíncrona de Lección 7) |

**Política de autoescalado (ciclo CloudWatch → Alarma → ASG, aplicado a ECS):**
- Métrica: utilización de CPU por tarea (target tracking, objetivo 65%)
- Mínimo: 2 tareas por zona de disponibilidad (operación normal)
- Máximo: 20 tareas por zona de disponibilidad (capacidad para CyberDay)
- Cooldown: período de enfriamiento entre eventos de escalado, para evitar oscilación (lanzar y apagar tareas repetidamente)

## Pilares de AWS Well-Architected Framework

| Pilar | ¿Cómo lo aborda esta decisión? |
|---|---|
| **Performance Efficiency** | Fargate lanza nuevas tareas en segundos, permitiendo que la capacidad reaccione casi en tiempo real al inicio de un pico de tráfico |
| **Cost Optimization** | Se paga por vCPU/memoria mientras la tarea corre, sin instancias EC2 ociosas durante los meses de tráfico normal |
| **Reliability** | Múltiples tareas distribuidas, con health checks automáticos que reemplazan tareas fallidas sin intervención manual |
| **Operational Excellence** | Fargate elimina la gestión de parches de sistema operativo, relevante porque el equipo de ADC ya tiene carga operativa gestionando el ERP/WMS on-premise (ADR-003) |

## Alternativas consideradas

| Opción | Ventajas | Desventajas | ¿Por qué se descartó / eligió? |
|---|---|---|---|
| **ECS + Fargate** (elegida) | Arranque en segundos, sin gestión de servidores, escalado granular por tarea | Requiere disciplina de imágenes de contenedor y observabilidad por tarea | Elegida: coincide exactamente con el patrón de tráfico "continuo y variable" de ADC |
| EC2 + Auto Scaling Group | Control total del sistema operativo, útil para cargas legacy | Arranque de 1-3 minutos — demasiado lento frente a un pico súbito de CyberDay; suma carga operativa de parches de SO | Descartada como motor principal, el tiempo de reacción no es competitivo para este caso de uso |
| AWS Lambda | Arranque en milisegundos, pago por invocación, cero gestión de servidores | No es el mejor ajuste para servir una aplicación web con lógica de sesión sostenida; su caso ideal es eventos esporádicos/batch | Descartada como motor principal, se evalúa como candidata específica para la mensajería asíncrona en ADR-007 |
| Amazon EKS (Kubernetes gestionado) | Máxima flexibilidad y portabilidad multi-plataforma | Curva de aprendizaje y complejidad de orquestación no se justifican para un proyecto de un solo proveedor cloud, sin requisito de portabilidad | Descartada por sobre-ingeniería frente al alcance actual de ADC |

## Métricas de éxito

- **SLA/SLO:** 99.95% de disponibilidad durante CyberDay (heredado del contexto de negocio)
- **Nueva métrica de este ADR — Tiempo de reacción de escalado:** ≤ 2 minutos entre que CloudWatch detecta el umbral y la nueva capacidad está sirviendo tráfico (health check incluido), significativamente más rápido que el rango de 2-5 minutos típico de EC2 mencionado en el material, gracias al arranque en segundos de Fargate

## Diseño ideal vs. restricción del AWS Academy Learner Lab

| Aspecto | Diseño ideal (producción) | Lo que se implementa/documenta en el Lab | Motivo de la brecha |
|---|---|---|---|
| Política de escalado | Target tracking + Predictive Scaling (anticipa el pico de CyberDay con datos históricos) | Solo target tracking reactivo | ADC (proyecto académico) no cuenta con historial de tráfico real suficiente para entrenar un modelo predictivo |
| Segmentación de servicios | Cada microservicio (catálogo, checkout, pagos) como un servicio ECS independiente, escalando de forma autónoma | Un servicio ECS único que agrupa la lógica de la plataforma | Simplificación por alcance y tiempo de sesión del Lab; documentado explícitamente como simplificación, no como diseño final recomendado |
| Pipeline de imágenes | CI/CD completo (CodeBuild → Amazon ECR) en cada cambio de código | Mecanismo documentado, sin pipeline desplegado | Fuera del alcance de tiempo disponible en el Lab para este módulo |

## Costos estimados

- **Fuente:** AWS Pricing Calculator (capacidad base) + cálculo manual con tarifas oficiales (capacidad CyberDay) — https://calculator.aws/#/estimate?id=72e9154621e5e6eeb1a748b1b0d9aaa2cfd01d4d
- **Resultado:** $179.71 USD/mes ($144.16 capacidad base + $35.55 capacidad CyberDay)
- **Detalle completo:** ver `docs/04-escalabilidad-computo/costos.md`

## Consecuencias

**Positivas:**
- Tiempo de reacción ante picos significativamente menor que con EC2
- Sin carga operativa de parches de sistema operativo
- Costo proporcional al tráfico real, no a capacidad reservada todo el año

**Negativas / riesgos:**
- Fargate tiene un costo por vCPU/memoria más alto que un EC2 equivalente corriendo 24/7 a alta utilización, si en el futuro ADC tuviera una carga perfectamente estable y predecible, convendría reevaluar EC2 con Reserved Instances
- La simplificación a un solo servicio ECS (en vez de microservicios independientes) limita el escalado granular por función, documentado como deuda técnica aceptada para el alcance de este proyecto

## Referencias

- Amazon ECS: https://aws.amazon.com/ecs/
- AWS Fargate: https://aws.amazon.com/fargate/
- AWS Well-Architected Framework - Performance Efficiency Pillar