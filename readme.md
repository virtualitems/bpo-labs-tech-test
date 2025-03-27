# AWS Tickets

- [Diagrama de arquitectura](./diagram.png)

- [Flujo principal](./flow.md)

- [Esenarios de error](./error.md)

- [Medidas de seguridad](./security.md)

## Información adicional

### Se asume que

- Se tiene un dominio registrado y se apunta a **Amazon Route 53**.

- Se tiene un certificado SSL en **AWS Certificate Manager**.

- Se tiene un usuario IAM con permisos para crear recursos.

- Se tienen los grupos de seguridad configurados para permitir tráfico.

- Se tiene un frontend con Client Side Rendering para construír la GUI sin requerir Server Side Rendering.

### ¿Por qué lambdas en lugar de contenedores?

- **Escalabilidad**: Lambda escala automáticamente con la demanda y se limita a la ejecución en vez de crear instancias para operaciones sensillas.

- **Operaciones transaccionales**: Las operaciones requeridas son transaccionales, dedicadas sólo a actualizar registros en Aurora o DynamoDB.

- **Costo**: Se paga por ejecución, no por tiempo de contenedor, lo que reduce costos en comparación a crear un Load Balancer con un grupo de instancias que cobran cada una por tiempo de actividad.

### ¿Por qué DynamoDB en lugar de RDS para los tickets?

- **Velocidad**: DynamoDB es más rápido para operaciones de lectura y los tickets los consultarán constantemente los usuarios.

### ¿Por qué Aurora en lugar de un EC2 con Postgres?

- **Escalabilidad**: Aurora es más escalable y tolerante a fallos que un EC2 con Postgres.

- **Resiliencia**: Aurora se puede replicar automáticamente en múltiples zonas de disponibilidad y puede tomar snapshots (de forma segura) para restaurar en caso de fallo.

### ¿Archivos estáticos para el frontend?

- Amazon S3 y CloudFront

### ¿Dónde puedo ver logs?

- **Amazon CloudWatch** para logs de Lambda y API Gateway.

### Servicios de control de costos

- Se usaría **AWS Budgets** para monitorear gastos con alertas y **AWS Trusted Advisor** para recomendaciones de ahorro.