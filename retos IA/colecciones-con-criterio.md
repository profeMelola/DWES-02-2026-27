# Colecciones con criterio: no todo es un `ArrayList`

> **Idea clave:** `ArrayList` es una buena opción por defecto, pero no sirve para todo.
> Antes de elegir colección hay **cuatro preguntas** que casi nunca vienen en el enunciado,
> igual que en el reto de la especificación incompleta del cliente.

---

## 1. El cliente dice: "quiero ver los pedidos del almacén"

Volvemos a la LogísticaApp. El encargado del almacén pide:

> «Quiero una pantalla con los pedidos pendientes, los urgentes primero. En cada pedido quiero ver sus etiquetas (frágil, refrigerado…) y abajo un resumen de cuántos pedidos hay por provincia.»

Cada frase esconde una decisión sobre qué colección usar:

| Lo que dice el cliente | Pregunta que debes hacerte | Colección |
|---|---|---|
| "Las etiquetas del pedido" | ¿Puede aparecer "frágil" dos veces? No tiene sentido | `Set` |
| "Cuántos pedidos por provincia" | ¿Necesito buscar por una clave (la provincia)? | `Map` |
| "Por provincia", en un listado | ¿Importa el orden en que se muestran? Sí, alfabético | `TreeMap` |
| "Los urgentes primero" | ¿Quién sale primero? ¿Es solo orden en el listado o "el siguiente que se prepara"? | `List` ordenada o `PriorityQueue` |

La última fila es la pregunta 4 del reto de requisitos vagos ("¿«salgan antes» significa orden en el listado o prioridad de envío?"). **La respuesta del cliente cambia la colección.**

---

## 2. Las cuatro preguntas

| Pregunta | Si la respuesta es… | Usa |
|---|---|---|
| ¿Puede haber **duplicados**? | No | `Set` |
| ¿Busco por una **clave**? | Sí | `Map` |
| ¿Importa el **orden**? | Orden en que se añadieron | `LinkedHashSet` / `LinkedHashMap` |
| | Ordenado (alfabético, por fecha…) | `TreeSet` / `TreeMap` |
| | Me da igual | `HashSet` / `HashMap` |
| ¿**Quién sale primero**? | El último que entró (LIFO) | `Deque` como pila |
| | El primero que entró (FIFO) | `Deque` / `Queue` como cola |
| | El más prioritario | `PriorityQueue` |
| Ninguna de las anteriores | Lista con orden y posibles duplicados | `List` (`ArrayList`) |

### Las familias de un vistazo

| Interfaz | Qué es | Implementación habitual |
|---|---|---|
| `List` | Secuencia con posición y duplicados | `ArrayList` |
| `Set` | Sin duplicados | `HashSet`, `LinkedHashSet`, `TreeSet` |
| `Map` | Pares clave → valor, sin claves repetidas | `HashMap`, `LinkedHashMap`, `TreeMap` |
| `Deque` | Se añade y se saca por los extremos: pila o cola | `ArrayDeque` |
| `Queue` | Cola | `ArrayDeque`, `PriorityQueue` |

### Hash, Linked y Tree: la misma idea en `Set` y en `Map`

| Prefijo | Orden al recorrer | Buscar / añadir | Requisito |
|---|---|---|---|
| `Hash…` | Ninguno (y puede cambiar) | O(1) | `equals()` y `hashCode()` |
| `LinkedHash…` | El de inserción | O(1) | `equals()` y `hashCode()` |
| `Tree…` | Ordenado | O(log n) | `Comparable` o un `Comparator` |

> 🔙 **Repaso:** por qué `Hash…` necesita `equals()` y `hashCode()`, y qué significan O(1) y O(log n), está en el reto **Algoritmos y complejidad**.

---

## 3. `List`: la opción por defecto (cuando de verdad lo es)

Usa `List` cuando importa el orden, puede haber duplicados o accedes por posición:

```java
// Las líneas de un pedido: orden de inserción, y puede haber dos líneas del mismo producto
List<LineaPedido> lineas = new ArrayList<>();
```

Deja de ser buena opción cuando empiezas a hacer esto:

