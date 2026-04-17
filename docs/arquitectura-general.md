# Arquitectura General — Sistema de Compra de entradas a conciertos.

---

## Descripción

API REST en Java (Spring Boot) para gestionar compra de entradas a conciertos.

---

## Diagrama de Contexto

```
  ┌───────────────┐        ┌──────────────────────────────┐       ┌──────────────────┐
  │   Cliente     │ ───▶   │  Sistema de Venta de Entradas │──────▶│ Pasarela de Pago│
  └───────────────┘        │      (Tickets Concert)       │       │ (Stripe/PayPal) │
                           │      API REST                │       └──────────────────┘
  ┌───────────────┐        │      Backend (Spring/Node)   │
  │   Organizador │ ───▶   │      Puerto 8080             │──────▶┌──────────────────┐
  └───────────────┘        └──────────────────────────────┘       │ Servicio Email  │
                                                                  │ (Notificaciones)│
                                                                  └──────────────────┘
```

---

## Capas del sistema

```
┌────────────────────────────────────────────┐
│              Capa de API                   │
│   Controladores REST · DTOs                │
│   TicketController                         │
│   EventController                          │
├────────────────────────────────────────────┤
│           Capa de Aplicación               │
│   TicketService                            │
│   EventService                             │
│   PaymentOrchestrator                      │
│   NotificationService                      │
├────────────────────────────────────────────┤
│             Capa de Dominio                │
│   Entidades: Ticket, Event, User           │
│   Interfaces: TicketRepository, EventRepo  │
│   PaymentStrategy                          │
│   PricingStrategy                          │
├────────────────────────────────────────────┤
│          Capa de Infraestructura           │
│   JpaTicketRepository                      │
│   JpaEventRepository                       │
│   StripePaymentService / PayPalService     │
│   EmailService (SMTP / API externa)        │
│   Configuración Spring / JPA               │
└────────────────────────────────────────────┘
```


---

## Decisiones registradas

| ADR | Decisión | Principio | Impacto principal |
|-----|----------|-----------|-------------------|
| [ADR-001](../adr/ADR-001-srp-servicio-notificaciones.md) | Separar `NotificacionService` | SRP | Clases con una sola razón de cambio |
| [ADR-002](../adr/ADR-002-ocp-calculo-multas.md) | Patrón Strategy para multas | OCP | Nuevos tipos sin modificar código existente |
| [ADR-003](../adr/ADR-003-dip-repositorio-prestamos.md) | Interfaz `PrestamoRepositorio` | DIP | Tests unitarios sin base de datos |
| [ADR-004](../adr/ADR-004-lsp-tipos-de-libro.md) | Jerarquía de libros con interfaces | LSP | Sustitución segura sin excepciones inesperadas |
| [ADR-005](../adr/ADR-005-isp-gestion-usuarios.md) | Segregar interfaz de usuarios | ISP | Cada actor depende solo de lo que usa |
