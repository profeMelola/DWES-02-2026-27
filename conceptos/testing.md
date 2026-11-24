# Testing: primer contacto

Un **test** es código que ejecuta tu código y comprueba que hace lo que debe. Si no lo hace, se pone en **rojo**.

¿Para qué sirve?
- Detectar fallos antes de que lleguen al usuario.
- Cambiar código sin miedo: si rompes algo, un test te avisa.
- Comprobar código que no has escrito tú (un compañero o una IA). **El test decide si ese código vale.**

## 1. Estructura de un test: AAA

| Fase | Pregunta | En código |
|---|---|---|
| **Arrange** (preparar) | ¿Qué necesito tener listo? | Crear objetos y datos |
| **Act** (actuar) | ¿Qué acción pruebo? | Una sola llamada |
| **Assert** (comprobar) | ¿Qué debería haber pasado? | `assertEquals`, `assertThrows`… |

Reglas de oro: **un comportamiento por test** y un nombre que se lea como una frase (`userNoPuedeBorrarAunqueConozcaLaUrl`).

## 2. Tipos de pruebas

| | Unitaria | Slice (una capa) | Integración |
|---|---|---|---|
| **Qué se prueba** | Una clase sola | Una capa de Spring | La aplicación entera |
| **Dependencias** | Simuladas (mocks) | Las demás capas simuladas | Todas reales (BD incluida) |
| **Anotación** | `@ExtendWith(MockitoExtension.class)` | `@WebMvcTest`, `@JdbcTest` | `@SpringBootTest` |
| **Velocidad** | Milisegundos | Segundos | La más lenta |
| **Si falla, el fallo está…** | En esa clase | En esa capa | En algún sitio |

**La pirámide de tests:** muchas unitarias en la base (baratas y precisas) y pocas de integración arriba (caras, pero prueban que todo encaja).

### `SeguridadTest` es una prueba de integración

```java
@SpringBootTest
@AutoConfigureMockMvc
class SeguridadTest { ... }
```

- `@SpringBootTest` arranca **la aplicación completa**: controladores, servicios, repositorios, H2 y `SecurityConfig`.
- `@AutoConfigureMockMvc` nos da `MockMvc`, que simula peticiones HTTP **sin navegador ni servidor**.
- Por eso `adminPuedeBorrar` borra **de verdad** el libro de H2: no hay nada simulado.

Para la seguridad es lo correcto: queremos comprobar que filtros, roles, CSRF, `UsuarioDetailsService` y BCrypt funcionan **juntos**.

## 3. JUnit 5

`@Test` marca un método como test. Las comprobaciones se hacen con `Assertions`:

| Assert | Comprueba que… | Ejemplo |
|---|---|---|
| `assertEquals(esperado, real)` | Son iguales | `assertEquals(3, borrados)` |
| `assertNotEquals(a, b)` | Son distintos | `assertNotEquals(0, lista.size())` |
| `assertTrue(condición)` | Es verdadero | `assertTrue(lista.isEmpty())` |
| `assertFalse(condición)` | Es falso | `assertFalse(dto.getTitulo().isBlank())` |
| `assertNull(objeto)` | Es `null` | `assertNull(libro.getEdicion())` |
| `assertNotNull(objeto)` | No es `null` | `assertNotNull(dto)` |
| `assertThrows(Excepcion.class, () -> ...)` | Lanza esa excepción | `assertThrows(RecursoNoEncontradoException.class, () -> service.buscarPorIsbn("x"))` |
| `assertDoesNotThrow(() -> ...)` | No lanza ninguna | `assertDoesNotThrow(() -> service.borrar(1L))` |
| `assertAll(...)` | Comprueba varias a la vez y avisa de todos los fallos | `assertAll(() -> assertEquals(...), () -> assertEquals(...))` |

> Ojo con el orden: **primero el valor esperado, después el real**. Al revés, el mensaje de error te despista.

### Ejemplo: una clase sin dependencias

El mapper no depende de nada: basta con un `new`. Ni Spring ni base de datos.

> **¿Pero el mapper no es un `@Component` que Spring inyecta?** Sí, y no importa. `@Component` solo le dice a Spring que cree el objeto e inyecte; sigue siendo una clase Java normal. En un test unitario lo creamos nosotros con `new`, sin arrancar Spring. Esta es otra ventaja de la inyección por constructor: puedes crear cualquier componente a mano y pasarle lo que necesite.

```java
class LibroMapperTest {

    private final LibroMapper mapper = new LibroMapper();

    @Test
    void toDtoCopiaIsbnYTitulo() {
        // Arrange
        Libro libro = new Libro();
        libro.setIsbn("9788490000076");
        libro.setTitulo("Rayuela");

        // Act
        LibroDto dto = mapper.toDto(libro);

        // Assert
        assertEquals("9788490000076", dto.getIsbn());
        assertEquals("Rayuela", dto.getTitulo());
    }
}
```

## 4. Mockito: simular dependencias

