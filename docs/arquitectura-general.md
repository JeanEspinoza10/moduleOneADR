# Arquitectura General — Sistema de Biblioteca Digital (SBD)

---

## Descripción

API REST en Java (Spring Boot) para gestionar préstamos de libros: usuarios, catálogo, préstamos y notificaciones.

---

## Diagrama de Contexto

```
  ┌───────────────┐        ┌─────────────────────────┐       ┌─────────────┐
  │ Bibliotecario │ ───▶   │  Sistema de Biblioteca  │──────▶│ Servidor    │
  └───────────────┘        │     Digital (SBD)       │       │ de correo   │
                           │     API REST            │       └─────────────┘
  ┌───────────────┐        │     Spring Boot 3       │
  │     Usuario   │ ───▶   │     Puerto 8080         │
  └───────────────┘        └─────────────────────────┘
```

---

## Capas del sistema

```
┌────────────────────────────────────┐
│         Capa de API                │
│   Controladores REST · DTOs        │
├────────────────────────────────────┤
│       Capa de Aplicación           │
│   PrestamoService                  │
│   NotificacionService              │
│   MultaFactory                     │
├────────────────────────────────────┤
│         Capa de Dominio            │
│   Entidades: Prestamo, Usuario,    │
│   Libro · Interfaces: Repositorios │
│   EstrategiaMulta                  │
├────────────────────────────────────┤
│      Capa de Infraestructura       │
│   JpaPrestamoRepositorio           │
│   Implementaciones de multas       │
│   Configuración Spring / JPA       │
└────────────────────────────────────┘
```

**Regla de dependencias:** las capas superiores dependen de las inferiores **solo a través de interfaces**. La infraestructura nunca es importada por el dominio.

---

## Decisiones registradas

| ADR | Decisión | Principio | Impacto principal |
|-----|----------|-----------|-------------------|
| [ADR-001](../adr/ADR-001-srp-servicio-notificaciones.md) | Separar `NotificacionService` | SRP | Clases con una sola razón de cambio |
| [ADR-002](../adr/ADR-002-ocp-calculo-multas.md) | Patrón Strategy para multas | OCP | Nuevos tipos sin modificar código existente |
| [ADR-003](../adr/ADR-003-dip-repositorio-prestamos.md) | Interfaz `PrestamoRepositorio` | DIP | Tests unitarios sin base de datos |
| [ADR-004](../adr/ADR-004-lsp-tipos-de-libro.md) | Jerarquía de libros con interfaces | LSP | Sustitución segura sin excepciones inesperadas |
| [ADR-005](../adr/ADR-005-isp-gestion-usuarios.md) | Segregar interfaz de usuarios | ISP | Cada actor depende solo de lo que usa |
