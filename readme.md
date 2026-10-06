# Programación web con Spring Framework & Spring Boot

## Primeros pasos

### Spring Boot ¿Qué es y cómo funciona?

Spring Boot es una herramienta dentro del framework Spring.

Es una extensión del framework Spring que tiene como finalidad simplificar la creación y configuración inicial de aplicaciones web.

Solo necesita una configuración básica, que permite el uso de librerías, integración con otras herramientas o incluso otros proyectos Spring.



### Spring Platform

Es un conjunto de proyectos open source desarrollados en Java con el objetivo de agilizar el proceso de desarrollo de aplicaciones.

Cuenta con cantidad de herramientas (tecnologías) que tienen como objetivo facilitar el trabajo de los desarrolladores.


### Estructura básica de un proyecto Spring

```
src/
 ├── main/
 │    ├── java/
 │    │    └── com.example.myapp/         // Paquete base de la aplicación
 │    │         ├── MyAppApplication.java // Clase principal con @SpringBootApplication
 │    │         ├── controller/           // Controladores REST
 │    │         ├── service/              // Lógica de negocio
 │    │         ├── repository/           // Acceso a la base de datos
 │    │         ├── model/                // Clases del modelo de datos 
 │    │         ├── entity/               // Entidades JPA
 │    │         └── config/               // Configuración personalizada
 │    └── resources/
 │         ├── application.properties     // Configuración principal de Spring Boot
 │         ├── application.yml            // (Alternativa) Configuración en formato YAML
 │         ├── static/                    // Archivos estáticos (HTML, CSS, JS)
 │         ├── templates/                 // Plantillas Thymeleaf o Freemarker
 │         └── db/                        // Scripts SQL (opcional)
 └── test/
      ├── java/                           // Pruebas unitarias y de integración

```

## Principales anotaciones

| Anotación | Rol en la aplicación | Tipo |
|----------|-----------------------|------|
| @SpringBootApplication | Clase principal de inicio | Configuración |
| @RestController | Controlador REST (devuelve JSON) | Web |
| @Controller | Controlador MVC (devuelve vistas) | Web |
| @Service | Lógica de negocio | Capa de servicio |
| @Repository | Acceso a datos | Capa de persistencia |
| @Entity | Entidad de base de datos | Modelo |
| @Configuration | Define beans/configuración | Configuración |

Un **controlador REST** es una clase de Spring que recibe peticiones HTTP (GET, POST, PUT, DELETE, etc.) y devuelve datos (normalmente en formato JSON o XML).
No devuelve vistas HTML, como haría un controlador tradicional (@Controller), sino respuestas de datos.

Por eso es la base para construir un **API RESTful.**

[Ir a las anotaciones de Spring](./APOYO_TEORIA/componentes-spring.md)

## Primeras aplicaciones

### Primera aplicación Spring "Hola Mundo"
https://www.jetbrains.com/help/idea/your-first-spring-application.html

### Segunda aplicación Spring
https://www.jetbrains.com/help/idea/spring-support-tutorial.html


## Apoyo teoría
- Java en general
    - [Patrones de diseño](./APOYO_TEORIA/patrones.md)
    - [Api Stream](./APOYO_TEORIA/api-stream.md)
    - [Comparator](./APOYO_TEORIA/Comparators.md)
- [Api REST](./APOYO_TEORIA/API-REST.md) 
- [Anotaciones Servicio y Controlador](./APOYO_TEORIA/Anotaciones-Servicio-Controlador.md)
- [Validaciones](./APOYO_TEORIA/Validaciones.md)
- [Spring Security](./APOYO_TEORIA/SpringSecurity.md)
- [JPA](./APOYO_TEORIA/JPA.md)
     - [Comparativa entre JPA y SpringDataJPA](./APOYO_TEORIA/Comparativa_JPA_vs_SpringDataJPA.md)
     - [Asociaciones](./APOYO_TEORIA/JPA-%20Asociaciones.md)
     - [Paginación](./APOYO_TEORIA/Pageable.md)
     - [JPA Avanzado](./APOYO_TEORIA/findAll_Example_QBE.md)
- [Thymeleaf](./APOYO_TEORIA/Thymeleaf.md)

## Webs de referencia

https://spring.io/

https://docs.spring.io/spring-framework/reference/web/webmvc.html

https://www.jetbrains.com/idea/spring/
___

## Página principal del curso
[VOLVER PÁGINA PRINCIPAL](https://github.com/profeMelola/DWES-00-2025-26)

## Licencia

<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/"><img alt="Licencia de Creative Commons" style="border-width:0" src="https://i.creativecommons.org/l/by-nc-sa/4.0/88x31.png" /></a><br />Este obra está bajo una <a rel="license" 
