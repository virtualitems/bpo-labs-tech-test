## 🛡️ Prevención: Ataques de bots y scraping masivo

### 🔐 Mitigación automática con WAF + Bot Control

1. El usuario o bot realiza múltiples solicitudes al frontend o API.
2. La petición llega primero a **Amazon CloudFront** (si es sitio web) o a **Amazon API Gateway** (si es API).
3. Ambas están protegidas con **AWS WAF** asociado a la Web ACL `waf-ticketing-api`.
4. **Reglas activas en WAF:**
   - Reglas administradas por AWS: `AWSManagedRulesCommonRuleSet`, `AWSManagedRulesBotControlRuleSet`.
   - Reglas personalizadas: bloqueo por IP, User-Agent sospechoso, tasas anormales.
5. Si se detecta comportamiento malicioso:
   - WAF responde con HTTP 403 sin pasar al backend.
   - El evento se registra en **AWS CloudWatch Logs**.

---

## 🛡️ Prevención: Automatización de compras (scalpers)

### 🔐 Validación por autenticación + anti-abuso

1. El bot intenta automatizar la compra de boletos accediendo a `PUT /tickets` o `POST /orders`.
2. **Amazon API Gateway** verifica el token JWT de **Amazon Cognito**.
   - Solo usuarios autenticados pueden reservar.
3. Para flujos sensibles como `PUT /tickets`, se puede activar un **CAPTCHA** en el frontend.
4. Se registra la IP del usuario en logs internos o en Redis para aplicar **rate limit individual**:
   - Ej: no permitir más de 5 reservas por minuto por usuario o IP.
5. Si se detectan patrones anómalos, WAF puede bloquear la IP temporalmente.

---

## 🛡️ Prevención: Inyección SQL y comandos maliciosos

### 🔐 Validación de entrada + ORM seguro

1. El atacante intenta enviar SQL malicioso en campos como `email`, `eventId`, etc.
2. Todos los endpoints en **Amazon API Gateway** pasan el payload a **AWS Lambda**.
3. Las Lambdas:
   - Validan tipos, caracteres y formatos
   - Usan **consultas parametrizadas** con bindings en Aurora:
     ```sql
     SELECT * FROM orders WHERE user_id = :userId
     ```
4. No hay concatenación directa de strings SQL.
5. Si la validación falla, se retorna error 400 sin acceder a la base de datos.

---

## 🛡️ Prevención: Reintentos de pago maliciosos o spoofing de webhooks

### 🔐 Validación estricta del webhook del proveedor de pagos

1. Un atacante intenta falsificar el webhook `POST /payments/webhook`.
2. La solicitud llega a **Amazon API Gateway**.
3. La **AWS Lambda** (`lambda-payments-webhook`) valida:
   - La firma HMAC del webhook.
   - El `origin` o `host` permitido.
   - Que no sea una repetición (`idempotency check`).
4. Si falla la validación:
   - Se responde con HTTP 403.
---

## 🛡️ Prevención: Enumeración de usuarios o scraping de datos personales

### 🔐 Respuestas genéricas

1. El atacante intenta usar `POST /auth/signup` o `POST /auth/login` para descubrir si un email ya existe.
2. **Amazon Cognito** no revelaría si el email está registrado por defecto.
3. Si se usa una Lambda como wrapper:
   - Se retorna siempre el mismo mensaje: "Si tus datos son válidos, recibirás un correo".
4. Adicionalmente:
   - Se puede aplicar **rate limiting por IP o por email** en **ElastiCache for Redis**.
   - Se bloquean IPs sospechosas con **AWS WAF** o mediante integración con servicios de inteligencia (lista negra).

---

## 🛡️ Prevención: Uso de credenciales robadas o tráfico sospechoso

1. El atacante intenta iniciar sesión desde una IP o dispositivo inusual.
2. **Amazon Cognito** puede activarse con **MFA adaptable** (requiere login desde dispositivo confiable).