# ADR-003 — Abstraer la persistencia de compras de entradas a entradas mediante interfaz

**Fecha:** 2025-04-15
**Estado:** ✅ Aceptado
**Principio SOLID:** D — Dependency Inversion Principle (DIP)

---

## Contexto

El servicio `CompraEntradaService` presenta una dependencia directa hacia `CompraEntradaRepository`, el cual corresponde a una interfaz de Spring Data JPA.
Si bien esta relación puede parecer adecuada en un primer análisis, en realidad introduce un acoplamiento explícito entre la capa de negocio y una tecnología de infraestructura (JPA).

Este tipo de dependencia compromete principios de diseño como la separación de responsabilidades y la independencia de la lógica de negocio, dificultando la evolución del sistema y limitando la posibilidad de sustituir la tecnología de persistencia en el futuro sin afectar la capa de servicio.

## Riesgos identificados
 - Acoplamiento tecnológico: La lógica de negocio queda vinculada a una implementación específica de persistencia.

 - Reducción de flexibilidad: Se limita la capacidad de reemplazar JPA por otra solución (ej. JDBC, MongoDB, servicios externos).

 - Impacto en pruebas: Las pruebas unitarias de la capa de negocio dependen de una infraestructura concreta, lo que complica la simulación o aislamiento.

**Código actual (con el problema):**

```java
// PrestamoService.java — depende directamente de la infraestructura JPA
@Service
public class CompraEntradaService {

    // ❌ Spring Data JPA es un detalle de infraestructura
    private final CompraEntradaRepository repository; // extiende JpaRepository<Prestamo, Long>

    public CompraEntradaService(CompraEntradaRepository repository) {
        this.repository = repository;
    }

    public Prestamo registrarCompra(Usuario usuario, Entrada entrada) {
        Compra compra = new Compra(usuario, entrada, LocalDate.now().plusDays(14));
        return repository.save(compra); // método de JpaRepository
    }

    public List<Compra> obtenerCompraActivos(Long usuarioId) {
        // ❌ Llama a un método derivado del nombre del campo JPA
        return repository.findByUsuarioIdAndDevueltoFalse(usuarioId);
    }
}
```

**¿Cuál es el problema?**

El servicio `CompraEntradaService` (capa de negocio, alto nivel) depende directamente de `CompraEntradaRepository`, el cual extiende `JpaRepository` (capa de infraestructura, bajo nivel). Este diseño genera las siguientes implicaciones:

 - Pruebas unitarias comprometidas: Para validar `CompraEntradaService` es necesario levantar el contexto de Spring y una base de datos en memoria (H2). Esto incrementa la complejidad, ralentiza la ejecución y vuelve las pruebas más frágiles.

 - Rigidez tecnológica: Un cambio en la tecnología de persistencia (por ejemplo, migrar a MongoDB o integrar un servicio REST externo) obligaría a modificar la lógica de negocio en `CompraEntradaService`.

 - Dirección de dependencia incorrecta: La capa de negocio queda subordinada a la infraestructura, cuando la relación debería ser inversa. El negocio debe definir sus contratos y la infraestructura adaptarse a ellos.

---

## Decisión

Para mejorar la arquitectura, se sugiere introducir una abstracción de repositorio en la capa de dominio. De esta forma:

 - `CompraEntradaService` dependerá de una interfaz propia del dominio.

 - La implementación concreta (`JpaRepository`, MongoDB, REST, etc.) quedará encapsulada en la capa de infraestructura.

 - Se logra mayor independencia tecnológica, facilidad de pruebas unitarias y alineación con principios de arquitectura limpia.

**Código corregido:**

