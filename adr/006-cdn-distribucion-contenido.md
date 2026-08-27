# ADR-006: Disponibilidad de contenido mediante CDN

**Estado:** Aceptado
**Fecha:** 2026-08-27
**Lección relacionada:** Lección 6 - Disponibilidad de contenidos de aplicaciones en cloud

## Contexto

ADC sirve su catálogo de productos a usuarios en Chile, Perú y Colombia.
Hasta ADR-001, el acceso de lectura al catálogo (`adc-catalog-media` y
`adc-catalog-docs`) se resolvía con URLs firmadas de S3 generadas por el
Servicio de Catálogo  sin distribución geográfica, cada solicitud viaja
hasta el bucket en la región de origen.

Aplicando el marco de clasificación del material de esta lección (público /
interno / confidencial / restringido) a los datos de ADC:

| Contenido | Clasificación | Justificación |
|---|---|---|
| Imágenes/videos de producto (`adc-catalog-media`) | **Público** | Sin datos personales; diseñado para visualización masiva, incluso por buscadores (SEO) |
| Fichas técnicas (`adc-catalog-docs`) | **Público** | Documentación de producto, sin información comercial sensible |
| Datos transaccionales (pedidos, pagos) | **Restringido** | Ya protegido en RDS, subred privada (ADR-002, ADR-003) |
| Métricas de costos/márgenes comerciales | **Confidencial** | Mencionado como ejemplo en el material; fuera del alcance técnico de este proyecto |

Esta clasificación revela que exigir URLs firmadas para **leer** el catálogo
(decisión de ADR-001) no se ajusta a la naturaleza pública de ese contenido  es fricción sin beneficio de seguridad real, y además impide aprovechar
cacheo en el edge para reducir latencia hacia los 3 países.

## Decisión

Se implementa **Amazon CloudFront** como CDN delante de ambos buckets S3
del catálogo, usando **Origin Access Control (OAC)** para que los buckets
permanezcan privados (sin acceso público directo a S3), mientras
CloudFront sirve el contenido **públicamente**, acorde a su clasificación.

- **Se elimina** la URL firmada de lectura (GET) definida en ADR-001  el
  contenido público no la necesita.
- **Se mantiene sin cambios** la URL firmada de subida (PUT) desde el CMS,
  publicar contenido sigue siendo una acción restringida al personal
  autorizado.
- Una única distribución CloudFront con **dos orígenes** (comportamiento
  por ruta, similar al enrutamiento por contenido del ALB en ADR-005):
  `/media/*` → `adc-catalog-media`, `/docs/*` → `adc-catalog-docs`.
- **Política de caché:** TTL por defecto 24 h; invalidación evitada mediante
  *cache-busting* por versión de objeto (el Servicio de Catálogo incluye el
  `versionId` de S3 en la URL), en vez de invalidaciones manuales.

## Pilares de AWS Well-Architected Framework

| Pilar | ¿Cómo lo aborda esta decisión? |
|---|---|
| **Performance Efficiency** | Los Edge Locations de CloudFront acercan el contenido a usuarios en CL/PE/CO sin necesitar réplicas multi-región de S3 |
| **Cost Optimization** | Reduce solicitudes GET directas a S3 (facturadas) al servir desde caché; el cache-busting por versión evita el costo de invalidaciones manuales frecuentes |
| **Security** | OAC mantiene los buckets S3 sin acceso público directo; la clasificación explícita del contenido (según el marco del material) justifica qué sí y qué no se expone abiertamente |
| **Operational Excellence** | Elimina la generación de URLs firmadas de lectura para contenido que nunca debió requerirlas |

## Alternativas consideradas

| Opción | Ventajas | Desventajas | ¿Por qué se descartó / eligió? |
|---|---|---|---|
| **CloudFront + OAC** (elegida) | Reduce latencia y costo; mantiene S3 privado sin exponerlo directamente | Requiere gestionar una capa adicional (distribución, comportamientos de caché) | Elegida: se ajusta a la clasificación "público" del contenido y al alcance geográfico de ADC |
| Mantener URLs firmadas GET (statu quo ADR-001) | Sin cambios adicionales | No reduce latencia para usuarios LATAM; cada solicitud golpea S3 directamente; fricción de seguridad sin beneficio real para contenido público | Descartada  no se ajusta a la naturaleza del contenido una vez clasificado correctamente |
| CloudFront con Signed URLs/Cookies (todo el contenido restringido) | Máximo control de acceso | Contradice la clasificación "público"; innecesario para catálogo sin licenciamiento especial | Descartada por ahora  candidata si en el futuro ADC vende contenido premium/licenciado |
| Multi-Region S3 + Route 53 latency-based routing (sin CDN) | Sin capa de caché externa | Requiere replicar buckets completos en múltiples regiones; mayor costo y complejidad que activar un CDN | Descartada |

## Métricas de éxito

- **SLA/SLO:** 99.9% (heredado del contexto de negocio para el catálogo)
- **Nueva métrica de este ADR  Cache hit ratio:** ≥ 85% (objetivo de eficiencia del CDN; un ratio bajo indicaría TTLs mal configurados o cache-busting demasiado agresivo)

## Diseño ideal vs. restricción del AWS Academy Learner Lab

| Aspecto | Diseño ideal (producción) | Lo que se implementa/documenta en el Lab | Motivo de la brecha |
|---|---|---|---|
| Price Class de CloudFront | "Use all edge locations"  máxima cobertura global, incluyendo LATAM | Price Class reducida (menor costo) | El Lab tiene créditos limitados; se documenta el impacto: menor cobertura de edge locations cercanas a CL/PE/CO |
| Dominio y certificado | Dominio propio de ADC con certificado ACM | Dominio por defecto `*.cloudfront.net` | Misma brecha que ADR-005  ADC no tiene dominio real registrado |
| AWS WAF en CloudFront | Reglas gestionadas contra ataques comunes | Fuera de alcance  no hay lección dedicada a seguridad perimetral en este módulo | Misma nota que ADR-005 |

## Costos estimados

- **Fuente:** AWS Pricing Calculator (Pay as you go, región South America) - https://calculator.aws/#/estimate?id=2bd261830a00baab7b2d45a591dbd28631299362
- **Resultado:** $3.97 USD/mes ($3.85 transferencia de datos + $0.12 solicitudes HTTPS), dentro del free tier real de CloudFront
- **Detalle completo:** ver `docs/06-disponibilidad-contenido-cdn/costos.md`

## Consecuencias

**Positivas:**
- Menor latencia percibida para usuarios en los 3 países de operación
- Reduce costo de solicitudes directas a S3 al servir desde caché
- La clasificación explícita de contenido queda documentada y es reutilizable si ADC agrega más tipos de datos en el futuro

**Negativas / riesgos:**
- El cache-busting por versión requiere que el Servicio de Catálogo (Lección 1) incluya correctamente el `versionId` en cada URL  un error ahí serviría contenido desactualizado desde el edge hasta que expire el TTL
- Reducir la Price Class en el Lab significa que la prueba en el entorno académico no refleja fielmente la latencia real que tendría un usuario en Lima o Bogotá

## Referencias

- Amazon CloudFront: https://aws.amazon.com/cloudfront/
- Origin Access Control: https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-restricting-access-to-origin.html