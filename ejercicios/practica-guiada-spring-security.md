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
| [3](#paso-3-usuarios-en-memoria-con-bcrypt) | Usuarios en memoria con BCrypt | `UserDetailsService`, `PasswordEncoder` |
| [4](#paso-4-usuarios-y-roles-en-h2-con-jdbc) | Usuarios y roles en H2 con JDBC | Nuestro `UserDetailsService` (SRP) |
| [5](#paso-5-autorización-admin-y-user) | Autorización ADMIN / USER | `hasRole`, orden de las reglas, 403 |
| [6](#paso-6-vistas-secauthorize-y-página-403) | Vistas: `sec:authorize` y página 403 | Ocultar no es proteger |
| [7](#paso-7-tests-con-mockmvc-y-withmockuser) | Tests con MockMvc y `@WithMockUser` | Comprobar la seguridad sin navegador |

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

<img width="448" height="344" alt="imagen" src="https://github.com/user-attachments/assets/99158c8d-3493-4d53-be4f-be84233ae683" />


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

> **Para pensar.** El `curl` falla por dos motivos a la vez: no tiene sesión y no envía token CSRF. ¿Cuál de los dos comprueba antes Spring Security? (Pista: mira el orden de los filtros en la teoría.)

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

| Bloque | Filtro que configura | Qué hace aquí |
|---|---|---|
| `authorizeHttpRequests` | `AuthorizationFilter` | Abre la portada, `/error`, los CSS, Bootstrap y la consola H2. El resto exige login. |
| `formLogin` | `UsernamePasswordAuthenticationFilter` | `GET /login` muestra **nuestra** vista; `POST /login` lo procesa Spring. `permitAll()` abre `/login`, si no nadie podría entrar. |
| `logout` | `LogoutFilter` | `POST /logout` cierra la sesión y redirige a `/login?logout`. |
| `csrf` y `headers` | `CsrfFilter` y cabeceras | Excepciones **solo** para la consola H2. CSRF sigue activo para la aplicación. |

> **¿Por qué `/vendor/**`?** Ahí están Bootstrap y sus iconos (en local, para trabajar sin internet). Si no los abres, la página de login se verá sin estilos: el navegador pide el CSS y Spring Security lo redirige al login.

> ⚠️ La consola H2 abierta y sin CSRF es **solo para desarrollo**. En producción no existiría.

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

En el paso 6 este botón solo se verá cuando haya alguien identificado.

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

## Paso 3: usuarios en memoria con BCrypt

El usuario `user` con contraseña aleatoria no sirve. Vamos a definir nuestros usuarios, primero en memoria.

Añade a `SecurityConfig` dos beans:

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();
}

@Bean
public UserDetailsService usuariosEnMemoria(PasswordEncoder passwordEncoder) {
    UserDetails admin = User.builder()
            .username("admin")
            .password(passwordEncoder.encode("1234"))
            .roles("ADMIN")
            .build();

    UserDetails ana = User.builder()
            .username("ana")
            .password(passwordEncoder.encode("1234"))
            .roles("USER")
            .build();

    return new InMemoryUserDetailsManager(admin, ana);
}
```

| Pieza | Qué hace |
|---|---|
| `PasswordEncoder` | Spring lo usa para comparar la contraseña escrita con el hash guardado (`matches`). |
| `UserDetailsService` | Le dice a Spring dónde buscar los usuarios. Aquí, en memoria. |
| `User.builder()` | Clase de Spring Security que implementa `UserDetails`. |
| `.roles("ADMIN")` | Crea la authority `ROLE_ADMIN` (añade el prefijo). |

Con estos dos beans, Spring Boot ya sabe autenticar. **No hace falta** declarar `AuthenticationProvider` ni `AuthenticationManager`.

### Comprueba

- Al arrancar, **ya no aparece** "Using generated security password": Spring ve que tenemos nuestro propio `UserDetailsService`.
- Entra como `admin` / `1234` y como `ana` / `1234`.
- Entra como `ana` y **borra un libro**. Funciona.

> **Para pensar.** `ana` es USER y ha podido borrar. ¿Ha fallado la autenticación o la autorización? (Respuesta: ninguna de las dos. Todavía no hemos escrito ninguna regla de autorización: solo pedimos estar autenticado.)

> **Para pensar.** Añade temporalmente un `System.out.println(passwordEncoder.encode("1234"));` dentro del método y arranca dos veces. ¿Sale el mismo hash? ¿Por qué el login sigue funcionando?

---

## Paso 4: usuarios y roles en H2 con JDBC

Los usuarios en memoria se pierden al cambiar el código y no se pueden gestionar. Los pasamos a la base de datos **con las mismas capas que el resto de la aplicación**:

```
UsuarioDetailsService (service)  ──usa──▶  UsuarioRepository (interfaz)
        │                                          ▲
        │ devuelve User (UserDetails)              │ implementa
        ▼                                          │
  Spring Security                       UsuarioRepositoryJdbc ──▶ H2 (tabla usuario)
```

### 4.1 La tabla

En `schema.sql`, añade el `DROP` **al principio**, junto a los otros:

```sql
DROP TABLE IF EXISTS usuario;
```

Y la tabla **al final**:

```sql
-- Usuarios de la aplicación (Spring Security)
-- password: hash BCrypt (60 caracteres), nunca la contraseña en claro
-- rol: sin prefijo ROLE_ (lo añade Spring con roles(...))
CREATE TABLE usuario (
                       id       INT          AUTO_INCREMENT PRIMARY KEY,
                       username VARCHAR(50)  NOT NULL,
                       password VARCHAR(60)  NOT NULL,
                       rol      VARCHAR(20)  NOT NULL,
                       activo   BOOLEAN      NOT NULL DEFAULT TRUE,
                       CONSTRAINT uq_usuario_username UNIQUE (username),
                       CONSTRAINT ck_usuario_rol CHECK (rol IN ('ADMIN', 'USER'))
);
```

> **¿Por qué un solo rol por usuario?** Para varios roles haría falta una relación N:M (tablas `usuario`, `rol` y `usuario_rol`). La dejamos para JPA en la UT04. Con un rol por usuario tenemos de sobra para esta aplicación.

### 4.2 Los datos

Al final de `data.sql`:

```sql
-- Usuarios. La contraseña de los tres es 1234 (guardada como hash BCrypt).
-- Aunque la contraseña es la misma, los hashes son distintos: cada uno lleva su propia sal.
INSERT INTO usuario (username, password, rol, activo) VALUES ('admin', '$2a$10$FE6Xa2yaXAVUf9v7DDQoF.bjxEGeJujmNH7o5T91IUG3qjc3z05Py', 'ADMIN', TRUE);
INSERT INTO usuario (username, password, rol, activo) VALUES ('ana',   '$2a$10$07vZe7nhH1Oehrm51orcpeer3uYX1JPi5lLBWb.HW.JMX.llYoIaK', 'USER',  TRUE);
INSERT INTO usuario (username, password, rol, activo) VALUES ('luis',  '$2a$10$EQIZl.C1bGRS.QitQ.2aPuM62n.iRSiiLBqTCIUwFC21c9U9FPdyK', 'USER',  FALSE);
```

> **¿Y si quiero otra contraseña?** Genera el hash con el `System.out.println(passwordEncoder.encode("..."))` del paso 3 y cópialo. En la BD **nunca** se escribe la contraseña en claro: si pones `'1234'`, el login fallará y en la consola verás el aviso `Encoded password does not look like BCrypt`.

### 4.3 El modelo

`model/Usuario.java`:

```java
/**
 * Fila de la tabla USUARIO.
 * POJO sin dependencias de Spring Security: NO implementa UserDetails.
 * La traducción a UserDetails la hace UsuarioDetailsService (SRP).
 */
@Data
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

### 4.4 El repositorio

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

> **SOLID: principio de segregación de interfaces (ISP).** `LibroRepository` y `AutorRepository` extienden `GenericRepository` porque necesitan el CRUD completo. `UsuarioRepository` solo necesita buscar por nombre. Obligarle a implementar `save`, `update` y `delete` sería añadir métodos que nadie usa.

`repository/UsuarioRepositoryJdbc.java`, con el mismo estilo que `AutorRepositoryJdbc`:

```java
@Repository
@RequiredArgsConstructor
public class UsuarioRepositoryJdbc implements UsuarioRepository {

    private final DataSource dataSource;

    @Override
    public Optional<Usuario> findByUsername(String username) {
        String sql = "SELECT id, username, password, rol, activo FROM usuario WHERE username = ?";
        try (Connection con = dataSource.getConnection();
             PreparedStatement ps = con.prepareStatement(sql)) {
            ps.setString(1, username);
            try (ResultSet rs = ps.executeQuery()) {
                return rs.next() ? Optional.of(mapRow(rs)) : Optional.empty();
            }
        } catch (SQLException e) {
            throw new AccesoDatosException("Error al buscar el usuario " + username, e);
        }
    }

    private Usuario mapRow(ResultSet rs) throws SQLException {
        return new Usuario(
                rs.getInt("id"),
                rs.getString("username"),
                rs.getString("password"),
                rs.getString("rol"),
                rs.getBoolean("activo"));
    }
}
```

### 4.5 Nuestro `UserDetailsService`

`service/UsuarioDetailsService.java`:

```java
/**
 * Adaptador entre nuestro modelo y Spring Security:
 * busca el Usuario en la BD y lo devuelve como UserDetails.
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

Lo que hace Spring con esta clase en cada login:

1. Llama a `loadUserByUsername("ana")`.
2. Si no existe → `UsernameNotFoundException` → `/login?error`.
3. Si existe pero está desactivado → `/login?error`.
4. Compara la contraseña escrita con el hash usando `passwordEncoder.matches(...)`.
5. Si coincide → guarda el usuario en el `SecurityContext` y este en la sesión.

> **SOLID: responsabilidad única (SRP).** `Usuario` no implementa `UserDetails`. Representar un usuario de la biblioteca y adaptarlo a Spring Security son dos responsabilidades, y cada una tiene su clase. `UsuarioDetailsService` hace de **adaptador**. (Ver el apartado 7 de la teoría.)

### 4.6 ¡Borra los usuarios en memoria!

Elimina de `SecurityConfig` el bean `usuariosEnMemoria`. **Deja** el `PasswordEncoder`.

> ⚠️ **Error típico.** Si dejas los dos `UserDetailsService` (el de memoria y el nuestro), Spring Boot no sabe cuál usar y **no configura la autenticación**: ningún login funcionará. Debe haber **uno solo**.

### Comprueba

| Prueba | Resultado esperado |
|---|---|
| `admin` / `1234` | Entra. |
| `ana` / `1234` | Entra. |
| `luis` / `1234` | **No entra**: está desactivado (`activo = FALSE`). |
| `pepe` / `1234` | No entra: no existe. Mismo mensaje que con contraseña mala. |
| Consola H2: `SELECT * FROM usuario;` | Tres hashes distintos para la misma contraseña. |

---

## Paso 5: autorización ADMIN y USER

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

De momento, el 403 muestra la página de error genérica de la aplicación ("Se ha producido un error inesperado"). Lo mejoramos en el paso 6.

> **Reto 1.** Mueve la regla `.requestMatchers("/libros", "/autores").hasAnyRole(...)` y cámbiala por `.requestMatchers("/libros/**", "/autores/**").hasAnyRole(...)`, colocándola **antes** de las de ADMIN. Entra como `ana` e intenta borrar. ¿Qué pasa? ¿Por qué? Déjalo como estaba.

> **Reto 2.** Quita `"/libros/guardar"` de la regla de ADMIN. Como `ana`, ¿puedes abrir el formulario de edición? ¿Y si le envías el formulario con las herramientas del navegador o con un cliente HTTP? Conclusión: hay que proteger **el POST que guarda**, no solo la página que muestra el formulario.

---

## Paso 6: vistas, `sec:authorize` y página 403

### 6.1 La dependencia

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

### 6.2 La barra de navegación

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

### 6.3 Ocultar botones a quien no es ADMIN

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

### 6.4 Página 403

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

## Paso 7: tests con MockMvc y `@WithMockUser`

Probar la matriz del paso 5 a mano con tres usuarios es lento y fácil de olvidar. Lo automatizamos.

### 7.1 Dependencias

En el `pom.xml`:

1. **Borra** esta línea de `<properties>`: con ella Maven no ejecuta los tests.

```xml
<maven.test.skip>true</maven.test.skip>
```

2. Sustituye la dependencia comentada `spring-boot-starter-webmvc-test` por estas dos:

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

### 7.2 La clase de test

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
                .andExpect(redirectedUrlPattern("**/login"));
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

    // ---------- Autenticación real (UserDetailsService + BCrypt + H2) ----------

    @Test
    void loginCorrectoConUsuarioDeLaBaseDeDatos() throws Exception {
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
| `@WithMockUser(roles = "USER")` | Simula un usuario **ya autenticado** con ese rol. **No** pasa por nuestro `UsuarioDetailsService` ni por la BD: prueba solo la **autorización**. |
| `.with(csrf())` | Añade un token CSRF válido a la petición, como haría Thymeleaf. |
| `formLogin("/login").user(...).password(...)` | Hace un login **de verdad**: usa nuestro `UsuarioDetailsService`, BCrypt y los datos de `data.sql`. Prueba la **autenticación**. |
| `authenticated()` / `unauthenticated()` | Comprueban si, tras la petición, hay alguien autenticado. |

> **¿Por qué `redirectedUrlPattern("**/login")` y no `redirectedUrl("/login")`?** Según la versión, Spring Security redirige con la URL completa (`http://localhost/login`) o relativa (`/login`). El patrón vale para las dos.

Ejecuta los tests desde IntelliJ (botón ▶ junto a la clase) o con `./mvnw test`. Deben pasar los 10.

> **Reto 4.** Rompe la seguridad a propósito y mira qué test falla:
> - Cambia `.hasRole("ADMIN")` por `.authenticated()` en la regla de libros.
> - Pon `TRUE` en el `activo` de `luis`.
> - Cambia en `UsuarioDetailsService` `.roles(...)` por `.authorities(...)`.
>
> Para cada cambio: ¿qué test falla y por qué? ¿Lo habrías detectado probando a mano?

> **Reto 5.** Escribe los tests equivalentes para `/autores`: USER puede listar, USER no puede abrir `/autores/editar/1`, ADMIN sí.

---

## Resumen: qué ficheros hemos tocado

```
pom.xml                                 + security, thymeleaf-extras, tests; − maven.test.skip
src/main/java/es/daw/bibliotecajdbc/
├── config/
│   ├── SecurityConfig.java             NUEVO  (pasos 2, 3, 5)
│   └── WebConfig.java                  + vista /login
├── model/Usuario.java                  NUEVO  (paso 4)
├── repository/
│   ├── UsuarioRepository.java          NUEVO  (paso 4)
│   └── UsuarioRepositoryJdbc.java      NUEVO  (paso 4)
└── service/UsuarioDetailsService.java  NUEVO  (paso 4)
src/main/resources/
├── schema.sql                          + tabla usuario
├── data.sql                            + 3 usuarios con hash BCrypt
└── templates/
    ├── login.html                      NUEVO  (paso 2)
    ├── error/403.html                  NUEVO  (paso 6)
    ├── fragments/layout.html           + usuario en la barra y logout (pasos 2 y 6)
    ├── libros/lista.html               + sec:authorize (paso 6)
    └── autores/lista.html              + sec:authorize (paso 6)
src/test/java/es/daw/bibliotecajdbc/
└── SeguridadTest.java                  NUEVO  (paso 7)
```

Fíjate en lo que **no** hemos tocado: ningún controlador, servicio ni repositorio de libros o autores. La seguridad se añade **alrededor** de la aplicación, en los filtros, sin mezclarse con la lógica de negocio.

## Errores típicos

| Síntoma | Causa probable |
|---|---|
| El login no funciona con ningún usuario | Hay dos `UserDetailsService` (no borraste el de memoria) o falta el bean `PasswordEncoder`. |
| Consola: `Encoded password does not look like BCrypt` | En `data.sql` hay una contraseña en claro en vez del hash. |
| Un formulario da 403 aunque el usuario tenga permiso | El formulario usa `action` en vez de `th:action`: falta el token CSRF. |
| Excepción: `ROLE_ADMIN cannot start with ROLE_` | En la BD el rol está guardado con prefijo y usas `.roles(...)`. |
| Nadie tiene permiso aunque el rol es correcto | Mezcla de `.authorities("ADMIN")` con `hasRole("ADMIN")`. |
| Un USER puede editar o borrar | Una regla general (`/libros/**`) está antes que la concreta de ADMIN. |
| La página de login sale sin estilos | Falta `/vendor/**` (o `/css/**`) en `permitAll()`. |
| La página 403 no se ve, sale el login | Falta `/error` en `permitAll()`. |
| La consola H2 se queda en blanco | Faltan las excepciones de `csrf` o de `frameOptions` para `/h2-console/**`. |
| Código copiado de internet no compila | Usa APIs eliminadas en Spring Security 7 (`WebSecurityConfigurerAdapter`, `authorizeRequests`, `antMatchers`, `.and()`). Ver la teoría. |

## Imports que pueden confundir

| Clase | Elige |
|---|---|
| `User` | `org.springframework.security.core.userdetails.User` |
| `UserDetails`, `UserDetailsService`, `UsernameNotFoundException` | `org.springframework.security.core.userdetails` |
| `PasswordEncoder` | `org.springframework.security.crypto.password.PasswordEncoder` |
| `BCryptPasswordEncoder` | `org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder` |
| `InMemoryUserDetailsManager` | `org.springframework.security.provisioning.InMemoryUserDetailsManager` |
| `SecurityFilterChain` | `org.springframework.security.web.SecurityFilterChain` |
| `HttpSecurity` | `org.springframework.security.config.annotation.web.builders.HttpSecurity` |
| `AutoConfigureMockMvc` | `org.springframework.boot.webmvc.test.autoconfigure.AutoConfigureMockMvc` (nuevo paquete en Boot 4) |
| `csrf()` (estático) | `SecurityMockMvcRequestPostProcessors.csrf` |
| `formLogin()` (estático) | `SecurityMockMvcRequestBuilders.formLogin` |
| `authenticated()`, `unauthenticated()` (estáticos) | `SecurityMockMvcResultMatchers` |
| `get()`, `post()` (estáticos) | `MockMvcRequestBuilders` |
| `status()`, `redirectedUrl()`, `redirectedUrlPattern()` (estáticos) | `MockMvcResultMatchers` |