| Si haces… | Estás pidiendo… |
|---|---|
| `if (!lista.contains(x)) lista.add(x);` | Un `Set` |
| Recorrer la lista buscando por id o por ISBN | Un `Map` |
| `lista.remove(0)` para sacar "el siguiente" | Una cola (`ArrayDeque`) |

---

## 4. `Set`: sin duplicados

```java
// Roles de un usuario: ADMIN no puede aparecer dos veces
Set<String> roles = new HashSet<>();
roles.add("ADMIN");
roles.add("ADMIN");   // no hace nada
roles.size();         // 1
```

| Necesito… | Usa |
|---|---|
| Solo que no se repitan | `HashSet` |
| Que no se repitan y mostrarlas en el orden en que se añadieron (etiquetas de un pedido) | `LinkedHashSet` |
| Que no se repitan y salgan ordenadas (desplegable de provincias) | `TreeSet` |

### ⚠️ La trampa del `TreeSet` con `Comparator`

Un `TreeSet` **no usa `equals()`**: considera iguales dos elementos si el `Comparator` devuelve 0.

```java
Set<Libro> porTitulo = new TreeSet<>(Comparator.comparing(Libro::getTitulo));
porTitulo.add(new Libro("978-84-01", "Dune"));
porTitulo.add(new Libro("978-84-02", "Dune"));   // otra edición, otro ISBN
porTitulo.size();   // 1 ❌ ¡la segunda edición ha desaparecido sin error!
```

Solución: desempatar por el campo que identifica al objeto.

```java
new TreeSet<>(Comparator.comparing(Libro::getTitulo).thenComparing(Libro::getIsbn));
```

---

## 5. `Map`: buscar por clave

### Ejemplo: el carrito de la compra

```java
// producto → cantidad, en el orden en que el usuario los añadió
Map<Long, Integer> carrito = new LinkedHashMap<>();

carrito.merge(productoId, 1, Integer::sum);   // si no está, pone 1; si está, suma 1
```

Con una `List<LineaCarrito>` tendrías que recorrer la lista para saber si el producto ya estaba.

### Ejemplo: resumen por provincia para Thymeleaf

```java
// En el servicio
Map<String, List<Pedido>> porProvincia = pedidos.stream()
        .collect(Collectors.groupingBy(Pedido::getProvincia,
                                       TreeMap::new,              // ← ordenado por provincia
                                       Collectors.toList()));
```

```html
<!-- En la vista -->
<tr th:each="entrada : ${porProvincia}">
    <td th:text="${entrada.key}">Provincia</td>
    <td th:text="${entrada.value.size()}">0</td>
</tr>
```

Sin `TreeMap::new`, `groupingBy` devuelve un `HashMap` y **las provincias salen en un orden aleatorio**.

| Necesito… | Usa |
|---|---|
| Solo buscar por clave | `HashMap` |
| Buscar por clave y mostrar en orden de inserción (carrito) | `LinkedHashMap` |
| Buscar por clave y mostrar ordenado por la clave (resumen por provincia) | `TreeMap` |

> ⚠️ `Map.of(...)` y `Set.of(...)` no garantizan ningún orden, y el orden **puede cambiar cada vez que arrancas la aplicación**. No los uses para algo que se muestra en pantalla en un orden concreto.

---

## 6. LIFO y FIFO: pilas y colas

| | Pila (LIFO) | Cola (FIFO) |
|---|---|---|
| Sale primero… | El **último** que entró | El **primero** que entró |
| Vida real | Pila de platos | Cola del supermercado |
| En una web | Botón "deshacer", historial de "volver" | Emails pendientes de enviar, pedidos por orden de llegada |
| En Java | `Deque<T> pila = new ArrayDeque<>()` | `Queue<T> cola = new ArrayDeque<>()` |
| Añadir | `push(x)` | `offer(x)` |
| Sacar | `pop()` | `poll()` |
| Mirar sin sacar | `peek()` | `peek()` |

```java
// Deshacer cambios en un formulario: el último cambio es el primero que se deshace
Deque<String> cambios = new ArrayDeque<>();
cambios.push("título cambiado");
cambios.push("precio cambiado");
cambios.pop();   // "precio cambiado"

// Emails de confirmación: se envían en el orden en que se generaron
Queue<Email> pendientes = new ArrayDeque<>();
pendientes.offer(email1);
pendientes.offer(email2);
pendientes.poll();   // email1
```

