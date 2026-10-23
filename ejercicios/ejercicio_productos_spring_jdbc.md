# Ejercicio: Productos con Spring y JDBC

![alt text](image.png)

## Contexto

Vais a construir el acceso a datos de una tabla `producto` directamente en **Spring Boot**, con **JDBC** (`DataSource`, `Connection`, `PreparedStatement`, `ResultSet`).

Es la primera parte de un ejercicio en dos fases:

1. **Ahora (UT02): JDBC.** Escribís vosotros el SQL y el mapeo de filas a objetos. Así veréis qué hay debajo de lo que hace JPA.
2. **Más adelante (UT04): JPA.** Sustituiréis la implementación por JPA manteniendo la misma interfaz, y comprobaréis qué trabajo os quita.

No usamos todavía JPA/Hibernate. Aquí solo hay JDBC, con una diferencia importante: **la conexión no la creáis vosotros, os la da Spring**.

## Punto de partida

Cread un proyecto Spring Boot nuevo llamado **`ProductosJdbc`** con las dependencias:

- Spring Web
- Spring Boot DevTools
- H2 Database
- JDBC API (`spring-boot-starter-jdbc`)
- Lombok (opcional)

### `application.properties`

```properties
spring.application.name=productosJdbc

spring.datasource.url=jdbc:h2:mem:productosdb
spring.datasource.driverClassName=org.h2.Driver
spring.datasource.username=sa
spring.datasource.password=

spring.h2.console.enabled=true
spring.h2.console.path=/h2-console
```

### `src/main/resources/schema.sql`

```sql
CREATE TABLE producto (
    id     INT AUTO_INCREMENT PRIMARY KEY,
    nombre VARCHAR(100) NOT NULL,
    precio DOUBLE       NOT NULL
);
```

### `src/main/resources/data.sql`

```sql
INSERT INTO producto (nombre, precio) VALUES ('Teclado', 25.50);
INSERT INTO producto (nombre, precio) VALUES ('Ratón', 12.90);
INSERT INTO producto (nombre, precio) VALUES ('Monitor', 149.99);
```

Spring Boot ejecuta estos dos ficheros automáticamente al arrancar con una base de datos embebida como H2.

## Estructura de paquetes

```
es.daw.productosjdbc
├── model        → Producto
├── repository   → ProductoRepository (interfaz)
│                  ProductoRepositoryJdbc (implementación)
└── runner       → ComprobacionRunner (solo para probar)
```

## Pasos a seguir

### 1. Crea la entidad `Producto`

Una clase Java normal (POJO), sin anotaciones de persistencia:

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class Producto {
    private Integer id;
    private String nombre;
    private double precio;
}
```

### 2. Define la interfaz `ProductoRepository`

```java
public interface ProductoRepository {
    void save(Producto producto) throws SQLException;
    Optional<Producto> findById(Integer id) throws SQLException;
    List<Producto> findAll() throws SQLException;
    void update(Producto producto) throws SQLException;
    void delete(Integer id) throws SQLException;
}
```

El resto de la aplicación trabajará **contra esta interfaz**, nunca contra la clase concreta. Cuando pasemos a JPA solo cambiará la implementación.

### 3. Implementa `ProductoRepositoryJdbc`

Spring crea el `DataSource` a partir de `application.properties` y os lo **inyecta por constructor**. No hay `DriverManager.getConnection(...)` ni `new` de conexiones en ningún sitio.

```java
@Repository
public class ProductoRepositoryJdbc implements ProductoRepository {

    private final DataSource dataSource;

    public ProductoRepositoryJdbc(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    @Override
    public Optional<Producto> findById(Integer id) throws SQLException {
        String sql = "SELECT id, nombre, precio FROM producto WHERE id = ?";
        try (Connection con = dataSource.getConnection();
             PreparedStatement ps = con.prepareStatement(sql)) {
            ps.setInt(1, id);
            try (ResultSet rs = ps.executeQuery()) {
                return rs.next() ? Optional.of(mapRow(rs)) : Optional.empty();
            }
        }
    }

    @Override
    public List<Producto> findAll() throws SQLException {
        // Mismo patrón: try-with-resources + dataSource.getConnection()
        throw new UnsupportedOperationException("Pendiente de implementar");
    }

    @Override
    public void save(Producto producto) throws SQLException {
        throw new UnsupportedOperationException("Pendiente de implementar");
    }

    @Override
    public void update(Producto producto) throws SQLException {
        throw new UnsupportedOperationException("Pendiente de implementar");
    }

    @Override
    public void delete(Integer id) throws SQLException {
        throw new UnsupportedOperationException("Pendiente de implementar");
    }

    private Producto mapRow(ResultSet rs) throws SQLException {
        return new Producto(
                rs.getInt("id"),
                rs.getString("nombre"),
                rs.getDouble("precio"));
    }
}
```

Completad `findAll`, `save`, `update` y `delete` siguiendo el mismo patrón.

Condiciones:

- Siempre `PreparedStatement` con parámetros `?`. **Nunca concatenéis valores en el SQL.**
- Siempre `try-with-resources` para cerrar `Connection`, `PreparedStatement` y `ResultSet`.
- `findById` devuelve `Optional.empty()` si no existe, **nunca `null`**.
- En `save`, recuperad el `id` generado (`Statement.RETURN_GENERATED_KEYS`) y asignadlo al objeto.

### 4. Comprobad que funciona

Cread una clase `ComprobacionRunner` que implemente `CommandLineRunner`, reciba `ProductoRepository` **por constructor** y, al arrancar:

1. Muestre `findAll()` por consola.
2. Guarde un producto nuevo y muestre su `id`.
3. Lo modifique y lo vuelva a buscar con `findById`.
4. Lo borre y compruebe que `findById` devuelve vacío.

Podéis comprobar los datos en la consola H2: `http://localhost:8080/h2-console` (JDBC URL: `jdbc:h2:mem:productosdb`).

## Lo que tenéis que poder explicar al terminar

- ¿Quién crea el `DataSource` y la conexión? ¿Qué os ahorra Spring respecto a abrir la conexión vosotros mismos?
- Nunca escribís `new ProductoRepositoryJdbc(...)`. ¿Quién crea esa clase y cómo llega hasta el `CommandLineRunner`?
- ¿Por qué el runner recibe `ProductoRepository` y no `ProductoRepositoryJdbc`? ¿Qué principio SOLID se refuerza?
- ¿Qué problema evita usar `PreparedStatement` con `?` en lugar de concatenar el SQL?
- Mirad el código de `ProductoRepositoryJdbc`: ¿qué partes son siempre repetitivas? Esa es la parte que JPA os va a quitar.

## Entrega

Subid el proyecto completo a vuestro repositorio personal, en la carpeta del ejercicio correspondiente a UT02, con un `README.md` breve indicando cómo arrancarlo y cómo probarlo desde la consola H2 (`/h2-console`).

## Parte 2 (UT04)

Sobre este mismo proyecto, sustituiremos `ProductoRepositoryJdbc` por una versión con JPA. La interfaz y el código que la usa no deberían cambiar.
