# ADR-005: Balanceo de carga y disponibilidad multi-AZ

**Estado:** Aceptado
**Fecha:** 2026-08-26
**Lección relacionada:** Lección 5 - Disponibilidad de aplicaciones en la red

## Contexto

Con el modelo híbrido (ADR-003) y la capa de cómputo escalable (ADR-004) ya
definidos, queda pendiente resolver dos piezas que conectan todo lo
construido hasta ahora:

1. **Cómo entra el tráfico de los usuarios** a la subred pública de la VPC
   (dejada vacía intencionalmente desde ADR-003) y llega a las tareas
   Fargate en la subred privada.
2. **Cómo sale el tráfico** de la subred privada hacia internet (necesario,
   por ejemplo, para que las tareas Fargate descarguen imágenes de
   contenedor o se comuniquen con el proveedor de pagos externo).

Según el material de la lección, una arquitectura de alta disponibilidad
típica combina tres elementos: distribución en múltiples AZs, balanceo de
carga, y escalado dinámico, los dos últimos ya resueltos en ADR-004; este
ADR resuelve la distribución en AZs a nivel de red.

## Decisión

**Application Load Balancer (Capa 7 — Aplicación):**

Se despliega un **Application Load Balancer**, distribuido en las dos
zonas de disponibilidad de la subred pública, con un **target group**
apuntando a las tareas Fargate del ADR-004. Algoritmo de distribución:
**Least Outstanding Requests** (no Round Robin).

**NAT Gateway:**

Se despliega **NAT Gateway** para dar salida a internet a los recursos de
la subred privada (RDS para parches menores, servicio de sincronización,
tareas Fargate para descarga de imágenes de contenedor).

## Justificación: ¿por qué Capa 7 y no Capa 4?

| Tipo de balanceador | Capa OSI | ¿Calza con ADC? |
|---|---|---|
| **Application Load Balancer (elegido)** | 7 (Aplicación) | Sí, puede enrutar por contenido HTTP (ruta `/catalogo` vs `/checkout`), hacer *health checks* basados en código de respuesta HTTP (no solo TCP), y terminar SSL/TLS |
| Network Load Balancer | 4 (Transporte) | No, solo distribuye por IP/puerto, sin visibilidad del contenido de la solicitud; no permite el enrutamiento futuro por microservicio mencionado como diseño ideal en ADR-004 |
| Classic Load Balancer | 4/7 híbrido | No, tecnología legacy que AWS ya no recomienda para cargas nuevas |

## Justificación: ¿por qué Least Outstanding Requests y no Round Robin?

El material clasifica las estrategias de distribución en Round Robin,
Least Connections e IP Hash. Round Robin asigna solicitudes en secuencia
sin considerar la carga real de cada tarea, funciona bien si todas las
solicitudes tardan lo mismo, pero **no es el caso de ADC**: una solicitud
de checkout (que toca RDS) toma más tiempo de procesamiento que una
solicitud de catálogo (que solo lee de S3/caché). Least Outstanding
Requests dirige el tráfico a la tarea con menos solicitudes activas en
proceso, evitando que una tarea ocupada con checkouts lentos siga
recibiendo más tráfico del que puede procesar.

## Pilares de AWS Well-Architected Framework

| Pilar | ¿Cómo lo aborda esta decisión? |
|---|---|
| **Reliability** | ALB distribuido en 2 AZs + *health checks* HTTP automáticos que retiran tareas no saludables del reparto de tráfico sin intervención manual |
| **Performance Efficiency** | El enrutamiento por contenido (Capa 7) deja preparado el camino para separar catálogo/checkout en servicios independientes sin rediseñar la capa de red |
| **Cost Optimization** | Se evalúa explícitamente el trade-off de NAT Gateway único vs. redundante (ver Alternativas y Diseño ideal vs. Lab) |
| **Security** | El ALB es el único componente con exposición directa a internet; toda la capa de cómputo y datos permanece en la subred privada |

## Alternativas consideradas

