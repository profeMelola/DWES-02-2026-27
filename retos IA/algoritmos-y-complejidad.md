# Algoritmos y complejidad: la pregunta que la IA no hace

> **Idea clave:** un algoritmo puede funcionar bien con 10 datos y ser un desastre con 1.000.000.
> Elegir bien depende de **información que normalmente no viene en el enunciado**,
> igual que en el reto de la especificación incompleta del cliente.

---

## 1. El cliente dice: "quiero buscar libros por ISBN"

Parece un requisito claro. No lo es. Antes de escribir una línea (o de pedírsela a la IA) faltan preguntas:

| Pregunta al cliente | Por qué cambia el algoritmo |
|---|---|
| ¿Cuántos libros hay? ¿100 o 2 millones? | Con 100 da igual el algoritmo. Con 2 millones, no. |
| ¿Los datos están ordenados? | La búsqueda binaria **solo** funciona con datos ordenados. |
| ¿Se busca mucho y se inserta poco, o al revés? | Mantener datos ordenados o indexados tiene un coste al insertar. |
| ¿La memoria es un problema? | Un `HashMap` es rapidísimo, pero ocupa memoria extra. |

**¿Y si los libros están en una base de datos?** Entonces el algoritmo lo elige la BD, pero las preguntas siguen importando: tú decides si la columna tiene **índice** y cómo escribes la consulta.

| Consulta | Coste | Por qué |
|---|---|---|
| `WHERE isbn = ?` con `isbn` `UNIQUE` o `PRIMARY KEY` | O(log n) | La BD crea un índice automáticamente: es como tener la columna ordenada |
| `WHERE isbn = ?` sin índice | O(n) | La BD lee la tabla fila a fila, como tu bucle `for` |
| `WHERE titulo LIKE '%sombra%'` | O(n) | Con el comodín al principio el índice no sirve |

Y el índice tampoco es gratis: ocupa espacio y hace más lentos los `INSERT` y `UPDATE`. Es la misma pregunta de antes: ¿se busca mucho o se inserta mucho?

**La IA no hace estas preguntas.** Te da *una* solución que funciona y asume el resto. Tu trabajo es saber qué preguntar y comprobar si lo que ha asumido encaja con tu caso.

---

## 2. Qué es la notación Big O (sin fórmulas)

Big O responde a una pregunta: **si los datos se multiplican por 10, ¿cuánto más tarda mi código?**

Piensa en una biblioteca física:

| Big O | Nombre | En la biblioteca | En código |
|---|---|---|---|
| **O(1)** | Constante | Sabes la estantería y el hueco exactos: vas directo | `array[5]`, `map.get(isbn)` |
| **O(log n)** | Logarítmica | Libros ordenados: abres por la mitad, descartas media biblioteca, repites | Búsqueda binaria |
| **O(n)** | Lineal | Miras los libros uno a uno hasta encontrarlo | Un `for` que recorre la lista |
| **O(n log n)** | Linearítmica | Ordenar toda la biblioteca de forma inteligente | `list.sort(...)` |
| **O(n²)** | Cuadrática | Para cada libro, lo comparas con **todos** los demás | Dos `for` anidados sobre la misma lista |
| **O(2ⁿ)** | Exponencial | Probar todas las combinaciones posibles de libros | Fuerza bruta con combinaciones |

### Cuántos pasos supone cada una

| n (datos) | O(1) | O(log n) | O(n) | O(n log n) | O(n²) |
|---|---|---|---|---|---|
| 10 | 1 | 3 | 10 | 33 | 100 |
| 1.000 | 1 | 10 | 1.000 | 10.000 | 1.000.000 |
| 1.000.000 | 1 | 20 | 1.000.000 | 20.000.000 | 1.000.000.000.000 |

👉 Con 10 datos todo es instantáneo. **Por eso en clase "todo funciona"** y los problemas aparecen en producción.

---

## 3. Búsqueda: tres formas de encontrar un libro

### 3.1 Búsqueda lineal — O(n)

```java
public Libro buscarPorIsbn(List<Libro> libros, String isbn) {
    for (Libro libro : libros) {          // miro uno a uno
        if (libro.getIsbn().equals(isbn)) {
            return libro;
        }
    }
    return null;
}
```

- ✅ Funciona siempre, con datos desordenados.
- ❌ Si el libro está al final (peor caso), recorre la lista entera.

### 3.2 Búsqueda binaria — O(log n)

```java
// ¡REQUISITO! El array tiene que estar ORDENADO
int[] anios = {1950, 1967, 1984, 1997, 2005, 2018, 2024};

int posicion = Arrays.binarySearch(anios, 2005);   // devuelve 4
```

Cómo funciona por dentro: mira el centro (`1997`). ¿2005 es mayor? Descarta la mitad izquierda. Mira el centro de lo que queda… En 3 pasos lo encuentra.

