# Colecciones con criterio: no todo es un `ArrayList`

> **Idea clave:** en nuestras aplicaciones los datos viven en la base de datos, y **buscar, filtrar, ordenar y contar se hace con SQL**.
> Las colecciones de Java organizan lo que ya está **en memoria**: el resultado de una consulta, el carrito de la sesión, los roles de un usuario, los datos de un formulario.
> Elegir bien la colección es saber **qué garantiza cada una**.

> ⚠️ Las referencias a la notación Big O, como O(1) u O(n), se explican en el reto **Algoritmos y complejidad**.

---

## 1. Antes de elegir colección: ¿esto lo tiene que hacer la base de datos?

| Si en Java haces… | Lo correcto es… |
|---|---|
| `findAll()` y recorrer la lista buscando un ISBN | `WHERE isbn = ?` (`findByIsbn`) |
| `findAll().stream().filter(...)` | `WHERE ...` |
| `lista.sort(...)` sobre lo que devuelve el repositorio | `ORDER BY ...` |
| Recorrer la lista para contar pedidos por provincia | `GROUP BY provincia` |
| Meter el resultado en un `Set` para quitar repetidos | `DISTINCT`, o una `PRIMARY KEY` / `UNIQUE` que impida que existan |

La BD tiene índices y está hecha para esto. Si traes la tabla entera a memoria para filtrarla en Java, con 10 filas no lo notas y con 100.000 sí.

> 🤖 **Es un error muy típico de la IA:** `findAll()` seguido de un `stream().filter(...)`. Funciona, y por eso no lo detectas.

---

## 2. Estructuras lineales

### 2.1 Arrays y listas

Colecciones ordenadas con acceso por posición. Un array tiene tamaño fijo; una `List` puede crecer, pero insertar o borrar en medio es O(n).

**Lo que devuelve un repositorio es una `List`, y está bien que lo sea:**

```java
List<Libro> findAll();   // SELECT ... ORDER BY titulo
```

- El **orden** lo pone la consulta (`ORDER BY`), y una `List` lo respeta.
- Los **duplicados** no existen: la clave primaria ya lo garantiza. No necesitas un `Set<Libro>`.

### 2.2 Pilas (LIFO)

*Last In, First Out*: el último que entra es el primero que sale, como una pila de platos.

| Operación | Qué hace |
|---|---|
| `push(x)` | Añade arriba |
| `pop()` | Quita el de arriba |
| `peek()` | Mira el de arriba sin quitarlo |

**Dónde la ves en nuestras aplicaciones: la pila de llamadas.** Cada vez que un método llama a otro, Java apila una llamada. Cuando salta una excepción, la traza que ves en consola **es esa pila**:

```
java.lang.NullPointerException: ...
    at com.biblioteca.service.LibroService.guardar(LibroService.java:42)        ← cima: aquí falló
    at com.biblioteca.controller.LibroController.guardar(LibroController.java:58)
    at ...                                                                       ← llamadas de Spring y Tomcat
```

Por eso se lee **de arriba abajo**: arriba está la última llamada, donde falló. Busca la primera línea que sea de tu paquete.

Si un método se llama a sí mismo sin fin, la pila se llena: `StackOverflowError`.

> 🔜 **Con la API REST y con JPA:** si `Autor` tiene una lista de `Libro` y cada `Libro` apunta a su `Autor`, convertirlos a JSON o hacer `toString()` puede entrar en un bucle infinito (autor → libro → autor…) y acabar en `StackOverflowError`.

**En Java:** `Deque<T> pila = new ArrayDeque<>()`. **No uses la clase `Stack`**: es de Java 1.0 y la propia documentación recomienda `ArrayDeque`.

### 2.3 Colas (FIFO)

*First In, First Out*: el primero que entra es el primero que sale, como la cola del supermercado.

| Operación | En Java (`Queue`) | Qué hace |
|---|---|---|
| *enqueue* | `offer(x)` | Añade al final |
| *dequeue* | `poll()` | Saca el del frente |

