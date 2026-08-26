# Estimación de Costos: Lección 4 - Escalabilidad de Servicios de Cómputo

**Fuente:** AWS Pricing Calculator (Bloque 1) + cálculo manual con tarifas oficiales (Bloque 2)
**Link de la estimación (Bloque 1):** https://calculator.aws/#/estimate?id=72e9154621e5e6eeb1a748b1b0d9aaa2cfd01d4d
**Región asumida:** US East (N. Virginia) - `us-east-1`
**Fecha de estimación:** 2026-08-25

## Supuestos de diseño

| Componente | Configuración |
|---|---|
| Fargate — Capacidad base | 4 tareas, 1 vCPU / 2 GB c/u, continuas (730 h/mes) |
| Fargate — Capacidad CyberDay | 36 tareas adicionales, 1 vCPU / 2 GB c/u, ~20 h/mes (promedio anualizado de 4 eventos de 60h/año) |

## Nota metodológica importante

El campo "Number of tasks or pods" de la calculadora usa la unidad **"per
day"**, pensada para cargas que se relanzan diariamente (ej. un batch job
diario), no para tareas que corren de forma continua. Usarlo directamente
con "Average duration = 730 horas" produce un **doble conteo**: la
calculadora convierte "N tareas por día" a un equivalente mensual
(N × 730/24 días) y luego multiplica otra vez por las 730 horas completas,
inflando el resultado en un factor de ~30x.

**Solución aplicada:**
- **Bloque 1 (capacidad base, activa todos los días del mes):** se
  aprovechó una equivalencia matemática, "4 tareas por día × 24 horas de
  duración" da el mismo resultado que "4 tareas continuas × 730 horas"
  (24 × 30.4 días ≈ 730), sin doble conteo. Verificado con "Show
  calculations": **$144.16/mes**, coincide con el cálculo manual de
  referencia ($144.16 exacto: 4×1×730×$0.04048 + 4×2×730×$0.004445).
- **Bloque 2 (capacidad CyberDay, NO ocurre todos los días del mes):** esa
  misma equivalencia no aplica, porque CyberDay ocurre solo 4 veces al año,
  no todos los días. Forzar el campo "per day" habría requerido inventar un
  valor artificial (ej. "1.18 tareas/día") sin sentido de negocio real. Se
  optó por un **cálculo manual**, usando las tarifas oficiales que la
  propia calculadora reveló en "Show calculations" del intento anterior
  ($0.04048/vCPU-hora, $0.004445/GB-hora):

    - 36 tareas × 1 vCPU × 20 horas × $0.04048/vCPU-hora = $29.15
    - 36 tareas × 2 GB × 20 horas × $0.004445/GB-hora = $6.40
    - Total Bloque 2 = $29.15 + $6.40 = **$35.55 USD/mes**

## Resultado final

| Concepto | Costo mensual estimado (USD) |
|---|---|
| Fargate - Capacidad base (verificado en calculadora) | 144.16 |
| Fargate - Capacidad CyberDay (cálculo manual) | 35.55 |
| **Total Lección 4** | **179.71** |

*(Nota: estimación de diseño, no refleja el consumo real de créditos del AWS Academy Learner Lab.)*

## Lectura del resultado

La capacidad de reserva para CyberDay representa ~20% del costo total de
esta lección ($35.55 de $179.71), a pesar de multiplicar por 9 la capacidad
base (de 4 a 40 tareas). Esto es la evidencia numérica de por qué Auto
Scaling es más barato que sobreaprovisionar: si ADC mantuviera las 40 tareas
encendidas las 730 horas del mes "por si acaso", el costo de cómputo sería
más de 10 veces mayor a los $179.71 actuales, ese es justamente el "gasto
invisible" que el material de la lección identifica como el principal
riesgo de la capacidad fija.

**Costo total acumulado del proyecto hasta la Lección 4:** $231.66 (L1+L2+L3)
+ $179.71 (L4) = **$411.37 USD/mes**.