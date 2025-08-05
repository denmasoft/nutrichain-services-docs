# DOCUMENTACIÓN TÉCNICA: Sistema de Microservicios NutriChain Logistics

## 1. Visión General de la Solución

Este documento detalla la arquitectura y las decisiones de implementación para el sistema de microservicios de NutriChain Logistics. El sistema está compuesto por tres servicios principales desarrollados de forma independiente (`Almacén`, `Tienda`, `Reportes`) y un servicio de catálogo preexistente, todos orquestados a través de Docker.

El objetivo es construir un sistema robusto, escalable y mantenible, siguiendo las mejores prácticas de la industria como Clean Code, principios SOLID y un enfoque de "API-first".

## 2. Decisiones de Arquitectura

### 2.1. Patrón de Microservicios

Se ha adoptado un patrón de microservicios para desacoplar las responsabilidades del negocio.

-   **Ventajas:**
    -   **Escalabilidad Independiente:** Cada servicio (p. ej., `Tienda`) puede escalarse horizontalmente sin afectar a los otros.
    -   **Flexibilidad Tecnológica:** Permite usar la mejor tecnología para cada trabajo (`Node.js` para I/O intensivo, `Python` para análisis de datos, `Nest.js` para una estructura robusta).
    -   **Resiliencia:** Un fallo en el servicio de `Reportes` no afectará la capacidad de crear pedidos en `Tienda`.
    -   **Equipos Autónomos:** Facilita que diferentes equipos trabajen en paralelo.

### 2.2. Comunicación entre Servicios

La comunicación se implementará principalmente de forma síncrona a través de **APIs REST con JSON**.

-   **Tienda -> Almacén:** Al crear un pedido, el servicio `Tienda` realiza una llamada HTTP directa al servicio `Almacén` para validar y actualizar el stock.
    -   **Justificación:** Esta operación es crítica y requiere una confirmación inmediata. Si falla, el pedido no puede ser creado.
    -   **Consideración a Futuro:** Para una mayor resiliencia, se podría evolucionar hacia un patrón de comunicación asíncrono (p. ej., usando RabbitMQ o Kafka). La `Tienda` publicaría un evento `PedidoCreado` y el `Almacén` se suscribiría para procesar la deducción de stock. Esto aumenta la resiliencia pero introduce complejidad (consistencia eventual).

-   **Reportes -> Almacén:** El servicio de `Reportes` accederá directamente a la base de datos de `Almacén` (MySQL).
    -   **Justificación:** Para la generación de reportes analíticos (OLAP), es mucho más eficiente consultar directamente la base de datos que realizar miles de llamadas API. Este patrón se conoce como "Base de Datos por Servicio Compartida" (solo para lectura), lo cual es aceptable para casos de uso de reporting.

### 2.3. Estrategia de Base de Datos (Polyglot Persistence)

Se utiliza una base de datos diferente para cada servicio según sus necesidades, siguiendo el principio de "Polyglot Persistence".

-   **Almacén (MySQL en Supabase):** Se necesita una base de datos relacional y transaccional (ACID) para garantizar la consistencia del inventario. MySQL es una elección sólida y madura.
-   **Tienda (MongoDB en MongoDB Atlas):** Los pedidos tienen una estructura de documento que puede variar (diferentes productos, datos del cliente, etc.). Una base de datos NoSQL como MongoDB ofrece la flexibilidad necesaria para este modelo.
-   **Reportes (Lectura de MySQL en Supabase):** No posee una base de datos propia, sino que lee de la de `Almacén`.

**Nota sobre Supabase:** El requisito indica el uso de Supabase. Supabase es principalmente un proveedor de **PostgreSQL**. Para cumplir estrictamente con los requisitos de MySQL y MongoDB, se utilizarían servicios cloud dedicados como **Amazon RDS for MySQL** o **PlanetScale** para MySQL, y **MongoDB Atlas** para MongoDB. En esta solución, se proporcionarán las variables de entorno para conectar a estos servicios, asumiendo que están provisionados. Para la parte de MySQL, usaremos las credenciales que Supabase nos daría para su instancia de Postgres, asumiendo que fuera MySQL a efectos del ejercicio.

### 2.4. Seguridad

La seguridad se basa en **JSON Web Tokens (JWT)**.

-   **Flujo:** Un servicio de autenticación (no incluido en esta prueba, pero asumido) generaría un token JWT para un usuario.
-   **Implementación:** Cada endpoint protegido en los microservicios (`Almacén`, `Tienda`, `Reportes`) incluirá un middleware o un guard que:
    1.  Extrae el token del header `Authorization: Bearer <token>`.
    2.  Verifica la firma del token usando una clave secreta compartida.
    3.  Valida la expiración y otros claims.
    4.  Si es válido, permite el acceso; de lo contrario, devuelve un error `401 Unauthorized`.

### 2.5. Cross-Cutting Concerns (Preocupaciones Transversales)

Para mantener el código limpio y seguir el principio DRY, las funcionalidades comunes se implementan como middleware o decoradores globales.

-   **Rate Limiting:** Previene ataques de fuerza bruta o abuso de la API. Se implementará un límite de peticiones por IP en un período de tiempo.
-   **Idempotencia:** Crítico para APIs que modifican datos. El cliente puede enviar un header `Idempotency-Key` (un UUID). El servidor almacena esta clave por un tiempo. Si recibe una petición con la misma clave, en lugar de procesarla de nuevo, devuelve la respuesta de la petición original. Esto previene, por ejemplo, la creación de pedidos duplicados por reintentos de red.
-   **CORS:** Configurado a nivel de producción para permitir únicamente las peticiones desde los dominios autorizados (p. ej., el front-end de NutriChain).
-   **Logging:** Se implementará un logger estructurado (JSON) para facilitar el análisis en herramientas como ELK Stack o Datadog. Se registrarán las peticiones, errores y eventos de negocio clave.
-   **Versioning:** Las APIs serán versionadas en la URL (p. ej., `/api/v1/...`) para permitir la evolución de la API sin romper clientes existentes.
-   **Traducciones (i18n):** Los mensajes de error y de éxito serán traducibles, detectando el idioma preferido del cliente a través del header `Accept-Language`.

## 3. Despliegue y Orquestación

-   **Docker:** Cada microservicio se empaquetará en su propia imagen de Docker, garantizando un entorno de ejecución consistente.

---