# DOCUMENTACIÓN DE ANÁLISIS DEL SOFTWARE — DIAGRAMAS DE SECUENCIA

## PROYECTO: RENTAMOVIL — PLATAFORMA PARA EMPRESA DE ALQUILER DE VEHÍCULOS

**INTEGRANTES:** Thiago Fabian Rojas Guevara — David Santiago Erez Rodríguez — Nicole Dayana Ramírez Vargas

**SERVICIO NACIONAL DE APRENDIZAJE — SENA — ANÁLISIS Y DESARROLLO DE SOFTWARE 3145556**

**2026**

---

## 1. Introducción

Este documento presenta los diagramas de secuencia de los flujos principales de RentaMovil: el
intercambio de mensajes entre el actor, el API Gateway, los microservicios y la base de datos para
cada operación. Es la segunda de las cuatro vistas dinámicas del análisis del software; los casos
de uso, los diagramas de actividad y los de estados están en documentos separados dentro de esta
misma carpeta.

**Nota sobre los nombres de servicio:** los diagramas usan los nombres de despliegue físico de los
microservicios (`api-gateway`, `iam`, `fleet-maintenance`, `booking-reservation`,
`payment-billing`, `telemetry-gps`), no los nombres conceptuales de Bounded Context del SRS. Los
flujos de Rental Execution y Notification, que en el SRS aparecen como contextos propios, se
ejecutan físicamente dentro de `booking-reservation` — son parte de su mismo ciclo de vida y no
necesitan desplegarse como servicios separados.

---

## SEQ-01 — Registro de cliente

| Campo | Valor |
|-------|-------|
| Actor | Visitante |
| RF relacionado | RF1.1 |
| Precondición | El visitante no tiene cuenta en el sistema |
| Postcondición | Se crea una Person y un User con rol `CLIENT` |

```mermaid
sequenceDiagram
    actor Visitante
    participant GW as API Gateway
    participant IAM as iam
    participant DB as BD (person, user)

    Visitante->>GW: POST /auth/register
    GW->>IAM: Reenvía la solicitud
    IAM->>DB: ¿El email o el username ya existen?
    alt Email o username ya registrados
        DB-->>IAM: Sí existen
        IAM-->>GW: 409 (EMAIL_ALREADY_EXISTS / USERNAME_TAKEN)
        GW-->>Visitante: 409
    else Datos disponibles
        DB-->>IAM: No existen
        IAM->>DB: INSERT Person + User (role = CLIENT)
        DB-->>IAM: OK
        IAM-->>GW: 201 Created
        GW-->>Visitante: 201 Created
    end
```

---

## SEQ-02 — Inicio de sesión

| Campo | Valor |
|-------|-------|
| Actor | Cliente (o cualquier usuario registrado) |
| RF relacionado | RF1.2 |
| Precondición | El usuario tiene una cuenta activa |
| Postcondición | Se emiten access token y refresh token |

```mermaid
sequenceDiagram
    actor Usuario
    participant GW as API Gateway
    participant IAM as iam
    participant DB as BD (user)

    Usuario->>GW: POST /auth/login {credenciales}
    GW->>GW: ¿Esta IP superó 10 intentos fallidos en los últimos 5 min?
    alt IP bloqueada temporalmente
        GW-->>Usuario: 429 Too Many Requests
    else IP habilitada
        GW->>IAM: Reenvía la solicitud
        IAM->>DB: Busca el usuario y valida la contraseña
        alt Credenciales inválidas
            DB-->>IAM: No coincide
            IAM-->>GW: 401 Unauthorized
            GW->>GW: Registra un intento fallido para esta IP
            GW-->>Usuario: 401 Unauthorized
        else Credenciales válidas y status = ACTIVE
            DB-->>IAM: Usuario válido
            IAM->>IAM: Genera access token (30 min) y refresh token (7 días)
            IAM-->>GW: 200 OK {accessToken, refreshToken}
            GW-->>Usuario: 200 OK
        end
    end
```

---

## SEQ-03 — Crear una reserva

| Campo | Valor |
|-------|-------|
| Actor | Cliente |
| RF relacionado | RF3.1, RF2.2 |
| Precondición | Cliente autenticado; el vehículo existe |
| Postcondición | `Reservation` creada en `PENDING_PAYMENT`; el Administrador queda notificado |

