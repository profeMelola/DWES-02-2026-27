# Chuleta de Thymeleaf

Resumen de lo que usa este proyecto (y algo más que puede caer en el examen). Los ejemplos salen de las plantillas de `src/main/resources/templates/`.

---

## 1. Esqueleto de una plantilla

```html
<!DOCTYPE html>
<html lang="es" xmlns:th="http://www.thymeleaf.org">   <!-- espacio de nombres th: -->
```

- El controlador devuelve el **nombre lógico** de la vista: `return "productos/lista";` → `templates/productos/lista.html`.
- `return "redirect:/productos";` → no pinta una plantilla, manda al navegador a otra URL (patrón PRG, ver `patron-PRG.md`).
- El texto que hay dentro de las etiquetas (`<td th:text="${p.nombre}">Nombre</td>`) es solo de **maqueta**: Thymeleaf lo sustituye. Así el HTML se ve bien abierto directamente en el navegador (*natural templates*).

---

## 2. Tipos de expresión

| Sintaxis | Nombre | Para qué | Ejemplo del proyecto |
|---|---|---|---|
| `${...}` | Variable | Leer datos del modelo | `${p.nombre}` |
| `*{...}` | Selección | Campo del objeto fijado con `th:object` | `*{nombre}` |
| `#{...}` | Mensaje | Textos de `messages.properties` (i18n) | `#{app.titulo}` |
| `@{...}` | URL | Enlaces con el contexto de la app | `@{/productos}` |
| `~{...}` | Fragmento | Trozo de otra plantilla | `~{fragments/layout :: navbar}` |

`${p.nombre}` llama a `p.getNombre()`, y `${p.fabricante.nombre}` navega por el objeto: `p.getFabricante().getNombre()`.

---

## 3. Atributos `th:*`

| Atributo | Qué hace | Ejemplo |
|---|---|---|
| `th:text` | Pone el texto de la etiqueta (escapa el HTML) | `<td th:text="${p.codigo}">1</td>` |
| `th:utext` | Igual, pero **sin** escapar el HTML (ojo con XSS) | `th:utext="${html}"` |
| `th:each` | Bucle `for-each` | `<tr th:each="p : ${productos}">` |
| `th:if` / `th:unless` | Pinta la etiqueta solo si se cumple o no se cumple la condición | `th:if="${#lists.isEmpty(productos)}"` |
| `th:switch` / `th:case` | `switch` (`th:case="*"` hace de `default`) | `<p th:case="'admin'">` |
| `th:href` / `th:src` / `th:action` | URL de un enlace, imagen o formulario | `th:action="@{/productos/guardar}"` |
| `th:value` | Atributo `value` | `<option th:value="${f.codigo}">` |
| `th:object` | Fija el objeto del formulario (para usar `*{}`) | `th:object="${productoForm}"` |
| `th:field` | Enlaza un campo del formulario con una propiedad (`id` + `name` + `value`) | `th:field="*{precio}"` |
| `th:errors` | Muestra los errores de validación de un campo | `th:errors="*{nombre}"` |
| `th:classappend` | **Añade** una clase CSS sin borrar las que ya tiene | `th:classappend="${...} ? 'is-invalid'"` |
| `th:class` | **Sustituye** la clase CSS | `th:class="${activo} ? 'on' : 'off'"` |
| `th:checked` / `th:selected` / `th:disabled` | Atributos booleanos | `th:disabled="${!editable}"` |
| `th:attr` | Cualquier otro atributo | `th:attr="data-id=${p.codigo}"` |
| `th:fragment` | **Define** un fragmento | `<nav th:fragment="navbar">` |
| `th:replace` / `th:insert` | **Usa** un fragmento (sección 7) | `th:replace="~{fragments/layout :: navbar}"` |
| `th:block` | Etiqueta "fantasma": hace su trabajo y desaparece del HTML | `<th:block th:replace="~{... :: scripts}">` |

---

## 4. Operadores dentro de las expresiones

| Operador | Significado | Ejemplo del proyecto |
|---|---|---|
| `cond ? a : b` | Ternario | `${fabricante.codigo == null} ? 'Nuevo fabricante' : ('Editar fabricante ' + ${fabricante.codigo})` |
| `cond ? a` | Ternario sin `else` (si es falso no hace nada) | `${#fields.hasErrors('nombre')} ? 'is-invalid'` |
| `a ?: b` | **Elvis**: `a` si no es `null`; si no, `b` | `${mensaje} ?: 'Se ha producido un error inesperado.'` |
| `+` | Concatenar | `${titulo} + ' · LogísticaApp'` |
| `\|...\|` | Sustitución literal (concatenar sin `+`) | `\|Editar producto ${p.codigo}\|` |
| `?.` | Navegación segura (no falla si es `null`) | `${p.fabricante?.nombre}` |
| `==`, `!=`, `gt`, `lt`, `ge`, `le` | Comparaciones (`gt` y `lt` evitan `<` y `>` dentro del HTML) | `${p.precio gt 100}` |
| `and`, `or`, `not` / `!` | Lógicos | `${a and not b}` |
| `_` | *No-op*: deja el texto de maqueta | `th:text="${x} ?: _"` |

