# Programar en la era de la IA

<img width="396" height="461" alt="image" src="https://github.com/user-attachments/assets/ac9636ab-4503-4640-9486-ab5c75d44b1c" />

- **La función de la programación ha cambiado.** Ya no se trata solo de escribir sintaxis o teclear código, sino de saber pensar, estructurar problemas y validar soluciones.
- **La IA genera código, no criterio.** Copia y ensambla patrones conocidos, pero no entiende la lógica del negocio, la arquitectura de la aplicación ni las consecuencias reales de un fallo en producción.
- **El riesgo del "vibe coding" o copiar sin entender.** Si no sabes programar, no puedes comprobar si el código que te da la IA es seguro, eficiente o correcto. Y cuando falle (y fallará), no sabrás depurarlo ni encontrar los errores ocultos.
- **Las bases no han muerto, han cambiado de sitio.** Escribir código a mano es hoy solo una parte del trabajo. Lo que antes se aprendía para escribir código, ahora se necesita para **leerlo, revisarlo y decidir si es válido**. El diseño, la depuración compleja y la integración de sistemas siguen requiriendo pensamiento humano.

## ¿Fundamentos o frameworks?

Las empresas ya no buscan a alguien que escriba un CRUD de memoria; eso lo hace la IA en segundos. Buscan a alguien capaz de mirar ese CRUD y ver que cualquiera puede borrar registros, que se puede inyectar SQL en el buscador o que la página tarda diez segundos cuando hay cien mil filas. Y eso solo lo ve quien entiende cómo funciona por dentro.

- **Los frameworks cambian; los fundamentos se quedan.** Spring, Thymeleaf o JPA cambiarán de versión o se sustituirán por otros. Saber leer código, depurar y detectar un fallo de seguridad o de rendimiento sirve con cualquier tecnología.
- **Conocer muchas herramientas por encima no basta.** Quien conoce diez frameworks pero no sabe verificar lo que hace ninguno es justo el perfil que la IA ya puede sustituir.

## La realidad del mercado: proyectos legacy

En clase trabajamos con las últimas versiones, pero en el mundo real gran parte del trabajo consiste en **mantener, migrar o ampliar proyectos legacy**: código heredado con años de antigüedad y versiones obsoletas de lenguajes y frameworks.

- En estos entornos la IA pierde gran parte de su eficacia: mezcla sintaxis de distintas versiones, inventa funciones que no existían en esa época o rompe dependencias delicadas sin darse cuenta.
- Cuando te toque mantener una aplicación crítica que lleva diez años en producción, la IA no te va a sacar las castañas del fuego.
- Ahí solo sobrevive quien sabe leer código ajeno, entender cómo funciona la tecnología por debajo y depurar paso a paso.

## ¿Por qué un examen práctico sin IA?

- **Orquestar es dirigir y verificar.** Sin saber leer código ni localizar un error, no se orquesta: se confía a ciegas. Ese criterio se construye programando sin ayuda.
- **Cuanto más se delega, más criterio hace falta.** Con agentes trabajando en paralelo se genera más código del que nadie puede leer con calma, así que hay que saber decidir rápido qué se acepta y qué no.
- **A un junior se le contrata por su criterio.** Hay quien encuentra trabajo sin saber programar, pero son casos sueltos, no una tendencia. Precisamente porque la IA ya hace las tareas típicas de un junior, lo que se valora en él es su criterio.
- **Aprobar con IA sin saber no es una ventaja.** Solo retrasa el momento de descubrirlo: en la entrevista o en las prácticas en empresa.

## ¿Por qué habrá un examen con IA en la 2ª evaluación?

- **Adaptarse sin ir más rápido que las empresas.** A día de hoy, y podría cambiar en el futuro, las empresas que se toman en serio la selección evalúan primero sin IA y después con IA.
- **FFE:** porque en las FFE y en vuestro primer empleo la vais a usar a diario, y queremos que lleguéis sabiendo dirigirla con rigor, no improvisando

## ¿Cómo será el examen con IA?

- **No se trata de IA sí o IA no.** Se trata de que la nota diga la verdad sobre lo que sabéis hacer.
- **Que funcione no es suficiente.** Una solución que funciona no garantiza el aprobado.
- **Tendréis que defender lo que entregáis.** Explicar cómo funciona y modificarlo en directo, sin IA. Si no sabes explicar tu código, ese código no cuenta como tuyo.

## Posible formato: sin IA y con IA en la misma prueba

Es posible que en la 2ª evaluación las dos pruebas prácticas se hagan en un mismo examen, en dos fases:

1. **Sin internet: revisar como programadores.** Partiréis de un proyecto supuestamente generado con IA. Tendréis que leerlo, entenderlo y detectar sus problemas: capas lógicas mal implementadas, patrones mal aplicados, errores en la base de datos, fallos de seguridad, bugs... Entregaréis un informe con los fallos encontrados, dónde están y cómo los corregiríais.
2. **Con IA: corregir con eficiencia.** Una vez entregado el informe, usaréis la IA para implementar las correcciones y mejoras que habéis propuesto. La IA os ayuda a corregir lo que vosotros habéis detectado.

Es lo que se hace en una empresa: revisar código ajeno o generado, decidir qué falla y por qué, y después usar las herramientas para resolverlo rápido.

## Conclusión

- El criterio para usar la IA de forma eficiente solo se adquiere programando sin ella.

- El modelo tradicional de aprendizaje, en el que el junior gana experiencia peleándose con problemas reales bajo la tutela de un senior, se está rompiendo. El junior que no ha pasado por esa “lucha productiva” no podrá revisar ni corregir lo que genera la IA. Por eso los fundamentos importan más que antes, no menos.

- Ocurre lo mismo que con la calculadora. Su llegada no hizo que dejáramos de aprender a sumar. Quien no sabe sumar acepta cualquier número que aparezca en la pantalla.
  - Con la IA la diferencia es todavía mayor. La calculadora no se equivoca si se usa bien; la IA sí, y además sus errores parecen correctos: código que compila y funciona en el caso fácil, pero que es inseguro, ineficiente o rompe la arquitectura. Para detectarlos hay que saber programar.
- Dejar de aprender a programar porque existe la IA nos dejaría sin la capacidad de decidir qué le pedimos y de comprobar si lo que nos devuelve está bien.

> **Cuidado con el "síndrome del impostor".** Hay estudiantes brillantes que sienten que usar IA es "hacer trampas" incluso cuando ya tienen una buena base. Una vez superada la fase sin IA, no usarla en el día a día te hace menos competitivo. ¡Úsala!
