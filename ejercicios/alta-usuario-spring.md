# Ejercicio 1. Alta de usuario: de Servlet/JSP a Spring MVC

> **Mismo problema, otra herramienta.** Rehacemos el [Alta de usuario de la UD1](https://github.com/profeMelola/DWES-01-2026-27/blob/main/ejercicios/alta-usuario.md) con **Spring Boot + Thymeleaf**. La funcionalidad es idéntica: lo que cambia es *quién hace cada cosa*.

## Objetivo

- Crear el primer proyecto Spring Boot con **Spring Initializr** desde IntelliJ.
- Traducir cada pieza del Servlet/JSP a su equivalente en Spring MVC.
- Sacar la lectura del fichero del controlador a un `@Service` (separar **presentación** de **lógica**).

## Lo que se pone en juego

| UD1 (Jakarta EE) | UD2 (Spring MVC) |
|---|---|
| `@WebServlet("/alta")` + `doGet()` | `@GetMapping("/alta")` |
| `doPost()` | `@PostMapping("/alta")` |
| `request.getParameter("nombre")` | `@RequestParam String nombre` |
| `request.getParameterValues("nivel")` | `@RequestParam List<String> nivel` |
| `request.setAttribute("x", v)` | `model.addAttribute("x", v)` |
| `getRequestDispatcher("/formulario.jsp").forward(...)` | `return "formulario";` |
| `formulario.jsp` (en `webapp/`) | `formulario.html` (en `resources/templates/`) |
| `<c:forEach>` / `<c:if>` / `${...}` | `th:each` / `th:if` / `th:text="${...}"` |
| `WEB-INF/datos/tecnologias.txt` | `resources/datos/tecnologias.txt` (classpath) |
| `FileUtil` estático + `init()` | `OpcionesService` (`@Service`, inyectado) |
| `try/catch` + forward a `error.jsp` | `@ExceptionHandler` → `error.html` |
| Desplegar el `.war` en Tomcat | Ejecutar el `main()` → Tomcat **embebido** |

---

## Enunciado

### 1. Crear el proyecto

IntelliJ → **New Project → Spring Boot** (Initializr):

| Campo | Valor |
|---|---|
| Name / Artifact | `alta-usuario-spring` |
| Group | `es.daw` |
| Package name | `es.daw.altausuario` |
| Type | Maven |
| JDK / Java | 25 |
| Packaging | Jar |
| Spring Boot | la última estable que ofrezca (4.1.x) |
| Dependencias | **Spring Web**, **Thymeleaf**, **Spring Boot DevTools** |

Arranca el proyecto *vacío* y abre `http://localhost:8080`. Sale la **Whitelabel Error Page**: Spring funciona, pero aún no hay ningún controlador para `/`.

### 2. Estructura a construir

```
src/main/
├── java/es/daw/altausuario/
│   ├── AltaUsuarioSpringApplication.java   ← ya lo genera Initializr
│   ├── controller/AltaController.java      ← sustituye a AltaServlet
│   ├── service/OpcionesService.java        ← sustituye a FileUtil
│   └── exception/FicheroNoEncontradoException.java
└── resources/
    ├── application.properties
    ├── datos/tecnologias.txt
    ├── datos/niveles.txt
    ├── static/css/estilos.css              ← el CSS que antes estaba dentro de cada JSP
    └── templates/
        ├── index.html
        ├── formulario.html
        ├── confirmacion.html
        └── error.html
```

### 3. Requisitos

1. `GET /` muestra la portada con el botón *"Darme de alta"*, que lleva por `GET` a `/alta`.
2. `GET /alta` muestra el formulario. Las opciones de **tecnología** y **nivel** salen de los `.txt` (ninguna escrita a mano en el HTML).
3. `POST /alta` recoge `nombre`, `email`, `tecnologia` y `nivel` (selección múltiple) y muestra la confirmación.
4. Si el `nombre` llega vacío, se vuelve al formulario con un mensaje de error **sin perder los datos introducidos** (incluida la selección de las listas).
5. El controlador **no lee ficheros**: pide las listas a `OpcionesService`.
6. Si falta un fichero de datos se muestra `error.html` con el mensaje de la excepción.

---

## Solución guiada

### Controlador a completar: `AltaController.java`

```java
@Controller
public class AltaController {

    private final OpcionesService opcionesService;

    // Inyección por constructor: NO hacemos new OpcionesService(), nos lo da Spring
    public AltaController(OpcionesService opcionesService) {
        this.opcionesService = opcionesService;
    }

    @GetMapping("/")
    public String inicio() {
        return "index";              // → templates/index.html
    }

    @GetMapping("/alta")
    public String mostrarFormulario(Model model) {
        // 1. Añadir al modelo las listas de tecnologías y niveles
        // 2. Devolver el nombre de la vista
    }

    @PostMapping("/alta")
    public String procesarFormulario(@RequestParam String nombre,
                                     @RequestParam String email,
                                     @RequestParam String tecnologia,
                                     @RequestParam(name = "nivel", required = false) List<String> niveles,
                                     Model model) {
        // 1. Si nombre está vacío → mensajeError + datos introducidos + listas → "formulario"
        // 2. Si no → datos al modelo → "confirmacion"
    }
}
```

> **¿Por qué `required = false` en `nivel`?** Igual que en la UD1: si no se marca ninguna opción de un `<select multiple>`, el navegador **no envía** el parámetro. Con `required = true` (valor por defecto) Spring respondería con un 400.

### Servicio: `OpcionesService.java`

Mismo algoritmo que `FileUtil`; solo cambia de dónde sale el `InputStream`:

```java
@Service
public class OpcionesService {

    public List<String> getTecnologias() { return leerFichero("datos/tecnologias.txt"); }
    public List<String> getNiveles()     { return leerFichero("datos/niveles.txt"); }

    private List<String> leerFichero(String ruta) {
        ClassPathResource recurso = new ClassPathResource(ruta);   // antes: sc.getResourceAsStream(...)
        if (!recurso.exists()) {
            throw new FicheroNoEncontradoException("No se encuentra el fichero " + ruta);
        }
        try (BufferedReader br = new BufferedReader(
                new InputStreamReader(recurso.getInputStream(), StandardCharsets.UTF_8))) {
            return br.lines().filter(l -> !l.isBlank()).map(String::trim).toList();
        } catch (IOException e) {
            throw new FicheroNoEncontradoException("Error leyendo " + ruta, e);
        }
    }
}
```

### Plantilla a completar: `formulario.html`

```html
<!DOCTYPE html>
<html lang="es" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>Formulario de alta</title>
    <link rel="stylesheet" th:href="@{/css/estilos.css}">
</head>
<body class="claro">
<div class="form-card">
    <h1>Formulario de alta</h1>

    <!-- Solo se pinta si existe mensajeError -->
    <div class="error-msg" th:if="..." th:text="...">Mensaje de error</div>

    <form th:action="@{/alta}" method="post">
        <label for="nombre">Nombre</label>
        <input type="text" id="nombre" name="nombre" th:value="...">

        <label for="email">Email</label>
        <input type="email" id="email" name="email" th:value="..." required>

        <label for="tecnologia">Tecnología</label>
        <select id="tecnologia" name="tecnologia">
            <option th:each="t : ${tecnologias}" th:value="..." th:text="..." th:selected="...">Tecnología</option>
        </select>

        <label for="nivel">Nivel (puedes marcar varios)</label>
        <select id="nivel" name="nivel" multiple>
            <!-- Pista: #lists.contains(lista, elemento) -->
            <option th:each="n : ${listaNiveles}" ...>Nivel</option>
        </select>

        <button type="submit">Enviar</button>
    </form>
</div>
</body>
</html>
```

> **Fijaos:** el HTML se puede abrir directamente en el navegador y se ve (con los textos de ejemplo). Los atributos `th:*` solo los entiende Thymeleaf en el servidor. Un JSP, en cambio, no se puede abrir sin servidor.

### Plantilla a completar: `confirmacion.html`

```html
<h1 th:text="|¡Te has dado de alta correctamente, ${nombre}!|">¡Alta correcta!</h1>
<dl>
    <dt>Email</dt><dd th:text="...">email</dd>
    <dt>Tecnología</dt><dd th:text="...">tecnología</dd>
    <dt>Nivel</dt><dd th:text="${#strings.listJoin(niveles, ', ')}">niveles</dd>
</dl>
<a class="volver" th:href="@{/}">&larr; Volver al inicio</a>
```

### Gestión de errores: `@ExceptionHandler`

```java
@ExceptionHandler(FicheroNoEncontradoException.class)
public String gestionarErrorFichero(FicheroNoEncontradoException e, Model model) {
    model.addAttribute("mensajeError", e.getMessage());
    return "error";
}
```

---

## Cómo probarlo

1. Ejecutar `AltaUsuarioSpringApplication` (o `./mvnw spring-boot:run`; en la solución, sin wrapper, `mvn spring-boot:run`) y abrir `http://localhost:8080`.
2. Botón → `GET /alta` → formulario con las listas cargadas desde los `.txt`.
3. Enviar con el nombre vacío → vuelve al formulario con el error y **con los datos que habías escrito**.
4. Enviar correcto → confirmación con los niveles separados por comas.
5. Añadir una línea a `tecnologias.txt` y recargar: aparece sin tocar Java ni HTML (DevTools recarga solo).
6. Renombrar `target/classes/datos/tecnologias.txt` y recargar `/alta` → `error.html`.
7. Abrir `http://localhost:8080/datos/tecnologias.txt` → **404**. Igual que `WEB-INF`: solo se sirve lo que está en `static/`.
8. Pasar los tests: `./mvnw test` (o `mvn test`).

---

## Para pensar (y comentar en clase)

**1. ¿Dónde está ahora el `HttpServletRequest`? ¿Ha desaparecido?**

 No ha desaparecido: lo maneja Spring.                                                                         
  - Spring Boot lleva dentro un Tomcat embebido, así que por debajo sigue habiendo Servlets.                                                                    
  - Hay un único Servlet, el DispatcherServlet, que recibe todas las peticiones (patrón Front Controller).                                                      
                                                                                 Cuando llega POST /alta, el DispatcherServlet:                                                               
  - Recibe el HttpServletRequest de Tomcat.                                                                                                                    
  - Busca el método que atiende esa URL (@PostMapping("/alta")).                                                                                               
  - Hace por nosotros el request.getParameter("nombre") y nos lo pasa como @RequestParam String nombre.                                                        
  - Llama a nuestro método.                                                                                                                                    
  - Con el String que devolvemos ("confirmacion") genera templates/confirmacion.html. Esto sustituye al forward.    

**2. ¿Por qué el controlador ya no tiene `try/catch`?**

La excepción es unchecked. FicheroNoEncontradoException extiende RuntimeException, así que el compilador no obliga a capturarla. 

En la UD1 era checked y había que poner try/catch en cada Servlet.                                                                                                                 
Los errores se gestionan en un solo sitio. La excepción pasa por el controlador sin que este la capture y llega al DispatcherServlet. Este la envía a GlobalExceptionHandler (@ControllerAdvice), que devuelve la vista error.                                                                    

Antes había try/catch y forward a error.jsp en cada Servlet. Ahora hay una sola clase que gestiona los errores de todos los controladores. 

**3. ¿Qué tendría que cambiar si mañana las tecnologías vienen de una base de datos? ¿Y el controlador?**

Solo cambia el servicio (y se añade la capa de datos). El controlador no cambia nada.                                                                         
                                                                             
El controlador solo llama a opcionesService.getTecnologias() y recibe una List<String>. No sabe de dónde salen los datos.                                     
                                                                               Habría que hacer esto:                                                                                                                                        
  - Añadir JPA y H2 al pom.xml.                                                                                                                                 
  - Crear la entidad Tecnologia y el repositorio TecnologiaRepository.                                                                                          
  - En OpcionesService, leer del repositorio en vez del fichero. 

**4. En `confirmacion.html`, escribe como nombre `<b>Bart</b>`. ¿Qué pasaba en el JSP con `${nombre}`? ¿Y con `th:text`?**

  **- En el JSP con ${nombre}:** la Expression Language no escapa el HTML. El texto se copia tal cual en la página, así que el navegador lo interpreta y sale Bart en negrita. Es una vulnerabilidad XSS (Cross-Site Scripting).
  
  **- En Thymeleaf con th:text:** se escapa automáticamente. Thymeleaf convierte < en &lt; y > en &gt;, así que en pantalla se ve el texto literal `<b>Bart</b>`, sin negrita y sin ejecutar nada. La opción      
  segura es la que se usa por defecto.

