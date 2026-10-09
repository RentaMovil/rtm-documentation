# DOCUMENTACIÓN DE ANÁLISIS DEL SOFTWARE — DIAGRAMAS DE ACTIVIDAD

## PROYECTO: RENTAMOVIL — PLATAFORMA PARA EMPRESA DE ALQUILER DE VEHÍCULOS

**INTEGRANTES:** Thiago Fabian Rojas Guevara — David Santiago Erez Rodríguez — Nicole Dayana Ramírez Vargas

**SERVICIO NACIONAL DE APRENDIZAJE — SENA — ANÁLISIS Y DESARROLLO DE SOFTWARE 3145556**

**2026**

---

## 1. Introducción

Este documento presenta los diagramas de actividad de los dos procesos de negocio más complejos de
RentaMovil: el ciclo completo de una reserva y el ciclo de mantenimiento de un vehículo. Es la
tercera de las cuatro vistas dinámicas del análisis del software; los casos de uso, los diagramas
de secuencia y los de estados están en documentos separados dentro de esta misma carpeta.

---

## ACT-01 — Proceso de reserva y pago

| Campo | Valor |
|-------|-------|
| RF relacionados | RF3.1, RF3.3, RF5.1, RF5.2, RF4.1, RF4.2 |
| Descripción | Recorre una reserva desde que el Cliente elige el vehículo hasta que el alquiler termina y queda facturado, incluyendo los caminos de rechazo, reintento, cancelación y expiración automática |

```mermaid
flowchart TD
    A([Inicio: Cliente autenticado]) --> B[Explorar catálogo y elegir vehículo]
    B --> C[Seleccionar fechas, sucursales y seguro opcional]
    C --> D{¿Acepta términos y condiciones?}
    D -- No --> C
    D -- Sí --> E[Reserva creada: PENDING_PAYMENT]
    E -- Sube comprobante --> G[Reserva: PENDING_REVIEW]
    E -. 24h sin comprobante aprobado .-> I[Reserva: CANCELLED]
    G --> H{Administrador revisa el comprobante}
    H -- Rechaza, permite reintento --> E
    H -- Rechaza y cancela --> I
    H -- Aprueba --> J[Reserva: CONFIRMED + factura generada]
    J -- Cliente cancela, ≥3 días antes --> I
    J --> M[Día de recogida: se registra el Rental]
    M --> N[Vehicle: RENTED, GPS activo]
    N --> O[Día de devolución: se registra el retorno]
    O --> P[Vehicle: AVAILABLE, Reserva: COMPLETED]
    P --> Z([Fin])
    I --> Z
```

---

## ACT-02 — Proceso de mantenimiento de un vehículo

| Campo | Valor |
|-------|-------|
| RF relacionados | RF8.3, RF8.7 |
| Descripción | Recorre el ciclo de vida de un mantenimiento, desde que se programa hasta que se completa o se cancela |

```mermaid
flowchart TD
    A([Inicio]) --> B{¿Vehicle.status == RETIRED?}
    B -- Sí --> X([No se puede programar mantenimiento])
    B -- No --> C[Mantenimiento programado: SCHEDULED]
    C --> D{El Administrador...}
    D -- cancela antes de iniciar --> E[Mantenimiento: CANCELLED]
    D -- inicia --> F{¿Vehicle.status == AVAILABLE?}
    F -- No --> G([Rechazado: el vehículo no está disponible])
    F -- Sí --> H[Mantenimiento: IN_PROGRESS, Vehicle: MAINTENANCE]
    H --> I{El Administrador...}
    I -- completa --> J[Mantenimiento: COMPLETED, Vehicle: AVAILABLE]
    I -- cancela --> K[Mantenimiento: CANCELLED, Vehicle: AVAILABLE]
    J --> Z([Fin: el registro queda de solo lectura])
    K --> Z
    E --> Z
```

---

## Referencias

| Título del documento | Referencia |
|------------------------|------------|
| Especificación de Requisitos de Software | `01-informe-especificacion-requisitos/SRS.md` (este repositorio) |
| Casos de uso | `01-Casos-de-Uso.md` (esta carpeta) |
| Diagramas de secuencia | `02-Diagramas-de-Secuencia.md` (esta carpeta) |
| Diagramas de estados | `04-Diagramas-de-Estados.md` (esta carpeta) |