- ✅ Rapidísima: 1 millón de datos → unos 20 pasos.
- ❌ Si los datos **no** están ordenados, el resultado es basura (no da error, da un resultado incorrecto).

### 3.3 Búsqueda con hash — O(1) de media

```java
Map<String, Libro> librosPorIsbn = new HashMap<>();
librosPorIsbn.put("978-84-376-0494-7", libro);

Libro encontrado = librosPorIsbn.get("978-84-376-0494-7");   // directo
```

- ✅ Acceso directo por clave, da igual cuántos libros haya.
- ❌ Ocupa más memoria y solo sirve para buscar **por la clave exacta** (no para "libros entre 1990 y 2000").

#### ¿Cómo sabe el `HashMap` dónde está el libro?

Imagina una biblioteca con **10 estanterías**, numeradas del 0 al 9. La regla es:

> El libro va a la estantería que indica el **último dígito de su ISBN**.

- ¿Buscas el ISBN `978-84-376-0494-7`? → Vas directo a la **estantería 7**. No miras las otras 9.
- En la estantería 7 puede haber varios libros (todos los que terminan en 7). Ahí sí miras cuál es el tuyo.

Java hace exactamente esto con dos métodos:

| Método | Qué hace | En la biblioteca |
|---|---|---|
| `hashCode()` | Calcula un número que indica **en qué estantería** mirar | El último dígito del ISBN |
| `equals()` | Comprueba si **este** es el libro que buscas | Leer el ISBN completo |

Por eso tu clase `Libro` necesita los dos, y los dos tienen que usar **el mismo criterio** (el ISBN):

```java
@Override
public boolean equals(Object o) {
    if (!(o instanceof Libro otro)) return false;
    return isbn.equals(otro.isbn);
}

@Override
public int hashCode() {
    return isbn.hashCode();
}
```

Con Lombok: `@EqualsAndHashCode(of = "isbn")`.

**¿Qué pasa si no los pones?** Java compara si son *el mismo objeto en memoria*, no si tienen el mismo ISBN:

```java
Libro a = new Libro("978-84-376-0494-7", "Cien años de soledad");
Libro b = new Libro("978-84-376-0494-7", "Cien años de soledad");

Set<Libro> libros = new HashSet<>();
libros.add(a);

libros.contains(b);   // false ❌ (sin equals/hashCode)
                      // true  ✅ (con equals/hashCode por ISBN)
```

---

## 4. El error más típico: el O(n²) escondido

Este código **parece** inocente y es el que suele generar la IA:

```java
// ¿Qué libros de la lista A están también en la lista B?
List<Libro> comunes = new ArrayList<>();
for (Libro libro : listaA) {               // n veces
    if (listaB.contains(libro)) {          // contains() recorre listaB → n veces
        comunes.add(libro);
    }
}
// Total: n × n = O(n²)
```

`contains()` en una `List` es un bucle oculto. Solución:

```java
Set<Libro> setB = new HashSet<>(listaB);   // O(n) una sola vez
List<Libro> comunes = new ArrayList<>();
for (Libro libro : listaA) {               // n veces
    if (setB.contains(libro)) {            // O(1)
        comunes.add(libro);
    }
}
// Total: O(n)
```

> ⚠️ Para que `HashSet` funcione con tus clases necesitan `equals()` y `hashCode()` bien definidos (por ejemplo, por ISBN).

### Bucles ocultos que debes reconocer

| Código | Coste real | Por qué |
|---|---|---|
| `list.contains(x)` | O(n) | Recorre la lista |
| `list.indexOf(x)` | O(n) | Recorre la lista |
| `arrayList.remove(0)` | O(n) | Desplaza todos los elementos |
| `repositorio.findById(id)` dentro de un `for` | n consultas a la BD | Cada vuelta es un viaje a la base de datos |
| `list.sort(...)` dentro de un `for` | O(n² log n) | Ordenas una y otra vez lo mismo |

El penúltimo es el más peligroso en nuestras aplicaciones: **una consulta SQL dentro de un bucle**. Con 10 autores no lo notas. Con 10.000, la página tarda minutos.

---

## 5. Ordenación: no la programes, entiéndela

| Algoritmo | Complejidad | Idea | ¿Lo usarás? |
|---|---|---|---|
| Burbuja, inserción, selección | O(n²) | Comparar e intercambiar vecinos | Solo para aprender |
| Quicksort | O(n log n) de media | Elegir un pivote, separar menores y mayores, repetir | Lo usa Java por dentro para tipos primitivos (`int[]`) |
| Mergesort | O(n log n) siempre | Partir en mitades, ordenar cada una, mezclar | Base del algoritmo que usa Java para objetos |
| Recuento / radix | O(n) | Contar cuántas veces aparece cada valor | Casos muy concretos (rangos pequeños de enteros) |

