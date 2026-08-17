# ADR-002: Estrategia de backup y recuperación de datos

**Estado:** Aceptado
**Fecha:** 2026-08-18
**Lección relacionada:** Lección 2 - Estrategias de almacenamiento y respaldo en la nube

## Contexto

ADC tiene dos categorías de datos en la nube con requisitos de recuperación
muy distintos (definidos en `docs/00-contexto-negocio/contexto-y-requerimientos.md`):

| Tipo de dato | SLA | RTO | RPO |
|---|---|---|---|
| Catálogo de productos (S3) | 99.9% | ≤ 4 h | ≤ 24 h *(re-sincronizable)* |
| Datos transaccionales (pedidos, pagos - RDS) | 99.95% durante CyberDay | ≤ 1 h | ≤ 15 min |

El ERP/WMS on-premise queda **fuera del alcance** de esta estrategia: su
respaldo es gestionado por el equipo de TI interna de ADC, tal como se
definió en el contexto de negocio. Esta decisión evita que el equipo cloud
asuma responsabilidad operativa sobre sistemas que no controla, un punto
que el material de la lección identifica explícitamente como parte de los
"elementos contractuales" a definir con claridad en cualquier estrategia
híbrida.

El versionado de S3 ya activado en **ADR-001** protege contra sobrescrituras
accidentales del catálogo, pero no es, por sí solo, una estrategia de backup
completa, no protege contra la eliminación del bucket, y no cubre en
absoluto los datos transaccionales, que no viven en S3.

## Decisión

Se implementa un **plan de backup centralizado con AWS Backup**, con
políticas diferenciadas por criticidad de dato:

**Plan "adc-backup-transaccional"** (RDS - pedidos y pagos):
- Backups automáticos continuos con transaction log backup (RPO ≤ 15 min, capacidad nativa de RDS).
- Snapshot diario vía AWS Backup, retenido 7 días (recuperación operativa rápida).
- Snapshot semanal, retenido 4 semanas.
- Snapshot mensual, retenido 12 meses (retención extendida por tratarse de datos de pagos, sujeta a criterios de cumplimiento normativo/financiero).
- RDS desplegado en modo **Multi-AZ** para failover automático ante falla de infraestructura (~60-120 seg), complementario al backup, no un sustituto: Multi-AZ resuelve caída de infraestructura, el backup resuelve corrupción lógica o eliminación accidental de datos.

**Plan "adc-backup-catalogo"** (S3 - `adc-catalog-media` y `adc-catalog-docs`):
- Versionado (ya activo desde ADR-001) como primera línea de defensa.
- Backup adicional vía AWS Backup, con snapshot **semanal** (no diario), retenido 90 días.
- **Sin Cross-Region Replication continua,**  decisión refinada respecto a lo mencionado como "ideal" en ADR-001 (ver sección de Alternativas).

## Pilares de AWS Well-Architected Framework

| Pilar | ¿Cómo lo aborda esta decisión? |
|---|---|
| **Reliability** | Backups automáticos + Multi-AZ cubren tanto corrupción de datos como falla de infraestructura, con mecanismos distintos para cada escenario |
| **Cost Optimization** | La frecuencia y retención de cada plan es proporcional a la criticidad real del dato, no se aplica la misma política costosa a datos que toleran RPO de 24h |
| **Security** | AWS Backup cifra los backups en reposo mediante KMS; el acceso al backup vault se controla vía políticas IAM independientes del acceso a los datos originales |
| **Operational Excellence** | Un plan centralizado (en vez de configuraciones dispersas por servicio) da visibilidad única para auditoría y cumplimiento, relevante para los "elementos contractuales" de respaldo que exige el negocio |

## Alternativas consideradas

