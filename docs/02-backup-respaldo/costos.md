# Estimación de Costos: Lección 2 - Backup y Recuperación

**Fuente:** AWS Pricing Calculator
**Link de la estimación:** [Estimación Costo RDS + AWS Backup (RDS + S3)](https://calculator.aws/#/estimate?id=24d935398ba490591c616bcc1f6847275731b9e6)
**Región asumida:** US East (N. Virginia) - `us-east-1`
**Fecha de estimación:** 2026-08-17

## Supuestos de diseño

| Componente | Configuración |
|---|---|
| RDS PostgreSQL | `db.t3.medium`, Multi-AZ, 100 GB gp3, On-Demand, 730 h/mes |
| RDS — Backup Storage nativo | 50 GB adicionales sobre el tramo gratuito |
| AWS Backup — RDS | 100 GB dato primario, crecimiento anual 15%, cambio diario 1%; retención diaria 7d + semanal 4 sem + mensual 12 meses (warm) |
| AWS Backup — S3 catálogo | 520 GB dato primario, crecimiento anual 20%, cambio diario 0.3%; retención semanal 13 semanas (warm) |

**Nota sobre almacenamiento en frío:** la calculadora de AWS Backup no ofrece
campos de transición a cold storage para los tipos de recurso RDS Backup ni
S3 Backup (a diferencia de EFS, que sí los expone). Esta es una limitación
de la herramienta de estimación, no del servicio real. En un despliegue de
producción, AWS Backup sí soporta cold storage para ambos tipos de recurso.
Por eso los 12 meses de retención mensual definidos en ADR-002 se cargan
acá como retención completa en warm storage, lo que **sobreestima** el costo
real de producción (donde existiría ahorro adicional al mover snapshots
antiguos a frío después de 90 días).

## Resultado de la calculadora

| Concepto | Costo mensual estimado (USD) |
|---|---|
| RDS PostgreSQL (instancia Multi-AZ + storage + backup nativo) | 133.60 |
| AWS Backup - RDS | 16.91 |
| AWS Backup - S3 (catálogo) | 34.97 |
| **Total Lección 2** | **185.48** |

*(Nota: estimación de diseño, no refleja el consumo real de créditos del AWS Academy Learner Lab.)*

## Lectura del resultado

La instancia Multi-AZ ($105.85 de los $133.60 de RDS) sigue siendo el
componente que domina el costo total — coherente con que es el único
recurso que corre 24/7 sin relación con el tráfico real.

Un hallazgo interesante: **el backup de S3 ($34.97) cuesta más del doble
que el backup de RDS ($16.91)**, a pesar de proteger el dato menos crítico
del negocio. Dos factores lo explican:

1. **Volumen:** el catálogo (520 GB) es más de 5 veces más grande que la
   base de datos transaccional (100 GB).
2. **Soporte de backup incremental:** RDS Backup soporta snapshots
   incrementales de forma nativa (cada respaldo solo almacena el cambio
   respecto al anterior), mientras que la política configurada para S3
   (snapshots semanales completos), no se beneficia del mismo mecanismo de
   optimización. Esto es un dato a tener en cuenta para la Lección 8: si el
   catálogo sigue creciendo, este costo escalará más rápido que el de la
   base de datos, incluso siendo el dato menos crítico.

**Costo total acumulado del proyecto hasta la Lección 2:** $9.23 (Lección 1)
+ $185.48 (Lección 2) = **$194.71 USD/mes**.