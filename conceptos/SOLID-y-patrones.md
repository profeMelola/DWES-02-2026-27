# SOLID y patrones de diseño en Spring MVC

## 1. La idea en 30 segundos

Un programa que funciona no es suficiente: tiene que poder **cambiarse sin romperse**. Los principios SOLID y los patrones son reglas que ya han demostrado ayudar a conseguirlo.

En Spring MVC se resumen en tres hábitos:

1. **Cada clase hace una sola cosa** (controlador, servicio, repositorio).
2. **Dependes de interfaces, no de implementaciones** y dejas que Spring te inyecte lo que necesitas.
3. **Si añadir una función obliga a modificar código que ya funciona, algo está mal diseñado.**

> En los ejemplos se omiten los `import` y el `throws SQLException` para que el código se lea mejor.

## 2. Punto de partida: el proyecto `Productos`

```
es.daw.productos
├── controller   → ProductoController        (HTTP y vistas)
├── service      → ProductoService            (reglas de negocio)
├── repository   → ProductoRepository         (interfaz)
│                  ProductoRepositoryJdbc     (implementación)
├── model        → Producto
└── form         → ProductoForm               (datos del formulario)
```

```mermaid
flowchart LR
    N[Navegador] --> C[Controller]
    C --> V[Vista Thymeleaf]
    C --> S[Service]
    S --> R["«interface»<br/>ProductoRepository"]
    J[ProductoRepositoryJdbc] -.->|implementa| R
    A[ProductoRepositoryJPA] -.->|implementa| R
    M[ProductoRepositoryListaMemoria] -.->|implementa| R
```

Regla de oro: **las flechas van siempre hacia la derecha**. El controlador conoce al servicio, el servicio conoce a la interfaz del repositorio, y nadie conoce a la implementación.

## 3. Los cinco principios SOLID

### S · Single Responsibility (una sola responsabilidad)

**Regla:** una clase debe tener **un solo motivo para cambiar**.

**Señal de alarma:** un controlador que abre conexiones, escribe SQL y calcula precios.

❌ **Mal:** el controlador lo hace todo.

```java
@Controller
public class ProductoController {

    @GetMapping("/productos/{id}")
    public String ver(@PathVariable int id, Model model) throws SQLException {
        // 1) acceso a datos: abre conexión, SELECT, ResultSet...
        // 2) lógica de negocio: if (precio > 100) { descuento... }
        // 3) preparar la vista
        return "producto";
    }
}
```

✅ **Bien:** cada capa con su trabajo.

```java
@Controller
@RequestMapping("/productos")
public class ProductoController {

    private final ProductoService service;

    public ProductoController(ProductoService service) {
        this.service = service;
    }

    @GetMapping("/{id}")
    public String ver(@PathVariable int id, Model model) {
        model.addAttribute("producto", service.buscar(id));   // solo HTTP y vista
        return "producto";
    }
}

@Service
public class ProductoService {

    private final ProductoRepository repository;

    public ProductoService(ProductoRepository repository) {
        this.repository = repository;
    }

    public Producto buscar(int id) {                          // solo reglas de negocio
        return repository.findById(id)
                .orElseThrow(() -> new ProductoNoEncontradoException(id));
    }
}
```

> `ProductoNoEncontradoException` es una excepción propia que extiende `RuntimeException`.

**Prueba rápida para comprobar si lo estás haciendo mal:** ¿el servicio usa `Model`, `HttpServletRequest` o `ResultSet`? Entonces mezcla capas.

---

### O · Open/Closed (abierto a ampliar, cerrado a modificar)

**Regla:** para añadir una función nueva, **escribes una clase nueva** en lugar de editar una que ya funciona.

**Señal de alarma:** un `if / else if` que crece cada vez que aparece un caso nuevo.

❌ **Mal:** cada descuento nuevo obliga a tocar el método.