En Java, lo correcto casi siempre es:

```java
libros.sort(Comparator.comparing(Libro::getTitulo));      // O(n log n), estable
```

**Estable** significa que si dos libros tienen el mismo título, mantienen el orden en que estaban. Importa cuando ordenas por varios criterios.

---

## 6. Técnicas algorítmicas en una línea

| Técnica | Idea | Ejemplo muy básico |
|---|---|---|
| **Divide y vencerás** | Partir el problema, resolver trozos, juntar | Búsqueda binaria, mergesort |
| **Programación dinámica** | Apuntar resultados para no recalcularlos | Fibonacci guardando los valores ya calculados |
| **Voraz (greedy)** | Elegir lo mejor en cada paso, sin mirar atrás | Dar cambio con las monedas más grandes primero |
| **Backtracking** | Probar un camino; si no funciona, retroceder | Resolver un sudoku o un laberinto |
| **Dos punteros** | Un índice al principio y otro al final, avanzando hacia el centro | Comprobar si una palabra es palíndromo |

> 💡 **¿Quieres profundizar?** Pide a la IA que te explique cada técnica con un ejercicio muy sencillo y que te proponga otro para resolverlo tú. Por ejemplo: *"Explícame la técnica de dos punteros con un ejemplo básico en Java y proponme un ejercicio para practicarla sin darme la solución."*

---

## 7. Usar la IA con criterio

### 7.1 Pregunta mal vs. pregunta bien

| ❌ Prompt pobre | ✅ Prompt con contexto |
|---|---|
| "Haz un método que busque libros por ISBN" | "Tengo unos 500.000 libros en memoria, se busca por ISBN miles de veces al día y casi nunca se insertan. ¿Qué estructura uso y qué complejidad tiene la búsqueda y la inserción?" |
| "Haz que esto vaya más rápido" | "Este método tarda mucho con 10.000 autores. ¿Tiene algún bucle oculto o consultas dentro de un bucle?" |

Fíjate: el buen prompt **contiene las respuestas a las preguntas del apartado 1**. Si no sabes hacer esas preguntas, no sabes darle ese contexto a la IA.

### 7.2 Checklist para revisar el código que te da la IA

1. **¿Cuántos bucles hay, y están anidados?** Cuenta también los ocultos (`contains`, `indexOf`, `remove`).
2. **¿Hay accesos a BD dentro de un bucle?**
3. **¿Asume datos ordenados?** Si usa búsqueda binaria, ¿quién garantiza el orden?
4. **¿Usa `HashMap`/`HashSet` con una clase propia?** ¿Tiene `equals()` y `hashCode()`?
5. **¿Sigue los patrones de clase?** Una solución "más eficiente" que se salta las capas no es mejor solución.
6. **¿Puedes explicar por qué funciona?** Si no, no lo entregues: pregúntale a la IA hasta entenderlo, o escríbelo tú.

### 7.3 Pídele que se justifique… y compruébalo

> "Dime la complejidad de este método y señala la línea que la determina."

La IA puede equivocarse al calcular la complejidad, sobre todo con bucles ocultos. **Tú tienes que saber verificarlo**: ese es el fundamento que no puedes delegar.

---

## 8. Practica: ¿qué complejidad tiene?

```java
// A
int primero = libros.get(0).getAnio();

// B
int total = 0;
for (Libro l : libros) total += l.getPaginas();

// C
for (Libro a : libros)
    for (Libro b : libros)
        if (a != b && a.getTitulo().equals(b.getTitulo())) System.out.println("Repetido");

// D
for (Autor autor : autores) {
    int n = libroRepository.countByAutorId(autor.getId());
    System.out.println(autor.getNombre() + ": " + n);
}

// E
libros.sort(Comparator.comparing(Libro::getAnio));
```

<details>
<summary>Soluciones</summary>

| | Complejidad | Comentario |
|---|---|---|
| A | O(1) | Acceso directo por posición (en `ArrayList`) |
| B | O(n) | Un recorrido |
| C | O(n²) | Bucles anidados. Mejor: un `HashSet<String>` de títulos → O(n) |
| D | O(n) en Java, pero **n consultas SQL** | Con muchos autores, mejor una sola consulta con `GROUP BY` |
| E | O(n log n) | Ordenación de Java |

</details>

---

## Resumen en tres frases

1. **Big O dice cómo empeora tu código cuando crecen los datos**, no cuánto tarda hoy con 10 registros.
2. **El algoritmo correcto depende del contexto** (cuántos datos, si están ordenados, qué se hace más), y ese contexto hay que preguntarlo.
3. **La IA da soluciones que funcionan; tú decides si sirven.** Para eso necesitas reconocer bucles ocultos, estructuras de datos y patrones de diseño.
