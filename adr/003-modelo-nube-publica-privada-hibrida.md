# ADR-003: Modelo de nube - pública, privada e híbrida

**Estado:** Aceptado
**Fecha:** 2026-08-19
**Lección relacionada:** Lección 3 - Arquitecturas de nubes públicas, privadas e híbridas

## Contexto

ADC tiene dos realidades tecnológicas distintas, ya identificadas en el
contexto de negocio:

- **ERP y WMS:** on-premise en Santiago, con licenciamiento vigente por 3
  años más y dependencia de hardware físico de picking en bodega. No es
  candidato a migración dentro del horizonte de este proyecto.
- **Plataforma de e-commerce** (catálogo, checkout, futura capa de
  cómputo): candidata natural a nube pública, por la necesidad de absorber
  picos de tráfico de 10-20x durante CyberDay sin sobredimensionar
  infraestructura fija todo el año.

Aplicando el árbol de decisión de 4 preguntas del material de la lección:

| # | Pregunta | Respuesta para ERP/WMS | Respuesta para plataforma e-commerce |
|---|---|---|---|
| 1 | ¿La ley/contrato/auditoría exige que el dato viva en infraestructura bajo control físico? | No es un mandato legal explícito, pero sí hay un bloqueo contractual (licenciamiento vigente) | No aplica |
| 2 | ¿Hay un sistema legacy o dependencia física que no puede migrarse en este horizonte? | **Sí** → detiene el árbol acá: **Nube híbrida** | No |
| 3 | ¿La demanda es variable, estacional o difícil de predecir? | — | **Sí** → Nube pública con autoescalado (se formaliza en Lección 4) |

Esto confirma con un criterio explícito, no solo intuitivo, que ADC necesita
un **modelo híbrido a nivel organizacional**, compuesto por un componente
que se queda on-premise y un componente que vive completamente en nube
pública.

## Decisión

Se adopta un **modelo de nube híbrido** para ADC:

- **ERP/WMS:** permanece on-premise (nube privada de facto), gestionado por
  TI interna, fuera del alcance operativo de este proyecto.
- **Plataforma de e-commerce:** se despliega íntegramente en **AWS (nube
  pública)**, dentro de una VPC dedicada.
- **Enlace híbrido:** conexión mediante **AWS Site-to-Site VPN**, entre el
  datacenter de ADC y la VPC en AWS.

**Qué cruza el enlace híbrido (y qué no):**

| Dirección | Dato que cruza | Dato que NO cruza |
|---|---|---|
| Cloud → On-premise | Evento de "pedido confirmado y pagado" (para iniciar picking en WMS), se detalla en ADR-007 | Datos de tarjeta/pago (quedan tokenizados en el proveedor de pagos y en RDS cloud) |
| On-premise → Cloud | Actualización de stock/inventario (para mantener el catálogo y el checkout precisos) | Datos financieros del ERP, identidad corporativa (Active Directory), información contable |

Minimizar lo que cruza el enlace es una decisión deliberada: el propio
material de la lección identifica la "complejidad de integración e
incompatibilidades" como la principal desventaja de un modelo híbrido, acotar
el tráfico cruzado a eventos de negocio puntuales, en vez de sincronización
masiva de datos, reduce esa superficie de riesgo.

## Pilares de AWS Well-Architected Framework

| Pilar | ¿Cómo lo aborda esta decisión? |
|---|---|
| **Reliability** | AWS Site-to-Site VPN provisiona automáticamente 2 túneles redundantes por conexión, sin configuración adicional |
| **Cost Optimization** | VPN tiene costo y tiempo de aprovisionamiento mucho menor que Direct Connect, adecuado al volumen de sincronización actual (eventos puntuales, no tráfico constante de alto volumen) |
| **Operational Excellence** | Minimizar el tipo de dato que cruza el enlace híbrido reduce la complejidad de integración, en línea con la advertencia del material sobre incompatibilidades en modelos híbridos |
| **Security** | Los datos financieros/regulados nunca cruzan el enlace; el tráfico híbrido se limita a eventos de negocio no sensibles (confirmación de pedido, niveles de stock) |