```mermaid
sequenceDiagram
    actor Cliente
    participant GW as API Gateway
    participant BR as booking-reservation
    participant FM as fleet-maintenance
    participant DB as BD (reservation)
    participant NOT as Notification

    Cliente->>GW: POST /reservations {vehículo, fechas, sucursales, seguro?, aceptaTérminos}
    GW->>GW: Valida JWT
    GW->>BR: Reenvía (con los claims del usuario)
    BR->>BR: Valida JWT y permiso reservations:create
    BR->>FM: ¿El vehículo está disponible en ese rango de fechas?
    FM-->>BR: Disponible
    BR->>BR: Calcula vehicle_subtotal, insurance_subtotal y total_amount
    BR->>DB: INSERT Reservation (status = PENDING_PAYMENT)
    DB-->>BR: OK
    BR-->>NOT: evento ReservationCreated
    NOT-->>NOT: Notifica al Administrador
    BR-->>GW: 201 Created {reservationId, status: PENDING_PAYMENT}
    GW-->>Cliente: 201 Created
```

---

## SEQ-04 — Pagar una reserva (revisión del comprobante)

| Campo | Valor |
|-------|-------|
| Actores | Cliente, Administrador |
| RF relacionado | RF5.1 |
| Precondición | Reserva en `PENDING_PAYMENT` |
| Postcondición | Reserva `CONFIRMED` (aprobado) o `PENDING_PAYMENT`/`CANCELLED` (rechazado) |

```mermaid
sequenceDiagram
    actor Cliente
    actor Admin as Administrador
    participant GW as API Gateway
    participant PB as payment-billing
    participant BR as booking-reservation
    participant CLD as Cloudinary
    participant DB as BD (payment, reservation)
    participant NOT as Notification

    Cliente->>CLD: Sube el comprobante de pago
    CLD-->>Cliente: URL del archivo
    Cliente->>GW: POST /payments {reservationId, bankAccountId, amount, receiptUrl}
    GW->>PB: Reenvía la solicitud
    PB->>PB: ¿amount == reservation.total_amount?
    alt El monto no coincide
        PB-->>GW: 400 AMOUNT_MISMATCH
        GW-->>Cliente: 400
    else Monto correcto
        PB->>DB: INSERT Payment (status = PENDING_REVIEW)
        PB->>BR: Reservation.status = PENDING_REVIEW
        PB-->>NOT: evento PaymentReceiptUploaded
        NOT-->>NOT: Notifica a los Administradores
        PB-->>GW: 201 Created
        GW-->>Cliente: 201 Created

        Admin->>GW: PATCH /payments/{id}/review {decisión}
        GW->>PB: Reenvía la solicitud
        alt El Administrador aprueba
            PB->>DB: Payment.status = APPROVED
            PB->>BR: Reservation.status = CONFIRMED
            PB-->>NOT: eventos PaymentCompleted, ReservationConfirmed, InvoiceGenerated
            NOT-->>NOT: Notifica al cliente
            PB-->>GW: 200 OK
            GW-->>Admin: 200 OK
        else El Administrador rechaza
            PB->>DB: Payment.status = REJECTED
            PB->>BR: Reservation.status = PENDING_PAYMENT o CANCELLED (según la decisión)
            PB-->>NOT: evento PaymentRejected
            NOT-->>NOT: Notifica al cliente
            PB-->>GW: 200 OK
            GW-->>Admin: 200 OK
        end
    end
```

---

## SEQ-05 — Ejecutar el alquiler (recogida, devolución y factura)

| Campo | Valor |
|-------|-------|
| Actor | Administrador |
| RF relacionado | RF4.1, RF4.2, RF5.2 |
| Precondición | Reserva en `CONFIRMED` |
| Postcondición | `Rental` completado, `Vehicle` disponible de nuevo, factura generada |

```mermaid
sequenceDiagram
    actor Admin as Administrador
    participant GW as API Gateway
    participant BR as booking-reservation
    participant FM as fleet-maintenance
    participant TG as telemetry-gps
    participant PB as payment-billing
    participant DB as BD (rental, reservation)

    Admin->>GW: POST /rentals {reservationId, initialMileage, gpsId}
    GW->>BR: Reenvía la solicitud
    BR->>BR: ¿Reservation.status == CONFIRMED?
    BR->>DB: INSERT Rental (status = IN_PROGRESS)
    BR->>FM: Vehicle.status = RENTED
    BR->>TG: Inicia el registro de Location para este GPS
    BR-->>GW: 201 Created {rentalId}
    GW-->>Admin: 201 Created

    Note over TG: Mientras el alquiler está en curso,<br/>telemetry-gps registra Location cada ≤ 60s

    Admin->>GW: PATCH /rentals/{id}/return {finalMileage}
    GW->>BR: Reenvía la solicitud
    BR->>BR: ¿finalMileage >= initialMileage?
    BR->>DB: Rental.status = COMPLETED
    BR->>FM: Vehicle.status = AVAILABLE
    BR->>DB: Reservation.status = COMPLETED
    BR->>TG: Detiene el registro de Location
    BR-->>PB: evento RentalCompleted
    PB->>PB: Genera Invoice (invoice_number, PDF)
    PB-->>GW: 200 OK {facturaDisponible: true}
    GW-->>Admin: 200 OK
```

