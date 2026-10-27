# Patrón PRG (Post / Redirect / Get)

PRG significa **Post / Redirect / Get**. Es la forma de terminar un `POST` que ha modificado datos: en vez de devolver una página directamente, se manda al navegador a otra URL que se pide con `GET`.

En el proyecto ProductosJDBC se aplica en el endpoint `POST /productos/guardar` de `ProductoController`:

```java
@PostMapping("/guardar")
public String guardar(@Valid @ModelAttribute ProductoForm productoForm,
                      BindingResult result,
                      Model model,
                      RedirectAttributes redirectAttributes) {

    if (result.hasErrors()) {
        // Volvemos al formulario: hay que recargar el desplegable
        model.addAttribute("fabricantes", fabricanteService.listar());
        return "productos/formulario";
    }

    if (productoForm.getCodigo() == null) {
        Producto creado = productoService.crear(productoForm.aProducto());
        redirectAttributes.addFlashAttribute("mensaje",
                "Producto creado con código " + creado.getCodigo());
    } else {
        productoService.actualizar(productoForm.aProducto());
        redirectAttributes.addFlashAttribute("mensaje",
                "Producto " + productoForm.getCodigo() + " actualizado");
    }

    // PRG (Post/Redirect/Get): si el usuario pulsa F5 no se reenvía el formulario
    return "redirect:/productos";
}
```

## El problema que resuelve

Imagina que `guardar` terminara así:

```java
model.addAttribute("productos", productoService.listar());
return "productos/lista";   // renderiza la vista directamente
```

El flujo sería este:

```
Navegador ── POST /productos/guardar (codigo=null, nombre=...) ──▶ Servidor
Navegador ◀── 200 OK + HTML de la lista ─────────────────────── Servidor
```

La barra de direcciones se queda en `/productos/guardar` y la última petición del navegador es ese **POST**. Si el usuario pulsa **F5**, el navegador avisa con "¿Reenviar el formulario?" y, si acepta, **vuelve a mandar el mismo POST**. Como `codigo` es `null`, se llama otra vez a `productoService.crear(...)` y se **crea un producto duplicado**.

Lo mismo pasa con los botones "Atrás" y "Adelante", o si alguien guarda esa URL en favoritos.

## Con PRG

```java
return "redirect:/productos";
```

```
1. Navegador ── POST /productos/guardar ──────────▶ Servidor   (crea el producto)
2. Navegador ◀── 302 Found, Location: /productos ── Servidor
3. Navegador ── GET /productos ───────────────────▶ Servidor   (listar())
4. Navegador ◀── 200 OK + HTML de la lista ──────── Servidor
```

- El prefijo `redirect:` hace que Spring no busque una plantilla. Responde con un **302** y la cabecera `Location`.
- El navegador hace por su cuenta un **GET** nuevo a `/productos`.
- Ahora la URL es `/productos` y la última petición es un GET. **F5 solo vuelve a cargar el listado**, que no modifica nada y se puede repetir sin problema.

## `RedirectAttributes` y los *flash attributes*

Hay una pega: el redirect es **otra petición**, así que lo que metas en el `Model` del POST se pierde. Para mostrar "Producto creado con código 7", el mensaje tiene que sobrevivir al salto, y para eso está:

```java
redirectAttributes.addFlashAttribute("mensaje", "Producto creado con código " + creado.getCodigo());
```

1. Spring guarda `mensaje` temporalmente en la **sesión** antes de mandar el 302.
2. En el GET a `/productos`, lo saca de la sesión, lo mete en el `Model` y **lo borra**.
3. La vista `lista` lo muestra con `${mensaje}`.
4. Si haces F5 en el listado, el mensaje ya no aparece, porque solo dura una petición (de ahí el nombre "flash").

> `addAttribute`, sin "Flash", haría algo distinto: lo pondría en la URL como parámetro (`/productos?mensaje=...`).

## ¿Y cuando hay errores de validación?

```java
if (result.hasErrors()) {
    model.addAttribute("fabricantes", fabricanteService.listar());
    return "productos/formulario";   // sin redirect
}
```

Aquí **no** se aplica PRG, y está bien así:

- No se ha guardado nada, así que reenviar el POST no causa duplicados (la validación volvería a fallar).
- Hace falta renderizar en **la misma petición** para no perder lo que el usuario escribió (`productoForm`) ni los errores de `BindingResult`. Con un redirect se perderían.

## Resumen

| Situación                         | Qué devuelve el POST      | Por qué                                              |
|-----------------------------------|---------------------------|------------------------------------------------------|
| Los datos se guardan bien         | `redirect:/productos`     | F5 no repite la operación; el mensaje va como flash  |
| Falla la validación               | `productos/formulario`    | Se conservan los datos escritos y los errores        |

El endpoint `POST /productos/borrar/{codigo}` sigue el mismo patrón por la misma razón: modifica datos, así que termina en `redirect:`.
