# ADR-003 — Abstraer la persistencia de compras de entradas a conciertos mediante interfaz

**Fecha:** 2025-04-15
**Estado:** ✅ Aceptado
**Principio SOLID:** D — Dependency Inversion Principle (DIP)

---

## Contexto

`PrestamoService` depende directamente de `PrestamoRepository`, una interfaz de Spring Data JPA. Aunque esto parece inofensivo, la dependencia sobre JPA (una tecnología de infraestructura) está acoplada a la capa de negocio:

**Código actual (con el problema):**

```java
// PrestamoService.java — depende directamente de la infraestructura JPA
@Service
public class PrestamoService {

    // ❌ Spring Data JPA es un detalle de infraestructura
    private final PrestamoRepository repository; // extiende JpaRepository<Prestamo, Long>

    public PrestamoService(PrestamoRepository repository) {
        this.repository = repository;
    }

    public Prestamo registrarPrestamo(Usuario usuario, Libro libro) {
        Prestamo prestamo = new Prestamo(usuario, libro, LocalDate.now().plusDays(14));
        return repository.save(prestamo); // método de JpaRepository
    }

    public List<Prestamo> obtenerPrestamosActivos(Long usuarioId) {
        // ❌ Llama a un método derivado del nombre del campo JPA
        return repository.findByUsuarioIdAndDevueltoFalse(usuarioId);
    }
}
```

**¿Cuál es el problema?**

`PrestamoService` (lógica de negocio, alto nivel) depende de `PrestamoRepository` que extiende `JpaRepository` (infraestructura, bajo nivel). Esto significa:

- Para probar `PrestamoService` en una prueba unitaria, necesitamos levantar el contexto de Spring y una base de datos en memoria (H2), lo cual hace las pruebas lentas y frágiles.
- Si se decide cambiar la tecnología de persistencia (por ejemplo, a MongoDB o a un servicio REST externo), hay que modificar `PrestamoService`.
- La dirección de la dependencia es incorrecta: el negocio depende de la infraestructura, cuando debería ser al revés.

---

## Decisión

Definimos una interfaz propia `PrestamoRepositorio` en la capa de dominio, con solo los métodos que el negocio necesita. `PrestamoService` depende de esa interfaz. La implementación con JPA (`JpaPrestamoRepositorio`) vive en la capa de infraestructura y es quien depende del detalle tecnológico.

**Código corregido:**

```java
// PrestamoRepositorio.java — interfaz propia en la capa de dominio
// No importa nada de Spring ni de JPA
public interface PrestamoRepositorio {

    Prestamo guardar(Prestamo prestamo);

    Optional<Prestamo> buscarPorId(Long id);

    List<Prestamo> listarActivosPorUsuario(Long usuarioId);
}


// PrestamoService.java — depende solo de la abstracción del dominio
@Service
public class PrestamoService {

    private final PrestamoRepositorio repositorio; // ✅ interfaz propia, no JPA
    private final NotificacionService notificaciones;

    public PrestamoService(PrestamoRepositorio repositorio,
                           NotificacionService notificaciones) {
        this.repositorio = repositorio;
        this.notificaciones = notificaciones;
    }

    public Prestamo registrarPrestamo(Usuario usuario, Libro libro) {
        Prestamo prestamo = new Prestamo(usuario, libro, LocalDate.now().plusDays(14));
        Prestamo guardado = repositorio.guardar(prestamo); // interfaz, no JPA
        notificaciones.notificarPrestamo(usuario, libro);
        return guardado;
    }

    public List<Prestamo> obtenerPrestamosActivos(Long usuarioId) {
        return repositorio.listarActivosPorUsuario(usuarioId);
    }
}


// --- CAPA DE INFRAESTRUCTURA ---

// SpringDataPrestamoRepo.java — repositorio interno de Spring Data (detalle)
public interface SpringDataPrestamoRepo extends JpaRepository<Prestamo, Long> {
    List<Prestamo> findByUsuarioIdAndDevueltoFalse(Long usuarioId);
}


// JpaPrestamoRepositorio.java — adaptador que conecta la interfaz del dominio con JPA
@Repository
public class JpaPrestamoRepositorio implements PrestamoRepositorio {

    private final SpringDataPrestamoRepo jpaRepo;

    public JpaPrestamoRepositorio(SpringDataPrestamoRepo jpaRepo) {
        this.jpaRepo = jpaRepo;
    }

    @Override
    public Prestamo guardar(Prestamo prestamo) {
        return jpaRepo.save(prestamo);
    }

    @Override
    public Optional<Prestamo> buscarPorId(Long id) {
        return jpaRepo.findById(id);
    }

    @Override
    public List<Prestamo> listarActivosPorUsuario(Long usuarioId) {
        return jpaRepo.findByUsuarioIdAndDevueltoFalse(usuarioId);
    }
}


// --- PRUEBAS UNITARIAS ---

// InMemoryPrestamoRepositorio.java — implementación en memoria para tests
// No necesita Spring, ni H2, ni ninguna base de datos
public class InMemoryPrestamoRepositorio implements PrestamoRepositorio {

    private final Map<Long, Prestamo> almacen = new HashMap<>();
    private long nextId = 1;

    @Override
    public Prestamo guardar(Prestamo prestamo) {
        prestamo.setId(nextId++);
        almacen.put(prestamo.getId(), prestamo);
        return prestamo;
    }

    @Override
    public Optional<Prestamo> buscarPorId(Long id) {
        return Optional.ofNullable(almacen.get(id));
    }

    @Override
    public List<Prestamo> listarActivosPorUsuario(Long usuarioId) {
        return almacen.values().stream()
            .filter(p -> p.getUsuarioId().equals(usuarioId) && !p.isDevuelto())
            .collect(Collectors.toList());
    }
}


// PrestamoServiceTest.java — prueba unitaria sin Spring ni base de datos
class PrestamoServiceTest {

    private PrestamoService servicio;
    private InMemoryPrestamoRepositorio repositorio;

    @BeforeEach
    void setUp() {
        repositorio = new InMemoryPrestamoRepositorio();
        NotificacionService notificaciones = mock(NotificacionService.class);
        servicio = new PrestamoService(repositorio, notificaciones);
    }

    @Test
    void registrarPrestamo_debeGuardarYAsignarId() {
        Usuario usuario = new Usuario(1L, "Ana López", "ana@mail.com", "ESTUDIANTE");
        Libro libro = new Libro(10L, "Clean Code", "Robert C. Martin");

        Prestamo resultado = servicio.registrarPrestamo(usuario, libro);

        assertNotNull(resultado.getId());
        assertEquals(1, repositorio.listarActivosPorUsuario(1L).size());
    }
}
```