```java
public double precioFinal(double precio, String tipo) {
    if (tipo.equals("NAVIDAD"))          return precio * 0.90;
    else if (tipo.equals("BLACKFRIDAY")) return precio * 0.70;
    else                                 return precio;
}
```

✅ **Bien:** patrón **Strategy** con beans de Spring. Cada descuento es una clase y **sabe a qué tipo se aplica**.

```java
public interface Descuento {
    boolean soporta(String tipo);
    double aplicar(double precio);
}

@Component
public class DescuentoNavidad implements Descuento {
    public boolean soporta(String tipo) { return "NAVIDAD".equals(tipo); }
    public double aplicar(double precio) { return precio * 0.90; }
}

@Service
public class PrecioService {

    // Spring inyecta en la lista TODOS los beans que implementan Descuento
    private final List<Descuento> descuentos;

    public PrecioService(List<Descuento> descuentos) {
        this.descuentos = descuentos;
    }

    public double precioFinal(double precio, String tipo) {
        for (Descuento d : descuentos) {
            if (d.soporta(tipo)) {
                return d.aplicar(precio);
            }
        }
        return precio;   // ningún descuento aplica
    }
}
```

Para añadir `BLACKFRIDAY` creas `DescuentoBlackFriday` con `@Component` y **no tocas `PrecioService`**: Spring lo mete en la lista solo.

> **"¡Pero sigue habiendo un `if`!"** Sí, pero **nunca crece**. El `if / else if` del ejemplo malo tenía una rama por cada descuento; este es siempre el mismo, haya 2 descuentos o 20.---

---

### L · Liskov Substitution (sustitución)

**Regla:** si el servicio funciona con una implementación de la interfaz, **tiene que funcionar igual con cualquier otra**. Cambiar `ProductoRepositoryJdbc` por `ProductoRepositoryMemoria` no puede romper nada.

**Idea clave:** una interfaz no es solo una lista de métodos, es una **promesa**. Implementarla significa cumplir la promesa, no solo conseguir que compile.

> **Analogía:** cualquier cargador USB-C vale para tu móvil. Uno que encaja en el puerto pero no carga "implementa" USB-C... y rompe el contrato.

**Señal de alarma:** una implementación que devuelve `null`, lanza `UnsupportedOperationException` o se comporta distinto a las demás.

#### 1. La promesa (el contrato)

```java
public interface ProductoRepository {

    /** Devuelve el producto, o Optional.empty() si no existe. Nunca null. */
    Optional<Producto> findById(Integer id);

    /** Devuelve todos los productos, o una lista vacía si no hay. Nunca null. */
    List<Producto> findAll();
}
```

#### 2. El servicio confía en la promesa

```java
public Producto buscar(int id) {
    return repository.findById(id)
            .orElseThrow(() -> new ProductoNoEncontradoException(id));
}
```

El servicio no sabe qué implementación tiene debajo, y no debe saberlo.

#### 3. Una implementación que no la cumple

❌ **Mal:** compila, pero rompe la promesa.

```java
@Repository
public class ProductoRepositoryMemoria implements ProductoRepository {

    private final Map<Integer, Producto> datos = new ConcurrentHashMap<>();

    @Override
    public Optional<Producto> findById(Integer id) {
        if (!datos.containsKey(id)) {
            return null;                          // promete Optional y devuelve null
        }
        return Optional.of(datos.get(id));
    }
}
```

Mismo servicio, misma llamada `buscar(99)` con un id que no existe:

| Implementación | Resultado |
|---|---|
| `ProductoRepositoryJdbc` | `ProductoNoEncontradoException` → error controlado ✅ |
| `ProductoRepositoryMemoria` | `NullPointerException` en el **servicio** 💥 |

**El error salta en el servicio, pero el culpable es el repositorio.** Por eso estos fallos cuestan tanto de encontrar.

✅ **Bien:**

```java
@Override
public Optional<Producto> findById(Integer id) {
    return Optional.ofNullable(datos.get(id));    // si no existe: Optional.empty()
}
```

---