**Dónde la ves:** cuando todos los hilos del servidor están ocupados, las peticiones nuevas **esperan en una cola** y se atienden por orden de llegada. También son colas las tareas que se procesan en orden, como los correos pendientes de enviar.

**En Java:** `Queue<T> cola = new ArrayDeque<>()`.

> ¿Y "los pedidos urgentes primero"? Eso no es una cola: es `ORDER BY urgente DESC, fecha_creacion`.

---

## 3. Estructuras asociativas

### 3.1 Mapas: clave → valor

Guardan pares clave-valor con acceso O(1) de media por clave. La clave tiene que ser **única e inmutable**: un id, un ISBN, un código de pedido.

**Spring ya los usa por todas partes:**

| Dónde | Clave → valor |
|---|---|
| `model.addAttribute("libros", libros)` | nombre → objeto para la vista |
| `session.setAttribute("carrito", carrito)` | nombre → objeto de la sesión |
| Parámetros de una petición | nombre del campo → valor |
| Un objeto JSON (API REST) | nombre de la propiedad → valor |

**Caso práctico: el carrito de la compra en la sesión.** Todavía no está en la BD: se guarda en memoria hasta que el usuario confirma el pedido.

Con una `List`, cada vez que se añade un producto hay que recorrer la lista para saber si ya estaba:

```java
// ❌ Con List
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

lineas.merge(productoId, cantidad, Integer::sum);    // si no está, lo añade; si está, suma
lineas.remove(productoId);                           // quitar un producto
```

Al mostrar el carrito, carga los productos con **una sola consulta** usando las claves (`WHERE id IN (...)`), no un `findById` por cada línea (ver "consultas dentro de un bucle" en Algoritmos y complejidad).

### 3.2 Conjuntos (Sets)

Colecciones de elementos únicos. Operaciones: pertenencia en O(1), unión, intersección y diferencia.

**Caso práctico: los roles de un usuario.** Un usuario no puede tener dos veces el rol `ADMIN`. Ahora tenemos un rol por usuario, pero cuando un usuario pueda tener varios (relación N:M, con JPA) será un `Set`:

```java
Set<String> roles = Set.of("ADMIN", "USER");
roles.contains("ADMIN");   // pertenencia: O(1)
```

> Spring Security ya lo hace así: el `User` que construyes en tu `UserDetailsService` guarda sus permisos en un `Set`.

**Las operaciones de conjuntos sirven de verdad.** En un formulario para editar los roles de un usuario (casillas de verificación), ¿qué filas hay que insertar y cuáles borrar en la tabla `usuario_rol`?

```java
Set<String> actuales = Set.of("USER", "EDITOR");   // lo que hay en la BD
Set<String> nuevos   = Set.of("USER", "ADMIN");    // lo que llega del formulario

Set<String> aInsertar = new HashSet<>(nuevos);
aInsertar.removeAll(actuales);   // diferencia: [ADMIN]  → INSERT

Set<String> aBorrar = new HashSet<>(actuales);
aBorrar.removeAll(nuevos);       // diferencia: [EDITOR] → DELETE
```

Sin conjuntos, borrarías todos los roles y los volverías a insertar.

---

## 4. `equals()` y `hashCode()`: por la clave de negocio

Si guardas objetos tuyos en un `Set`, los usas como clave de un `Map`, llamas a `contains()` o los comparas con `assertEquals` en un test, Java necesita saber cuándo dos objetos son "el mismo". El criterio es **el campo que lo identifica de forma única**: el ISBN de un libro, el código de un pedido.

```java
@EqualsAndHashCode(onlyExplicitlyIncluded = true)
public class Pedido {
    @EqualsAndHashCode.Include
    private String codigo;
    private LocalDate fecha;
    private boolean urgente;
}
```

Por qué hacen falta los dos métodos, y el problema de `@Data`, está en **Algoritmos y complejidad**.

---

## 5. Resumen: qué estructura y cuánto cuesta

