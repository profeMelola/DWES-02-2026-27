# Logging con SLF4J: que la aplicación cuente lo que hace

**Logging** es dejar que la aplicación escriba mensajes sobre lo que va haciendo: qué petición llegó, qué se borró, qué error saltó. En desarrollo los ves en la consola; en producción quedan en un fichero.

| | `System.out.println` | Depurador 🐞 | Logging |
|---|---|---|---|
| ¿Cuándo sirve? | Nunca en código real | En tu ordenador, mientras desarrollas | **Siempre**, también en producción |
| ¿Se puede apagar? | No: hay que borrarlo | — | Sí, por niveles, sin tocar el código |
| ¿Dice cuándo y dónde? | No | — | Sí: fecha, hora, nivel y clase |
| ¿Queda guardado? | No | No | Sí, si lo mandas a un fichero |

> En producción **no hay depurador**. Si algo falla a las 3 de la mañana, el log es lo único que te cuenta qué pasó.

## SLF4J y Logback

| Pieza | Qué es | ¿Hay que añadirla? |
|---|---|---|
| **SLF4J** | La **interfaz** con la que escribes los logs (`log.info(...)`) | No: viene con cualquier starter de Spring Boot |
| **Logback** | La **implementación** que los escribe en consola o fichero | No: es la que trae Spring Boot por defecto |

Es lo mismo que hacemos con los repositorios: **programamos contra la interfaz** (SLF4J) y la implementación (Logback) se puede cambiar sin tocar nuestro código.

## Cómo se usa

Con Lombok, basta una anotación en la clase. Crea un campo llamado `log`:

```java
@Slf4j
@Service
@RequiredArgsConstructor
public class AutorService {

    public long borrar(Long id) {
        long libros = libroRepository.countByAutorId(id);
        autorRepository.delete(id);
        log.info("Autor {} borrado junto con {} libros", id, libros);
        return libros;
    }
}
```

Sin Lombok, la línea equivalente es:

```java
private static final Logger log = LoggerFactory.getLogger(AutorService.class);
```

Salida en la consola:

```
2026-11-27T10:15:32.481+01:00  INFO 12345 --- [nio-8080-exec-3] e.d.b.service.AutorService : Autor 3 borrado junto con 2 libros
```

### Usa `{}`, no concatenes

| ❌ Mal | ✅ Bien |
|---|---|
| `log.debug("Buscando ISBN " + isbn);` | `log.debug("Buscando ISBN {}", isbn);` |

Cada `{}` se sustituye por el siguiente argumento, en orden. Si el nivel `DEBUG` está apagado, con `{}` el mensaje ni siquiera se construye.

## Niveles

De más grave a menos grave. Si configuras un nivel, ves ese y **todos los de arriba**.

| Nivel | Cuándo usarlo | Ejemplo en la biblioteca |
|---|---|---|
| `ERROR` | Algo ha fallado y no se ha podido hacer | No se puede conectar con la BD (`AccesoDatosException`) |
| `WARN` | Algo raro, pero la aplicación sigue | Se pide un ISBN que no existe (404) |
| `INFO` | Hechos importantes del negocio | Se crea, edita o borra un libro o un autor |
| `DEBUG` | Detalles para el programador | Filtros que llegan al listado, datos del DTO |
| `TRACE` | Muchísimo detalle, casi nunca | Cada fila leída del `ResultSet` |

Por defecto, Spring Boot muestra desde `INFO` hacia arriba.

### Errores: pasa la excepción como último argumento

```java
@ExceptionHandler(AccesoDatosException.class)
public String accesoDatos(AccesoDatosException e, Model model) {
    log.error("Error de acceso a datos", e);   // imprime también la traza completa
    ...
}

@ExceptionHandler(RecursoNoEncontradoException.class)
public String noEncontrado(RecursoNoEncontradoException e, Model model) {
    log.warn("Recurso no encontrado: {}", e.getMessage());
    ...
}
```

## Configuración en `application.properties`

```properties
# Nuestro código, con detalle
logging.level.es.daw.bibliotecajdbc=DEBUG

# Ver qué hace Spring Security por dentro (¡muy útil cuando sale un 403!)
logging.level.org.springframework.security=DEBUG

# Guardar también en un fichero
logging.file.name=logs/biblioteca.log
```

El nivel se configura **por paquete**: puedes poner tu código en `DEBUG` y dejar Spring en `INFO`.

> Con `org.springframework.security=DEBUG`, al hacer login o recibir un 403 ves en la consola cada filtro por el que pasa la petición y por qué se deniega. Es la misma cadena que vimos en el panel Frames del depurador.

## Qué NO hay que loguear

| Nunca | Por qué |
|---|---|
| Contraseñas (ni siquiera el hash) | El log lo leen más personas y se copia a otros sitios |
| Datos personales (DNI, teléfono, tarjeta…) | Protección de datos (RGPD) |
| Tokens o claves de API | Quien lea el log puede suplantarte |
| El objeto entero si tiene datos sensibles | `log.info("{}", usuario)` llama a `toString()`, y el de Lombok lo imprime **todo** |

> 🤖 Ojo con el código de la IA: suele poner `log.debug("Login: {}", usuario)` o loguear el DTO del formulario completo. Funciona, y puede filtrar datos sensibles.

## Práctica

1. Añade `@Slf4j` a `AutorService` y `LibroService`: un `log.info` al crear, editar y borrar.
2. En `GlobalExceptionHandler`: `log.warn` para el 404 y `log.error(..., e)` para el 500.
3. Un `log.debug` en `LibroController` con los filtros que llegan al listado.
4. Arranca y prueba. Después cambia `logging.level.es.daw.bibliotecajdbc` a `INFO`: los `DEBUG` desaparecen sin tocar el código.
5. Activa `logging.level.org.springframework.security=DEBUG`, entra como `ana` e intenta borrar un libro. Busca en la consola por qué se deniega.

## Imports que pueden confundir

| Qué | Import correcto | Trampa del autocompletado |
|---|---|---|
| `@Slf4j` | `lombok.extern.slf4j.Slf4j` | `@Log`, `@Log4j2`, `@CommonsLog` (otras librerías) |
| `Logger` | `org.slf4j.Logger` | `java.util.logging.Logger`, `org.apache.logging.log4j.Logger` |
| `LoggerFactory` | `org.slf4j.LoggerFactory` | `org.apache.logging.log4j.LogManager` |

## Resumen

1. **Logging = la memoria de la aplicación en producción**, donde no hay depurador.
2. **`@Slf4j` + `log.info("... {}", valor)`**, y el nivel se elige en `application.properties` sin tocar el código.
3. **Nunca contraseñas ni datos personales** en el log.
