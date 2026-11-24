# Práctica guiada: securizar BibliotecaJDBC con Spring Security

> UT02 · RA4 — Sesiones, autenticación y autorización
> Teoría: [conceptos/spring-security.md](../conceptos/spring-security.md)
> Proyecto de partida: [BibliotecaJDBC del examen de la 1ª evaluación](./refuerzo/EXAMEN-1ev-ALUMNO.zip) (sin resolver)

## Objetivo

Partimos de BibliotecaJDBC, una aplicación que funciona pero **no tiene seguridad**: cualquiera puede dar de alta, editar o borrar. Al terminar:

| Quién | Qué puede hacer |
|---|---|
| Anónimo (sin login) | Ver la portada. Todo lo demás → al login. |
| `ana` (USER) | Consultar y filtrar libros y autores. |
| `admin` (ADMIN) | Además, dar de alta, editar y borrar. |
| `luis` (USER, desactivado) | Nada: no puede iniciar sesión. |

La contraseña de los tres usuarios es `1234`.

## Pasos

| Paso | Qué haces | Qué aprendes |
|---|---|---|
| [0](#paso-0-destripar-la-aplicación-sin-seguridad) | Destripar la aplicación sin seguridad | Por qué hace falta |
| [1](#paso-1-añadir-solo-la-dependencia) | Añadir solo la dependencia | Comportamiento por defecto |
| [2](#paso-2-securityfilterchain-y-login-propio) | `SecurityFilterChain` y login propio | Rutas públicas, login, logout, CSRF |
| [3](#paso-3-nuestros-usuarios-y-nuestro-userdetailsservice) | Nuestros usuarios y nuestro `UserDetailsService` | Patrón Builder, BCrypt, modelo, repositorio y adaptador (SRP, ISP, DIP) |
| [4](#paso-4-autorización-admin-y-user) | Autorización ADMIN / USER | `hasRole`, orden de las reglas, 403 |
| [5](#paso-5-vistas-secauthorize-y-página-403) | Vistas: `sec:authorize` y página 403 | Ocultar no es proteger |
| [6](#paso-6-tests-con-mockmvc-y-withmockuser) | Tests con MockMvc y `@WithMockUser` | Comprobar la seguridad sin navegador |

> Como en el resto del material, el código Java va **sin `package` ni `import`**. Cuando IntelliJ te ofrezca varias clases con el mismo nombre, consulta la [tabla de imports](#imports-que-pueden-confundir) del final.

---

## Paso 0: destripar la aplicación sin seguridad

Arranca el proyecto tal cual y comprueba lo que puede hacer **cualquiera**.

1. Abre `http://localhost:8080/libros`. Sin identificarte, edita un libro y borra otro.
2. Ahora sin usar la aplicación: escribe directamente en el navegador `http://localhost:8080/libros/editar/9788490000496`. Se abre el formulario de *Rayuela*.
3. Borra un libro **sin abrir el navegador**, desde un terminal:

```bash
curl -i -X POST http://localhost:8080/libros/borrar/9788490000076
```

Recarga `/libros`: *Bestiario* ha desaparecido.

> **Para pensar.** El botón de borrar está en un formulario POST con confirmación JavaScript. ¿Sirve de algo esa confirmación frente al `curl`? ¿Qué le falta a la aplicación para impedirlo?

---

## Paso 1: añadir solo la dependencia

Lo mejor es añadir el starter:

<img width="224" height="172" alt="imagen" src="https://github.com/user-attachments/assets/99158c8d-3493-4d53-be4f-be84233ae683" />


O añadir directamente en el `pom.xml`, dentro de `<dependencies>`:

```xml
<!-- Spring Security -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

Recarga Maven y arranca. **No escribas ni una línea de código.** Observa:

| Prueba | Qué pasa |
|---|---|
| Abrir `http://localhost:8080/` | Te lleva a un **login generado** por Spring, en `/login`. Hasta la portada está protegida. |
| Consola de IntelliJ | `Using generated security password: 3f2a...`. El usuario es `user`. La contraseña **cambia en cada arranque**. |
| Entrar con `user` y esa contraseña | La aplicación funciona entera, también editar y borrar. |
| Repetir el `curl` del paso 0 con otro ISBN | El libro **ya no se borra**. Anota el código de estado que devuelve. |
| Borrar desde el botón de la aplicación | **Sí funciona**: Thymeleaf añade solo el token CSRF a los formularios con `th:action`. |
| `http://localhost:8080/h2-console` | Bloqueada. Lo arreglamos en el paso 2. |

> **Para pensar.** El `curl` falla por dos motivos a la vez: no tiene sesión y no envía token CSRF. ¿Cuál de los dos comprueba antes Spring Security? (Pista: mira el orden de los [filtros en la teoría](https://github.com/profeMelola/DWES-02-2026-27/blob/main/conceptos/spring-security.md#6-la-cadena-de-filtros).)

Esto es lo que hace Spring Security **por defecto**: todo cerrado. A partir de aquí, nuestro trabajo es **abrir** lo público y **decidir** quién puede hacer qué.

---

## Paso 2: `SecurityFilterChain` y login propio

### 2.1 La configuración de seguridad

Crea `config/SecurityConfig.java`:

```java
@Configuration
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
                // AUTORIZACIÓN: qué rutas son públicas
                .authorizeHttpRequests(auth -> auth
                        // Públicas: portada, página de error y recursos estáticos
                        .requestMatchers("/", "/error", "/css/**", "/vendor/**").permitAll()
                        // Consola H2: solo para desarrollo
                        .requestMatchers("/h2-console/**").permitAll()
                        // Cualquier otra cosa: estar identificado
                        .anyRequest().authenticated()
                )
                // AUTENTICACIÓN: formulario de login propio
                .formLogin(form -> form
                        .loginPage("/login")              // GET /login: nuestra vista
                        .defaultSuccessUrl("/libros")     // a dónde ir tras entrar
                        .permitAll()
                )
                // LOGOUT: POST /logout (con token CSRF)
                .logout(logout -> logout
                        .logoutSuccessUrl("/login?logout")
                        .permitAll()
                )
                // La consola H2 no envía token CSRF y se muestra dentro de un <iframe>
                .csrf(csrf -> csrf.ignoringRequestMatchers("/h2-console/**"))
                .headers(headers -> headers.frameOptions(frame -> frame.sameOrigin()));

        return http.build();
    }
}
```
> **¿Cómo se lee esta configuración?** `http` funciona como un builder: vamos encadenando bloques y al final `http.build()` construye la cadena de filtros.
>
> Cada bloque configura una parte de la seguridad (un filtro):
>
> | Bloque | Qué configura |
> |---|---|
> | `authorizeHttpRequests(auth -> ...)` | Quién puede entrar a cada URL |
> | `formLogin(form -> ...)` | El formulario de login |
> | `logout(logout -> ...)` | El cierre de sesión |
> | `csrf(csrf -> ...)` | La protección CSRF |
> | `headers(headers -> ...)` | Las cabeceras de seguridad |
>
> Dentro de cada bloque, `form`, `logout`, `csrf`... son el objeto que configura esa parte. Lo que escribes dentro de los paréntesis solo afecta a ese bloque.
>
> Si en internet ves el estilo antiguo, con `.and()` entre bloques, no lo copies: ya no compila en Spring Security 7.

> **¿Qué es `@Bean`?** Hasta ahora, para que Spring gestionara una clase nuestra le poníamos `@Component`, `@Service`, `@Repository` o `@Controller`. Pero `SecurityFilterChain` o `BCryptPasswordEncoder` son clases **de Spring Security**: no podemos abrirlas para anotarlas. Además, hay que configurarlas antes de usarlas.
>
> La solución es un **método** anotado con `@Bean` dentro de una clase `@Configuration`. Spring lo ejecuta **una vez** al arrancar, guarda el objeto que devuelve en su contenedor y lo inyecta donde se necesite, igual que cualquier otro componente.
>
> | | `@Component` / `@Service` / `@Repository` | `@Bean` |
> |---|---|---|
> | Se pone en | La **clase** | Un **método** de una clase `@Configuration` |
> | Quién crea el objeto | Spring, con el constructor | Nosotros, en el método (`return new ...`) |
> | Cuándo se usa | Clases **nuestras** | Clases **de terceros** o que hay que configurar al crearlas |
> | Ejemplo en el proyecto | `LibroService`, `AutorRepositoryJdbc` | `SecurityFilterChain`, `PasswordEncoder` |
>
> El resultado es el mismo: un componente gestionado por Spring que se puede inyectar.

| Bloque | Filtro que configura | Qué hace aquí |
|---|---|---|
| `authorizeHttpRequests` | `AuthorizationFilter` | Abre la portada, `/error`, los CSS, Bootstrap y la consola H2. El resto exige login. |
| `formLogin` | `UsernamePasswordAuthenticationFilter` | `GET /login` muestra **nuestra** vista; `POST /login` lo procesa Spring. `permitAll()` abre `/login`, si no nadie podría entrar. |
| `logout` | `LogoutFilter` | `POST /logout` cierra la sesión y redirige a `/login?logout`. |
| `csrf` y `headers` | `CsrfFilter` y cabeceras | Excepciones **solo** para la consola H2. CSRF sigue activo para la aplicación. |

> **¿Por qué `/vendor/**`?** Ahí están Bootstrap y sus iconos (en local, para trabajar sin internet). Si no los abres, la página de login se verá sin estilos: el navegador pide el CSS y Spring Security lo redirige al login.

> ⚠️ La consola H2 abierta y sin CSRF es **solo para desarrollo**. En producción no existiría.


### Otras configuraciones habituales en `SecurityConfig`

Casos que no usamos en la práctica, pero que verás en proyectos reales.

| Caso | Bloque | Configuración |
|---|---|---|
| El campo del formulario se llama `email`, no `username` | `formLogin` | `.usernameParameter("email")` |
| Ir **siempre** a `/libros` tras entrar, aunque el usuario viniera de otra página | `formLogin` | `.defaultSuccessUrl("/libros", true)` |
| Si falla el login, ir a otra URL en vez de `/login?error` | `formLogin` | `.failureUrl("/login?fallo")` |
| Volver a la portada tras salir | `logout` | `.logoutSuccessUrl("/")` |
| Borrar cookies al salir | `logout` | `.deleteCookies("JSESSIONID")` |
| Regla distinta según el método HTTP (GET libre, POST solo ADMIN) | `authorizeHttpRequests` | `.requestMatchers(HttpMethod.GET, "/libros/**").permitAll()` y `.requestMatchers(HttpMethod.POST, "/libros/**").hasRole("ADMIN")` |
| Toda una zona de administración solo para ADMIN | `authorizeHttpRequests` | `.requestMatchers("/admin/**").hasRole("ADMIN")` |
| Cerrar todo lo no previsto (más estricto que `authenticated()`) | `authorizeHttpRequests` | `.anyRequest().denyAll()` al final |
| Cambiar el tiempo de caducidad de la sesión | — | No va en `SecurityConfig`: `server.servlet.session.timeout=15m` en `application.properties` |

### 2.2 La vista de login (sin controlador)

La vista no tiene lógica, así que va en `WebConfig`, igual que la portada:

```java
@Override
public void addViewControllers(ViewControllerRegistry registry) {
    registry.addViewController("/").setViewName("index");
    registry.addViewController("/login").setViewName("login");
}
```

Crea `templates/login.html`:

```html
<!DOCTYPE html>
<html lang="es" xmlns:th="http://www.thymeleaf.org">
<head th:replace="~{fragments/layout :: head('Iniciar sesión')}"></head>
<body>
<nav th:replace="~{fragments/layout :: navbar}"></nav>

<div class="container" style="max-width: 420px;">

  <h2 class="titulo-seccion mb-4"><i class="bi bi-box-arrow-in-right"></i> Iniciar sesión</h2>

  <!-- /login?error: lo añade Spring Security si las credenciales fallan -->
  <div th:if="${param.error}" class="alert alert-danger">
    <i class="bi bi-exclamation-triangle"></i> Usuario o contraseña incorrectos
  </div>
  <!-- /login?logout: lo añadimos nosotros en logoutSuccessUrl -->
  <div th:if="${param.logout}" class="alert alert-success">
    <i class="bi bi-check-circle"></i> Has cerrado la sesión
  </div>

  <!-- th:action añade el token CSRF. Los nombres username y password los espera Spring -->
  <form th:action="@{/login}" method="post" class="card card-body shadow-sm">
    <div class="mb-3">
      <label for="username" class="form-label">Usuario</label>
      <input type="text" id="username" name="username" class="form-control" required autofocus>
    </div>
    <div class="mb-3">
      <label for="password" class="form-label">Contraseña</label>
      <input type="password" id="password" name="password" class="form-control" required>
    </div>
    <button type="submit" class="btn btn-primary w-100">Entrar</button>
  </form>

</div>

<th:block th:replace="~{fragments/layout :: scripts}"></th:block>
</body>
</html>
```

Detalles importantes:

- El formulario va a `POST /login`, pero **no escribimos ningún controlador** para eso: lo atiende el filtro de Spring Security.
- Los campos **tienen que llamarse** `username` y `password`.
- Mensaje de error genérico, sin decir si falla el usuario o la contraseña: así no revelamos qué usuarios existen.

### 2.3 Botón de salir (provisional)

Para poder cerrar sesión mientras probamos, añade en `fragments/layout.html`, dentro de la barra, justo después del `<div class="navbar-nav">...</div>`:

```html
<!-- Logout por POST: así lo protege el token CSRF -->
<form th:action="@{/logout}" method="post" class="ms-auto m-0">
  <button type="submit" class="btn btn-outline-light btn-sm">
    <i class="bi bi-box-arrow-right"></i> Salir
  </button>
</form>
```

> **¿Por qué un formulario y no un enlace?** Porque un enlace es GET y CSRF solo protege POST, PUT y DELETE. Con un GET, cualquier web podría cerrarte la sesión con algo tan simple como `<img src="http://localhost:8080/logout">`.

En el paso 5 este botón solo se verá cuando haya alguien identificado.

### Comprueba

| Prueba | Resultado esperado |
|---|---|
| `http://localhost:8080/` | Portada visible y con estilos, sin login. |
| Pulsar "Gestionar libros" | Redirige a **nuestro** login. |
| Entrar con `user` y la contraseña de la consola | Llegas a `/libros`. |
| Pulsar "Salir" | Vuelves al login con "Has cerrado la sesión". |
| Contraseña incorrecta | "Usuario o contraseña incorrectos". La URL es `/login?error`. |
| `/h2-console` | Accesible de nuevo (JDBC URL: `jdbc:h2:mem:bibliotecadb`). |

> **Para pensar.** Abre las herramientas del navegador (F12 → Aplicación → Cookies) y mira el valor de `JSESSIONID` antes y después de hacer login. ¿Cambia? ¿Por qué lo hace Spring Security?

---

## Paso 3: nuestros usuarios y nuestro `UserDetailsService`

El usuario `user` con contraseña aleatoria no sirve. Vamos a tener **nuestros propios usuarios**, organizados por capas como el resto de la aplicación: un modelo `Usuario`, un repositorio con interfaz y un servicio que **adapta** nuestro `Usuario` a lo que espera Spring Security.

```
UsuarioDetailsService (service)  ──usa──▶  UsuarioRepository (interfaz)
        │                                          ▲
        │ devuelve User (UserDetails)              │ implementa
        ▼                                          │
  Spring Security                       UsuarioRepositoryMemoria (lista en memoria)
```

> **¿Y la base de datos?** De momento los usuarios están en una lista. En la UT04, con JPA, crearemos otra implementación de `UsuarioRepository` que lea de la BD. **Solo cambiará esa clase**: el servicio y la configuración de seguridad no se tocarán.

### 3.0 Antes de nada: refactorizar a Builder

Vamos a crear un modelo nuevo, `Usuario`. Antes, arreglamos cómo se crean los objetos en el resto del proyecto.

> Teoría: [Builder: crear objetos con muchos atributos](https://github.com/profeMelola/DWES-02-2026-27/blob/main/conceptos/SOLID-y-patrones.md#builder-crear-objetos-con-muchos-atributos)

#### Refactoriza

1. Añade `@Builder` a `Libro` y a `Autor`. **No quites** `@NoArgsConstructor` ni `@AllArgsConstructor` (en la teoría tienes por qué).
2. Busca todos los `new Libro(` y `new Autor(` del proyecto (`Ctrl + Shift + F`) y cámbialos por el builder.

Ejemplo resuelto, `AutorRepositoryJdbc.mapRow`:

```java
// Antes
return new Autor(rs.getInt("id"), rs.getString("nombre"));

// Después
return Autor.builder()
        .id(rs.getInt("id"))
        .nombre(rs.getString("nombre"))
        .build();
```

El resto lo haces tú...

> **Para pensar.** Con el builder, ¿qué pasa si en el mapper te olvidas de poner `.titulo(...)`? ¿Compila? ¿Qué valor tiene el título? `LibroDto` tiene `@NotBlank` en el título: ¿te salva? ¿Quién lo impide entonces?

<details>
<summary>Respuesta</summary>

Compila sin problema y el título queda a `null` (el valor por defecto de un `String`). `@NotBlank` **no te salva**: valida lo que escribe el usuario en el formulario, y el formulario sí traía título. El error es del programador, en el mapper.

Lo impide la base de datos, porque la columna es `NOT NULL`: el `INSERT` falla y el usuario ve una página de error en lugar de un mensaje en el formulario.

Moraleja: el builder no obliga a rellenar nada. Hay que rellenar todos los atributos, y la base de datos es la última barrera, no la primera.

</details>

### 3.1 El `PasswordEncoder`

Añade a `SecurityConfig` este bean:

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();
}
```

Spring lo usa en cada login para comparar la contraseña escrita con el hash guardado (`matches`). Nunca guardamos contraseñas en claro: solo su hash BCrypt (ver el apartado 5 de la teoría).

### 3.2 El modelo

`model/Usuario.java`:

```java
/**
 * Usuario de la biblioteca.
 * POJO sin dependencias de Spring Security: NO implementa UserDetails.
 * La traducción a UserDetails la hace UsuarioDetailsService (SRP).
 */
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Usuario {

    private Integer id;
    private String username;
    private String password;   // hash BCrypt
    private String rol;        // ADMIN o USER, sin prefijo ROLE_
    private boolean activo;
}
```

### 3.3 El repositorio: la interfaz

`repository/UsuarioRepository.java`:

```java
/**
 * Solo lo que necesita la autenticación.
 * No extiende GenericRepository: nadie da de alta ni borra usuarios desde la aplicación (ISP).
 */
public interface UsuarioRepository {

    Optional<Usuario> findByUsername(String username);
}
```

> **[SOLID: principio de segregación de interfaces (ISP)](https://github.com/profeMelola/DWES-02-2026-27/blob/main/conceptos/SOLID-y-patrones.md#i--interface-segregation-interfaces-peque%C3%B1as).** `LibroRepository` y `AutorRepository` extienden `GenericRepository` porque necesitan el CRUD completo. `UsuarioRepository` solo necesita buscar por nombre. Obligarle a implementar `save`, `update` y `delete` sería añadir métodos que nadie usa.

### 3.4 El repositorio: implementación en memoria

`repository/UsuarioRepositoryMemoria.java`:

```java
/**
 * Usuarios en memoria. En la UT04 se sustituirá por una implementación con JPA.
 * Contraseña de los tres: 1234 (guardada como hash BCrypt).
 */
@Repository
public class UsuarioRepositoryMemoria implements UsuarioRepository {

```java
    private final List<Usuario> usuarios = List.of(
            Usuario.builder()
                    .id(1).username("admin").rol("ADMIN").activo(true)
                    .password("$2a$10$FE6Xa2yaXAVUf9v7DDQoF.bjxEGeJujmNH7o5T91IUG3qjc3z05Py")
                    .build(),
            Usuario.builder()
                    .id(2).username("ana").rol("USER").activo(true)
                    .password("$2a$10$07vZe7nhH1Oehrm51orcpeer3uYX1JPi5lLBWb.HW.JMX.llYoIaK")
                    .build(),
            Usuario.builder()
                    .id(3).username("luis").rol("USER").activo(false)
                    .password("$2a$10$EQIZl.C1bGRS.QitQ.2aPuM62n.iRSiiLBqTCIUwFC21c9U9FPdyK")
                    .build()
    );

    @Override
    public Optional<Usuario> findByUsername(String username) {
        return usuarios.stream()
                .filter(u -> u.getUsername().equals(username))
                .findFirst();
    }
}
```

Fíjate:

- La contraseña de los tres es `1234`, pero los **hashes son distintos**: cada uno lleva su propia sal.
- `luis` está **desactivado** (`activo = false`).
- Los roles van **sin** prefijo `ROLE_`.

> Los hashes ya vienen generados. **Nunca** escribas la contraseña en claro en la lista: si pones `"1234"`, el login fallará y en la consola verás el aviso `Encoded password does not look like BCrypt`.

### 3.5 Nuestro `UserDetailsService`

`service/UsuarioDetailsService.java`:

```java
/**
 * Adaptador entre nuestro modelo y Spring Security:
 * busca el Usuario y lo devuelve como UserDetails.
 */
@Service
@RequiredArgsConstructor
public class UsuarioDetailsService implements UserDetailsService {

    private final UsuarioRepository usuarioRepository;

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        Usuario usuario = usuarioRepository.findByUsername(username)
                .orElseThrow(() -> new UsernameNotFoundException("No existe el usuario " + username));

        return User.builder()
                .username(usuario.getUsername())
                .password(usuario.getPassword())      // el hash: Spring lo compara con matches()
                .roles(usuario.getRol())              // ADMIN → authority ROLE_ADMIN
                .disabled(!usuario.isActivo())
                .build();
    }
}
```
> **¿Te suena `User.builder()`?** Es el mismo patrón que acabas de aplicar en el 3.0. Aquí no lo genera Lombok: Spring Security lo trae hecho en su clase `User`.

Lo que hace Spring con esta clase en cada login:

1. Llama a `loadUserByUsername("ana")`.
2. Si no existe → `UsernameNotFoundException` → `/login?error`.
3. Si existe pero está desactivado → `/login?error`.
4. Compara la contraseña escrita con el hash usando `passwordEncoder.matches(...)`.
5. Si coincide → guarda el usuario en el `SecurityContext`, y este en la sesión.

Con este servicio y el `PasswordEncoder`, Spring Boot ya sabe autenticar. **No hace falta** declarar `AuthenticationProvider` ni `AuthenticationManager`.

> **SOLID: responsabilidad única (SRP).** `Usuario` no implementa `UserDetails`. Representar un usuario de la biblioteca y adaptarlo a Spring Security son dos responsabilidades, y cada una tiene su clase. `UsuarioDetailsService` hace de **adaptador**. (Ver el apartado 7 de la teoría.)

> **SOLID: inversión de dependencias (DIP).** El servicio depende de la **interfaz** `UsuarioRepository`, no de `UsuarioRepositoryMemoria`. Por eso en la UT04 podremos cambiar la lista por una base de datos sin tocarlo.

> ⚠️ **Error típico.** Debe haber **un solo** `UserDetailsService`. Si declaras otro (por ejemplo, copiando un ejemplo de internet con `InMemoryUserDetailsManager`), Spring Boot no sabe cuál usar y **no configura la autenticación**: ningún login funcionará.

### Comprueba

> ⚠️ **¿Sigues dentro sin haber hecho login?** Si en el paso 2 entraste como `user`, tu sesión sigue viva: DevTools guarda las sesiones entre reinicios y Spring Security no vuelve a comprobar al usuario en cada petición, se fía de la sesión. Pulsa **Salir** (o usa una ventana de incógnito).

| Prueba | Resultado esperado |
|---|---|
| Arrancar | **Ya no aparece** "Using generated security password": Spring ve que tenemos nuestro propio `UserDetailsService`. |
| `admin` / `1234` | Entra. |
| `ana` / `1234` | Entra. |
| `luis` / `1234` | **No entra**: está desactivado. |
| `pepe` / `1234` | No entra: no existe. Mismo mensaje que con la contraseña mala. |
| `ana` borra un libro | **Funciona**. Lo arreglamos en el paso 4. |

> **Para pensar.** `ana` es USER y ha podido borrar. ¿Ha fallado la autenticación o la autorización? (Respuesta: ninguna de las dos. Todavía no hemos escrito ninguna regla de autorización: solo pedimos estar autenticado.)

> **Para pensar.** Pon un punto de ruptura (*breakpoint*) en `loadUserByUsername` y haz login. Navega después por varias páginas: ¿se vuelve a ejecutar? ¿Por qué?

---

## Paso 4: autorización ADMIN y USER

Ya sabemos **quién** es cada uno. Falta decidir **qué puede hacer**. Primero, las rutas de la aplicación:

| Ruta | Método | Qué hace | ¿Quién? |
|---|---|---|---|
| `/libros` | GET | Listar y filtrar | ADMIN, USER |
| `/libros/nuevo` | GET | Formulario de alta | ADMIN |
| `/libros/editar/{isbn}` | GET | Formulario de edición | ADMIN |
| `/libros/guardar` | POST | Guardar alta o edición | ADMIN |
| `/libros/borrar/{isbn}` | POST | Borrar | ADMIN |
| `/autores` | GET | Listar | ADMIN, USER |
| `/autores/nuevo`, `/autores/editar/{id}`, `/autores/guardar`, `/autores/borrar/{id}` | GET / POST | Alta, edición, borrado | ADMIN |

Sustituye el bloque `authorizeHttpRequests` de `SecurityConfig` por este:

```java
.authorizeHttpRequests(auth -> auth
        // Públicas: portada, página de error y recursos estáticos
        .requestMatchers("/", "/error", "/css/**", "/vendor/**").permitAll()
        // Consola H2: solo para desarrollo
        .requestMatchers("/h2-console/**").permitAll()
        // Solo ADMIN: altas, ediciones y borrados
        .requestMatchers("/libros/nuevo", "/libros/editar/**",
                "/libros/guardar", "/libros/borrar/**").hasRole("ADMIN")
        .requestMatchers("/autores/nuevo", "/autores/editar/**",
                "/autores/guardar", "/autores/borrar/**").hasRole("ADMIN")
        // ADMIN y USER: consultar y filtrar
        .requestMatchers("/libros", "/autores").hasAnyRole("ADMIN", "USER")
        // Cualquier otra cosa: al menos estar identificado
        .anyRequest().authenticated()
)
```

Claves:

- **Gana la primera regla que coincide.** Por eso van de lo más concreto a lo más general, y `anyRequest()` al final.
- `hasRole("ADMIN")` busca la authority `ROLE_ADMIN`, la que creó `.roles("ADMIN")` en nuestro `UsuarioDetailsService`.
- La regla protege la **URL**, la pidas como la pidas: con el botón, escribiéndola o con `curl`.

### Comprueba

| Prueba | Anónimo | `ana` (USER) | `admin` |
|---|---|---|---|
| `/` | ✅ | ✅ | ✅ |
| `/libros` y filtrar | → login | ✅ | ✅ |
| `/libros/nuevo` | → login | ❌ 403 | ✅ |
| Escribir `/libros/editar/9788490000496` | → login | ❌ 403 | ✅ |
| Pulsar el botón de borrar | — | ❌ 403 | ✅ |
| `/autores/editar/1` | → login | ❌ 403 | ✅ |

De momento, el 403 muestra la página de error genérica de la aplicación ("Se ha producido un error inesperado"). Lo mejoramos en el paso 5.

> **Reto 1.** Mueve la regla `.requestMatchers("/libros", "/autores").hasAnyRole(...)` y cámbiala por `.requestMatchers("/libros/**", "/autores/**").hasAnyRole(...)`, colocándola **antes** de las de ADMIN. Entra como `ana` e intenta borrar. ¿Qué pasa? ¿Por qué? Déjalo como estaba.

> **Reto 2.** Quita `"/libros/guardar"` de la regla de ADMIN. Como `ana`, ¿puedes abrir el formulario de edición? ¿Y si le envías el formulario con las herramientas del navegador o con un cliente HTTP? Conclusión: hay que proteger **el POST que guarda**, no solo la página que muestra el formulario.

---

## Paso 5: vistas, `sec:authorize` y página 403

### 5.1 La dependencia

En el `pom.xml`:

```xml
<!-- sec:authorize y sec:authentication en Thymeleaf -->
<dependency>
    <groupId>org.thymeleaf.extras</groupId>
    <artifactId>thymeleaf-extras-springsecurity6</artifactId>
</dependency>
```

Sin versión: la gestiona Spring Boot.

En cada plantilla donde uses `sec:`, añade el espacio de nombres a la etiqueta `<html>`:

```html
<html lang="es" xmlns:th="http://www.thymeleaf.org"
      xmlns:sec="http://www.thymeleaf.org/extras/spring-security">
```

| Atributo | Para qué |
|---|---|
| `sec:authorize="isAuthenticated()"` | Mostrar solo si hay alguien identificado. |
| `sec:authorize="!isAuthenticated()"` | Mostrar solo a anónimos. |
| `sec:authorize="hasRole('ADMIN')"` | Mostrar solo a ADMIN. |
| `sec:authentication="name"` | Escribir el nombre del usuario. |

### 5.2 La barra de navegación

En `fragments/layout.html`, **sustituye el formulario provisional de logout** del paso 2 por esto:

```html
<!-- Usuario identificado: nombre, rol y botón de salir -->
<div sec:authorize="isAuthenticated()" class="ms-auto d-flex align-items-center gap-3">
  <span class="navbar-text">
    <i class="bi bi-person-circle"></i> <span sec:authentication="name">usuario</span>
    <span sec:authorize="hasRole('ADMIN')" class="badge bg-warning text-dark">ADMIN</span>
  </span>
  <!-- Logout por POST: así lo protege el token CSRF -->
  <form th:action="@{/logout}" method="post" class="m-0">
    <button type="submit" class="btn btn-outline-light btn-sm">
      <i class="bi bi-box-arrow-right"></i> Salir
    </button>
  </form>
</div>

<!-- Anónimo: enlace al login -->
<a sec:authorize="!isAuthenticated()" th:href="@{/login}" class="btn btn-outline-light btn-sm ms-auto">
  <i class="bi bi-box-arrow-in-right"></i> Entrar
</a>
```

### 5.3 Ocultar botones a quien no es ADMIN

En `libros/lista.html`, añade `sec:authorize="hasRole('ADMIN')"` a cinco elementos:

```html
<!-- Botón de alta -->
<a sec:authorize="hasRole('ADMIN')" th:href="@{/libros/nuevo}" class="btn btn-success">

<!-- Cabeceras de las columnas -->
<th sec:authorize="hasRole('ADMIN')" class="col-accion">Editar</th>
<th sec:authorize="hasRole('ADMIN')" class="col-accion">Borrar</th>

<!-- Celdas de cada fila -->
<td sec:authorize="hasRole('ADMIN')">
    <a th:href="@{/libros/editar/{isbn}(isbn=${l.isbn})}" ...>
</td>
<td sec:authorize="hasRole('ADMIN')">
    <form th:action="@{/libros/borrar/{isbn}(isbn=${l.isbn})}" ...>
</td>
```

Haz lo mismo en `autores/lista.html` (botón "Añadir autor", cabeceras "Editar" y "Borrar" y sus celdas).

> Oculta **la cabecera y la celda**. Si ocultas solo una de las dos, la tabla se descuadra.

### 5.4 Página 403

Spring Boot busca automáticamente una plantilla con el código de error en `templates/error/`. Crea `templates/error/403.html`:

```html
<!DOCTYPE html>
<html lang="es" xmlns:th="http://www.thymeleaf.org">
<head th:replace="~{fragments/layout :: head('Acceso denegado')}"></head>
<body>
<nav th:replace="~{fragments/layout :: navbar}"></nav>

<div class="container text-center py-4">
  <i class="bi bi-shield-lock-fill fs-1 text-danger"></i>
  <h2 class="mt-2">403 · Acceso denegado</h2>
  <p class="lead">
    No tienes permiso para acceder a <code th:text="${path}">/ruta</code>
  </p>
  <a th:href="@{/libros}" class="btn btn-primary">Volver a los libros</a>
</div>

<th:block th:replace="~{fragments/layout :: scripts}"></th:block>
</body>
</html>
```

Funciona gracias a que en el paso 2 abrimos `/error` con `permitAll()`: Spring Security responde 403 y Spring Boot muestra `error/403.html`. `${path}` lo rellena Spring Boot con la URL que se intentó abrir.

### Comprueba

| Prueba | Resultado esperado |
|---|---|
| Anónimo en la portada | Botón "Entrar" en la barra. |
| `ana` en `/libros` | Su nombre en la barra, sin insignia ADMIN. Sin botón "Añadir libro" ni columnas Editar y Borrar. |
| `admin` en `/libros` | Nombre con insignia ADMIN y todos los botones. |
| `ana` escribe `/libros/nuevo` | Página **403 · Acceso denegado** con la ruta. |

> **Reto 3: ocultar un botón no protege.** En `SecurityConfig`, comenta las dos reglas `hasRole("ADMIN")`. Entra como `ana`: los botones siguen ocultos. Ahora escribe en el navegador `/libros/editar/9788490000496`, cambia el título y guarda. ¿Qué ha pasado? Explica qué protege `sec:authorize` y qué protege la `SecurityFilterChain`. Vuelve a descomentar las reglas.

---

## Paso 6: tests con MockMvc y `@WithMockUser`

Probar la matriz del paso 4 a mano con tres usuarios es lento y fácil de olvidar. Lo automatizamos.

### 6.1 Dependencias

En el `pom.xml`:

1. **Borra** esta línea de `<properties>`: con ella Maven no ejecuta los tests.

```xml
<maven.test.skip>true</maven.test.skip>
```

2. Revisa las dependencias de test en el `pom.xml`:
   - `spring-boot-starter-webmvc-test` (MockMvc) ya está, pero **comentada**: descoméntala.
   - `spring-boot-starter-security-test` (`@WithMockUser`, `csrf()`, `formLogin()`): si en el paso 1 añadiste Spring Security como starter de IntelliJ, se habrá añadido sola. Si no está, añádela.

```xml
<!-- Tests: MockMvc y @WithMockUser -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webmvc-test</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security-test</artifactId>
    <scope>test</scope>
</dependency>
```

### 6.2 La clase de test

Crea `src/test/java/es/daw/bibliotecajdbc/SeguridadTest.java`:

```java
@SpringBootTest
@AutoConfigureMockMvc
class SeguridadTest {

    @Autowired
    private MockMvc mockMvc;

    // ---------- Autorización (usuarios simulados con @WithMockUser) ----------

    @Test
    void anonimoEsRedirigidoAlLogin() throws Exception {
        mockMvc.perform(get("/libros"))
                .andExpect(status().is3xxRedirection())
                .andExpect(redirectedUrl("/login"));
    }

    @Test
    void portadaEsPublica() throws Exception {
        mockMvc.perform(get("/"))
                .andExpect(status().isOk());
    }

    @Test
    @WithMockUser(roles = "USER")
    void userPuedeVerLibros() throws Exception {
        mockMvc.perform(get("/libros"))
                .andExpect(status().isOk());
    }

    @Test
    @WithMockUser(roles = "USER")
    void userNoPuedeAbrirFormularioDeAlta() throws Exception {
        mockMvc.perform(get("/libros/nuevo"))
                .andExpect(status().isForbidden());
    }

    @Test
    @WithMockUser(roles = "USER")
    void userNoPuedeBorrarAunqueConozcaLaUrl() throws Exception {
        mockMvc.perform(post("/libros/borrar/9788490000076").with(csrf()))
                .andExpect(status().isForbidden());
    }

    @Test
    @WithMockUser(roles = "ADMIN")
    void adminPuedeBorrar() throws Exception {
        // Ojo: borra de verdad el libro de H2. Usa un ISBN que ningún otro test necesita.
        mockMvc.perform(post("/libros/borrar/9788490001479").with(csrf()))
                .andExpect(status().is3xxRedirection())
                .andExpect(redirectedUrl("/libros"));
    }

    @Test
    @WithMockUser(roles = "ADMIN")
    void sinTokenCsrfNiSiquieraAdminPuedeBorrar() throws Exception {
        mockMvc.perform(post("/libros/borrar/9788490000076"))
                .andExpect(status().isForbidden());
    }

    // ---------- Autenticación real (UsuarioDetailsService + BCrypt + repositorio) ----------

    @Test
    void loginCorrectoConUsuarioDelRepositorio() throws Exception {
        mockMvc.perform(formLogin("/login").user("admin").password("1234"))
                .andExpect(authenticated().withUsername("admin").withRoles("ADMIN"));
    }

    @Test
    void loginConContrasenaIncorrecta() throws Exception {
        mockMvc.perform(formLogin("/login").user("admin").password("mala"))
                .andExpect(unauthenticated());
    }

    @Test
    void usuarioDesactivadoNoPuedeEntrar() throws Exception {
        mockMvc.perform(formLogin("/login").user("luis").password("1234"))
                .andExpect(unauthenticated());
    }
}
```

| Herramienta | Qué hace |
|---|---|
| `@SpringBootTest` + `@AutoConfigureMockMvc` | Arranca la aplicación completa (con H2 y la seguridad) y nos da un `MockMvc` para lanzar peticiones sin servidor ni navegador. |
| `@WithMockUser(roles = "USER")` | Simula un usuario **ya autenticado** con ese rol. **No** pasa por nuestro `UsuarioDetailsService`: prueba solo la **autorización**. |
| `.with(csrf())` | Añade un token CSRF válido a la petición, como haría Thymeleaf. |
| `formLogin("/login").user(...).password(...)` | Hace un login **de verdad**: usa nuestro `UsuarioDetailsService`, BCrypt y los usuarios de `UsuarioRepositoryMemoria`. Prueba la **autenticación**. |
| `authenticated()` / `unauthenticated()` | Comprueban si, tras la petición, hay alguien autenticado. |

> **¿Por qué `redirectedUrlPattern("**/login")` y no `redirectedUrl("/login")`?** Según la versión, Spring Security redirige con la URL completa (`http://localhost/login`) o relativa (`/login`). El patrón vale para las dos.

Ejecuta los tests desde IntelliJ (botón ▶ junto a la clase) o con `./mvnw test`. Deben pasar los 10.

> **Reto 4.** Rompe la seguridad a propósito y mira qué test falla:
> - Cambia `.hasRole("ADMIN")` por `.authenticated()` en la regla de libros.
> - Pon `true` en el `activo` de `luis` en `UsuarioRepositoryMemoria`.
> - Cambia en `UsuarioDetailsService` `.roles(...)` por `.authorities(...)`.
>
> Para cada cambio: ¿qué test falla y por qué? ¿Lo habrías detectado probando a mano?

> **Reto 5.** Escribe los tests equivalentes para `/autores`: USER puede listar, USER no puede abrir `/autores/editar/1`, ADMIN sí.

---

## Resumen: qué ficheros hemos tocado

```
pom.xml                                    + security, thymeleaf-extras, tests; − maven.test.skip
src/main/java/es/daw/bibliotecajdbc/
├── config/
│   ├── SecurityConfig.java                NUEVO  (pasos 2, 3, 4)
│   └── WebConfig.java                     + vista /login
├── mapper/
│   ├── AutorMapper.java                   new → builder (paso 3.0)
│   └── LibroMapper.java                   new → builder (paso 3.0)
├── model/
│   ├── Autor.java                         + @Builder (paso 3.0)
│   ├── Libro.java                         + @Builder (paso 3.0)
│   └── Usuario.java                       NUEVO  (paso 3)
├── repository/
│   ├── AutorRepositoryJdbc.java           new → builder (paso 3.0)
│   ├── LibroRepositoryJdbc.java           new → builder (paso 3.0)
│   ├── UsuarioRepository.java             NUEVO  (paso 3)
│   └── UsuarioRepositoryMemoria.java      NUEVO  (paso 3)
└── service/UsuarioDetailsService.java     NUEVO  (paso 3)
src/main/resources/templates/
├── login.html                             NUEVO  (paso 2)
├── error/403.html                         NUEVO  (paso 5)
├── fragments/layout.html                  + usuario en la barra y logout (pasos 2 y 5)
├── libros/lista.html                      + sec:authorize (paso 5)
└── autores/lista.html                     + sec:authorize (paso 5)
src/test/java/es/daw/bibliotecajdbc/
└── SeguridadTest.java                     NUEVO  (paso 6)
```

Fíjate en lo que **no** hemos tocado por la seguridad: ningún controlador ni servicio de libros o autores. Los cambios en mappers y repositorios del paso 3.0 son una refactorización, no seguridad. La seguridad se añade **alrededor** de la aplicación, en los filtros, sin mezclarse con la lógica de negocio.

