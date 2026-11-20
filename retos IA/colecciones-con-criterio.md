# Colecciones con criterio: no todo es un `ArrayList`

> **Idea clave:** en nuestras aplicaciones los datos viven en la base de datos, y **buscar, filtrar, ordenar y contar se hace con SQL**.
> Las colecciones de Java organizan lo que ya está **en memoria**: el resultado de una consulta, el carrito de la sesión, los roles de un usuario.
> Saber qué garantiza cada colección te permite revisar con criterio el código que te da la IA.

**Dos notaciones que aparecen en este documento:**

| Notación | Significa |
|---|---|
| **O(1)** | Tarda lo mismo con 10 datos que con un millón |
| **O(n)** | Si duplicas los datos, duplicas el tiempo (recorre los elementos uno a uno) |

---

## 1. Antes de elegir colección: ¿esto lo tiene que hacer la base de datos?

| Si en Java haces… | Lo correcto es… |
|---|---|
| `findAll()` y recorrer la lista buscando un ISBN | `WHERE isbn = ?` (`findByIsbn`) |
| `findAll().stream().filter(...)` | `WHERE ...` |
| `lista.sort(...)` sobre lo que devuelve el repositorio | `ORDER BY ...` |
| Recorrer la lista para contar pedidos por provincia | `GROUP BY provincia` |

La base de datos está hecha para esto. Si traes la tabla entera a memoria para filtrarla en Java, con 10 filas no lo notas, pero con 100.000 la página se vuelve lenta.

> 🤖 **Es un error muy típico de la IA:** `findAll()` seguido de `stream().filter(...)`. Funciona, y por eso cuesta detectarlo.

---

## 2. Estructuras lineales

### 2.1 Arrays y listas

Colecciones ordenadas con acceso por posición. Un array tiene tamaño fijo; una `List` puede crecer.

**Lo que devuelve un repositorio es una `List`, y está bien que lo sea:**

```java
List<Libro> findAll();   // SELECT ... ORDER BY titulo
```

- El **orden** lo pone la consulta (`ORDER BY`), y una `List` lo mantiene.
- **No hay duplicados** porque la clave primaria de la tabla ya lo impide. No necesitas un `Set`.

### 2.2 Pilas (LIFO)

*Last In, First Out*: el último que entra es el primero que sale, como una pila de platos.

| Operación | Qué hace |
|---|---|
| `push(x)` | Añade arriba |
| `pop()` | Quita el de arriba |
| `peek()` | Mira el de arriba sin quitarlo |

**Dónde la ves: la traza de un error.** Cada vez que un método llama a otro, Java apila esa llamada. Cuando salta una excepción, lo que ves en consola **es esa pila**:

```
java.lang.NullPointerException: ...
    at com.biblioteca.service.LibroService.guardar(LibroService.java:42)        ← cima: aquí falló
    at com.biblioteca.controller.LibroController.guardar(LibroController.java:58)
    at ...                                                                       ← llamadas de Spring
```

Por eso se lee **de arriba abajo**: arriba está la última llamada, la que falló. Busca la primera línea que sea de tu código.

Si un método se llama a sí mismo sin parar, la pila se llena y salta `StackOverflowError` ("desbordamiento de pila").

**En Java:** `Deque<String> pila = new ArrayDeque<>();`. No uses la clase `Stack`: es de las primeras versiones de Java y la propia documentación recomienda `ArrayDeque`.

### 2.3 Colas (FIFO)

*First In, First Out*: el primero que entra es el primero que sale, como la cola del supermercado.

| Operación | En Java | Qué hace |
|---|---|---|
| Encolar | `offer(x)` | Añade al final |
| Desencolar | `poll()` | Saca el primero |

**Dónde la ves:** cuando el servidor está ocupado, las peticiones nuevas esperan en una cola y se atienden por orden de llegada. También son colas las tareas que se procesan en orden, como los correos pendientes de enviar.

**En Java:** `Queue<Email> cola = new ArrayDeque<>();`.

> ¿Y "los pedidos urgentes primero"? Eso no es una cola: es `ORDER BY urgente DESC, fecha`.

---

## 3. Estructuras asociativas

### 3.1 Mapas: clave → valor

Guardan pares clave-valor. Buscar por clave es O(1): vas directo, sin recorrer nada. La clave tiene que ser **única y no cambiar**: un id, un ISBN, un código de pedido.

**Spring ya los usa:** `model.addAttribute("libros", libros)` guarda un par nombre → objeto para la vista, y la sesión funciona igual.

**Caso práctico: el carrito de la compra en la sesión.** Todavía no está en la BD: se guarda en memoria hasta que el usuario confirma el pedido.

Con una `List`, cada vez que se añade un producto hay que recorrer la lista para ver si ya estaba:

