# ADR-0003: Estrategia de Seguridad — Identidad Federada, Multitenancy y Cifrado Extremo a Extremo

- **Estado:** Propuesto
- **Fecha:** 2026-10-13
- **Decisores:** Kevin Castro, Julian Ospina (Equipo de Arquitectura), Auditor / Seguridad
- **Proyecto:** Aprende 360 — Plataforma Integradora LMS
- **Stakeholders afectados:** Auditor / Seguridad, Estudiante Colegio, Administrador LMS, Directivo / Decano
- **Vistas 4+1 relacionadas:** Vista Lógica, Vista de Escenarios (+1), Vista Física

> **Nota de estado:** Esta decisión se encuentra en estado **PROPUESTO**. Requiere validación del comité de seguridad y una prueba de penetración (SAST/DAST) antes de ser ratificada como *Aceptada*.

---

## 1. Contexto y Declaración del Problema

Aprende 360 presta servicio a instituciones heterogéneas (colegios, universidades y administrativos) sobre una misma plataforma, gestionando datos personales de **menores de edad**. El diagnóstico identificó dos riesgos críticos:

- **R-03** — Falta de aislamiento multi-tenancy entre colegios y universidades → posible filtración de datos académicos entre instituciones.
- **R-05** — Incumplimiento de normativas de protección de datos de menores (Habeas Data / GDPR).

Adicionalmente, la plataforma requiere integrarse con identidades institucionales y registrar de forma confiable las modificaciones de calificaciones para auditoría académica.

> ¿Qué mecanismo de identidad, aislamiento y cifrado garantiza el cumplimiento normativo y la separación estricta de datos entre instituciones sin degradar la experiencia de acceso?

---

## 2. Factores de Decisión (Decision Drivers)

- **ASR-04** — Autenticación federada transparente vía SSO (Google Workspace, Microsoft Entra ID / LDAP) (ESC-07).
- **ASR-05** — Aislamiento lógico estricto (multitenancy) entre instituciones.
- **ASR-07** — Tráfico cifrado en tránsito (TLS 1.3) y datos personales cifrados en reposo (AES-256).
- **ASR-09** — Registro inmutable de modificaciones de calificaciones y respuestas de examen.
- **ESC-03** — Bloqueo de inyección SQL por WAF con registro de alerta.
- **RES-03** — Cumplimiento estricto de leyes de protección de datos y *habeas data* para menores.

---

## 3. Alternativas Consideradas

### 3.1 Identidad
| Opción | Evaluación |
|--------|-----------|
| Cuentas locales gestionadas por Moodle | No federada, difícil de administrar a escala, sin control central institucional. |
| **SSO federado mediante OAuth2 / SAML2 / LDAP delegado en el IdP institucional** | Acceso unificado, sin duplicar credenciales y con MFA en origen. **Elegida (propuesta).** |

### 3.2 Aislamiento multi-tenancy
| Opción | Evaluación |
|--------|-----------|
| Instancia Moodle separada por institución | Aislamiento fuerte pero insostenible en costo y operación para el equipo (RES-06). |
| **Multitenancy lógico con RBAC + *tenant scoping* + categorías/roles Moodle** | Un único clúster lógico (aprovecha ADR-0001) con separación estricta por institución. **Elegida (propuesta).** |

### 3.3 Protección perimetral y cifrado
| Opción | Evaluación |
|--------|-----------|
| Firewall de red tradicional (capa 4) | No inspecciona payloads de aplicación. |
| **WAF de capa 7 + TLS 1.3 + cifrado en reposo AES-256 + logs inmutables** | Inspección profunda, cumplimiento normativo y trazabilidad. **Elegida (propuesta).** |

---

## 4. Decisión Propuesta

Se propone adoptar un modelo de **seguridad en profundidad** compuesto por:

1. **Identidad federada (SSO):** autenticación delegada al IdP institucional vía OAuth2 / SAML2 / LDAP, con el módulo *Auth* de Moodle en el borde del cluster.
2. **Aislamiento multi-tenancy lógico:** RBAC + *tenant scoping* por institución sobre el cluster de [ADR-0001](0001-patron-arquitectonico.md), con cifrado de datos en reposo sobre la capa de [ADR-0002](0002-sistema-de-persistencia.md).
3. **Protección y trazabilidad:** WAF de capa 7 en el *Ingress*, TLS 1.3 en tránsito, AES-256 en reposo y **log inmutable** de cambios de calificaciones (Append-Only en S3 con *Object Lock*).

```
[Usuario] → [WAF L7 / ALB] → [IdP SSO: OAuth2/SAML2/LDAP]
                                   │ (token validado)
                                   ▼
                          [Pods Moodle + RBAC tenant-scoped]
                                   ├─ TLS 1.3 en tránsito
                                   ├─ AES-256 en reposo (RDS/S3) → ADR-0002
                                   └─ Logs inmutables (S3 Object Lock)
```

---

## 5. Consecuencias

**Positivas**
- Cumple el requisito regulatorio de protección de datos de menores (mitiga **R-05**).
- Elimina la filtración entre instituciones mediante aislamiento lógico estricto (mitiga **R-03**).
- Trazabilidad auditable e inmutable de calificaciones (ASR-09).
- Autenticación unificada < 1,0 s sin gestionar credenciales propias (ESC-07).

**Negativas / Compromisos**
- El **WAF L7 y el cifrado estricto TLS 1.3 introducen ~15-20 ms de latencia** por petición (Trade-off 1: Desempeño vs. Seguridad), compensada con caché de borde (CDN).
- Aumenta la carga operativa del WAF y de la gestión de secretos/tokens.
- La implementación de *Object Lock* y del modelo *tenant-scoped* exige rediseñar plugins personalizados.

---

## 6. Mecanismo de Verificación

- Escaneo **SAST/DAST** y **pruebas de penetración** del endpoint de autenticación (ESC-03: 0 vulnerabilidades explotadas, IP bloqueadas < 1 s).
- Prueba de autenticación federada con **SAML2 / OAuth2** validando sesión < 1,0 s.
- Pruebas de aislamiento multi-tenant: un usuario de una institución no puede acceder a recursos de otra.
- Verificación de **inmutabilidad** del log de auditoría de calificaciones.

---

## 7. Trazabilidad y Enlaces

- **Diagrama de contexto (C4 Nivel 1):** [`../diagrams/c4-contexto.puml`](../diagrams/c4-contexto.puml)
- **Diagrama de secuencia del proceso clave:** [`../diagrams/secuencia-proceso-clave.puml`](../diagrams/secuencia-proceso-clave.puml)
- **Índice de arquitectura:** [`../README.md`](../README.md)
- **ADR relacionadas:** [ADR-0001 — Patrón Arquitectónico](0001-patron-arquitectonico.md) · [ADR-0002 — Sistema de Persistencia](0002-sistema-de-persistencia.md)

| Necesidad | ASR / Escenario | ADR | Verificación |
|-----------|-----------------|-----|--------------|
| Protección de datos de menores | ASR-05 / ESC-03 | ADR-0003 | SAST/DAST + pentest |
| Acceso unificado por SSO | ASR-04 / ESC-07 | ADR-0003 | Prueba SAML2/OAuth2 < 1 s |
| Auditoría de calificaciones | ASR-09 | ADR-0003 | Verificación de log inmutable |

---

## 8. Estado de Cumplimiento de la Decisión

**Estado: PROPUESTO.** Pendiente de aprobación por el comité de seguridad y de la prueba de penetración. Al aprobarse, se promoverá el estado a *Aceptado* y se actualizará la matriz de trazabilidad del `README.md`.