| Opción | Ventajas | Desventajas | ¿Por qué se descartó / eligió? |
|---|---|---|---|
| **AWS Backup centralizado con políticas diferenciadas** (elegida) | Un solo panel de control, políticas auditable, consistentes con la clasificación de datos del negocio | Requiere entender un servicio adicional (AWS Backup) más allá de las funciones nativas de cada servicio | Elegida: escala mejor a medida que el proyecto suma más lecciones (ej. cómputo en Lección 4) que también podrían necesitar backup |
| Backups nativos por servicio, sin orquestador central (snapshots manuales de RDS + reglas de lifecycle de S3 por separado) | Más simple de entender inicialmente | Sin visibilidad unificada; políticas inconsistentes entre servicios; más difícil de auditar para cumplimiento | Descartada: no escala bien y dificulta demostrar cumplimiento normativo ante el negocio |
| Herramienta de backup híbrida de terceros (Veeam/Commvault, mencionada en el manual para escenarios on-prem + cloud) | Un solo panel para on-premise y cloud a la vez | El ERP/WMS on-premise está explícitamente fuera de alcance de este proyecto; agregar licenciamiento de terceros no se justifica solo para el lado cloud | Descartada por alcance del proyecto |
| Cross-Region Replication continua para **todo** (catálogo + transaccional) | RPO mínimo posible en ambos casos | Costo y complejidad innecesarios para el catálogo, que ya tolera RPO ≤ 24h y es re-sincronizable desde su fuente | Descartada para el catálogo — se reconsidera el "ideal" propuesto en ADR-001 con mejor criterio de costo/beneficio |

## Métricas de éxito

- **SLA/SLO:** 99.95% (transaccional durante CyberDay) / 99.9% (catálogo), heredado del contexto de negocio
- **RTO/RPO:**
  - Transaccional: RTO ≤ 1 h (Multi-AZ failover + snapshot restore) / RPO ≤ 15 min (transaction log backup continuo)
  - Catálogo: RTO ≤ 4 h / RPO ≤ 24 h (versionado + snapshot semanal)

## Diseño ideal vs. restricción del AWS Academy Learner Lab

| Aspecto | Diseño ideal (producción) | Lo que se implementa/documenta en el Lab | Motivo de la brecha |
|---|---|---|---|
| Copia de backups en región secundaria | Copiar snapshots del backup vault a una región DR distinta | Diseñado y documentado, no desplegado | El Lab restringe a una única región (mismo motivo que ADR-001) |
| Backup Vault Lock (inmutabilidad WORM) | Activo para los backups de datos de pago, evitando eliminación incluso por un administrador | Configuración documentada, no aplicada | El rol `LabRole` no tiene permisos para configurar políticas de Vault Lock |
| Prueba de restauración (restore drill) | Ejercicio trimestral programado, restaurando a un ambiente aislado | Documentado como runbook, no ejecutado en este proyecto | El tiempo de sesión del Lab (~4h) no permite un ciclo completo de prueba de restauración sin agotar el tiempo disponible para el resto del módulo |

## Costos estimados

- **Fuente:** [Estimación Costo RDS + AWS Backup (RDS + S3)](https://calculator.aws/#/estimate?id=24d935398ba490591c616bcc1f6847275731b9e6)
- **Resultado:** $185.48 USD/mes ($133.60 instancia RDS Multi-AZ + $16.91 AWS Backup RDS + $34.97 AWS Backup S3)
- **Detalle completo:** ver `docs/02-backup-respaldo/costos.md`

## Consecuencias

**Positivas:**
- Políticas de backup auditable y trazables desde un solo panel
- Separación clara de responsabilidad: ADC cloud no gestiona backup del ERP/WMS on-premise
- La retención extendida (12 meses) para datos de pago da soporte a futuras auditorías financieras

**Negativas / riesgos:**
- Un solo backup vault mal configurado (permisos IAM) podría afectar múltiples servicios a la vez, se mitiga con políticas IAM específicas por plan, no un acceso genérico
- La decisión de no usar CRR continua para el catálogo asume que el RPO de 24h seguirá siendo válido; si el negocio reduce ese SLA en el futuro, esta decisión debe revisarse

## Referencias

- AWS Backup: https://aws.amazon.com/backup/
- AWS Well-Architected Framework - Reliability Pillar