## Alternativas consideradas

| Opción | Ventajas | Desventajas | ¿Por qué se descartó / eligió? |
|---|---|---|---|
| **Nube híbrida con VPN Site-to-Site** (elegida) | Bajo costo, aprovisionamiento en horas, redundancia nativa de 2 túneles | Latencia variable (aceptable porque el tráfico cruzado es asíncrono, no en tiempo real) | Elegida: se ajusta al volumen y naturaleza actual del tráfico híbrido |
| Todo en nube pública (migrar también ERP/WMS) | Arquitectura homogénea, más simple de operar | Bloqueada por licenciamiento contractual vigente y dependencia de hardware físico de picking | Descartada, inviable en el horizonte de este proyecto (respuesta "Sí" en pregunta 2 del árbol) |
| Todo on-premise (sin nube) | Control total | No resuelve el problema central del proyecto: absorber picos de 10-20x sin sobredimensionar todo el año | Descartada, contradice el objetivo de negocio |
| Nube híbrida con AWS Direct Connect | Latencia estable y predecible | Costo fijo mensual alto y aprovisionamiento de semanas, no justificado para tráfico de sincronización puntual/asíncrono | Descartada por ahora, se documenta como posible upgrade futuro si el volumen de sincronización crece |

## Métricas de éxito

- **SLA/SLO:** heredado del contexto de negocio (99.9-99.95% según tipo de dato)
- **Nueva métrica de este ADR - Tolerancia de sincronización de inventario:** ≤ 15 min entre un cambio de stock en el WMS y su reflejo en el catálogo cloud (evita sobreventa de productos sin stock durante CyberDay)

## Diseño ideal vs. restricción del AWS Academy Learner Lab

| Aspecto | Diseño ideal (producción) | Lo que se implementa/documenta en el Lab | Motivo de la brecha |
|---|---|---|---|
| Túnel VPN completo (extremo a extremo) | Conexión funcional entre el datacenter real de ADC y AWS | Se configura y documenta el lado AWS (Virtual Private Gateway / Customer Gateway), sin túnel activo end-to-end | El "datacenter on-premise" de ADC es un escenario ficticio, no existe un extremo físico real contra el cual conectar en el Lab |
| Región | `sa-east-1` (São Paulo) | Región disponible en el Lab | Misma brecha documentada en ADR-001 |
| Upgrade a Direct Connect | Evaluación futura si el volumen de sincronización crece | Fuera de alcance, decisión de negocio, no restricción técnica del Lab | N/A |

## Costos estimados

- **Fuente:** AWS Pricing Calculator - https://calculator.aws/#/estimate?id=d8903dcc89494a5193874bf709a07f0f3d03a2f4
- **Resultado:** $36.95 USD/mes ($36.50 VPN Connection + $0.45 Data Transfer)
- **Detalle completo:** ver `docs/03-modelo-nube-hibrido/costos.md`

## Consecuencias

**Positivas:**
- Evita una migración forzada e inviable del ERP/WMS
- Acota el riesgo de seguridad del enlace híbrido a datos no sensibles
- Deja documentado un camino de evolución claro (VPN → Direct Connect) si el negocio lo requiere

**Negativas / riesgos:**
- La latencia variable de VPN podría no ser suficiente si en el futuro se necesita sincronización de stock en tiempo real (no solo cada 15 min), de ocurrir, este ADR debe revisarse
- Depender de un enlace híbrido introduce un punto de integración adicional que no existe en una arquitectura 100% cloud

## Referencias

- AWS Site-to-Site VPN: https://aws.amazon.com/vpn/
- AWS Direct Connect: https://aws.amazon.com/directconnect/
- AWS Well-Architected Framework