Los literales de texto van entre **comillas simples**: `'Nuevo producto'`.

**¿Qué cuenta como verdadero en `th:if`?** Cualquier valor que no sea `null`, salvo `false`, el número `0` y las cadenas `"false"`, `"off"` y `"no"`. Por eso `th:if="${mensaje}"` sirve para "si hay mensaje".

---

## 5. Objetos de utilidad (`#...`)

| Objeto | Para qué | Ejemplo |
|---|---|---|
| `#lists` | Listas | `${#lists.isEmpty(productos)}`, `${#lists.size(productos)}` |
| `#numbers` | Formatear números | `${#numbers.formatDecimal(p.precio, 1, 2, 'POINT')}` → `12.50` |
| `#strings` | Cadenas | `${#strings.toUpperCase(f.nombre)}`, `${#strings.isEmpty(x)}`, `${#strings.abbreviate(x, 20)}` |
| `#fields` | Errores de validación del formulario | `${#fields.hasErrors('nombre')}`, `${#fields.hasErrors('*')}` |
| `#temporals` | Fechas `java.time` (`LocalDate`, etc.) | `${#temporals.format(fecha, 'dd/MM/yyyy')}` |
| `#dates` | `java.util.Date` | `${#dates.format(d, 'dd/MM/yyyy')}` |
| `#aggregates` | Sumas y medias | `${#aggregates.sum(productos.![precio])}` |
| `#maps`, `#sets`, `#arrays` | Otras colecciones | `${#maps.containsKey(m, 'k')}` |
| `#objects`, `#bools` | Nulos y booleanos | `${#objects.nullSafe(x, 'N/D')}` |

`#numbers.formatDecimal(num, enteros mínimos, decimales, separador decimal)`. El separador puede ser `'POINT'`, `'COMMA'` o `'DEFAULT'` (el del idioma del navegador).

Ojo: es `#lists`, en **plural**.

---

## 6. Bucles: `th:each` y variable de estado

`th:each` funciona como un `for-each` de Java: **repite la etiqueta en la que está**, incluido su contenido, una vez por cada elemento de la lista.

```java
// Controlador
model.addAttribute("productos", productoService.listar());
```

```html
<!-- Plantilla -->
<tr th:each="p : ${productos}">
    <td th:text="${p.nombre}">Nombre</td>
</tr>
```

```html
<!-- HTML generado si hay 3 productos -->
<tr><td>Teclado</td></tr>
<tr><td>Ratón</td></tr>
<tr><td>Monitor</td></tr>
```

- `p` es el elemento de cada vuelta. El nombre lo eliges tú.
- `${productos}` es la lista que llega del modelo.
- `p` solo existe dentro de la etiqueta que lleva `th:each`.
- Si quieres repetir filas, `th:each` va en el `<tr>`, no en el `<table>`.

### Variable de estado

Sirve para saber **en qué vuelta del bucle estamos**. Se declara después del elemento, separada por una coma. Aquí se llama `st`, pero puede tener otro nombre:

```html
<tr th:each="p, st : ${productos}">
    <td th:text="${st.count}">1</td>
    <td th:text="${p.nombre}">Nombre</td>
</tr>
```

Estos son sus valores con una lista de 3 productos:

| `p.nombre` | `st.index` | `st.count` | `st.size` | `st.first` | `st.last` | `st.even` | `st.odd` |
|------------|------------|------------|-----------|------------|-----------|-----------|----------|
| Teclado    | 0          | 1          | 3         | `true`     | `false`   | `false`   | `true`   |
| Ratón      | 1          | 2          | 3         | `false`    | `false`   | `true`    | `false`  |
| Monitor    | 2          | 3          | 3         | `false`    | `true`    | `false`   | `true`   |

| Propiedad         | Valor                                 |
|-------------------|---------------------------------------|
| `index`           | Posición contando desde 0             |
| `count`           | Posición contando desde 1             |
| `size`            | Total de elementos                    |
| `first` / `last`  | `true` en el primero / en el último   |
| `even` / `odd`    | `true` si `count` es par / impar      |

Ojo: `even` y `odd` dependen de `count`, no de `index`. Por eso la primera fila es impar (`odd`).

### Variable de estado implícita: `pStat`