`AutorService` necesita `AutorRepository` y `LibroRepository`, que a su vez necesitan H2. Si queremos probar **solo el servicio**, sustituimos los repositorios por **mocks**: objetos falsos que no tocan la base de datos y responden lo que les digamos. Son como el doble de un actor: siguen el guion.

| Qué | Para qué | Código |
|---|---|---|
| `@Mock` | Crear el doble | `@Mock AutorRepository autorRepository;` |
| `@InjectMocks` | Crear la clase **real** y pasarle los dobles por el constructor | `@InjectMocks AutorService autorService;` |
| `when(...).thenReturn(...)` | Darle guion: "si te piden esto, devuelve aquello" | `when(libroRepository.countByAutorId(1L)).thenReturn(3L);` |
| `when(...).thenThrow(...)` | Que lance una excepción | `when(repo.findById(99L)).thenThrow(new AccesoDatosException("..."));` |
| `verify(mock).metodo(...)` | Comprobar que se le llamó | `verify(autorRepository).delete(1L);` |
| `verify(mock, never())` | Comprobar que **no** se le llamó | `verify(autorRepository, never()).save(any());` |
| `verify(mock, times(n))` | Que se le llamó n veces | `verify(libroRepository, times(1)).countByAutorId(1L);` |
| `any()` | "Con cualquier argumento" | `verify(repo).save(any());` |

```java
@ExtendWith(MockitoExtension.class)
class AutorServiceTest {

    @Mock AutorRepository autorRepository;     // doble
    @Mock LibroRepository libroRepository;     // doble
    @InjectMocks AutorService autorService;    // clase REAL que probamos

    @Test
    void borrarDevuelveCuantosLibrosTenia() {
        // Arrange: guion del doble
        when(libroRepository.countByAutorId(1L)).thenReturn(3L);

        // Act
        long borrados = autorService.borrar(1L);

        // Assert: el resultado y la interacción
        assertEquals(3, borrados);
        verify(autorRepository).delete(1L);
    }
}
```

> **Lo único real es `AutorService`; todo lo que tiene alrededor es atrezo.** Si el test falla, el fallo está en el servicio.

### ¿Y si el servicio también recibe el mapper?

`@InjectMocks` le pasa al constructor **todos** los dobles declarados en el test. Si el servicio recibe un mapper, tienes dos opciones:

| Opción | Código | Cuándo |
|---|---|---|
| Mapper **real** | `@Spy LibroMapper libroMapper = new LibroMapper();` | Lo normal: el mapper es sencillo y no toca la BD |
| Mapper **falso** | `@Mock LibroMapper libroMapper;` + `when(libroMapper.toDto(libro)).thenReturn(dto);` | Si quieres aislar el servicio por completo |

Si no declaras ninguno de los dos, `@InjectMocks` le pasa `null` y el test falla con `NullPointerException`.

## 5. TDD: primero el test

**Test Driven Development**: escribir el test **antes** que el código.

1. 🔴 **Rojo:** escribes el test de algo que todavía no existe. Falla.
2. 🟢 **Verde:** escribes el **mínimo** código para que pase.
3. 🔵 **Refactor:** mejoras el código; los tests te avisan si rompes algo.

Ejemplo. Nueva regla: *no se puede crear un autor con el nombre vacío*.

```java
@Test
void crearAutorConNombreEnBlancoLanzaExcepcion() {
    Autor autor = new Autor();
    autor.setNombre("   ");

    assertThrows(IllegalArgumentException.class, () -> autorService.crear(autor));
    verify(autorRepository, never()).save(any());   // y no llega a la BD
}
```

Hoy falla (rojo), porque `crear` guarda sin quejarse. Añade la comprobación en el servicio y pasará a verde.

**El test es la especificación escrita antes del código.** Seguiremos con testing y TDD en la UT03 sobre la API REST.

## Imports que pueden confundir

| Qué | Import correcto | Trampa del autocompletado |
|---|---|---|
| `@Test` | `org.junit.jupiter.api.Test` | `org.junit.Test` (JUnit 4) |
| `assertEquals`, `assertThrows`… | `static org.junit.jupiter.api.Assertions.*` | `org.junit.Assert` (JUnit 4), `junit.framework.Assert` |
| `@ExtendWith` | `org.junit.jupiter.api.extension.ExtendWith` | — |
| `MockitoExtension` | `org.mockito.junit.jupiter.MockitoExtension` | — |
| `@Mock`, `@InjectMocks` | `org.mockito.Mock`, `org.mockito.InjectMocks` | `@MockBean` (ya no existe en Spring Boot 4) |
| `when`, `verify`, `never`, `times` | `static org.mockito.Mockito.*` | — |
| `any()` | `static org.mockito.ArgumentMatchers.any` | `org.hamcrest.Matchers.any` |
| `@AutoConfigureMockMvc` | `org.springframework.boot.webmvc.test.autoconfigure` | Paquete de Spring Boot 3 (`boot.test.autoconfigure.web.servlet`) |
| `get()`, `post()` | `...test.web.servlet.request.MockMvcRequestBuilders` | `MockMvcResultMatchers` (ahí están `status()`, `redirectedUrl()`) |