> ⚠️ **No uses la clase `Stack`.** Es de Java 1.0, hereda de `Vector` y la propia documentación de Java recomienda `ArrayDeque`. La IA y muchos tutoriales antiguos la siguen proponiendo.

### Cola con prioridad: "los urgentes primero"

```java
Queue<Pedido> porPreparar = new PriorityQueue<>(
        Comparator.comparing(Pedido::isUrgente, Comparator.reverseOrder())   // true antes que false
                  .thenComparing(Pedido::getFechaCreacion));                 // después, el más antiguo

porPreparar.offer(pedidoNormal);
porPreparar.offer(pedidoUrgente);
porPreparar.poll();   // pedidoUrgente
```

### ⚠️ La trampa de la `PriorityQueue`

Solo `poll()` y `peek()` respetan la prioridad. **Si la recorres con un `for` o con `th:each`, el orden no está garantizado.**

| El cliente quiere… | Usa |
|---|---|
| Ver **todos** los pedidos con los urgentes arriba (pantalla del listado) | `List` + `sort(comparator)` |
| Saber **cuál es el siguiente** que hay que preparar | `PriorityQueue` + `poll()` |

Es la misma pregunta de requisitos vagos: ¿orden en el listado o prioridad real?

---

## 7. Lo que cambia en una aplicación web

### 7.1 Un bean lo comparten todos los usuarios

Un `@Service` o un `@Repository` existe **una sola vez** en la aplicación, y lo usan todas las peticiones a la vez. Una colección guardada como atributo es compartida:

```java
@Repository
public class PedidoRepositoryMemoria {
    private final Map<Long, Pedido> pedidos = new HashMap<>();   // ❌ dos peticiones a la vez pueden corromperlo
    private long siguienteId = 1;                                // ❌ dos pedidos pueden recibir el mismo id

    public Pedido guardar(Pedido pedido) {
        pedido.setId(siguienteId++);
        pedidos.put(pedido.getId(), pedido);
        return pedido;
    }
}
```

```java
@Repository
public class PedidoRepositoryMemoria {
    private final Map<Long, Pedido> pedidos = new ConcurrentHashMap<>();   // ✅ segura con varias peticiones
    private final AtomicLong siguienteId = new AtomicLong();              // ✅ cada llamada recibe un id distinto

    public Pedido guardar(Pedido pedido) {
        pedido.setId(siguienteId.incrementAndGet());
        pedidos.put(pedido.getId(), pedido);
        return pedido;
    }
}
```

`ConcurrentHashMap` protege el mapa, pero no tu lógica: "comprobar si hay stock y después descontarlo" sigue siendo un problema con dos peticiones a la vez. Es la pregunta 6 del reto de requisitos vagos (dos pedidos piden el último artículo). Con base de datos se resuelve con transacciones.

> Un bean con `@SessionScope` (por ejemplo, el carrito) es uno por usuario, así que ahí no hay este problema.

### 7.2 No dejes que modifiquen tus datos desde fuera

```java
public List<Pedido> listarPendientes() {
    return pedidos.values().stream()
            .filter(Pedido::isPendiente)
            .toList();   // lista inmodificable
}
```

Si el controlador intenta hacer `add()` o `remove()` sobre esa lista, salta una `UnsupportedOperationException`. Es intencionado: los datos se modifican a través del servicio, no tocando su colección. Lo mismo pasa con `List.of(...)`.

> 🔜 **Cuando trabajemos con JPA:** en las relaciones entre entidades, `Set` o `List` no da igual. Si cargas a la vez dos relaciones de tipo `List` con `JOIN FETCH`, Hibernate lanza `MultipleBagFetchException`. Y si usas `Set`, vuelves a necesitar `equals()` y `hashCode()` bien hechos (sin `@Data`).

---

## 8. Usar la IA con criterio

### 8.1 Pregunta mal vs. pregunta bien