### I · Interface Segregation (interfaces pequeñas)

**Regla:** ninguna clase debe estar obligada a implementar métodos que **no usa**, ni a depender de ellos.

**Idea clave:** mejor varias interfaces pequeñas que una grande "para todo".

**Señal de alarma:** métodos que devuelven `0` o `null`, o lanzan `UnsupportedOperationException`, *"porque en esta clase no tiene sentido"*.

**Escenario:** la tienda vende libros en papel (se envían) y ebooks (se descargan).

❌ **Mal:** una interfaz que sirve para todo.

```java
public interface Articulo {
    String getNombre();
    double getPrecio();
    double getPesoKg();        // para calcular el envío
    String getUrlDescarga();   // para productos digitales
}

public class LibroPapel implements Articulo {
    /* nombre, precio, peso... */

    @Override
    public String getUrlDescarga() {
        throw new UnsupportedOperationException();   // un libro de papel no se descarga
    }
}

public class Ebook implements Articulo {
    /* nombre, precio, url... */

    @Override
    public double getPesoKg() {
        return 0;                                     // ¿pesa 0 kg? ¿y si alguien calcula su envío?
    }
}
```

Las dos clases están **obligadas a mentir** para que el código compile. Y son justo las dos primeras formas de romper Liskov.

✅ **Bien:** una interfaz por capacidad.

```java
public interface Articulo {
    String getNombre();
    double getPrecio();
}

public interface Enviable {
    double getPesoKg();
}

public interface Descargable {
    String getUrlDescarga();
}

public class LibroPapel     implements Articulo, Enviable              { /* ... */ }
public class Ebook          implements Articulo, Descargable           { /* ... */ }
public class PackLibroEbook implements Articulo, Enviable, Descargable { /* ... */ }
```

Cada servicio pide **solo lo que necesita**:

```java
@Service
public class EnvioService {
    public double calcularCoste(Enviable articulo) {
        return 3.0 + articulo.getPesoKg() * 1.5;
    }
}

@Service
public class DescargaService {
    public String generarEnlace(Descargable articulo) {
        return articulo.getUrlDescarga() + "?token=" + UUID.randomUUID();
    }
}
```

**Lo que ganas:** `envioService.calcularCoste(ebook)` **ya no compila**. El error lo detecta el compilador, no un cliente al que le cobran el envío de un PDF.

| | Interfaz grande | Interfaces pequeñas |
|---|---|---|
| Métodos "de relleno" | Sí (`throw`, `return 0`) | No |
| Calcular el envío de un ebook | Compila y da un resultado absurdo | No compila |
| Añadir un producto que es las dos cosas | Igual que los demás | `implements Enviable, Descargable` |

**Prueba rápida:** ¿hay algún método implementado con `return 0`, `return null` o `throw new UnsupportedOperationException()` solo para cumplir la interfaz? Entonces la interfaz es demasiado grande: divídela.
---

### D · Dependency Inversion (inversión de dependencias)

**Regla:** las clases de alto nivel (el servicio) dependen de **interfaces**, no de implementaciones concretas. Y no crean sus colaboradores con `new`: **se los dan**.

**Señal de alarma:** un `new` de un repositorio o un servicio dentro de otra clase.

❌ **Mal:** el servicio crea y conoce la implementación.

```java
@Service
public class ProductoService {
    private final ProductoRepository repository = new ProductoRepositoryJdbc();   // acoplado
}
```

✅ **Bien:** inyección por constructor de la **interfaz**.

```java
@Service
public class ProductoService {

    private final ProductoRepository repository;      // interfaz + final

    public ProductoService(ProductoRepository repository) {
        this.repository = repository;                 // Spring decide cuál inyectar
    }
}
```

**Lo que ganas:** cambiar de JDBC a memoria (o a JPA en la UT04) **sin tocar el servicio**.

