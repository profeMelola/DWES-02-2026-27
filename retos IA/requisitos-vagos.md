# De requisito vago a especificación

La idea es simular el trabajo real: un cliente pide algo de forma imprecisa, y el valor está en convertir esa petición en algo comprobable antes y después de que la IA escriba el código.

### Ejemplo sobre la LogísticaApp

> «Queremos que los clientes puedan cancelar sus pedidos y que los pedidos urgentes salgan antes. Ah, y que no se pueda enviar más de lo que tenemos en el almacén.»

Pincha en cada ambigüedad para ver la pregunta que debería surgir y posibles respuestas del cliente.

<details>
<summary><strong>1. Cancelar</strong></summary>

**Pregunta:** ¿En qué estados se puede cancelar un pedido? ¿Un pedido ya enviado se puede cancelar?

**Posibles respuestas del cliente:**
- Solo se puede cancelar mientras está *Pendiente*. Una vez *Preparado* o *Enviado*, no.
- Se puede cancelar hasta que sale del almacén. Si ya está *Enviado*, se tramita como devolución, que queda fuera de este alcance.
- Se puede cancelar siempre, pero si ya está *Enviado* queda marcado como *Cancelado con incidencia* para revisión manual.

</details>

<details>
<summary><strong>2. Stock</strong></summary>

**Pregunta:** ¿Al cancelar se devuelve el stock al almacén?

**Posibles respuestas del cliente:**
- Sí, al cancelar se repone automáticamente la cantidad de cada línea del pedido.
- Solo si el pedido no había salido del almacén. Si estaba *Enviado*, el stock se repone cuando llegue la devolución física.
- No se toca el stock: lo ajusta a mano el encargado del almacén.

</details>

<details>
<summary><strong>3. Urgente</strong></summary>

**Pregunta:** ¿Quién decide que un pedido es urgente? ¿Cuesta más?

**Posibles respuestas del cliente:**
- Lo marca el cliente al hacer el pedido y tiene un recargo fijo de envío.
- Lo marca solo un administrador, por ejemplo para clientes preferentes. No tiene coste extra.
- Es automático: los pedidos de importe superior a un límite son urgentes.

</details>

<details>
<summary><strong>4. «Salgan antes»</strong></summary>

**Pregunta:** ¿Significa orden en el listado, prioridad de envío o ambas?

**Posibles respuestas del cliente:**
- Solo orden: en el listado del almacén los urgentes aparecen primero.
- Prioridad real: un urgente debe enviarse en el mismo día si se hace antes de una hora de corte.
- Ambas: aparecen primero y además tienen un plazo máximo de preparación.

</details>

<details>
<summary><strong>5. «Lo que tenemos»</strong></summary>

**Pregunta:** ¿El stock se comprueba al crear el pedido o al enviarlo?

**Posibles respuestas del cliente:**
- Al crear el pedido: si no hay stock suficiente, no se deja confirmar y se muestra un aviso.
- Al crear se reserva el stock, y al enviar se descuenta definitivamente.
- Se puede crear sin stock, pero no se puede pasar a *Enviado* hasta que haya existencias.

</details>

<details>
<summary><strong>6. Concurrencia</strong></summary>

**Pregunta:** ¿Qué pasa si dos pedidos piden el último artículo a la vez?

**Posibles respuestas del cliente:**
- Gana el primero que se confirma. El segundo recibe un aviso de falta de stock.
- Si uno de los dos es urgente, tiene preferencia.
- Ninguno se rechaza: el segundo queda *Pendiente de stock* hasta que haya reposición.

</details>

<details>
<summary><strong>7. Permisos</strong></summary>

**Pregunta:** ¿Puede cancelar el propio cliente o solo un administrador?

**Posibles respuestas del cliente:**
- El cliente puede cancelar sus propios pedidos. El administrador puede cancelar cualquiera.
- Solo el administrador cancela, a petición del cliente por teléfono o correo.
- El cliente solicita la cancelación y un administrador la aprueba.

</details>