| ❌ Prompt pobre | ✅ Prompt con contexto |
|---|---|
| "Haz un carrito de la compra" | "Haz un carrito en un bean con `@SessionScope`. Cada producto aparece una vez con su cantidad, y se muestran en el orden en que se añadieron. ¿Qué colección uso y por qué?" |
| "Ordena los pedidos por urgencia" | "Necesito mostrar todos los pedidos con los urgentes arriba y, aparte, obtener el siguiente que hay que preparar. ¿Uso la misma colección para las dos cosas?" |

El buen prompt **contiene las respuestas a las cuatro preguntas**.

### 8.2 La IA huele a rancio

Estas clases funcionan, pero son de hace más de 25 años. Si la IA las propone, pídele la alternativa actual:

| Si la IA propone… | Hoy se usa… |
|---|---|
| `Stack` | `ArrayDeque` (como `Deque`) |
| `Vector` | `ArrayList` |
| `Hashtable` | `HashMap` o `ConcurrentHashMap` |
| `LinkedList` como cola | `ArrayDeque` |
| `List lista = new ArrayList();` (sin `<Tipo>`) | `List<Pedido> lista = new ArrayList<>();` |

### 8.3 Checklist para revisar el código que te da la IA

1. **¿Usa `List` donde no debe haber duplicados o donde busca por clave?**
2. **¿Mete clases propias en un `Hash…`?** ¿Tienen `equals()` y `hashCode()`?
3. **¿Usa `TreeSet`/`TreeMap` con un `Comparator`?** ¿Desempata por el identificador?
4. **¿El orden en pantalla depende de un `HashMap`, `Map.of` o `Set.of`?**
5. **¿Recorre una `PriorityQueue` esperando que salga ordenada?**
6. **¿Hay una colección modificable como atributo de un `@Service` o `@Repository`?** ¿Es segura con varias peticiones a la vez?
7. **¿Usa alguna clase de la tabla 8.2?**

---

## 9. Practica: ¿qué colección usarías?

| | Situación |
|---|---|
| A | Los roles de un usuario (`ADMIN`, `USER`) |
| B | El carrito: producto → cantidad, mostrado en el orden en que se añadieron |
| C | Un desplegable con las provincias de los clientes, sin repetir y ordenadas |
| D | Botón "deshacer" los últimos cambios de un formulario |
| E | Emails de confirmación que se envían en el orden en que se generan |
| F | El almacén pulsa "siguiente pedido": urgentes primero y después el más antiguo |
| G | Un contador de visitas por libro, guardado en un `@Service` |
| H | Los pedidos de un cliente, en el orden en que los hizo |

<details>
<summary>Soluciones</summary>

| | Colección | Por qué |
|---|---|---|
| A | `Set<String>` (`HashSet`) | Sin duplicados; el orden da igual |
| B | `LinkedHashMap<Long, Integer>` | Búsqueda por producto + orden de inserción |
| C | `TreeSet<String>` | Sin duplicados + orden alfabético. También vale `stream().map(...).distinct().sorted().toList()` |
| D | `Deque<…>` con `ArrayDeque` (pila) | LIFO: el último cambio se deshace primero |
| E | `Queue<Email>` con `ArrayDeque` | FIFO: el primero que entra, el primero que sale |
| F | `PriorityQueue<Pedido>` + `poll()` | Prioridad con desempate. Si además hay que **mostrar** el listado: `List` ordenada |
| G | `ConcurrentHashMap<Long, Integer>` + `merge(id, 1, Integer::sum)` | Búsqueda por libro, compartido por todas las peticiones |
| H | `List<Pedido>` (`ArrayList`) | Orden y posición: aquí `ArrayList` es la respuesta correcta |

</details>

---

## Resumen en tres frases

1. **Antes de escribir `new ArrayList<>()`, hazte las cuatro preguntas:** ¿duplicados?, ¿clave?, ¿orden?, ¿quién sale primero?
2. **`Hash`, `LinkedHash` y `Tree` responden a la pregunta del orden**, y cada uno tiene su requisito (`equals`/`hashCode` o `Comparator`).
3. **En una aplicación web, una colección en un bean la comparten todos los usuarios**, y la IA no te lo va a avisar.