---

## SEQ-06 — Programar y completar un mantenimiento

| Campo | Valor |
|-------|-------|
| Actor | Administrador |
| RF relacionado | RF8.3, RF8.7 |
| Precondición | El vehículo no está `RETIRED` |
| Postcondición | Mantenimiento `COMPLETED`; vehículo `AVAILABLE` de nuevo |

```mermaid
sequenceDiagram
    actor Admin as Administrador
    participant GW as API Gateway
    participant FM as fleet-maintenance
    participant DB as BD (vehicle, vehicle_maintenance)
    participant NOT as Notification

    Admin->>GW: POST /maintenances {vehicleId, maintenanceTypeId, startDate}
    GW->>FM: Reenvía la solicitud
    FM->>FM: ¿Vehicle.status distinto de RETIRED?
    FM->>DB: INSERT VehicleMaintenance (status = SCHEDULED)
    FM-->>GW: 201 Created
    GW-->>Admin: 201 Created

    Admin->>GW: PATCH /maintenances/{id}/start
    GW->>FM: Reenvía la solicitud
    FM->>FM: ¿Vehicle.status == AVAILABLE?
    FM->>DB: VehicleMaintenance.status = IN_PROGRESS
    FM->>DB: Vehicle.status = MAINTENANCE
    FM-->>NOT: evento VehicleSentToMaintenance
    FM-->>GW: 200 OK
    GW-->>Admin: 200 OK

    Admin->>GW: PATCH /maintenances/{id}/complete {cost?, observations?}
    GW->>FM: Reenvía la solicitud
    FM->>DB: VehicleMaintenance.status = COMPLETED, end_date = hoy
    FM->>DB: Vehicle.status = AVAILABLE
    FM-->>NOT: evento MaintenanceCompleted
    FM-->>GW: 200 OK
    GW-->>Admin: 200 OK
```

---

## SEQ-07 — Rastreo GPS en tiempo real

| Campo | Valor |
|-------|-------|
| Actor | Administrador |
| RF relacionado | RF6.1 |
| Precondición | El vehículo está en estado `RENTED` |
| Postcondición | Se devuelve la última ubicación registrada |

```mermaid
sequenceDiagram
    participant GPS as Dispositivo GPS
    participant TG as telemetry-gps
    participant DB as BD (location)
    actor Admin as Administrador
    participant GW as API Gateway

    loop Cada ≤ 60 segundos mientras Vehicle.status == RENTED
        GPS->>TG: Reporta coordenadas actuales
        TG->>DB: INSERT Location
    end

    Admin->>GW: GET /vehicles/{id}/location
    GW->>TG: Reenvía la solicitud
    TG->>TG: Valida rol ADMIN o SUPER_ADMIN
    alt Un Cliente intenta acceder
        TG-->>GW: 403 Forbidden
    else Administrador autorizado
        TG->>DB: SELECT última Location
        DB-->>TG: {lat, lng, timestamp}
        TG-->>GW: 200 OK
        GW-->>Admin: 200 OK
    end
```

---

## Referencias

| Título del documento | Referencia |
|------------------------|------------|
| Especificación de Requisitos de Software | `01-informe-especificacion-requisitos/SRS.md` (este repositorio) |
| Casos de uso | `01-Casos-de-Uso.md` (esta carpeta) |
| Diagramas de actividad | `03-Diagramas-de-Actividad.md` (esta carpeta) |
| Diagramas de estados | `04-Diagramas-de-Estados.md` (esta carpeta) |
| Catálogo de eventos de dominio | repositorio `rtm-docs`, `02-domain/domain-events.md` |
| Catálogo de microservicios (despliegue físico) | repositorio `rtm-docs`, `09-microservices/service-catalog.md` |