Si no declaras la variable de estado, Thymeleaf la crea igualmente. Su nombre es el de la variable del bucle seguido de `Stat`:

```html
<!-- No declaramos st: usamos pStat directamente -->
<tr th:each="p : ${productos}">
    <td th:text="${pStat.count}">1</td>
    <td th:text="${p.nombre}">Nombre</td>
</tr>
```

| `th:each`                         | Variable de estado |
|-----------------------------------|--------------------|
| `p : ${productos}`                | `pStat`            |
| `f : ${fabricantes}`              | `fStat`            |
| `producto : ${productos}`         | `productoStat`     |
| `p, st : ${productos}`            | `st` (no `pStat`)  |

### Ejemplos

**Mostrar la posición y el total:**

```html
<p th:each="p, st : ${productos}"
   th:text="|Producto ${st.count} de ${st.size}: ${p.nombre}|">Producto</p>
```

Resultado: `Producto 1 de 3: Teclado`.

**Colorear las filas impares con `pStat`:**

```html
<tr th:each="p : ${productos}"
    th:classappend="${pStat.odd} ? 'table-secondary'">
    <td th:text="${p.nombre}">Nombre</td>
</tr>
```

La variable de estado ya se puede usar en la misma etiqueta que lleva `th:each`.

**Destacar el primer elemento:**

```html
<li th:each="p, st : ${productos}">
    <span th:text="${p.nombre}">Nombre</span>
    <strong th:if="${st.first}">(primero de la lista)</strong>
</li>
```

**Separar con comas, sin coma al final:**

```html
<span th:each="f : ${fabricantes}">
    <span th:text="${f.nombre}">Fabricante</span><span th:unless="${fStat.last}">, </span>
</span>
```

Resultado: `Acme, Bosch, Makita`.

### Tamaño de la lista

| Expresión                   | Dónde se puede usar                                              |
|-----------------------------|------------------------------------------------------------------|
| `${st.size}`                | Solo dentro del `th:each`, porque `st` no existe fuera del bucle |
| `${productos.size()}`       | En cualquier parte de la plantilla                               |
| `${#lists.size(productos)}` | En cualquier parte de la plantilla                               |

Las dos últimas leen directamente la lista del modelo, así que no dependen del bucle:

```html
<!-- Fuera del bucle: st no existe -->
<p th:text="|Total: ${productos.size()} productos|">Total</p>
<p th:text="|Total: ${#lists.size(productos)} productos|">Total</p>

<!-- Dentro del bucle: las tres opciones dan el mismo resultado -->
<tr th:each="p, st : ${productos}">
    <td th:text="|${st.count} de ${st.size}|">1 de 3</td>
    <td th:text="|${st.count} de ${productos.size()}|">1 de 3</td>
    <td th:text="|${st.count} de ${#lists.size(productos)}|">1 de 3</td>
</tr>
```

### Patrón "lista vacía" (`productos/lista.html`)

Si la lista está vacía, `th:each` no genera ninguna fila y la tabla se queda sin datos. Para mostrar un aviso, se añade una fila aparte:

```html
<tr th:if="${#lists.isEmpty(productos)}">
    <td colspan="6">No hay productos</td>
</tr>
<tr th:each="p : ${productos}"> ... </tr>
```

Son dos `<tr>` distintos porque, en una misma etiqueta, `th:each` se ejecuta antes que `th:if`. Si la lista está vacía, no se repite la etiqueta y el `th:if` nunca llega a evaluarse.
---

## 7. Fragmentos (`fragments/layout.html`)

**Definir** un fragmento, con o sin parámetros:

```html
<head th:fragment="head(titulo)"> ... <title th:text="${titulo} + ' · LogísticaApp'"> ... </head>
<nav  th:fragment="navbar"> ... </nav>
```

**Usarlo:**

```html
<head th:replace="~{fragments/layout :: head('Fabricantes')}"></head>
<nav  th:replace="~{fragments/layout :: navbar}"></nav>
```

- `fragments/layout` es la ruta desde `templates/`, sin `.html`.
- `::` separa la plantilla del nombre del fragmento.
- `head('Fabricantes')` pasa `'Fabricantes'` al parámetro `titulo`. También vale `head(${variable})` o `head(titulo='Fabricantes')`.
- Si el fragmento está en la misma plantilla: `~{:: nombre}` o `~{this :: nombre}`.

| | Resultado |
|---|---|
| `th:replace` | La etiqueta anfitriona **desaparece** y el fragmento ocupa su lugar |
| `th:insert` | La etiqueta anfitriona **se queda** y el fragmento va dentro |

El fragmento `scripts` usa `<th:block>` para meter varios `<script>` sin añadir una etiqueta envolvente.

---

## 8. URLs con `@{...}`

