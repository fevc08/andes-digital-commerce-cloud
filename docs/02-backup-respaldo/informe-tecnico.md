# Informe Técnico: Lección 2 - Estrategias de Almacenamiento y Respaldo en la Nube

## 1. Objetivo

Documentar cómo se implementa operativamente el plan de backup centralizado
definido en **[ADR-002](../../adr/002-estrategia-backup-recuperacion.md)**,
describiendo el flujo de respaldo y el procedimiento de recuperación para
cada tipo de dato de ADC.

## 2. Arquitectura de respaldo seleccionada

ADC utiliza **AWS Backup** como orquestador centralizado, con dos planes de
backup independientes, cada uno con su propia frecuencia y retención,
proporcional a la criticidad del dato que protege:

| | Datos transaccionales (RDS) | Catálogo de productos (S3) |
|---|---|---|
| **Mecanismo base** | Transaction log backup continuo (nativo de RDS) | Versionado (ADR-001) |
| **Snapshot vía AWS Backup** | Diario (retención 7 días) → Semanal (4 semanas) → Mensual (12 meses) | Semanal (retención 90 días) |
| **Redundancia adicional** | Multi-AZ (failover automático) | *(sin CRR — ver ADR-002)* |
| **RPO objetivo** | ≤ 15 min | ≤ 24 h |
| **RTO objetivo** | ≤ 1 h | ≤ 4 h |

## 3. Plan de backup transaccional (RDS)

Este plan protege la base de datos que soporta pedidos y pagos, el dato más
crítico del negocio, especialmente durante CyberDay.

**Cómo se sostiene el RPO ≤ 15 min:** RDS mantiene un respaldo continuo de
los transaction logs (WAL, write-ahead log), no solo snapshots periódicos.
Esto permite un **Point-in-Time Recovery (PITR)**: en caso de necesitar
restaurar, no se pierde todo lo ocurrido desde el último snapshot diario,
sino que se puede recuperar el estado de la base de datos hasta casi
cualquier minuto específico dentro del período de retención.

**Cómo se sostiene el RTO ≤ 1 h:** existen dos mecanismos distintos, para
dos escenarios distintos de falla:

- **Falla de infraestructura** (ej. la instancia o la zona de disponibilidad
  donde corre RDS deja de responder): **Multi-AZ** realiza un failover
  automático hacia la instancia réplica en la otra zona, típicamente en
  60-120 segundos. Este mecanismo no requiere intervención humana.
- **Corrupción lógica o eliminación accidental de datos** (ej. un query mal
  ejecutado borra registros): Multi-AZ no ayuda acá, porque el error se
  replica igual a ambas zonas. Se restaura desde el snapshot o mediante PITR
  a una nueva instancia, apuntando la aplicación a la instancia restaurada.
  Este proceso manual/semi-automatizado está presupuestado dentro de la
  ventana de 1 hora de RTO.

## 4. Plan de backup del catálogo (S3)

El catálogo tiene un perfil de riesgo distinto: los datos son
**re-sincronizables** desde su fuente de origen (el sistema de gestión de
productos), por lo que no se justifica una estrategia tan agresiva como la
transaccional.

- El **versionado** (activo desde ADR-001) es la primera línea de defensa:
  protege contra sobrescrituras accidentales del CMS sin necesitar un
  proceso de restauración formal, simplemente se recupera la versión
  anterior del objeto.
- El **snapshot semanal vía AWS Backup** protege contra un escenario más
  extremo: eliminación del bucket completo o corrupción masiva no cubierta
  por el versionado.

Esta combinación es suficiente para cumplir RPO ≤ 24 h sin necesitar
Cross-Region Replication continua, una decisión de costo/beneficio
documentada explícitamente en ADR-002.

## 5. Procedimiento de recuperación (runbook resumido)

**Escenario A - Falla de infraestructura en RDS:**
1. Multi-AZ detecta la falla y promueve automáticamente la réplica.
2. La aplicación reconecta usando el mismo endpoint DNS de RDS (sin cambio de configuración).
3. Tiempo estimado: 1-2 minutos, muy por debajo del RTO objetivo de 1 hora.

**Escenario B - Corrupción de datos transaccionales:**
1. Se identifica el punto en el tiempo previo al incidente.
2. Se restaura una nueva instancia RDS desde PITR o desde el snapshot más reciente anterior al incidente.
3. Se valida la integridad de los datos restaurados.
4. Se redirige la aplicación a la nueva instancia.
5. Tiempo estimado dentro de la ventana de 1 hora, dependiendo del tamaño de la base de datos.

**Escenario C - Pérdida de contenido del catálogo:**
1. Si es una sobrescritura puntual: se recupera la versión anterior del objeto directamente desde S3 (segundos).
2. Si es una pérdida masiva del bucket: se restaura desde el snapshot semanal de AWS Backup, aceptando hasta 24 h de posible desactualización, tal como está presupuestado en el RPO.

## 6. Relación con otras lecciones

- **Lección 1 (Almacenamiento de objetos):** el versionado de S3 activado en ADR-001 es la base sobre la que se construye el plan de backup del catálogo.
- **Lección 3 (Modelo de nube híbrido):** este informe asume que el ERP/WMS on-premise gestiona su propio backup de forma independiente, la justificación completa de esa separación de responsabilidad se formaliza en ADR-003.
- **Lección 5 (Disponibilidad de red):** el Multi-AZ mencionado aquí es específico de la capa de base de datos (RDS); el Multi-AZ de la capa de cómputo (ALB + Auto Scaling Group) se diseña por separado en ADR-005, con su propia justificación de tráfico y balanceo de carga.