# ADR-001 — Separar el envío de notificaciones del servicio de préstamos

**Fecha:** 2025-04-17
**Estado:** ✅ Aceptado
**Principio SOLID:** S — Single Responsibility Principle (SRP)

---

## Contexto

La clase `TicketService` es responsable de verificar si el usuario existe, calcular precio, procesar pago, guardar ticket de compra en DB y enviar correo de confirmación.

**Código actual (con el problema):**

```java
// TicketService.java — tiene más de una responsabilidad
@Service
public class TicketService {

    public void buyTicket(BuyTicketRequest request) {

        // 1. Validación
        if (request.getUserId() == null) {
            throw new IllegalArgumentException("UserId requerido");
        }

        // 2. Calcular precio
        double price = request.getBasePrice() * 1.18;

        // 3. Procesar pago
        // llamada directa a Stripe
        System.out.println("Procesando pago con Stripe...");

    }
}
```

**¿Cuál es el problema?**

`TicketService` tiene **3 razones para cambiar**:
- Si cambia la validación del usuario.
- Si cambia el cálculo de precios.
- Si cambia la paserla de pagos.

---

## Decisión

Extraemos las responsabilidades en componentes independientes.

**Código corregido:**

```java
// TicketValidator.java — responsabilidad única: validar la solicitud
@Component
public class TicketValidator {

    public void validate(BuyTicketRequest request) {
        if (request.getUserId() == null) {
            throw new IllegalArgumentException("UserId requerido");
        }
    }
}


// PricingService.java — responsabilidad única: calcular precios
@Component
public class PricingService {

    public double calculatePrice(double basePrice) {
        return basePrice * 1.18;
    }
}


// StripePaymentService.java - responsabilidad única: procesar pago
public interface PaymentService {
    void pay(double amount);
}

@Service
public class StripePaymentService implements PaymentService {

    @Override
    public void pay(double amount) {
        System.out.println("Pago realizado con Stripe: " + amount);
    }
}


// Clase Principal 
@Service
public class TicketService {

    private final TicketValidator validator;
    private final PricingService pricingService;
    private final PaymentService paymentService;
    private final TicketRepository ticketRepository;
    private final NotificationService notificationService;

    public TicketService(
        TicketValidator validator,
        PricingService pricingService,
        PaymentService paymentService,
        TicketRepository ticketRepository,
        NotificationService notificationService
    ) {
        this.validator = validator;
        this.pricingService = pricingService;
        this.paymentService = paymentService;
        this.ticketRepository = ticketRepository;
        this.notificationService = notificationService;
    }

    public void buyTicket(BuyTicketRequest request) {

        validator.validate(request);

        double price = pricingService.calculatePrice(request.getBasePrice());

        paymentService.pay(price);

    }
}


```

### Principio SOLID aplicado — SRP

> "Un módulo debe tener una, y solo una, razón para cambiar."

| Clase | Única razón de cambio |
|-------|----------------------|
| `TicketValidator` | Reglas para validar la soliicitud. |
| `PricingService` | Cálculo del precio |
| `StripePaymentService` | Proceso de pago |

**Antes:** un cambio en el proveedor de pago obligaba a modificar `TicketService`.  
**Después:** ese cambio solo afecta a `StripePaymentService`.


## Consecuencias

### Positivas
- `TicketService` queda enfocado en orquestar el flujo, lo que lo hace más claro y fácil de mantener.
- Cada componente (TicketValidator, PricingService, PaymentService) se puede probar de forma aislada con mocks simples.
- Cambios en la pasarela de pago no afectan la lógica principal; solo se modifica o agrega una implementación de PaymentService.

### Negativas / trade-offs
- Se incrementa el número de clases y dependencias, lo que puede ser excesivo en sistemas pequeños.
- El flujo de ejecución ya no está en un solo lugar, lo que puede dificultar la comprensión inicial.