| Caso | Sintaxis | Resultado |
|---|---|---|
| Ruta fija | `@{/productos}` | `/productos` |
| Variable de ruta | `@{/productos/editar/{codigo}(codigo=${p.codigo})}` | `/productos/editar/7` |
| Parámetro de consulta | `@{/productos(fab=${f.codigo})}` | `/productos?fab=3` |
| Varios | `@{/buscar(q=${txt}, pag=2)}` | `/buscar?q=...&pag=2` |
| Recurso estático | `@{/css/estilos.css}` | `static/css/estilos.css` |

Lo que va entre paréntesis rellena los `{...}` de la ruta. Lo que sobra se añade como `?clave=valor`. En el controlador se recoge con `@PathVariable` o `@RequestParam`.

Borrar se hace con un **`POST`** (un formulario con un botón), no con un enlace `GET`:

```html
<form th:action="@{/productos/borrar/{codigo}(codigo=${p.codigo})}" method="post"
      onsubmit="return confirm('¿Seguro?');">
    <button type="submit">Borrar</button>
</form>
```

---

## 9. Formularios + validación (`productos/formulario.html`)

```html
<form th:action="@{/productos/guardar}" th:object="${productoForm}" method="post" novalidate>

    <input type="hidden" th:field="*{codigo}">          <!-- vacío = alta, relleno = edición -->

    <input type="text" th:field="*{nombre}" class="form-control"
           th:classappend="${#fields.hasErrors('nombre')} ? 'is-invalid'">
    <div class="invalid-feedback" th:errors="*{nombre}">Error</div>

    <select th:field="*{codigoFabricante}">
        <option value="">-- Selecciona --</option>
        <option th:each="f : ${fabricantes}" th:value="${f.codigo}" th:text="${f.nombre}">F</option>
    </select>
</form>
```

Claves:
- `th:object` **exige** que el controlador haya metido ese objeto en el modelo: `model.addAttribute("productoForm", new ProductoForm())`.
- `th:field="*{nombre}"` genera `id="nombre" name="nombre" value="..."`. Al volver con errores, el campo **conserva lo que escribió el usuario**.
- En un `<select>`, `th:field` marca la `<option>` correcta como `selected` sin hacer nada más.
- `th:errors` pinta los mensajes de `@NotBlank`, `@Positive`, etc. Las clases `is-invalid` e `invalid-feedback` son de Bootstrap.
- `novalidate` desactiva la validación del navegador para que se vea la del servidor.

En el controlador:

```java
@PostMapping("/guardar")
public String guardar(@Valid @ModelAttribute ProductoForm productoForm, BindingResult result, Model model, ...) {
    if (result.hasErrors()) {
        model.addAttribute("fabricantes", fabricanteService.listar()); // ¡recargar el desplegable!
        return "productos/formulario";                                 // NO redirect: se perderían los errores
    }
    ...
    return "redirect:/productos";
}
```

- `BindingResult` va **justo detrás** del objeto con `@Valid`.
- El nombre del atributo en el modelo sale del tipo de la clase: `ProductoForm` → `productoForm`.

| Más de `#fields` | |
|---|---|
| `#fields.hasErrors('campo')` | ¿Tiene errores ese campo? |
| `#fields.hasErrors('*')` | ¿Hay algún error? |
| `#fields.errors('campo')` | Lista de mensajes del campo |
| `#fields.allErrors()` | Todos los mensajes |
| `#fields.hasGlobalErrors()` | Errores que no son de un campo concreto |

---

## 10. Mensajes flash (fragmento `alertas`)

```java
redirectAttributes.addFlashAttribute("mensaje", "Producto creado con código " + id);
redirectAttributes.addFlashAttribute("error", e.getMessage());
return "redirect:/productos";
```

```html
<div th:if="${mensaje}" class="alert alert-success"><span th:text="${mensaje}">Mensaje</span></div>
<div th:if="${error}"   class="alert alert-danger"><span th:text="${error}">Error</span></div>
```

Un atributo *flash* sobrevive **a un redirect** y llega al modelo de la siguiente petición. Si recargas la página, desaparece.

---

## 11. Página de error (`error.html`)

Spring Boot usa `templates/error.html` para los errores no controlados y le pasa `status`, `error`, `path`, etc. Nuestro `GlobalExceptionHandler` le pasa `titulo` y `mensaje`. Con Elvis sirve para los dos casos:

```html
<span th:text="${titulo} ?: (${status} + ' ' + ${error})">Error</span>
```

---

## 12. Comentarios

| Sintaxis | Efecto |
|---|---|
| `<!-- ... -->` | Comentario HTML normal: llega al navegador |
| `<!--/* ... */-->` | Comentario de Thymeleaf: se **elimina** al procesar la plantilla |


