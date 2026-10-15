# ADR-0001: Patrón Arquitectónico — Despliegue de Moodle en Contenedores sobre Kubernetes

- **Estado:** Aceptado
- **Fecha:** 2026-10-12
- **Decisores:** Kevin Castro, Julian Ospina (Equipo de Arquitectura), Equipo TI / DevOps
- **Proyecto:** Aprende 360 — Plataforma Integradora LMS
- **Stakeholders afectados:** Equipo TI / DevOps, Estudiante Universidad, Directivo / Decano
- **Vistas 4+1 relacionadas:** Vista de Desarrollo, Vista de Procesos, Vista Física

---

## 1. Contexto y Declaración del Problema

Moodle tradicionalmente se despliega de forma monolítica sobre servidores virtuales estáticos. En los periodos de evaluación de Aprende 360, la concurrencia pasa de ~200 a **5.000 usuarios activos simultáneos** (ASR-01 / ESC-01), lo que colapsa la capacidad de CPU y RAM de servidores fijos y provoca bloqueos en el motor de cuestionarios (Riesgo **R-01**).

La infraestructura de prueba local actual no define un mecanismo de escalado horizontal ni de alta disponibilidad que cumpla con el SLO de disponibilidad del **99,5 %** (ASR-02).

> ¿Cómo desplegar el núcleo del LMS de Aprende 360 para absorber picos de concurrencia elástica sin degradar la experiencia y cumpliendo el presupuesto de infraestructura?

---

## 2. Factores de Decisión (Decision Drivers)

- **ASR-01** — Soportar 5.000 cuestionarios simultáneos con respuesta por pregunta < 1,5 s.
- **ASR-02** — Disponibilidad del 99,5 % durante el periodo lectivo (ESC-02).
- **ASR-08** — Actualizar plugins sin detener los servicios core.
- **RES-01** — Mantener compatibilidad con el ecosistema Moodle 4.x (PHP / MySQL / PostgreSQL).
- **RES-02** — El costo Cloud no debe exceder el presupuesto por suscripción institucional.
- **RES-06** — Equipo reducido de 4 ingenieros DevOps para mantenimiento.
- **ESC-06** — Despliegue Canary sin downtime (Zero Downtime).

---

## 3. Alternativas Consideradas

| Opción | Descripción | Evaluación |
|--------|-------------|------------|
| **A. Servidores EC2 tradicionales** | Auto-scaling vertical (cambiar el tamaño de la instancia). | Escalado lento (minutos), requiere reinicio y no elimina el punto único de fallo. |
| **B. Serverless / AWS Fargate puro** | Ejecución sin gestión de servidores. | Costo muy elevado para cargas sostenidas y prolongadas; penaliza el presupuesto (RES-02). |
| **C. Contenedores Docker sobre Kubernetes (EKS/GKE) con HPA** | Empaquetar Moodle en imágenes y orquestar con escalado horizontal. | Escalado en < 90 s, capa web apátrida, alto aprovechamiento de recursos. **Elegida.** |

---

## 4. Decisión Adoptada

Se adopta el **empaquetamiento de Moodle en contenedores Docker orquestados con Kubernetes (Horizontal Pod Autoscaler — HPA)**, separando la **capa web apátrida (stateless)** de la **capa de persistencia**. Las sesiones y caché se delegan a Redis (ver [ADR-0002](0002-sistema-de-persistencia.md)) y los archivos a Object Storage S3.

El patrón arquitectónico resultante es un **monolito modular contenerizado**: se conserva el núcleo Moodle (RES-01) evitando el costo de una reescritura a microservicios, pero se despliega con capacidad de escala horizontal y sin estado en la capa de aplicación.

```
[Usuario] → [ALB / WAF] → [Service K8s] → [Pods Moodle replicados (HPA)]
                                              ├─ Redis (sesiones / MUC)   → ADR-0002
                                              ├─ RDS Aurora MySQL Multi-AZ → ADR-0002
                                              └─ S3 + CloudFront          → ADR-0002
```

---

## 5. Consecuencias

**Positivas**
- Escalado automático en **menos de 90 segundos** ante picos de demanda (mitiga R-01).
- Despliegues Canary sin downtime (ESC-06) y actualización de plugins sin parada del núcleo (ASR-08, RES-04).
- Mejor densidad de recursos y costo controlado mediante auto-scaling nocturno (RES-02, ESC-04).

**Negativas / Compromisos**
- Obliga a **externalizar el estado de sesión** fuera del contenedor (Redis) → acopla esta decisión con [ADR-0002](0002-sistema-de-persistencia.md).
- Aumenta la complejidad de los pipelines CI/CD y la curva de aprendizaje del equipo (RES-06).
- Introduce un nuevo componente de orquestación que requiere observabilidad y monitoreo.

---

## 6. Mecanismo de Verificación

- Pruebas de carga con **Apache JMeter** estresando el sistema hasta **5.000 usuarios concurrentes**, monitoreando el comportamiento del Horizontal Pod Autoscaler.
- Validación de despliegue **Canary en Staging** con métrica de **0 minutos de parada total**.

---

## 7. Trazabilidad y Enlaces

- **Diagrama de contexto (C4 Nivel 1):** [`../diagrams/c4-contexto.puml`](../diagrams/c4-contexto.puml)
- **Diagrama de secuencia del proceso clave:** [`../diagrams/secuencia-proceso-clave.puml`](../diagrams/secuencia-proceso-clave.puml)
- **Índice de arquitectura:** [`../README.md`](../README.md)
- **ADR relacionadas:** [ADR-0002 — Sistema de Persistencia](0002-sistema-de-persistencia.md) · [ADR-0003 — Estrategia de Seguridad](0003-estrategia-seguridad.md)

| Necesidad | ASR / Escenario | ADR | Verificación |
|-----------|-----------------|-----|--------------|
| Exámenes sin caídas | ASR-01 / ESC-01 | ADR-0001 | JMeter 5.000 hilos concurrentes |
| Cero tiempo de parada | ASR-02 / ESC-06 | ADR-0001 | Despliegue Canary en Staging |

---

## 8. Estado de Cumplimiento de la Decisión

**Estado: ACEPTADO.** Ratificado por el equipo de arquitectura e implementado en el diseño de producción de Aprende 360.
