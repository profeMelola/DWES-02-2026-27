# Base de datos en producción: lo que no se ve con 10 filas

> **Idea clave:** en clase la base de datos tiene 10 filas y todo va rápido.
> En producción tiene millones, y lo que decide si la aplicación responde en milisegundos o en minutos
> no es tu código Java, sino **cómo está montada la base de datos**.
> La IA te da consultas que funcionan; casi nunca te dice qué índice necesitan.

---

## 1. Índices

### 1.1 Qué es un índice

Piensa en el índice alfabético al final de un libro:

| Sin índice | Con índice |
|---|---|
| Para encontrar "transacción" lees el libro página a página | Vas al índice, buscas la palabra y saltas a la página |
| La BD lee **toda la tabla** fila a fila | La BD va **directa** a las filas que busca |

Con 10 filas no hay diferencia. Con 2 millones, es la diferencia entre milisegundos y segundos.

```sql
CREATE INDEX idx_libro_autor ON libro(autor_id);
```

### 1.2 Cuáles se crean solos

| Restricción | ¿Crea índice? |
|---|---|
| `PRIMARY KEY` | ✅ Siempre |
| `UNIQUE` (por ejemplo, `username`, `isbn`) | ✅ Siempre |
| `FOREIGN KEY` | ⚠️ Depende del motor: MySQL sí, PostgreSQL **no**. Compruébalo |

### 1.3 Dónde crear índices

En las columnas que aparecen a menudo en:

| Parte de la consulta | Ejemplo |
|---|---|
| `WHERE` | `WHERE autor_id = ?` |
| `JOIN ... ON` | `JOIN autor a ON l.autor_id = a.id` |
| `ORDER BY` | `ORDER BY fecha_creacion` |

**Índice de varias columnas:** `CREATE INDEX idx_pedido_prov_fecha ON pedido(provincia, fecha);` sirve para buscar por `provincia` o por `provincia` y `fecha`, pero **no** solo por `fecha`. El orden de las columnas importa, como en una guía telefónica ordenada por apellido y después por nombre: no puedes buscar a todos los que se llaman "Ana".

### 1.4 Los índices no son gratis

| Ventaja | Coste |
|---|---|
| Las búsquedas (`SELECT`) son mucho más rápidas | Ocupan espacio en disco |
| | Cada `INSERT`, `UPDATE` y `DELETE` tiene que actualizar también el índice |

Por eso **no se indexa todo**: solo lo que se consulta mucho. La pregunta es la de siempre: ¿en esta tabla se busca mucho o se inserta mucho?

> 🤖 Si le pides a la IA "optimiza mi base de datos", suele proponer índices en muchas columnas. No sabe qué consultas se hacen de verdad ni si la tabla recibe más lecturas o escrituras. Eso lo sabes tú.

### 1.5 Cuándo el índice no sirve aunque exista

| Consulta | Por qué no usa el índice |
|---|---|
| `WHERE titulo LIKE '%sombra%'` | Con el `%` al principio no sabe por dónde empezar. `LIKE 'sombra%'` sí lo usa |
| `WHERE UPPER(titulo) = 'DUNE'` | Al aplicar una función a la columna, el índice (que guarda `titulo` tal cual) no sirve |
| `WHERE activo = true` | Si la mitad de las filas son `true`, a la BD le compensa leer la tabla entera |
| Tabla con pocas filas | Leerla entera es más rápido que usar el índice |

> 🤖 La IA suele generar búsquedas con `LIKE '%texto%'` porque siempre "funciona". Antes de aceptarlo, pregunta al cliente: ¿basta con buscar los títulos que **empiezan** por el texto?

---

## 2. `EXPLAIN`: pregúntale a la BD qué va a hacer

Pon `EXPLAIN` delante de la consulta y la BD te dice **cómo** la va a ejecutar, sin ejecutarla:

```sql
EXPLAIN SELECT * FROM libro WHERE autor_id = 3;
```

| Si ves… | Significa |
|---|---|
| `tableScan` (H2), `ALL` (MySQL), `Seq Scan` (PostgreSQL) | ❌ Lee la tabla entera |
| El nombre de un índice | ✅ Usa el índice |

