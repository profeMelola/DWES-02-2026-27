# Guía de uso de la IA para programar

> La IA es una herramienta, no un sustituto de tu criterio. Tú diseñas, tú decides y tú respondes del código que entregas.

---

## Fase 1 · Pensar (sin IA y sin código)

### 1. Entiende el problema
- ¿Qué hay que resolver exactamente? Explícalo con tus palabras en 2-3 frases.
- ¿Qué entradas hay y qué salidas se esperan?
- Si no sabes explicarlo, todavía no puedes pedírselo a nadie, tampoco a la IA.

### 2. Conoce tus recursos
- **Estructura del proyecto:** carpetas, ficheros, capas (controladores, modelos, vistas...).
- **Limitaciones:** versión de PHP, framework, librerías permitidas, base de datos.
- **Excepciones y casos límite:** datos vacíos, valores incorrectos, errores de conexión...

### 3. Diseña
- **Diagrama de flujo funcional:** qué hace el programa, paso a paso, desde el punto de vista del usuario.
- **Diagrama de flujo técnico:** qué clases, métodos y llamadas intervienen.
- Piensa en **escalabilidad, reutilización y legibilidad** antes de escribir nada.
- Define los **casos de prueba**: qué resultado esperas para cada entrada.

---

## Fase 2 · Proteger

### 4. Separa los datos que NO compartes con la IA
- Credenciales, contraseñas, claves de API, ficheros `.env`.
- URLs de endpoints internos o privados.
- Constantes y reglas de negocio de la empresa.
- Datos personales de cualquier persona.

### 5. Lo que sí puedes compartir (con criterio)
- Nombres genéricos de clases, métodos y variables, **si la política de la empresa o del centro lo permite**.
- Fragmentos de código sin datos sensibles.
- Ante la duda: **pregunta antes de pegar**.

---

## Fase 3 · Preguntar a la IA

### 6. Prepara bien el prompt
- **Un chat por tarea.** No mezcles temas distintos en la misma conversación.
- **Divide el prompt en secciones:** contexto, objetivo, restricciones y formato de respuesta.
- Sé completo y concreto: no dejes nada para que la IA lo adivine.
- Evita pegar PDFs, imágenes o ficheros enteros si no hace falta: cuanto más contexto innecesario, peores respuestas.

### 7. Elige cómo pedirlo
- **Estructura:** le das tus clases, métodos y variables y le pides que los complete u organice.
- **Caso hipotético:** le planteas un ejemplo parecido para entender cómo resolverlo y luego lo aplicas tú.
- **Mixto:** combinas ambos.

### 8. Pide siempre el razonamiento
- Pide la explicación junto con el código.
- **Ojo:** la IA explica con la misma seguridad algo correcto que algo erróneo. El razonamiento sirve para que tú lo compruebes, no para creértelo.

### 9. Si entras en bucle, para
- Si llevas varios intentos sin avanzar, el problema suele estar en tu planteamiento, no en la IA.
- Vuelve a la Fase 1, revisa el diseño y abre un chat nuevo con un prompt mejor.

---

## Fase 4 · Verificar

### 10. Revisa línea por línea
- No copies nada que no entiendas.
- Si puedes, **escríbelo tú a mano** en lugar de copiar y pegar: así lo entiendes de verdad.
- Contrasta funciones y librerías con la **documentación oficial**: la IA a veces se inventa métodos o usa versiones antiguas.

### 11. Prueba en un entorno aislado
- Nunca pruebes directamente sobre datos o sistemas reales.
- Comprueba tus casos de prueba del paso 3, incluidos los casos límite.

### 12. Guarda versiones
- Haz un **commit antes de integrar** código generado, para poder volver atrás si algo falla.

---

## Resumen rápido

| Fase | Pregunta clave |
|---|---|
| 1. Pensar | ¿Sé explicar y diseñar la solución sin IA? |
| 2. Proteger | ¿Estoy compartiendo algo que no debo? |
| 3. Preguntar | ¿Mi prompt es claro, completo y de una sola tarea? |
| 4. Verificar | ¿Entiendo cada línea y la he probado? |

> **Regla de oro:** si no puedes defender tu código delante del profesor sin mirar el chat, no es tu código.
