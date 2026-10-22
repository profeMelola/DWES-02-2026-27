# Ejercicio 2. Personajes de Springfield en Spring MVC (autónomo)

> Migra a Spring Boot la práctica de Jakarta EE [Personajes de Springfield: primer contacto con el API Stream](https://github.com/profeMelola/DWES-01-2026-27/blob/main/ejercicios/simpsons.md).
> Esta vez **no hay solución guiada**: aplica directamente lo visto en el [Alta de usuario v2](./alta-usuario-spring-validacion.md) (DTO, `@ModelAttribute`, `@Valid`, `BindingResult`, `th:field`, `messages.properties`).

![alt text](https://raw.githubusercontent.com/profeMelola/DWES-01-2026-27/main/ejercicios/image-5.png)

## Punto de partida

- [Recursos de la práctica Jakarta](https://github.com/profeMelola/DWES-01-2026-27/tree/main/ejercicios/ejercicio-03-simpson/recursos): `PersonajeRepositorio`, `personajes.jsp`, `estilos.css`.
- La lógica de streams que ya hiciste en el `PersonajeServicio` de la UD1 se **reutiliza**.

## Proyecto

| Campo | Valor |
|---|---|
| Artifact | `simpsons-spring` |
| Package | `es.daw.simpsons` |
| Java | 25 |
| Dependencias | Spring Web, Thymeleaf, Validation, Lombok, DevTools |

---

## Requisitos

### 1. Capas

| Capa | Clase | Responsabilidad |
|---|---|---|
| Modelo | `Personaje` (`record en paquete model`) | Datos de un personaje (solo lectura) |
| Datos | `PersonajeRepositorio` (`@Repository`) | La lista en memoria de la UD1 |
| Lógica | `PersonajeServicio` (`@Service`) | Filtrar, ordenar y limitar con **streams** |
| Presentación | `PersonajeController` (`@Controller`) | Recibir el filtro, validarlo, llamar al servicio y elegir la vista |
| Formulario | `FiltroPersonajesDto` (clase + Lombok) | Los campos del formulario de búsqueda |

- Inyección **por constructor** en todas las capas.
- El controlador no contiene ningún stream: toda la lógica está en el servicio.

### 2. Funcionalidad

1. `GET /` redirige a `/personajes`.
2. `GET /personajes` muestra el formulario de filtros y la tabla de resultados.
3. Al entrar sin filtros se muestran **todos** los personajes, ordenados por nombre.
4. El formulario se envía por **GET** (los filtros quedan en la URL y se pueden compartir).
5. Se muestra el número de personajes encontrados.
6. Los menores de 18 años aparecen resaltados (clase CSS `menor`).
7. Al buscar, el formulario **conserva** los filtros elegidos (`th:object` + `th:field`).

### 3. Filtros del DTO y validaciones

| Campo | Tipo | Validación | Operación del stream |
|---|---|---|---|
| `lugar` | `String` | Opcional | `filter` |
| `ocupacion` | `String` | Opcional | `filter` |
| `edadMax` | `Integer` | Opcional. Entre 0 y 120 | `filter` |
| `soloFamilia` | `boolean` | — | `filter` por `principal` |
| `ordenarPor` | `String` | Obligatorio. Solo `nombre`, `apellido`, `edad` o `lugar` | `sorted(Comparator...)` |
| `descendente` | `boolean` | — | `reversed()` |
| `limite` | `Integer` | Opcional. Entre 1 y 50 | `limit(n)` |

- Las listas de **lugares** y **ocupaciones** de los `<select>` salen de los datos (sin repetidos y ordenadas), no escritas a mano. Cárgalas con métodos `@ModelAttribute`.
- Si hay errores de validación: se muestra cada error **junto a su campo** y **no** se muestra la tabla.
- Si al escribir una letra en un campo numérico el error es poco claro, búscalo en `BindingResult` y dale un mensaje propio en `messages.properties`.
- Todos los mensajes de error salen de `messages.properties`.
- Al ordenar por apellido, edad o lugar, los empates se resuelven por nombre.

### 4. Vista

- Una única plantilla `personajes.html` con el formulario y la tabla.
- Si no hay resultados, mostrar *"No hay personajes con esos filtros"* en lugar de la tabla vacía.
- Enlace **Limpiar filtros** que vuelve a `/personajes`.
- `estilos.css` en `static/css/`.

---



[← Volver a ejercicios](./readme.md)