```java
@Repository
@Profile("jdbc")
public class ProductoRepositoryJdbc implements ProductoRepository { /* ... */ }

@Repository
@Profile("memoria")
public class ProductoRepositoryMemoria implements ProductoRepository {
    private final Map<Integer, Producto> datos = new ConcurrentHashMap<>();
    /* ... */
}
```

```properties
# application.properties: cambias una línea y arranca con otra implementación
spring.profiles.active=memoria
```

> **Inyección por constructor, siempre.** Nada de `@Autowired` sobre atributos: IntelliJ te avisa con *Field injection is not recommended*. Cuando domines el constructor explícito puedes sustituirlo por `@RequiredArgsConstructor` de Lombok con atributos `final`.

> **¿Interfaz también para el servicio?** No por costumbre. Ponla cuando haya (o vaya a haber) varias implementaciones. En el repositorio sí, porque ahí cambiarás JDBC por JPA.

## 4. Patrones que usarás en la UT02

| Patrón | Problema que resuelve | En tu proyecto |
|---|---|---|
| **MVC** | Separar vista, lógica y datos | `@Controller` + Thymeleaf + `Model` |
| **Inyección de dependencias (IoC)** | No hacer `new` de colaboradores | Constructor + `@Service` / `@Repository` / `@Component` |
| **Repository** | Acceso a datos detrás de una interfaz | `ProductoRepository` |
| **Service** | Reglas de negocio en un solo sitio | `ProductoService` |
| **DTO / Form** | Mover datos a la vista sin exponer el modelo | `ProductoForm` |
| **Singleton** | Una sola instancia | Los beans de Spring lo son por defecto |
| **Strategy** | Cambiar un algoritmo sin `if / else` | `Descuento` + sus implementaciones |

### DTO / Form: datos del formulario

Según la convención de clase: **clase con Lombok para los formularios Thymeleaf** y **`record` para los DTO de una API REST**.

```java
@Data
public class ProductoForm {

    @NotBlank(message = "El nombre es obligatorio")
    private String nombre;

    @Positive(message = "El precio debe ser mayor que 0")
    private double precio;
}
```

```java
@PostMapping
public String crear(@Valid @ModelAttribute ProductoForm form, BindingResult errores) {
    if (errores.hasErrors()) {
        return "producto-form";                 // vuelve al formulario con los errores
    }
    service.crear(form);                        // el servicio convierte el form en Producto
    return "redirect:/productos";
}
```

**Por qué no usar `Producto` directamente en el formulario:** el formulario puede incluir campos que el modelo no debe recibir del navegador (por ejemplo, el `id`).

### Singleton: cuidado con el estado

Spring crea **una sola instancia** de cada `@Service` y la comparten **todos los usuarios a la vez**.

❌ **Mal:**

```java
@Service
public class ProductoService {
    private Producto ultimoVisto;   // lo ven y lo pisan todos los usuarios
}
```

✅ **Bien:** un servicio **no guarda datos de un usuario**. Si hay que recordar algo de cada usuario, va en la sesión (se verá con RA4) o se pasa como parámetro.

---

## 5. Chuleta de IntelliJ (Windows y Linux)

| Quiero... | Atajo |
|---|---|
| Generar constructor, getters... | `Alt + Insert` |
| Menú de refactorización (*Extract Interface*, *Extract Method*...) | `Ctrl + Alt + Shift + T` |
| Extraer un método | `Ctrl + Alt + M` |
| Renombrar | `Shift + F6` |
| Mover una clase a otro paquete | `F6` |
| Ir a las implementaciones de una interfaz | `Ctrl + Alt + B` |
| Ver quién usa esta clase | `Alt + F7` |
| Diagrama de clases (Ultimate) | `Ctrl + Alt + Shift + U` |
| Cambiar de perfil | *Run → Edit Configurations → Active profiles* |

**Flujo típico para separar capas:** sobre una clase `ProductoRepositoryJdbc`, `Ctrl + Alt + Shift + T` → *Extract Interface* → marcas los métodos y obtienes `ProductoRepository`.