```java
// ❌ Con List: recorrer para buscar
for (LineaCarrito linea : lineas) {
    if (linea.getProductoId().equals(productoId)) {
        linea.setCantidad(linea.getCantidad() + cantidad);
        return;
    }
}
lineas.add(new LineaCarrito(productoId, cantidad));
```

Con un `Map`, el producto es la clave y la cantidad el valor:

```java
// ✅ Con Map: producto → cantidad
Map<Long, Integer> lineas = new LinkedHashMap<>();   // Linked: mantiene el orden en que se añadieron

lineas.merge(productoId, cantidad, Integer::sum);    // si no está, lo añade; si está, suma la cantidad
lineas.remove(productoId);                           // quitar un producto
```

### 3.2 Conjuntos (Sets)

Colecciones de elementos **sin duplicados**. Comprobar si un elemento está es O(1).

**Caso práctico: los roles de un usuario.** Un usuario no puede tener dos veces el rol `ADMIN`. Ahora cada usuario tiene un solo rol, pero cuando pueda tener varios, serán un `Set`:

```java
Set<String> roles = new HashSet<>();
roles.add("ADMIN");
roles.add("ADMIN");        // no hace nada: ya estaba
roles.contains("ADMIN");   // true
```

**Las operaciones de conjuntos también sirven.** Al guardar un formulario para editar los roles de un usuario, ¿qué roles hay que añadir y cuáles quitar?

```java
Set<String> actuales = Set.of("USER", "EDITOR");   // lo que hay en la BD
Set<String> nuevos   = Set.of("USER", "ADMIN");    // lo que llega del formulario

Set<String> aAnadir = new HashSet<>(nuevos);
aAnadir.removeAll(actuales);   // [ADMIN]

Set<String> aQuitar = new HashSet<>(actuales);
aQuitar.removeAll(nuevos);     // [EDITOR]
```

**¿Y el orden?** Un `HashSet` no guarda ningún orden. Si lo necesitas, `LinkedHashSet` mantiene el orden en que se añadieron los elementos. Pero si el orden importa, normalmente lo que quieres es una `List`.

---

## 4. `equals()` y `hashCode()`

### 4.1 Para qué sirven

Cuando guardas objetos tuyos en un `Set`, los usas como clave de un `Map` o llamas a `contains()`, Java necesita saber cuándo dos objetos son **el mismo**.

Imagina una biblioteca con 10 estanterías. La regla es: *el libro va a la estantería que indica el último dígito de su ISBN*.

- ¿Buscas el ISBN `978-84-376-0494-7`? Vas directo a la **estantería 7** sin mirar las demás.
- En la estantería 7 puede haber varios libros. Ahí sí lees el ISBN completo para encontrar el tuyo.

Un `HashSet` o un `HashMap` hacen exactamente esto con dos métodos:

| Método | Qué hace | En la biblioteca |
|---|---|---|
| `hashCode()` | Calcula en qué "estantería" mirar | El último dígito del ISBN |
| `equals()` | Comprueba si es el objeto que buscas | Leer el ISBN completo |

Por eso hacen falta **los dos**, y los dos tienen que usar **el mismo campo**.

### 4.2 Qué pasa si no los pones

Java compara si son *el mismo objeto en memoria*, no si tienen el mismo ISBN:

```java
Libro a = new Libro("978-84-376-0494-7", "Cien años de soledad");
Libro b = new Libro("978-84-376-0494-7", "Cien años de soledad");

Set<Libro> libros = new HashSet<>();
libros.add(a);
libros.contains(b);   // false ❌ sin equals/hashCode
                      // true  ✅ con equals/hashCode por ISBN
```

### 4.3 Qué campo usar: la clave de negocio

El campo que identifica al objeto de forma única y no cambia: el ISBN de un libro, el código de un pedido, el `username` de un usuario.

```java
@EqualsAndHashCode(onlyExplicitlyIncluded = true)
public class Pedido {
    @EqualsAndHashCode.Include
    private String codigo;      // solo este campo decide si dos pedidos son el mismo
    private LocalDate fecha;
    private String estado;
}
```

### 4.4 Cuidado con `@Data`

`@Data` genera `equals()` y `hashCode()` con **todos** los campos. Entonces dos objetos del mismo pedido (mismo código) con distinto estado contarían como pedidos distintos: el `Set` tendría el mismo pedido dos veces.

---

## 5. Cuando lleguemos a JPA: ¿`List` o `Set` en las relaciones?

| Relación | Usa | Ejemplo |
|---|---|---|
| Muchos a muchos (`@ManyToMany`) | **`Set`** | Usuario ↔ roles |
| Uno a muchos (`@OneToMany`) | **`List`** | Autor → libros |