```java
// CompraEntradaRepository.java — interfaz propia en la capa de dominio
// No importa nada de Spring ni de JPA
public interface CompraEntradaRepository {

    Prestamo guardar(Compra compra);

    Optional<Compra> buscarPorId(Long id);

    List<Compra> listarActivosPorUsuario(Long usuarioId);
}


// CompraEntradaService.java — depende solo de la abstracción del dominio
@Service
public class CompraEntradaService {

    private final CompraEntradaRepository repositorio; // ✅ interfaz propia, no JPA
    private final NotificacionService notificaciones;

    public CompraEntradaService(CompraEntradaRepository repositorio,
                           NotificacionService notificaciones) {
        this.repositorio = repositorio;
        this.notificaciones = notificaciones;
    }

    public Compra registrarCompra(Usuario usuario, Entrada entrada) {
        Compra compra = new Compra(usuario, entrada, LocalDate.now().plusDays(14));
        Compra guardado = repositorio.guardar(compra); // interfaz, no JPA
        notificaciones.notificarPrestamo(usuario, entrada);
        return guardado;
    }

    public List<Compra> obtenerComprasActivos(Long usuarioId) {
        return repositorio.listarActivosPorUsuario(usuarioId);
    }
}


// --- CAPA DE INFRAESTRUCTURA ---

// SpringDataCompraRepo.java — repositorio interno de Spring Data (detalle)
public interface SpringDataCompraRepo extends JpaRepository<Compra, Long> {
    List<Compra> findByUsuarioIdAndDevueltoFalse(Long usuarioId);
}


// JpaCompraEntradoRepositorio.java — adaptador que conecta la interfaz del dominio con JPA
@Repository
public class JpaCompraEntradoRepositorio implements CompraEntradaRepository {

    private final SpringDataCompraRepo jpaRepo;

    public JpaCompraRepositorio(SpringDataCompraRepo jpaRepo) {
        this.jpaRepo = jpaRepo;
    }

    @Override
    public Compra guardar(Compra compra) {
        return jpaRepo.save(compra);
    }

    @Override
    public Optional<Compra> buscarPorId(Long id) {
        return jpaRepo.findById(id);
    }

    @Override
    public List<Compra> listarActivosPorUsuario(Long usuarioId) {
        return jpaRepo.findByUsuarioIdAndDevueltoFalse(usuarioId);
    }
}


// --- PRUEBAS UNITARIAS ---

// InMemoryPrestamoRepositorio.java — implementación en memoria para tests
// No necesita Spring, ni H2, ni ninguna base de datos
public class InMemoryCompraEntradaRepositorio implements CompraEntradaRepository {

    private final Map<Long, Compra> almacen = new HashMap<>();
    private long nextId = 1;

    @Override
    public Compra guardar(Compra compra) {
        compra.setId(nextId++);
        almacen.put(compra.getId(), compra);
        return compra;
    }

    @Override
    public Optional<Compra> buscarPorId(Long id) {
        return Optional.ofNullable(almacen.get(id));
    }

    @Override
    public List<Compra> listarActivosPorUsuario(Long usuarioId) {
        return almacen.values().stream()
            .filter(p -> p.getUsuarioId().equals(usuarioId) && !p.isDevuelto())
            .collect(Collectors.toList());
    }
}


// CompraEntradaServiceTest.java — prueba unitaria sin Spring ni base de datos
class CompraEntradaServiceTest {

    private CompraEntradaService servicio;
    private InMemoryCompraEntradaRepositorio repositorio;

    @BeforeEach
    void setUp() {
        repositorio = new InMemoryCompraEntradaRepositorio();
        NotificacionService notificaciones = mock(NotificacionService.class);
        servicio = new CompraEntradaService(repositorio, notificaciones);
    }

    @Test
    void registrarCompraEntrad_debeGuardarYAsignarId() {
        Usuario usuario = new Usuario(1L, "Ana López", "ana@mail.com", "ESTUDIANTE");
        Entrad entrada = new Entrada(10L, "Clean Code", "Robert C. Martin");

        Prestamo resultado = servicio.registrarCompra(usuario, entrada);

        assertNotNull(resultado.getId());
        assertEquals(1, repositorio.listarActivosPorUsuario(1L).size());
    }
}
```

### Principio SOLID aplicado — DIP

> "El Principio de Inversión de Dependencias (DIP) dentro de SOLID establece que los módulos de alto nivel no deben depender de módulos de bajo nivel, sino de abstracciones. Esto permite construir sistemas desacoplados, flexibles y fáciles de mantener."

**Dirección de dependencias:**

```
ANTES (incorrecta):
  CompraEntradaService ===> JpaRepository  (infraestructura)

DESPUÉS (correcta):
  CompraEntradaService         ===> CompraEntradaRepositorio  (interfaz del dominio)
  JpaCompraEntradaRepositorio  ===> CompraEntradaRepositorio  (interfaz del dominio)
```

Tanto el servicio como el repositorio JPA dependen de una **abstracción** común. La capa de infraestructura se ajusta al contrato definido por el dominio, en lugar de imponerle sus propios detalles.

### Alternativas consideradas

| Alternativa | Por qué se descartó |
|-------------|---------------------|
| Mockear `JpaRepository` directamente en los tests con Mockito | Funciona como parche, pero el acoplamiento permanece. Un cambio en la API de JPA puede romper los tests aunque el negocio no haya cambiado |
| Usar `@DataJpaTest` para todos los tests de servicio | Hace las pruebas de integración, no unitarias. Son más lentas y requieren más configuración |

---

## Consecuencias

### Positivas
- Los tests unitarios de `CompraEntradaService` no necesitan Spring ni base de datos: se ejecutan en milisegundos con `InMemoryCompraEntradaRepositorio`.
- Cambiar de MySQL a MongoDB requiere solo escribir `MongoCompraEntradaRepositorio implements CompraEntradaRepositorio`, sin tocar `CompraEntradaService`.
- `CompraEntradaService` pertenece limpiamente a la capa de dominio: no importa ninguna dependencia de infraestructura.

### Negativas / trade-offs
- Se añaden dos clases por cada entidad con repositorio: la interfaz del dominio y el adaptador JPA. Para sistemas pequeños puede parecer boilerplate.
- El equipo debe entender el patrón antes de contribuir; puede requerir una sesión de onboarding.

---

## Diagrama de capas resultante

```
┌───────────────────────────────────────────────────┐
│             Capa de Dominio                       │
│                                                   │
│   CompraEntradaService                            │
│        │                                          │
│        └──→  CompraEntradaRepositorio  (interfaz) │
└─────────────────────┬─────────────────────────────┘
                      │ implementa
┌─────────────────────▼─────────────────────────────┐
│          Capa de Infraestructura                  │
│                                                   │
│   JpaCompraEntradaRepositorio                     │
│   InMemoryCompraEntradaRepositorio  (tests)       │
└───────────────────────────────────────────────────┘
```