| Estructura | Garantiza | Acceso / búsqueda | En nuestras aplicaciones |
|---|---|---|---|
| `List` (`ArrayList`) | Orden y posición | Por posición O(1); buscar un valor O(n) | Resultado de una consulta |
| Pila (`ArrayDeque`) | Sale el último que entró | Cima O(1) | Pila de llamadas, trazas de error, deshacer |
| Cola (`ArrayDeque`) | Sale el primero que entró | Frente O(1) | Peticiones en espera, tareas en orden |
| `Map` (`HashMap`, `LinkedHashMap`) | Claves únicas | Por clave O(1) | Carrito, modelo, sesión, JSON |
| `Set` (`HashSet`) | Elementos únicos | Pertenencia O(1) | Roles, diferencias entre dos selecciones |

`Linked…` (`LinkedHashMap`, `LinkedHashSet`) mantiene el orden en que se añadieron los elementos; `Hash…` no garantiza ningún orden.

---

## 6. Usar la IA con criterio

### 6.1 Checklist para revisar el código que te da la IA

1. **¿Trae datos de la BD para filtrarlos, ordenarlos o contarlos en Java?** Eso es un `WHERE`, un `ORDER BY` o un `GROUP BY`.
2. **¿Hay una consulta dentro de un bucle?** Por ejemplo, un `findById` por cada línea del carrito.
3. **¿Usa una `List` donde busca por clave o donde no puede haber repetidos?** Piensa en `Map` o `Set`.
4. **¿Mete objetos tuyos en un `Set` o como clave de un `Map`?** ¿Tienen `equals()` y `hashCode()` por la clave de negocio?
5. **¿Usa clases desfasadas?**

| Si la IA propone… | Hoy se usa… |
|---|---|
| `Stack` | `ArrayDeque` |
| `Vector` | `ArrayList` |
| `Hashtable` | `HashMap` |

### 6.2 Pregunta mal vs. pregunta bien

| ❌ Prompt pobre | ✅ Prompt con contexto |
|---|---|
| "Haz un carrito de la compra" | "Haz un carrito guardado en la sesión. Cada producto aparece una vez con su cantidad, en el orden en que se añadió. Los productos están en la BD: cárgalos con una sola consulta al mostrar el carrito." |
| "Muestra los pedidos urgentes primero" | "Añade al repositorio JDBC un método que devuelva los pedidos pendientes, los urgentes primero y después por fecha, ordenados en la consulta SQL." |

---

## 7. Practica: ¿colección de Java o SQL?

| | Situación |
|---|---|
| A | Mostrar los libros de un autor |
| B | Los roles de un usuario que puede tener varios |
| C | El carrito de la compra antes de confirmar el pedido |
| D | Al guardar el formulario de roles, saber qué roles hay que borrar |
| E | Mostrar los pedidos con los urgentes arriba |
| F | Saber cuántos pedidos hay por provincia |
| G | Salta una excepción: ¿qué línea de la traza miras primero? |

<details>
<summary>Soluciones</summary>

| | Respuesta | Por qué |
|---|---|---|
| A | SQL: `WHERE autor_id = ?` → `List<Libro>` | Filtrar es trabajo de la BD; el resultado, una `List` |
| B | `Set` | No puede haber roles repetidos |
| C | `Map<Long, Integer>` (mejor `LinkedHashMap`) | Búsqueda por producto; todavía no está en la BD |
| D | Diferencia de conjuntos: `actuales.removeAll(nuevos)` | Operación típica de `Set` |
| E | SQL: `ORDER BY urgente DESC, fecha_creacion` | Ordenar es trabajo de la BD |
| F | SQL: `GROUP BY provincia` | Contar es trabajo de la BD |
| G | La de arriba (la primera de tu paquete) | La traza es una pila: arriba está la última llamada |

</details>

---

## Resumen en tres frases

1. **Buscar, filtrar, ordenar y contar se hace en la base de datos**, no trayendo la tabla a memoria.
2. **Las colecciones organizan lo que ya está en memoria:** `List` para resultados, `Map` para buscar por clave (carrito), `Set` para elementos únicos (roles).
3. **Pilas y colas están por debajo de lo que usas:** la traza de un error es una pila, y las peticiones que esperan al servidor forman una cola.
