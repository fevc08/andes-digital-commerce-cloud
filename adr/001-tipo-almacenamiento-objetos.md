# ADR-001: Selección de arquitectura de almacenamiento de objetos para el catálogo de productos

**Estado:** Aceptado
**Fecha:** 2026-08-18
**Lección relacionada:** Lección 1 - Arquitecturas de almacenamiento de objetos

## Contexto

ADC gestiona un catálogo digital de miles de SKUs (imágenes de producto, videos
cortos y fichas técnicas en PDF) que debe estar disponible para usuarios en
Chile, Perú y Colombia. El contenido es cargado y actualizado regularmente por
el equipo de Producto o Marketing a través de un panel de administración (CMS
interno).

El patrón de acceso a este catálogo es **impredecible por diseño**, un producto
puede permanecer con tráfico bajo durante meses y luego recibir un pico masivo
de accesos si se destaca en una campaña o CyberDay. Esto hace inadecuada
cualquier estrategia de optimización de costos basada solo en antigüedad del
archivo.

Según `docs/00-contexto-negocio/contexto-y-requerimientos.md`, el catálogo
tiene un SLA de disponibilidad de **99.9%**, RTO ≤ **4 h** y RPO ≤ **24 h**
(re-sincronizable desde el sistema de origen del catálogo).

## Decisión

Se implementa un modelo de almacenamiento de objetos en **Amazon S3**, separado
en dos buckets según el tipo de contenido y su patrón de acceso:

- **`adc-catalog-media`:** imágenes y videos de producto, clase de
  almacenamiento **S3 Intelligent-Tiering**.
- **`adc-catalog-docs`:** fichas técnicas y documentos PDF, clase
  **S3 Standard**.

El acceso público a los objetos **no se realiza mediante ACLs públicas**, sino
a través de **URLs firmadas (presigned URLs)** generadas por el servicio de
catálogo, con expiración corta. Esto sienta la base para la integración con
CloudFront que se formalizará en la Lección 6 (ADR-006).

## Pilares de AWS Well-Architected Framework

| Pilar | ¿Cómo lo aborda esta decisión? |
|---|---|
| **Cost Optimization** | S3 Intelligent-Tiering mueve automáticamente los objetos entre el tier de acceso frecuente e infrecuente según el uso real, sin arriesgar mover a un tier lento un producto que vuelve a ser tendencia |
| **Performance Efficiency** | Separar medios (peso alto, acceso variable) de documentos (peso bajo, siempre debe ser rápido) permite políticas independientes por tipo de contenido |
| **Reliability** | S3 ofrece 99.999999999% (11 nueves) de durabilidad; el versionado activo protege contra sobrescrituras accidentales desde el CMS |
| **Security** | Buckets privados sin acceso público directo; acceso controlado por URLs firmadas con expiración y políticas IAM del servicio de catálogo |

## Alternativas consideradas

| Opción | Ventajas | Desventajas | ¿Por qué se descartó / eligió? |
|---|---|---|---|
| **S3 Intelligent-Tiering + S3 Standard diferenciado** (elegida) | Optimiza costos automáticamente sin sacrificar rendimiento en documentos críticos | Requiere separar el contenido en dos buckets con criterios distintos | Elegida: se ajusta al patrón de acceso real e impredecible del catálogo |
| S3 Standard único para todo | Máxima simplicidad operativa | Sobrecosto en objetos de baja frecuencia (productos de temporada/descontinuados) | Descartada por ineficiencia de costos a escala |
| Lifecycle rules manuales por antigüedad (Standard → IA → Glacier) | Control fino y predecible | La antigüedad no predice el acceso real; un producto de hace 2 años puede reactivarse en campaña | Descartada: no resuelve el problema real del negocio |
| Multi-cloud (S3 + Google Cloud Storage / Azure Blob) | Evita vendor lock-in | Complejidad operativa innecesaria; el Learner Lab es exclusivamente AWS | Descartada por alcance del proyecto |

## Métricas de éxito

- **SLA/SLO:** 99.9% de disponibilidad del catálogo (heredado del contexto de negocio)
- **RTO/RPO:** RTO ≤ 4 h / RPO ≤ 24 h — el mecanismo concreto que sostiene este RPO (versionado + replicación) se detalla en **ADR-002 (Lección 2)**

## Diseño ideal vs. restricción del AWS Academy Learner Lab

| Aspecto | Diseño ideal (producción) | Lo que se implementa/documenta en el Lab | Motivo de la brecha |
|---|---|---|---|
| Región | `sa-east-1` (São Paulo), por latencia hacia CL/PE/CO | Región disponible en el Learner Lab (típicamente `us-east-1`) | El Lab fija/limita la región disponible para cuentas de estudiante |
| Cifrado en reposo | SSE-KMS con llave gestionada por el cliente y rotación automática | SSE-S3 (AES-256 gestionado por AWS) | El rol `LabRole` no tiene permisos para crear/administrar claves KMS propias |
| Replicación entre regiones | Cross-Region Replication activa para `adc-catalog-media` | Diseñada y documentada, no desplegada | Requiere una segunda región habilitada, fuera del alcance del Lab |

## Costos estimados

- **Fuente:** AWS Pricing Calculator - [Estimación - Almacenamiento de Objetos](https://calculator.aws/#/estimate?id=3f4ac8fd95e56a5ab22bfbc33b82685f8952e48a)
- **Resultado:** $9.23 USD/mes ($111.19 USD proyectado a 12 meses, incluyendo $0.43 USD de costo único inicial)
- **Detalle completo:** ver `docs/01-almacenamiento-objetos/costos.md`

## Consecuencias

**Positivas:**
- Optimización de costos automática sin intervención manual
- Elimina la necesidad de servidores de archivos propios
- Escalabilidad prácticamente ilimitada
- Acceso seguro y auditable vía URLs firmadas

**Negativas / riesgos:**
- S3 Intelligent-Tiering cobra un pequeño monitoreo por objeto (aplica a objetos ≥128 KB); se acepta porque las imágenes/videos de producto superan ese umbral con margen
- Mantener dos buckets con políticas distintas exige disciplina en la convención de nombres del CMS (ver informe técnico)

## Referencias

- Amazon S3 - Cloud object storage: https://aws.amazon.com/es/s3/
- AWS Well-Architected Framework