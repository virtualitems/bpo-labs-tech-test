## ❌ Error: Registro fallido

### `POST /auth/signup`

1. El usuario envía los datos desde el frontend.
2. **Amazon Cognito** detecta email ya registrado y lanza error.
3. La respuesta de error es devuelta por **Amazon API Gateway** al frontend.

---

## ❌ Error: Inicio de sesión inválido

### `POST /auth/login`

1. El usuario envía credenciales incorrectas.
2. **Amazon Cognito** rechaza la autenticación.
3. Se devuelve mensaje de error de login inválido al frontend.

---

## ❌ Error: Evento no encontrado

### `GET /events/{eventId}`

1. El usuario consulta un `eventId` que no existe.
2. **AWS Lambda** no encuentra resultado en **Amazon Aurora**.
3. La Lambda devuelve `404` al **Amazon API Gateway**.
4. API Gateway responde al cliente con error de recurso no encontrado.

---

## ❌ Error: Tickets no disponibles al reservar

### `PUT /tickets`

1. El usuario intenta reservar tickets ya reservados o adquiridos.
2. **AWS Lambda** en **DynamoDB** recibe `ConditionalCheckFailedException`.
3. La Lambda devuelve error al **Amazon API Gateway**.
4. El frontend recibe mensaje indicando que los tickets no están disponibles.

---

## ❌ Error: Creación de orden fallida tras reserva

### `PUT /tickets`

1. Los tickets son reservados exitosamente.
2. La llamada a **`lambda-orders-create`** falla al insertar en **Amazon Aurora**.
3. Se lanza rollback lógico: la Lambda libera los tickets en **DynamoDB**.
4. Se devuelve error al usuario indicando que no se pudo completar la orden.

---

## ❌ Error: Orden ya pagada o no existente en webhook

### `POST /payments/webhook`

1. El proveedor de pagos envía `orderId` inválido o duplicado.
2. **AWS Lambda** no encuentra la orden o ya tiene `status = completed`.
3. No se actualizan datos.
4. Se registra el intento fallido y se retorna `200 OK` sin acciones.

---

## ❌ Error: Tickets no encontrados al validar

### `GET /tickets/{ticketId}`

1. El escáner envía un `ticketId` inválido.
2. **AWS Lambda** no encuentra el item en **DynamoDB**.
3. Se devuelve mensaje de “ticket inválido” al escáner.

---

## ❌ Error: Usuario sin órdenes encontradas

### `GET /user/{userId}/orders`

1. El usuario autenticado no tiene órdenes en **Amazon Aurora**.
2. **AWS Lambda** retorna una lista vacía.
3. El frontend muestra mensaje de “sin compras registradas”.

---

## ❌ Error: Orden no encontrada

### `GET /orders/{orderId}`

1. El usuario solicita un `orderId` inexistente o que no le pertenece.
2. **AWS Lambda** no encuentra coincidencia válida en **Amazon Aurora**.
3. Se retorna `404` o `403` al frontend según el caso.