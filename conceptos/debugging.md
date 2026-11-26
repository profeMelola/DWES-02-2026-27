# Debugging: ver lo que pasa dentro de la aplicación

**Depurar (debuggear)** es ejecutar la aplicación **en pausa**: la paras en la línea que quieras y miras qué valor tiene cada variable en ese momento. Después avanzas línea a línea para ver qué ocurre.

¿Por qué no usar `System.out.println`?

| `System.out.println` | Depurador |
|---|---|
| Muestra solo lo que tú has decidido imprimir | Ves **todas** las variables y objetos |
| Hay que tocar el código y volver a arrancar | No tocas el código |
| Luego hay que borrarlos | No queda nada en el código |
| No ves quién llamó a quién | Ves la pila de llamadas completa |

## Conceptos básicos

| Concepto | Qué es |
|---|---|
| **Breakpoint** | Punto rojo en el margen de una línea: la ejecución se para **antes** de ejecutarla |
| **Variables** | Panel con los valores en ese momento. Los objetos se despliegan para ver sus campos |
| **Frames** (pila de llamadas) | Qué método llamó a cuál hasta llegar aquí. El de arriba es el actual |
| **Arrancar en modo debug** | El botón del **bicho** 🐞 (no el ▶️). Sin él, los breakpoints no hacen nada |

## Atajos de IntelliJ

| Tecla | Acción | Para qué |
|---|---|---|
| **F8** | Step Over | Ejecuta la línea y pasa a la siguiente |
| **F7** | Step Into | Entra **dentro** del método de esa línea |
| **Shift+F8** | Step Out | Termina el método actual y vuelve a quien lo llamó |
| **Alt+F9** | Run to Cursor | Avanza hasta la línea donde está el cursor |
| **F9** | Resume | Sigue hasta el siguiente breakpoint |
| **Alt+F8** | Evaluate Expression | Ejecuta código Java con la aplicación parada |
| **Ctrl+F8** | Toggle Breakpoint | Pone o quita un breakpoint en la línea |

---

## Ejemplo 1 · Editar un libro de punta a punta

**Objetivo:** ver en vivo cómo viajan los datos por las capas.

1. Breakpoint en la primera línea de `LibroController.guardar`.
2. Arranca en modo debug, entra como `admin`, edita un libro y pulsa Guardar.
3. En **Variables**, despliega el `LibroDto`: son los datos que llegan del formulario.
4. Con **F7** entra en el mapper y mira cómo el DTO se convierte en `Libro`.
5. Sigue con **F7** hasta el servicio y después hasta el repositorio, al `PreparedStatement`.

**Qué ves:** la arquitectura por capas en directo. Entre la vista y el controlador viaja el **DTO**; entre el controlador y el servicio, el **modelo**.

## Ejemplo 2 · Crear un libro con ISBN duplicado

**Objetivo:** encontrar de dónde sale un error.

1. Menú **Run → View Breakpoints** (Ctrl+Shift+F8) → **+** → **Java Exception Breakpoints** → `IsbnDuplicadoException`.
2. En modo debug, crea un libro con un ISBN que ya exista.
3. La aplicación se para **justo donde se lanza** la excepción, en el servicio.
4. Avanza con **F8** y observa cómo la captura el controlador y hace `rejectValue`.

**Qué ves:** el camino completo desde que se lanza la excepción hasta el mensaje rojo del formulario.

## Ejemplo 3 · Borrar un autor con la consola de H2 abierta

**Objetivo:** entender por qué el servicio cuenta los libros **antes** de borrar.

1. Breakpoint en `AutorService.borrar`.
2. Abre la consola de H2 en otra pestaña (`/h2-console`).
3. En modo debug, borra un autor que tenga libros.
4. Parado en el breakpoint, consulta en H2: `SELECT * FROM libro WHERE autor_id = ?`. Los libros siguen ahí.
5. Avanza con **F8** hasta pasar el `delete` y repite la consulta: han desaparecido.

**Qué ves:** el `ON DELETE CASCADE` de la base de datos en acción. Después del borrado ya no quedaría nada que contar.

## Ejemplo 4 · Hacer login

**Objetivo:** ver que la "magia" de Spring Security es código real.

1. Breakpoint en `UsuarioDetailsService.loadUserByUsername`.
2. En modo debug, entra con `admin` / `1234`.
3. Mira el panel **Frames**: decenas de clases de Spring Security (filtros) llamando hasta llegar a **tu** método.
4. En **Variables**, mira el `Usuario` que devuelve el repositorio: la contraseña es un **hash BCrypt**, no `1234`.
5. Avanza con **F8** y mira el `UserDetails` que construye el adaptador.

**Qué ves:** quién llama a tu código y cómo el adaptador convierte tu `Usuario` en lo que entiende Spring Security.

## Destripando: el bug de los argumentos cruzados

1. En el mapper, intercambia dos campos al construir el objeto (por ejemplo, el año y la edición, o el título y el autor).
2. Arranca y edita un libro: el dato aparece mal y no se sabe por qué.
3. Breakpoint en el mapper: en **Variables** se ve al instante qué campo recibe qué valor.

---

## Extras rápidos

| Truco | Cómo | Para qué |
|---|---|---|
| **Breakpoint condicional** | Clic derecho sobre el punto rojo → escribe la condición, p. ej. `isbn.equals("9788490000076")` | En un listado de 20 libros, se para solo en el que te interesa |
| **Evaluate Expression** | **Alt+F8** con la aplicación parada → `libroDto.getTitulo().trim()` o `autorRepository.findById(1L)` | Probar código sin modificar el programa |
| **Cambiar un valor** | Clic derecho sobre una variable → **Set Value** (F2) | Forzar un caso (por ejemplo, un año inválido) sin rellenar el formulario otra vez |
| **Depurar un test** | Botón 🐞 junto al test de `SeguridadTest` | Ver dentro del controlador el usuario simulado que crea `@WithMockUser` |

---

## Resumen

1. **Breakpoint + modo debug** (🐞): la aplicación se para y ves todas las variables.
2. **F8 avanza, F7 entra, Shift+F8 sale.**
3. Antes de llenar el código de `System.out.println`, **pon un breakpoint**.