| Opción | Ventajas | Desventajas | ¿Por qué se descartó / eligió? |
|---|---|---|---|
| **ALB + Least Outstanding Requests** (elegida) | Enrutamiento por contenido, distribución sensible a la carga real de cada tarea | Mayor costo que NLB | Elegida: se ajusta al patrón de tráfico HTTP heterogéneo de ADC |
| Network Load Balancer | Latencia mínima, throughput extremo | Sin visibilidad de contenido HTTP, no permite enrutamiento por ruta | Descartada, ADC no tiene requisitos de latencia extrema que justifiquen renunciar al enrutamiento por contenido |
| 2 NAT Gateways (uno por AZ) | Alta disponibilidad real, sin punto único de falla en la salida a internet | Duplica el costo mensual del NAT Gateway | Diseño ideal, ver brecha con el Lab abajo |
| 1 NAT Gateway (única AZ) | Menor costo | Punto único de falla para la salida a internet de toda la subred privada; tráfico cross-AZ adicional desde la AZ que no lo tiene local | Implementación en el Lab, con brecha documentada explícitamente |
| NAT Instance (EC2 propia) | Menor costo que NAT Gateway gestionado | Requiere gestión y parcheo manual, contradice el principio de mínima carga operativa ya aplicado con Fargate en ADR-004 | Descartada |

## Métricas de éxito

- **SLA/SLO:** 99.95% durante CyberDay (heredado del contexto de negocio)
- **Nueva métrica de este ADR - Tiempo de detección de tarea no saludable:** ~90 segundos (intervalo de *health check* de 30s × 2 verificaciones fallidas consecutivas antes de retirar la tarea del target group)

## Diseño ideal vs. restricción del AWS Academy Learner Lab

| Aspecto | Diseño ideal (producción) | Lo que se implementa/documenta en el Lab | Motivo de la brecha |
|---|---|---|---|
| NAT Gateway | 2 (uno por AZ), sin punto único de falla | 1 NAT Gateway (una sola AZ) | Costo duplicado no justificado para validar el diseño en el Lab; el trade-off queda documentado y sería el primer cambio a aplicar en un despliegue de producción real |
| Certificado SSL/TLS (HTTPS) | Certificado gestionado vía AWS Certificate Manager sobre un dominio real de ADC | Listener HTTP simple, sin dominio real disponible en el entorno académico | ADC es una empresa ficticia sin dominio registrado; el mecanismo se documenta pero no se puede validar end-to-end |
| AWS WAF sobre el ALB | Reglas gestionadas contra ataques comunes (especialmente relevante por el aumento de superficie durante CyberDay) | Fuera de alcance — no hay lección dedicada a seguridad perimetral en este módulo | Se documenta como recomendación futura, no como parte del alcance actual |

## Costos estimados

- **Fuente:** AWS Pricing Calculator - https://calculator.aws/#/estimate?id=f318505554209e32ff2dbdf2073eeefddd58c5d2
- **Resultado:** $65.50 USD/mes ($32.20 Application Load Balancer + $33.30 NAT Gateway)
- **Detalle completo:** ver `docs/05-disponibilidad-red/costos.md`

## Consecuencias

**Positivas:**
- Cierra el diseño de red completo: entrada (ALB) y salida (NAT) de la subred pública/privada
- El enrutamiento por contenido HTTP deja lista la evolución hacia microservicios independientes mencionada en ADR-004
- El trade-off de NAT Gateway queda documentado con criterio explícito, no como un descuido

**Negativas / riesgos:**
- El NAT Gateway único en el Lab es un punto único de falla que **no debe replicarse en producción** — riesgo aceptado solo para fines de validación académica
- Sin HTTPS real validado en el Lab, la configuración de seguridad de transporte queda documentada pero no probada end-to-end

## Referencias

- Elastic Load Balancing: https://aws.amazon.com/elasticloadbalancing/
- NAT Gateway: https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html
- AWS Well-Architected Framework - Reliability Pillar