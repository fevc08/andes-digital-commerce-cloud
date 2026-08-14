# Andes Digital Commerce (ADC): Arquitectura Cloud Conectada

> Proyecto ABP · Módulo 5 · Arquitecturas Cloud Básicas · SOFOFA

## 📌 Contexto

Andes Digital Commerce (ADC) es un retailer de e-commerce y logística que opera
en Chile, Perú y Colombia. Su plataforma de venta online enfrenta picos de
tráfico de 10-20x lo normal durante eventos tipo *CyberDay*, mientras que su
ERP y sistema de gestión de bodega (WMS) permanecen on-premise por
integraciones contractuales con proveedores logísticos locales.

Este proyecto documenta el diseño de una arquitectura cloud que resuelve, de
forma conectada e integral, los siguientes desafíos:

- Almacenamiento escalable del catálogo de productos
- Respaldo y recuperación de datos transaccionales
- Un modelo de nube híbrido justificado por la realidad del negocio
- Escalabilidad automática de cómputo para absorber picos de demanda
- Alta disponibilidad de la aplicación en múltiples zonas
- Distribución eficiente de contenido en 3 países
- Integración asíncrona entre servicios (pedido → pago → bodega)
- Estimación y control de costos en un modelo de tráfico altamente variable

## 🗂️ Estructura del repositorio

| Carpeta | Contenido |
|---|---|
| `docs/` | Informe técnico y costos por cada una de las 8 lecciones del módulo |
| `adr/` | Architecture Decision Records — una por cada decisión arquitectónica clave |
| `diagramas/` | Diagramas técnicos (.drawio) por etapa y el diagrama integrado final |
| `costos/` | Estimaciones consolidadas vía AWS Pricing Calculator |
| `entregable-final/` | Documento integrador y esquema de comunicación entre servicios |

## 🧭 Cómo navegar este proyecto

1. Empieza por `docs/00-contexto-negocio/` para entender el negocio y sus requisitos.
2. Cada carpeta `docs/0X-.../` corresponde 1:1 a una lección del módulo, y enlaza
   su(s) ADR(s) correspondiente(s) en `adr/`.
3. `entregable-final/documento-consolidado.md` integra todo el diseño en una
   sola narrativa, con el diagrama final embebido.

## ☁️ Servicios AWS utilizados

*(se irá completando a medida que avanzan las lecciones)*

## 🛠️ Entorno de trabajo

Diseño desarrollado y validado (donde aplica) sobre AWS Academy Learner Lab,
documentando explícitamente las diferencias entre el diseño ideal de
producción y las restricciones del entorno académico (rol `LabRole` sin
permisos IAM personalizados, créditos y tiempo de sesión limitados).

## 👤 Autor

Fidel Vera Chourio — SOFOFA, Módulo 5, Arquitectura Cloud