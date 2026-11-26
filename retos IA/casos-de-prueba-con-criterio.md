# Casos de prueba con criterio: un test verde no significa que el código esté bien

> **Idea clave:** la IA escribe tests muy rápido, pero suele mirar el código y escribir tests que confirman **lo que el código hace**, no **lo que debería hacer**.
> Si el código tiene un bug, el test lo da por bueno y sale verde.
> Decidir **qué casos hay que probar y cuál es el resultado correcto** es trabajo tuyo.

Teoría de apoyo: [Testing: primer contacto](../conceptos/testing.md)

---

## 1. El problema: tests que dan el bug por bueno

El cliente dice: *"al crear un autor, el nombre se guarda sin espacios al principio ni al final"*.

Este es el código, **con un bug**: alguien olvidó el `trim()`.

```java
public Autor crear(Autor autor) {
    return autorRepository.save(autor);
}
```

Le pedimos a la IA *"hazme los tests de `AutorService.crear`"*. Lee el código y escribe esto:

```java
@Test
void crearGuardaElAutor() {
    Autor autor = new Autor();
    autor.setNombre("  Ana María Matute  ");
    when(autorRepository.save(autor)).thenReturn(autor);

    Autor creado = autorService.crear(autor);

    assertEquals("  Ana María Matute  ", creado.getNombre());   // ✅ verde
}
```

El test **pasa**. Y está mal: comprueba que el nombre se guarda **con** espacios, que es justo el bug.

La IA no conoce el requisito del cliente. Solo ve el código, y da por hecho que el código es correcto.

---

## 2. Primero los casos, en castellano

Antes de escribir un test (o de pedírselo a la IA), apunta los casos y qué debe pasar en cada uno. Esa lista **es la especificación**.

> **Crear autor**
> - Con nombre normal → se guarda
> - Con espacios alrededor → se guarda **sin** espacios
> - Con el nombre vacío o en blanco → error, y **no llega** a la base de datos

Para no dejarte casos, repasa siempre estos tipos:

| Tipo de caso | Pregunta | Ejemplo en la biblioteca |
|---|---|---|
| **Camino feliz** | ¿Funciona en el caso normal? | Crear un libro con todos los datos correctos |
| **Datos inválidos** | ¿Qué pasa con datos vacíos o mal formados? | Título en blanco, ISBN con letras |
| **Duplicados** | ¿Qué pasa si ya existe? | Crear un libro con un ISBN que ya está → `IsbnDuplicadoException` |
| **No existe** | ¿Qué pasa si busco algo que no está? | Editar un ISBN inexistente → 404 |
| **Límites** | ¿Qué pasa justo en el borde? | Filtrar de 2000 a 2000: ¿sale un libro de 2000? |
| **Permisos** | ¿Quién puede hacerlo y quién no? | Un USER intenta borrar → 403 |
| **Efectos** | ¿Pasa algo que **no** debería? | Si hay error, no se llama a `save()` |

Fíjate en el caso de **límites**: a veces la respuesta no está en el enunciado. Hay que preguntársela al cliente, igual que en el reto de [requisitos vagos](./requisitos-vagos.md).

---

## 3. Cómo usar la IA con criterio

| ❌ Sin criterio | ✅ Con criterio |
|---|---|
| *"Hazme los tests de `AutorService`"* | *"Escribe un test JUnit 5 con Mockito para cada caso: (1) nombre normal → se guarda; (2) nombre con espacios → se guarda sin espacios; (3) nombre en blanco → `IllegalArgumentException` y no se llama a `save()`"* |
| La IA decide qué probar | **Tú** decides qué probar; la IA escribe la sintaxis |
| Aceptas el test porque sale verde | Compruebas que el test **falla** cuando el código está mal |

### La prueba de fuego: rompe el código

Un test que nunca falla no protege nada. Antes de darlo por bueno:

1. Mete un bug a propósito en el código (quita el `trim()`, cambia un `>` por `>=`).
2. Ejecuta el test.
3. **Si sigue en verde, el test no sirve.**

---

## 4. Reto

El listado `/libros` tiene un filtro por rango de años (`anioDesde`, `anioHasta`). Antes de mirar las soluciones, escribe en castellano la lista de casos que probarías.

<details>
<summary>Posibles casos</summary>

| Caso | Resultado esperado |
|---|---|
| Sin filtros | Todos los libros |
| Solo `anioDesde` | Libros de ese año en adelante |
| Solo `anioHasta` | Libros hasta ese año |
| Los dos | Libros dentro del rango |
| `anioDesde` = `anioHasta` | Libros de ese año (¿los límites se incluyen? **pregunta al cliente**) |
| `anioHasta` menor que `anioDesde` | Error en el formulario (`filtro.rango.invalido`) y no se busca |
| Rango sin libros | Lista vacía, **no** un error |
| Un USER usa el filtro | Puede (200). Un anónimo, no: va al login |

</details>

---