Pruébalo en la consola de H2: lanza el `EXPLAIN`, crea el índice y vuelve a lanzarlo.

> 🤖 Puedes pegar la salida de `EXPLAIN` a la IA para que te la explique. Pero la decisión se toma con lo que dice `EXPLAIN`, no con lo que la IA supone.

---

## 3. Tipos de datos y restricciones

| Mal | Bien | Por qué |
|---|---|---|
| Fechas en `VARCHAR` | `DATE`, `TIMESTAMP` | Como texto, `'10/02/2026'` se ordena antes que `'9/01/2026'` |
| Dinero en `DOUBLE` | `DECIMAL(10,2)` | `DOUBLE` tiene errores de redondeo: `0.1 + 0.2` no da exactamente `0.3` |
| Todo `VARCHAR(255)` | El tamaño real: `VARCHAR(13)` para un ISBN | Ocupa menos y documenta qué se espera |
| Sin restricciones | `NOT NULL`, `UNIQUE`, `CHECK` | La BD impide datos incorrectos aunque el código falle |

> 🤖 Revisa los `CREATE TABLE` que te da la IA: es frecuente que use `DOUBLE` para precios o `VARCHAR(255)` para todo.

---

## 4. Conexiones: el pool

Abrir una conexión con la base de datos es **lento**. Por eso no se abre una por cada consulta: se tiene un **pool**, un grupo de conexiones abiertas que se prestan y se devuelven.

- Spring Boot ya lo hace por ti: el `DataSource` que inyectas en tus repositorios JDBC **es un pool** (HikariCP, con 10 conexiones por defecto).
- Si no cierras una conexión, **no vuelve al pool**. Tras unas cuantas peticiones, el pool se queda sin conexiones y la aplicación se cuelga esperando.

```java
// ✅ try-with-resources: la conexión vuelve al pool aunque salte una excepción
try (Connection con = dataSource.getConnection();
     PreparedStatement ps = con.prepareStatement(sql)) {
    ...
}
```

---

## 5. Lo que haces tú desde el código

| ❌ Evita | ✅ Mejor | Por qué |
|---|---|---|
| `SELECT *` | `SELECT id, titulo, isbn` | Trae solo lo que vas a mostrar |
| Traer todas las filas para mostrar 20 | Paginar: `LIMIT 20 OFFSET 40` | Nadie lee 10.000 filas en una página |
| Una consulta dentro de un bucle (`for` + `findById`) | Una sola consulta con `JOIN` o `WHERE id IN (...)` | 1.000 vueltas = 1.000 viajes a la BD |
| Filtrar u ordenar en Java | `WHERE` y `ORDER BY` | La BD lo hace con índices |

---

## 6. Lo que hace un administrador en producción

| Tarea | Para qué |
|---|---|
| Copias de seguridad automáticas (y probar a restaurarlas) | Una copia que nunca se ha restaurado no se sabe si sirve |
| Activar el registro de **consultas lentas** | Ver qué consultas tardan más de X segundos y añadirles índices |
| Mantener las **estadísticas** actualizadas (`ANALYZE`) | La BD decide si usar un índice según lo que sabe de los datos |
| Usuario de la aplicación con **permisos mínimos** | La aplicación necesita `SELECT`, `INSERT`, `UPDATE`, `DELETE`, no `DROP TABLE` |
| Monitorizar espacio, conexiones y tiempos | Detectar problemas antes que los usuarios |

---

## 7. Usar la IA con criterio

La IA puede proponerte mejoras para la base de datos, pero **no conoce tus datos**: no sabe cuántas filas hay, qué consultas se repiten ni qué motor usas en producción. Antes de aplicar una mejora, necesitas criterio para decidir si tiene sentido.

### 7.1 Antes de aplicar una mejora que te propone la IA

