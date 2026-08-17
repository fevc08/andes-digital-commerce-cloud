# Informe Técnico: Lección 1 - Arquitectura de Almacenamiento de Objetos

## 1. Objetivo

Definir la estrategia de almacenamiento base del catálogo digital de Andes
Digital Commerce (ADC), integrándola con el flujo general de la arquitectura
(carga desde CMS, consumo desde el front-end de e-commerce).

La decisión completa, con alternativas evaluadas y pilares WAF, está
documentada en **[ADR-001](../../adr/001-tipo-almacenamiento-objetos.md)**.
Este informe se enfoca en el *cómo se integra* en el flujo operativo.

## 2. Arquitectura seleccionada

Se combinan dos patrones típicos de almacenamiento de objetos vistos en el
módulo:

- **Almacenamiento para aplicaciones web/móviles:** el catálogo se sirve como
  objetos accedidos vía URL, sin servidor de archivos tradicional.
- **Base para distribución de contenido multimedia:** la estructura de
  buckets se diseña ya pensando en la integración con CDN de la Lección 6,
  evitando rediseñar el modelo de datos más adelante.

## 3. Estructura de buckets y organización de objetos

| Bucket | Clase de almacenamiento | Contenido | Acceso |
|---|---|---|---|
| `adc-catalog-media` | S3 Intelligent-Tiering | Imágenes y videos de producto | Privado, vía presigned URL |
| `adc-catalog-docs` | S3 Standard | Fichas técnicas (PDF) | Privado, vía presigned URL |

**Convención de keys (organización interna del bucket):**

- `catalog-media/{pais}/{categoria}/{product_id}/{filename}`
- `catalog-docs/{pais}/{categoria}/{product_id}/{filename}`

Ejemplo: `catalog-media/cl/electronica/SKU-88213/imagen-principal.jpg`

Esta estructura permite, sin necesidad de una base de datos adicional:
- Filtrar/administrar contenido por país (relevante porque el catálogo puede
  variar entre Chile, Perú y Colombia).
- Aplicar políticas de acceso o auditoría por categoría si el negocio lo
  requiere a futuro.

**Metadatos de objeto:** cada objeto se etiqueta con `product_id`, `categoria`,
`pais` y `fecha_actualizacion` como *object tags*, lo que permite generar
reportes o políticas futuras sin depender de un sistema externo de indexado.

## 4. Flujo de integración con la arquitectura general

1. El equipo de Producto/Marketing sube o actualiza contenido desde el **CMS
   interno**.
2. El **servicio de catálogo** (backend) solicita a S3 una **URL firmada de
   subida (PUT)** con expiración corta; el CMS sube el archivo directamente al
   bucket correspondiente, sin pasar el binario por el backend.
3. El objeto queda almacenado en `adc-catalog-media` o `adc-catalog-docs`
   según su tipo, con versionado activo.
4. Cuando un usuario final visita el sitio, el **front-end de e-commerce**
   solicita al servicio de catálogo una **URL firmada de lectura (GET)** para
   mostrar la imagen/documento.
5. *(Extensión prevista en Lección 6):* este punto de acceso pasará a ser una
   distribución de **CloudFront** en lugar de URLs firmadas directas a S3,
   reduciendo latencia para usuarios en los 3 países.

Este flujo desacopla completamente el almacenamiento del servidor de
aplicación, el backend nunca maneja el binario del archivo, solo genera
permisos de acceso temporal. Esto reduce carga en el backend y es coherente
con el patrón "arquitectura orientada a URL" visto en el manual.

## 5. Justificación técnica resumida

La variable que más pesó en esta decisión fue el **patrón de acceso
impredecible** del catálogo (un producto puede reactivarse en cualquier
momento por una campaña). S3 Intelligent-Tiering resuelve esto de forma
nativa, sin que el equipo de ADC tenga que mantener reglas de ciclo de vida
manuales basadas en antigüedad, que es la alternativa más común, pero menos
adecuada para este caso de negocio específico (detalle completo de
alternativas evaluadas en ADR-001).

## 6. Relación con otras lecciones

- **Lección 2 (Backup y respaldo):** el versionado activado aquí es la base
  sobre la que se construye la estrategia de recuperación ante fallos.
- **Lección 6 (CDN):** la estructura de buckets y el modelo de acceso vía URL
  firmada están diseñados para conectarse directamente con CloudFront sin
  cambios estructurales.