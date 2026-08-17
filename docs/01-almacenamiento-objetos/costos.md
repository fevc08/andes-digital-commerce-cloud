# Estimación de Costos: Lección 1 - Almacenamiento de Objetos

**Fuente:** AWS Pricing Calculator
**Link de la estimación:** [Estimación - Almacenamiento de Objetos](https://calculator.aws/#/estimate?id=3f4ac8fd95e56a5ab22bfbc33b82685f8952e48a)
**Región asumida:** US East (N. Virginia) - `us-east-1`
**Fecha de estimación:** 2026-08-17

## Supuestos de diseño

| Bucket | Clase de almacenamiento | Volumen estimado | Tamaño promedio de objeto | Solicitudes/mes |
|---|---|---|---|---|
| `adc-catalog-docs` | S3 Standard | 20 GB | 2 MB | 200 PUT/COPY/POST/LIST · 5.000 GET |
| `adc-catalog-media` | S3 Intelligent-Tiering | 300 GB (Frequent 60%) + 150 GB (Infrequent 30%) + 50 GB (Archive Instant Access 10%) | 4 MB | 5.000 PUT/COPY/POST/LIST · 50.000 GET |

**Configuración relevante:**
- Tiers con demora de recuperación (Archive Access / Deep Archive Access) deshabilitados: 0%, el catálogo requiere disponibilidad instantánea.
- Lifecycle Transition requests: 0, cada bucket recibe objetos directamente en su clase, sin migración entre clases.
- Transferencia de datos: 10 GB/mes entrante (cargas desde CMS), 35 GB/mes saliente (consumo desde el front-end).

## Resultado de la calculadora

| Concepto | Costo |
|---|---|
| Costo único inicial (carga de datos) | $0.43 USD |
| **Costo mensual recurrente** | **$9.23 USD** |
| Costo total estimado a 12 meses | $111.19 USD |

*(Nota: es una estimación de diseño con AWS Pricing Calculator, calculada
sobre `us-east-1` por disponibilidad en el Learner Lab, no refleja el
consumo real de créditos del AWS Academy Learner Lab, que se factura de
forma distinta.)*

## Lectura del resultado

El costo mensual está compuesto principalmente por:
- Almacenamiento de los 500 GB de `adc-catalog-media` (el bucket dominante en volumen).
- Solicitudes GET, que son 10x más frecuentes que las PUT, coherente con un catálogo que se **consulta mucho más de lo que se actualiza**, típico de un front-end de e-commerce.
- El costo de *monitoring & automation* de Intelligent-Tiering (cargo por objeto, aplicado a los ~128.000 objetos estimados), que es el "costo de entrada" que se paga a cambio de no tener que mantener reglas de lifecycle manuales, trade-off ya justificado en ADR-001.

Este número (~$9 USD/mes) se va a usar como línea base en `docs/08-administracion-costos/` para el análisis de eficiencia consolidado del proyecto.