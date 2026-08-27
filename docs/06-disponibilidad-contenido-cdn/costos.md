# Estimación de Costos: Lección 6 - Disponibilidad de Contenidos (CDN)

**Fuente:** AWS Pricing Calculator (Pay as you go, región South America)
**Link de la estimación:** https://calculator.aws/#/estimate?id=2bd261830a00baab7b2d45a591dbd28631299362
**Región asumida (origen S3):** US East (N. Virginia) - `us-east-1`
**Región de tarifa (solicitudes de usuarios):** South America
**Fecha de estimación:** [completar]

## Supuestos de diseño

| Componente | Configuración |
|---|---|
| Modelo de precio | Pay as you go (no Flat Rate, ver justificación abajo) |
| Solicitudes HTTPS | 55.000/mes (mismo volumen base de la Lección 1) |
| Datos transferidos a usuarios | 35 GB/mes |
| Datos transferidos hacia el origen (POST/PUT) | 0 GB, el flujo de subida no pasa por CloudFront (ADR-006) |
| Invalidaciones manuales | 0, cache-busting por versión (ADR-006) |

## Por qué "Pay as you go" y no un plan Flat Rate

Los planes Flat Rate de CloudFront cobran un monto fijo mensual sin importar
el cache hit ratio real. Eso anularía la posibilidad de demostrar
numéricamente la eficiencia del diseño — con Pay as you go, el costo *sí*
refleja qué tan bien está funcionando el cacheo, que es precisamente la
métrica de éxito (≥85% cache hit ratio) definida en ADR-006.

## Resultado de la calculadora

| Concepto | Costo mensual estimado (USD) |
|---|---|
| Transferencia de datos a internet (South America) | 3.85 |
| Solicitudes HTTPS (South America) | 0.12 |
| Transferencia hacia el origen (POST/PUT) | 0.00 |
| **Total Lección 6** | **3.97** |

*(Nota: estimación de diseño, no refleja el consumo real de créditos del AWS Academy Learner Lab.)*

## Nota sobre el Free Tier

El volumen real de ADC (35 GB, 55.000 solicitudes) está completamente
dentro del nivel gratuito permanente de CloudFront (1.024 GB y 10 millones
de solicitudes/mes), en un despliegue real, este costo sería **$0**. La
calculadora de AWS excluye intencionalmente el free tier de sus
estimaciones (misma práctica ya documentada en la Lección 1), por lo que
$3.97/mes representa el costo "de diseño" sin depender de un beneficio
promocional que AWS podría modificar en el futuro.

## Lectura del resultado

El costo de CloudFront en sí es marginal ($3.97/mes) comparado con el resto
del proyecto, pero su verdadero impacto en costos no está en esta línea,
sino en lo que **reduce** en otro lado: al absorber el 85% del tráfico en
caché, CloudFront evita que ~46.750 de las 55.000 solicitudes mensuales
lleguen hasta S3. El costo de esas solicitudes de S3 evitadas ya estaba
incluido en la estimación de la Lección 1 (que asumía tráfico 100% directo
a S3, antes de que existiera esta capa de CDN). Esto se retoma como
hallazgo en la Lección 8: el costo combinado real de "almacenamiento +
distribución" es menor que la simple suma de L1 + L6, porque ambas
estimaciones se calcularon en momentos distintos del diseño y no deben
sumarse sin ajustar el tráfico ya absorbido por caché.

**Costo total acumulado del proyecto hasta la Lección 6:** $476.87 (L1-L5)
+ $3.97 (L6) = **$480.84 USD/mes**.