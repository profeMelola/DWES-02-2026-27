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

```html
<tr th:each="p, st : ${productos}">
    <td th:text="${st.count}">1</td>          <!-- 1, 2, 3... -->
    <td th:text="${p.nombre}">Nombre</td>
</tr>
```

| Propiedad de `st` | Valor |
|---|---|
| `index` / `count` | Posición desde 0 / desde 1 |
| `size` | Total de elementos |
| `first` / `last` | `true` en el primero / el último |
| `even` / `odd` | Fila par / impar |

Si no declaras la variable de estado, existe igualmente con el nombre `pStat` (variable + `Stat`).

**Patrón "lista vacía"** (`productos/lista.html`):

```html
<tr th:if="${#lists.isEmpty(productos)}">
    <td colspan="6">No hay productos</td>
</tr>
<tr th:each="p : ${productos}"> ... </tr>
```

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

---

## 13. Errores típicos en el examen

- `#list` en lugar de `#lists`.
- Comillas dobles dentro de un `th:` que ya va entre dobles: usa simples, `'texto'`.
- Olvidar `th:` (`href="@{...}"` no se procesa).
- `th:object` con un objeto que no está en el modelo → error `Neither BindingResult nor plain target object...`.
- Hacer `redirect` cuando hay errores de validación: se pierden los errores y lo que había escrito el usuario.
- No recargar en el modelo las listas del desplegable al volver al formulario con errores.
- Usar `<` o `>` en una expresión: mejor `lt` o `gt`.
- Olvidar `~{...}` en `th:replace`: funciona, pero está obsoleto.
