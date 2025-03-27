## 👨‍💼 Administración: Crear eventos y cargar tickets

### `POST /events`

1. Un usuario administrador envía los datos del evento a través de **Amazon API Gateway**.
2. Una **AWS Lambda** (`lambda-events-create`) persiste el evento en **Amazon Aurora**:
   ```sql
   INSERT INTO events (id, name, description, date_start, date_end, venue_id, created_at)
   VALUES (...);
   ```

---

### `POST /tickets`

1. El administrador carga los boletos de un evento a través de **Amazon API Gateway**.
2. Una **AWS Lambda** (`lambda-tickets-create`) guarda los tickets en **Amazon DynamoDB**:
   ```js
   PUT item INTO tickets_event_<eventId> {
     ticket_id, zone_id, seat_number, status: "free", created_at
   }
   ```

---

## 👤 Registro de usuario

### `POST /auth/signup`

1. El usuario envía los datos desde el frontend.
2. La solicitud pasa por **Amazon API Gateway**.
3. **Amazon Cognito** registra al usuario y devuelve un token JWT.
4. Una **AWS Lambda** (`lambda-auth-signup`) guarda información adicional en **Amazon Aurora**:
   ```sql
   INSERT INTO users (id, name, email, password_hash, document_number, created_at)
   VALUES (:id, :name, :email, :hash, :document_number, NOW());
   ```

---

## 👤 Inicio de sesión

### `POST /auth/login`

1. El frontend envía credenciales al endpoint de **Amazon Cognito**.
2. Cognito valida y responde con un token JWT.

---

## 🎟️ Explorar eventos y disponibilidad

### `GET /events`

1. El frontend llama al endpoint a través de **Amazon API Gateway**.
2. **AWS Lambda** (`lambda-events-list`) busca en **Amazon ElastiCache for Redis**:
   ```redis
   GET events:list
   ```
3. Si no hay caché, consulta **Amazon Aurora**:
   ```sql
   SELECT * FROM events WHERE deleted_at IS NULL;
   ```
4. Se guarda en Redis y se retorna.

---

### `GET /events/{eventId}`

1. El frontend consulta los detalles a través de **Amazon API Gateway**.
2. **AWS Lambda** (`lambda-events-detail`) busca en **Redis**:
   ```redis
   GET events:{eventId}
   ```
3. Si no existe, consulta **Amazon Aurora**:
   ```sql
   SELECT * FROM events WHERE id = :eventId;
   SELECT * FROM zones WHERE event_id = :eventId;
   ```

---

### `GET /events/{eventId}/tickets`

1. El frontend consulta disponibilidad por zona a través de **Amazon API Gateway**.
2. **AWS Lambda** (`lambda-tickets-availability`) escanea **Amazon DynamoDB**:
   ```js
   SCAN tickets_event_<eventId> WHERE status = "free"
   ```

---

## 🎟️ Reservar boletos y crear orden

### `PUT /tickets`

1. El usuario envía su selección de boletos vía **Amazon API Gateway**.
2. **AWS Lambda** (`lambda-tickets-reserve`) intenta reservar los tickets:
   ```js
   UPDATE ... SET status = "reserved", reserved_by = :userId, reservation_expires_at = :ttl
   CONDITION status = "free"
   ```
3. Si todos se reservan correctamente:
   - `lambda-tickets-reserve` llama internamente a **`lambda-orders-create`**:
     ```sql
     INSERT INTO orders (...) VALUES (...)
     ```
   - Luego, inicia una ejecución de **AWS Step Functions** (`sf-reservation-timeout`) para cada ticket reservado.
     - Dentro del flujo, tras un `Wait`, se invoca **`lambda-ticket-reservation-check`**.
     - Esta Lambda libera el ticket si sigue en estado `reserved`.

---

## 💳 Confirmación de pago

### `POST /payments/webhook`

1. El proveedor externo envía el webhook a través de **Amazon API Gateway**.
2. **AWS Lambda** (`lambda-payments-webhook`) valida la firma.
3. Consulta la orden en **Amazon Aurora**:
   ```sql
   SELECT * FROM orders WHERE id = :orderId;
   ```
4. Actualiza su estado:
   ```sql
   UPDATE orders SET status = 'completed';
   ```
5. Cambia el estado de los tickets en **Amazon DynamoDB**:
   ```js
   UPDATE ... SET status = "acquired", acquired_at, order_id
   CONDITION status = "reserved" OR status = "free"
   ```
6. Envía notificación por **Amazon SNS** o **Amazon SES**.

---

## 🧾 Ver historial y detalles de compra

### `GET /user/{userId}/orders`

1. El usuario autenticado consulta su historial vía **Amazon API Gateway**.
2. **AWS Lambda** (`lambda-orders-by-user`) consulta **Amazon Aurora**:
   ```sql
   SELECT * FROM orders WHERE user_id = :userId ORDER BY created_at DESC;
   ```

---

### `GET /orders/{orderId}`

1. El frontend consulta una orden específica vía **Amazon API Gateway**.
2. **AWS Lambda** (`lambda-orders-detail`) consulta **Amazon Aurora**:
   ```sql
   SELECT * FROM orders WHERE id = :orderId;
   ```

---

## 🎟️ Validación de ticket en el evento

### `GET /tickets/{ticketId}`

1. El lector escanea el ticket en el punto de ingreso.
2. La solicitud llega por **Amazon API Gateway**.
3. **AWS Lambda** (`lambda-tickets-validate`) consulta **Amazon DynamoDB**:
   ```js
   GET item FROM tickets_event_<eventId> WHERE ticket_id = :ticketId;
   ```
4. Valida:
   - `status === "acquired"`
   - `acquired_at != null`
5. Devuelve acceso permitido o denegado.