### Principio SOLID aplicado — DIP

> "Los módulos de alto nivel no deben depender de módulos de bajo nivel. Ambos deben depender de abstracciones."

**Dirección de dependencias:**

```
ANTES (incorrecta):
  PrestamoService  ──→  JpaRepository  (infraestructura)

DESPUÉS (correcta):
  PrestamoService         ──→  PrestamoRepositorio  (interfaz del dominio)
  JpaPrestamoRepositorio  ──→  PrestamoRepositorio  (interfaz del dominio)
```

Ambos módulos —el servicio y el repositorio JPA— apuntan hacia la **abstracción**. La infraestructura se adapta al contrato del dominio, no al revés.

### Alternativas consideradas

| Alternativa | Por qué se descartó |
|-------------|---------------------|
| Mockear `JpaRepository` directamente en los tests con Mockito | Funciona como parche, pero el acoplamiento permanece. Un cambio en la API de JPA puede romper los tests aunque el negocio no haya cambiado |
| Usar `@DataJpaTest` para todos los tests de servicio | Hace las pruebas de integración, no unitarias. Son más lentas y requieren más configuración |

---

## Consecuencias

### Positivas
- Los tests unitarios de `PrestamoService` no necesitan Spring ni base de datos: se ejecutan en milisegundos con `InMemoryPrestamoRepositorio`.
- Cambiar de MySQL a MongoDB requiere solo escribir `MongoPrestamoRepositorio implements PrestamoRepositorio`, sin tocar `PrestamoService`.
- `PrestamoService` pertenece limpiamente a la capa de dominio: no importa ninguna dependencia de infraestructura.

### Negativas / trade-offs
- Se añaden dos clases por cada entidad con repositorio: la interfaz del dominio y el adaptador JPA. Para sistemas pequeños puede parecer boilerplate.
- El equipo debe entender el patrón antes de contribuir; puede requerir una sesión de onboarding.

---

## Diagrama de capas resultante

```
┌──────────────────────────────────────────────┐
│             Capa de Dominio                  │
│                                              │
│   PrestamoService                            │
│        │                                     │
│        └──→  PrestamoRepositorio  (interfaz) │
└─────────────────────┬────────────────────────┘
                      │ implementa
┌─────────────────────▼────────────────────────┐
│          Capa de Infraestructura             │
│                                              │
│   JpaPrestamoRepositorio                     │
│   InMemoryPrestamoRepositorio  (tests)       │
└──────────────────────────────────────────────┘
```