| Pregunta | Por qué |
|---|---|
| **¿He medido que hay un problema?** | Sin `EXPLAIN` o sin el registro de consultas lentas, estás optimizando a ciegas |
| **¿Cuántas filas tendrá la tabla?** | Con 100 filas, ninguna mejora se notará |
| **¿Se lee mucho o se escribe mucho?** | Un índice acelera las lecturas y ralentiza las escrituras |
| **¿Qué motor uso en producción?** | Lo que vale en MySQL puede no valer en H2 o PostgreSQL, y al revés |
| **¿Qué cuesta?** | Espacio, escrituras más lentas, más piezas que mantener |
| **¿Puedo probarlo y deshacerlo?** | Primero en desarrollo, con `EXPLAIN` antes y después |

### 7.2 Propuestas típicas de la IA y qué revisar

| La IA propone… | Revisa… |
|---|---|
| "Añade índices a estas columnas" | ¿Aparecen esas columnas en los `WHERE`, `JOIN` u `ORDER BY` que se usan de verdad? |
| Un índice en una columna `true`/`false` | Si la mitad de las filas tienen cada valor, la BD lo ignorará |
| "Quita las `FOREIGN KEY` para ir más rápido" | Ganas muy poco y pierdes la garantía de que los datos son coherentes |
| "Usa una caché" o "pásate a NoSQL" | ¿Has probado antes un índice? Suele ser la solución más simple |
| Sintaxis de otro motor | Comprueba que funciona en el tuyo |

### 7.3 Checklist para revisar el código que te da la IA

1. **¿Las columnas del `WHERE`, `JOIN` y `ORDER BY` tienen índice?** La IA casi nunca los propone si no se lo pides.
2. **¿Usa `LIKE '%...'` o funciones sobre columnas en el `WHERE`?**
3. **¿Hay consultas dentro de un bucle?**
4. **¿Hace `SELECT *` o trae todas las filas sin paginar?**
5. **¿Cierra las conexiones** (try-with-resources)?
6. **¿Los tipos de datos son los adecuados** (fechas, dinero)?

### 7.4 Pregunta mal vs. pregunta bien

| ❌ Prompt pobre | ✅ Prompt con contexto |
|---|---|
| "Haz una consulta que busque pedidos por provincia" | "La tabla `pedido` tendrá unos 2 millones de filas y se busca por provincia y fecha constantemente. Dame la consulta y el índice que necesita, y explícame cómo comprobarlo con `EXPLAIN`." |
| "Mi aplicación va lenta, optimízala" | "Esta consulta tarda 3 segundos en una tabla de 500.000 filas. Esta es la salida de `EXPLAIN`: [pegar]. ¿Qué índice añadirías y qué coste tendría en los `INSERT`?" |

El buen prompt le da a la IA lo que no sabe: **cuántos datos hay, qué se consulta y qué has medido**.

---

## 8. Practica

| | Pregunta |
|---|---|
| A | La pantalla de libros filtra por autor y va lenta. ¿Qué haces primero? |
| B | ¿Necesita índice la columna `isbn` si es `UNIQUE`? |
| C | ¿Por qué no poner índices en todas las columnas? |
| D | `WHERE titulo LIKE '%amor%'` va lenta aunque `titulo` tiene índice. ¿Por qué? |
| E | Tras un rato funcionando, la aplicación se queda colgada sin dar error. Usa JDBC. ¿Qué revisas? |
| F | ¿En qué tipo guardas el precio de un producto? |

<details>
<summary>Soluciones</summary>

| | Respuesta |
|---|---|
| A | `EXPLAIN` de la consulta. Si lee la tabla entera, índice en `autor_id` |
| B | No: `UNIQUE` ya crea un índice |
| C | Ocupan espacio y hacen más lentos los `INSERT`, `UPDATE` y `DELETE` |
| D | Con `%` al principio el índice no sirve |
| E | Que todas las conexiones se cierren con try-with-resources: el pool se ha quedado sin conexiones |
| F | `DECIMAL(10,2)`, nunca `DOUBLE` |

</details>

---

## Resumen en tres frases

1. **Un índice hace que la BD vaya directa a las filas** en vez de leer la tabla entera, pero hace más lentas las escrituras: indexa lo que más se consulta.
2. **`EXPLAIN` te dice si tu consulta usa el índice**: no lo supongas, compruébalo.
3. **La IA te da consultas que funcionan con 10 filas;** tú tienes que pensar en cómo irán con millones.
