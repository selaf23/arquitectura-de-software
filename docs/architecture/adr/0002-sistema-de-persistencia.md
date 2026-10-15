# ADR-0002: Sistema de Persistencia — Capa Híbrida (RDS Aurora MySQL + Redis Cluster + Object Storage S3)

- **Estado:** Aceptado
- **Fecha:** 2026-10-13
- **Decisores:** Kevin Castro, Julian Ospina (Equipo de Arquitectura), Auditor / Seguridad
- **Proyecto:** Aprende 360 — Plataforma Integradora LMS
- **Stakeholders afectados:** Estudiante Universidad, Directivo / Decano, Proveedor Cloud / TI
- **Vistas 4+1 relacionadas:** Vista Lógica, Vista de Procesos, Vista Física

---

## 1. Contexto y Declaración del Problema

La persistencia tradicional de Moodle concentra base de datos, sesiones y archivos en un mismo nodo. En Aprende 360 esto genera tres problemas críticos:

1. **R-01** — Saturación de MySQL por concurrencia masiva en exámenes simultáneos (bloqueo de tablas de estado de cuestionario).
2. **R-02** — Agotamiento del almacenamiento local `MoodleData` por subida masiva de tareas (PDF, video, audio).
3. Desautenticaciones aleatorias y cuellos de botella de I/O cuando las sesiones se guardan en disco o en la BD en un entorno multinodo.

> ¿Cómo estructurar la capa de persistencia para soportar alta concurrencia, desacoplar los archivos masivos y mantener las sesiones consistentes entre nodos?

---

## 2. Factores de Decisión (Decision Drivers)

- **ASR-01** — 5.000 cuestionarios simultáneos con latencia baja (ESC-01).
- **ASR-03** — Archivos en Object Storage con latencia de descarga < 2 s (ESC-08).
- **ASR-06** — RPO < 15 min y RTO < 2 h ante fallas catastróficas (ESC-02).
- **ASR-07** — Cifrado en tránsito (TLS 1.3) y en reposo (AES-256).
- **RES-02** — Presupuesto Cloud fijo por suscripción.
- **RES-05** — El almacenamiento pesado debe desacoplarse del servidor web hacia S3 / Object Storage.
- **ESC-02 / ESC-08** — Failover automático y cero pérdida de archivos entregados.

---

## 3. Alternativas Consideradas

### 3.1 Base de datos relacional
| Opción | Evaluación |
|--------|-----------|
| MySQL en instancia única | Punto único de fallo, sin failover, incumple ASR-06. |
| **RDS Aurora MySQL Multi-AZ (1 writer + réplicas de lectura)** | Failover automático, lecturas derivadas y Managed (reduce carga DevOps). **Elegida.** |
| NoSQL (MongoDB / DynamoDB) | Rompe compatibilidad con el esquema relacional de Moodle (RES-01) y no aporta ventaja al modelo académico. |

### 3.2 Gestión de sesiones y caché
| Opción | Evaluación |
|--------|-----------|
| Sticky Sessions en el Load Balancer | Frágil ante caída de nodo, sin tolerancia a fallos. |
| Sesiones en base de datos MySQL | Saturación de I/O en la BD (agrava R-01). |
| **Redis Cluster dedicado (sesiones PHP + MUC)** | Lecturas < 2 ms, capa web apátrida, alta disponibilidad. **Elegida.** |

### 3.3 Almacenamiento de archivos (`MoodleData`)
| Opción | Evaluación |
|--------|-----------|
| Servidor NFS / AWS EFS compartido | Alto costo y latencia elevada en archivos pequeños. |
| Almacenamiento local sincronizado con `rsync` | Riesgo de desincronización entre nodos web. |
| **Bucket Amazon S3 + plugin oficial Object Storage + CloudFront CDN** | Almacenamiento prácticamente ilimitado, -70 % de I/O en web, descargas aceleradas. **Elegida.** |

---

## 4. Decisión Adoptada

Se adopta una **arquitectura de persistencia híbrida y desacoplada** compuesta por tres capas especializadas:

1. **Persistencia relacional:** **Amazon RDS Aurora MySQL Multi-AZ** (nodo escritor + réplicas de lectura) para registros académicos y calificaciones.
2. **Estado y caché:** **Redis Cluster (ElastiCache)** dedicado a sesiones PHP y al *Moodle Universal Cache* (MUC).
3. **Almacenamiento de objetos:** **Amazon S3** acoplado a **CloudFront CDN** para entregables y multimedia, mediante el plugin oficial de Moodle.

```
Pods Moodle (stateless)
   ├── sessions / MUC  →  Redis Cluster      (ADR-0002)
   ├── calificaciones  →  RDS Aurora MySQL   (writer + read replicas)
   └── archivos tarea  →  S3  →  CloudFront CDN
```

Esta decisión **depende de** [ADR-0001](0001-patron-arquitectonico.md), ya que el escalado horizontal de pods solo es viable si la capa de estado se externaliza.

---

## 5. Consecuencias

**Positivas**
- Lecturas de sesión < 2 ms y capa web **totalmente sin estado** (habilita ADR-0001).
- Reducción del **70 % de carga I/O** en los servidores web por el desacoplamiento a S3.
- Failover automático de la BD con RTO < 60 s (ESC-02) y almacenamiento escalable casi ilimitado.
- Cifrado en reposo (AES-256) habilitado de forma nativa en S3 y RDS (soporta ASR-07 y R-05).

**Negativas / Compromisos**
- Mayor dependencia de la API de S3 y costos variables por *data egress* en CloudFront.
- Introduce componentes de infraestructura intermedios (Redis, CDN) que requieren monitoreo.
- Riesgo de **vendor lock-in** parcial sobre el proveedor Cloud (RES-02 / Proveedor Cloud).

---

## 6. Mecanismo de Verificación

- Simulación de caída de un pod web durante una sesión de cuestionario, verificando que el estudiante **no pierda sesión** (Redis).
- Prueba de **subida simultánea de 500 archivos** confirmando persistencia en el bucket S3 y descarga vía CDN.
- Verificación de **failover Multi-AZ** con conmutación al nodo Read-Replica < 60 s.
- Revisión de las **métricas de facturación AWS S3** contra el presupuesto (RES-02).

---

## 7. Trazabilidad y Enlaces

- **Diagrama de contexto (C4 Nivel 1):** [`../diagrams/c4-contexto.puml`](../diagrams/c4-contexto.puml)
- **Diagrama de secuencia del proceso clave:** [`../diagrams/secuencia-proceso-clave.puml`](../diagrams/secuencia-proceso-clave.puml)
- **Índice de arquitectura:** [`../README.md`](../README.md)
- **ADR relacionadas:** [ADR-0001 — Patrón Arquitectónico](0001-patron-arquitectonico.md) · [ADR-0003 — Estrategia de Seguridad](0003-estrategia-seguridad.md)

| Necesidad | ASR / Escenario | ADR | Verificación |
|-----------|-----------------|-----|--------------|
| Persistencia de entregables | ASR-03 / ESC-08 | ADR-0002 | Subida de 500 archivos + CDN |
| BD sin caídas | ASR-06 / ESC-02 | ADR-0002 | Failover Multi-AZ < 60 s |
| Control de costos de almacenamiento | RES-02 / RES-05 | ADR-0002 | Métricas de facturación S3 |

---

## 8. Estado de Cumplimiento de la Decisión

**Estado: ACEPTADO.** Ratificado por el equipo de arquitectura. La implementación de la externalización de estado es requisito previo para el despliegue en producción de [ADR-0001](0001-patron-arquitectonico.md).