```java
@ManyToMany
private Set<Rol> roles = new HashSet<>();

@OneToMany(mappedBy = "autor")
private List<Libro> libros = new ArrayList<>();
```

**¿Por qué `Set` en muchos a muchos?** Los roles de un usuario se guardan en una tabla intermedia (`usuario_rol`). Si quitas un rol a un usuario que tiene tres:

- Con **`List`**, JPA no sabe qué fila corresponde a cada elemento: **borra las tres filas y vuelve a insertar las dos** que quedan.
- Con **`Set`**, sabe que cada rol es único: **borra solo esa fila**.

Además, un `Set` impide asignar dos veces el mismo rol.

**¿Y en uno a muchos?** Cada libro es una fila de su propia tabla, así que JPA lo borra o actualiza uno a uno. Una `List` funciona bien.

**Si usas `Set`, `equals()` y `hashCode()` por la clave de negocio (apartado 4.3), nunca por el `id`.** El `id` lo genera la base de datos: vale `null` hasta que guardas el objeto, y después cambia. Si metiste el objeto en un `HashSet` antes de guardarlo, al cambiar el `id` queda en la "estantería" equivocada y `contains()` ya no lo encuentra.

---

## 6. Resumen

| Estructura | Garantiza | En nuestras aplicaciones |
|---|---|---|
| `List` | Orden y posición | Resultado de una consulta; uno a muchos en JPA |
| Pila (`ArrayDeque`) | Sale el último que entró | Traza de un error |
| Cola (`ArrayDeque`) | Sale el primero que entró | Peticiones o tareas en espera |
| `Map` | Claves únicas, búsqueda directa | Carrito, modelo de la vista |
| `Set` | Elementos únicos | Roles; muchos a muchos en JPA |

---

## 7. Usar la IA con criterio

### 7.1 Checklist para revisar el código que te da la IA

1. **¿Trae datos de la BD para filtrarlos, ordenarlos o contarlos en Java?** Eso es un `WHERE`, un `ORDER BY` o un `GROUP BY`.
2. **¿Recorre una `List` para buscar por clave o para evitar repetidos?** Piensa en `Map` o `Set`.
3. **¿Mete objetos tuyos en un `Set` o como clave de un `Map`?** ¿Tienen `equals()` y `hashCode()` por la clave de negocio?
4. **¿Usa `@Data` donde importa la igualdad?**
5. **¿Usa clases antiguas?**

| Si la IA propone… | Hoy se usa… |
|---|---|
| `Stack` | `ArrayDeque` |
| `Vector` | `ArrayList` |
| `Hashtable` | `HashMap` |

### 7.2 Pregunta mal vs. pregunta bien

| ❌ Prompt pobre | ✅ Prompt con contexto |
|---|---|
| "Haz un carrito de la compra" | "Haz un carrito guardado en la sesión. Cada producto aparece una vez con su cantidad, en el orden en que se añadió. ¿Qué colección uso y por qué?" |
| "Muestra los pedidos urgentes primero" | "Añade al repositorio JDBC un método que devuelva los pedidos pendientes, los urgentes primero y después por fecha, ordenados en la consulta SQL." |

El buen prompt ya dice **dónde** se hace cada cosa: en la BD o en memoria.

---

## 8. Practica: ¿colección de Java o SQL?

| | Situación |
|---|---|
| A | Mostrar los libros de un autor |
| B | Los roles de un usuario que puede tener varios |
| C | El carrito de la compra antes de confirmar el pedido |
| D | Mostrar los pedidos con los urgentes arriba |
| E | Salta una excepción: ¿qué línea de la traza miras primero? |
| F | ¿Por qué campo haces `equals()` y `hashCode()` en `Usuario`? |

<details>
<summary>Soluciones</summary>

| | Respuesta | Por qué |
|---|---|---|
| A | SQL: `WHERE autor_id = ?` → `List<Libro>` | Filtrar es trabajo de la BD |
| B | `Set` | No puede haber roles repetidos |
| C | `Map<Long, Integer>` | Búsqueda por producto; todavía no está en la BD |
| D | SQL: `ORDER BY urgente DESC, fecha` | Ordenar es trabajo de la BD |
| E | La de arriba (la primera de tu código) | La traza es una pila: arriba está la última llamada |
| F | `username` | Es único y no cambia; el `id` es `null` hasta que se guarda |

</details>

---

## Resumen en tres frases

1. **Buscar, filtrar, ordenar y contar se hace en la base de datos**, no trayendo la tabla a memoria.
2. **`List` para resultados, `Map` para buscar por clave, `Set` para elementos únicos**, y `equals()`/`hashCode()` siempre por la clave de negocio.
3. **La IA da código que funciona; tú decides si sirve.** Para eso te basta con hacerte las preguntas de la checklist.
