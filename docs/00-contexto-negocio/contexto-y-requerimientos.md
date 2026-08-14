# Contexto de Negocio y Requerimientos - Andes Digital Commerce (ADC)

## 1. Perfil de la empresa

**Andes Digital Commerce (ADC)** es un retailer de comercio electrónico y
logística con operación en **Chile, Perú y Colombia**. Vende productos de
consumo masivo y electrónica a través de una plataforma web y app móvil,
con despacho propio en las tres capitales y despacho vía terceros en
regiones/ciudades secundarias.

## 2. Situación actual (estado on-premise)

| Sistema | Ubicación actual | Motivo de permanencia |
|---|---|---|
| ERP (finanzas, compras, inventario contable) | Datacenter propio, Santiago | Integraciones contractuales de largo plazo con proveedores logísticos locales; licenciamiento on-prem vigente por 3 años más |
| WMS (gestión de bodega) | Datacenter propio, Santiago | Depende de hardware de picking físico integrado al ERP |
| E-commerce (front-end, catálogo, carrito, checkout) | Servidores propios, capacidad fija | **Candidato a migración**, no escala ante eventos de alta demanda |

**Consecuencia arquitectónica:** el ERP/WMS on-premise no migra en este
proyecto. Esto es lo que obliga a un **modelo de nube híbrido real**
(Lección 3), y a diseñar la comunicación entre la nube y el on-premise como
**asíncrona y desacoplada** (Lección 7), en vez de asumir conectividad
directa y permanente.

## 3. Evento de negocio crítico: CyberDay

ADC participa en eventos de alto tráfico (CyberDay/CyberWeekend) 3-4 veces
al año, con ventanas de **48-72 horas continuas**.

| Métrica | Operación normal | Durante CyberDay |
|---|---|---|
| Usuarios concurrentes (estimado) | ~500 | ~5.000 – 10.000 (10-20x) |
| Requests/seg al front-end (estimado) | ~200 | ~2.000 – 4.000 |
| Tolerancia a caída del sitio | Baja | **Crítica**, cada minuto caído durante CyberDay tiene impacto directo en ingresos |

Este patrón de tráfico es el que justifica técnicamente **Auto Scaling**
(Lección 4) y **Load Balancing multi-AZ** (Lección 5): la arquitectura debe
crecer y decrecer con el tráfico, no operar sobredimensionada todo el año
por unos pocos días de pico.

## 4. Clasificación de datos y requerimientos por tipo

No todos los datos de ADC tienen el mismo nivel de criticidad. Definir esto
acá evita justificaciones genéricas ("hacemos backup porque es buena
práctica") y permite justificar **por qué cada mecanismo de respaldo o
disponibilidad es proporcional al dato que protege**.

| Tipo de dato | Dónde vive (propuesta) | SLA de disponibilidad | RTO | RPO |
|---|---|---|---|---|
| Catálogo de productos (imágenes, fichas técnicas) | S3 | 99.9% | ≤ 4 h | ≤ 24 h *(re-sincronizable desde el sistema de origen del catálogo)* |
| Datos transaccionales (pedidos, pagos) | RDS (motor relacional) | 99.95% durante CyberDay / 99.9% resto del año | ≤ 1 h | ≤ 15 min |
| Cola de eventos pedido→pago→bodega | SQS/SNS | 99.9% | N/A *(el propio diseño de colas es la mitigación de disponibilidad)* | N/A |
| ERP/WMS (on-premise) | Datacenter propio | Fuera de alcance de este proyecto | Gestionado por TI interna | Gestionado por TI interna |

> 📌 **Nota metodológica:** estas cifras son **supuestos de diseño
> razonables para un ejercicio académico**, no datos reales de una empresa
> existente. Se fijan aquí explícitamente para que todo el proyecto sea
> consistente y verificable, no se inventan de nuevo en cada lección.

## 5. Requerimiento de multi-región vs. restricción del Lab

**Diseño ideal:** dado que ADC opera en Chile, Perú y Colombia, la región
AWS más adecuada por latencia sería `sa-east-1` (São Paulo) o una estrategia
multi-región.

**Restricción del AWS Academy Learner Lab:** el Lab típicamente fija o
limita la región disponible (usualmente `us-east-1`), y no permite
configuraciones multi-cuenta/multi-región complejas.

**Decisión de este proyecto:** se documenta el diseño ideal (región LATAM)
en los ADR correspondientes, y se despliega/valida en el Lab sobre la
región disponible, dejando la brecha explícitamente registrada — igual que
haremos con Multi-AZ.

## 6. Principios que guiarán las decisiones (AWS Well-Architected Framework)

Cada ADR de este proyecto va a referenciar explícitamente uno o más de
estos pilares:

- **Reliability:** la plataforma debe seguir operando (o degradarse
  controladamente) ante fallos de instancia o zona, especialmente durante
  CyberDay.
- **Performance Efficiency:** el catálogo debe servirse rápido en los 3
  países sin sobredimensionar cómputo todo el año.
- **Cost Optimization:** el gasto debe ser proporcional al tráfico real
  (picos vs. operación normal), no una capacidad fija sobrecomprada.
- **Operational Excellence:** la integración con el ERP/WMS on-premise debe
  ser desacoplada y tolerante a fallos temporales de conectividad.
- **Security:** los datos transaccionales (pagos, datos de clientes) tienen
  mayor exigencia de protección que el catálogo público de productos.

## 7. Glosario de referencia para este proyecto

| Sigla | Significado | Uso en este proyecto |
|---|---|---|
| RTO | Recovery Time Objective (tiempo máximo aceptable para restaurar el servicio tras una falla) | Definido por tipo de dato en la sección 4 |
| RPO | Recovery Point Objective (máxima pérdida de datos aceptable, medida en tiempo) | Definido por tipo de dato en la sección 4 |
| SLA | Service Level Agreement (compromiso de disponibilidad medible) | Definido por tipo de dato en la sección 4 |
| WAF | AWS Well-Architected Framework | Pilares listados en la sección 6 |