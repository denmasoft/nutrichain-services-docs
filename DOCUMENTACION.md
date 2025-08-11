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


## 4. Herramientas de Inteligencia Artificial en el Ciclo de Desarrollo

Para acelerar el desarrollo, mejorar la calidad del código y mantener la consistencia en las mejores prácticas, el equipo de NutriChain Logistics ha adoptado un enfoque de **desarrollo asistido por IA**. El objetivo no es reemplazar el juicio del desarrollador, sino potenciarlo, automatizando tareas repetitivas y proporcionando un "par programador" virtual para la resolución de problemas y la generación de código.

### 4.1. Herramientas y Justificación de Uso

Se seleccionó un conjunto de herramientas de IA, cada una orientada a una fase específica del ciclo de vida del desarrollo.

#### 4.1.1. Asistentes de Código en el IDE (GitHub Copilot)

-   **Descripción:** Integrado directamente en el editor de código (VS Code), `GitHub Copilot` ofrece sugerencias de código en tiempo real, desde líneas individuales hasta funciones completas.
-   **Por qué se eligió:**
    -   **Generación de Boilerplate:** La creación de los microservicios `Almacén` y `Tienda` implicó una cantidad significativa de código repetitivo (configuración de servidores Express/Nest.js, definiciones de rutas, controladores básicos de CRUD). Copilot aceleró drásticamente esta fase, permitiendo a los desarrolladores centrarse en la lógica de negocio.
    -   **Implementación de Patrones:** Fue fundamental para generar rápidamente las implementaciones de los "Cross-Cutting Concerns". Por ejemplo, al necesitar un middleware de **Rate Limiting**, se le pudo pedir a Copilot que generara una implementación estándar usando librerías como `express-rate-limit`, adaptándola luego a nuestras necesidades específicas.
    -   **Pruebas Unitarias:** Se utilizó para generar los esqueletos de las pruebas unitarias para los servicios, asegurando que se cubrieran los casos de éxito, de error y los casos límite, reforzando así la robustez del sistema.

#### 4.1.2. LLMs de Propósito General (GPT-4 / Claude 3)

-   **Descripción:** Modelos de Lenguaje Grandes a los que se accede a través de una interfaz de chat para tareas más complejas como la refactorización, depuración, documentación y análisis comparativo.
-   **Por qué se eligieron:**
    -   **Análisis Arquitectónico:** Durante la fase de diseño, se utilizaron para debatir las decisiones descritas en la sección 2. Por ejemplo, se le presentó el escenario `Tienda -> Almacén` y se le pidió que "compare y contraste una implementación síncrona con REST vs. una asíncrona con RabbitMQ, incluyendo ventajas, desventajas y fragmentos de código para ambos enfoques en Node.js". Esto enriqueció la decisión final documentada.
    -   **Refactorización y Clean Code:** Para asegurar el cumplimiento de los principios SOLID, se le proporcionaban fragmentos de código que habían crecido en complejidad y se le solicitaba refactorizarlos. Por ejemplo: "Toma este controlador que valida el JWT y actualiza el stock, y sepáralo siguiendo el Principio de Responsabilidad Única".
    -   **Generación de Documentación:** Partes de este mismo documento, especialmente las justificaciones detalladas y las descripciones de las ventajas, fueron generadas como un primer borrador por un LLM y luego refinadas por el equipo técnico.

#### 4.1.3. Editores de Código Nativos de IA (Cursor)

-   **Descripción:** `Cursor` es un editor de código (un fork de VS Code) diseñado desde cero para la interacción con IA. A diferencia de un simple plugin, tiene una conciencia completa del contexto de todo el proyecto.
-   **Por qué se eligió:**
    -   **Conciencia del Contexto Completo:** Su principal ventaja fue la capacidad de realizar cambios a nivel de todo el repositorio. Por ejemplo, se pudo dar una instrucción como: "En todos los microservicios (`Almacén`, `Tienda`, `Reportes`), busca dónde se valida el JWT y asegúrate de que todos usen la variable de entorno `JWT_SECRET` y manejen los errores de token expirado de forma consistente". `Cursor` es capaz de encontrar y editar todos los archivos relevantes de una sola vez.
    -   **"Chatear con tu Código":** Se utilizó para depurar problemas complejos. En lugar de buscar manualmente, se le preguntaba al editor: "¿Dónde se define la función que deduce el stock del producto? Muéstrame todos los lugares donde se la llama". Esto agilizó radicalmente la navegación y el entendimiento del código heredado o complejo.

### 4.2. Consideraciones y Límites: El Límite de Tokens

Una limitación fundamental de las herramientas de IA actuales es el **límite de tokens**. Un `"token"` es una porción de texto (aproximadamente 4 caracteres en inglés). Cada modelo de IA tiene una **"ventana de contexto"** máxima, que es la cantidad total de tokens (incluyendo la pregunta del usuario y la respuesta de la IA) que puede procesar en una sola interacción.

-   **Impacto en el Desarrollo:** Esto significa que no se puede simplemente "pegar" un microservicio completo y pedir una refactorización. Si el código excede el límite de tokens (p. ej., 32,000 tokens para algunos modelos), la IA no podrá procesarlo.
-   **Estrategia Aplicada:**
    1.  **Modularidad:** El problema se abordó enviando archivos o clases individuales para su análisis, lo cual es una práctica que se alinea perfectamente con los principios de Clean Code y microservicios.
    2.  **Uso de Editores Conscientes del Contexto:** Herramientas como **Cursor** mitigan este problema. Internamente, indexan todo el código base y utilizan técnicas de búsqueda semántica (embeddings) para encontrar y enviar a la IA solo los fragmentos más relevantes para la consulta del usuario, gestionando la ventana de contexto de manera más eficiente.

### 4.3. Resumen del Impacto de la IA

La integración de estas herramientas no reemplazó la necesidad de desarrolladores expertos, sino que los transformó en **"supervisores de IA"**. El resultado fue:

-   **Aceleración del Ciclo de Desarrollo:** Reducción del tiempo dedicado a tareas repetitivas.
-   **Mejora de la Calidad del Código:** Facilidad para aplicar patrones complejos y refactorizar sobre la marcha.
-   **Consistencia:** Aseguramiento de que prácticas como el logging, el manejo de errores y la seguridad se implementaran de manera uniforme en todos los microservicios.

La IA actuó como un copiloto, permitiendo que el equipo de ingeniería se concentrara en resolver los problemas de negocio de alto nivel de NutriChain Logistics.

Claro, aquí tienes una ampliación de la documentación con una nueva sección que detalla los retos técnicos y sus soluciones, siguiendo el tono y estilo del documento original.

---

## 5. Retos Encontrados y Soluciones Aplicadas

Durante el diseño y la planificación de la implementación, se identificaron varios retos inherentes a la arquitectura de microservicios. A continuación, se describen estos desafíos y las soluciones estratégicas que se aplicaron para mitigarlos, garantizando la robustez y escalabilidad del sistema.

### 5.1. Reto: Acoplamiento Síncrono y Cascada de Fallos

-   **Problema:** La comunicación síncrona inicial entre `Tienda` y `Almacén` (una llamada HTTP REST para actualizar el stock) creaba un punto de acoplamiento fuerte.
-   **Impacto:** Si el servicio `Almacén` sufría una degradación de rendimiento o una caída, la funcionalidad crítica de creación de pedidos en `Tienda` quedaba completamente bloqueada, aunque el resto del servicio `Tienda` estuviera operativo. Este fenómeno se conoce como "cascada de fallos".
-   **Solución:**
    1.  **Implementación de Circuit Breaker:** Se integró un patrón de "Circuit Breaker" (interruptor de circuito) en el cliente HTTP del servicio `Tienda`. Si las llamadas al servicio `Almacén` comienzan a fallar consecutivamente, el circuito se "abre" y las peticiones subsiguientes fallan inmediatamente sin esperar un timeout, devolviendo un error controlado al usuario (p. ej., "No se puede verificar el stock en este momento, inténtelo más tarde"). Esto evita que `Tienda` sature sus propios recursos intentando contactar a un servicio caído.
    2.  **Estrategia de Reintentos con Backoff Exponencial:** Para fallos transitorios, en lugar de reintentar inmediatamente, el cliente HTTP implementa una estrategia de "backoff exponencial", esperando intervalos de tiempo crecientes entre cada reintento (p. ej., 1s, 2s, 4s). Esto da tiempo al servicio `Almacén` a recuperarse sin ser bombardeado por peticiones.
    3.  **Hoja de Ruta hacia la Asincronía:** Aunque la solución inicial es síncrona por su simplicidad, se definió en la hoja de ruta la transición a un modelo asíncrono basado en eventos como la solución definitiva a largo plazo, tal como se menciona en la sección 2.2.

### 5.2. Reto: Riesgo de Acoplamiento en la Capa de Datos

-   **Problema:** Permitir que el servicio `Reportes` lea directamente de la base de datos de `Almacén` introduce un acoplamiento a nivel de datos, lo cual es un anti-patrón en arquitecturas de microservicios puras.
-   **Impacto:**
    -   **Fragilidad:** Cualquier cambio en el esquema de la base de datos de `Almacén` (p. ej., renombrar una tabla o columna) podría romper el servicio `Reportes` de forma inesperada.
    -   **Degradación del Rendimiento:** Una consulta analítica compleja y pesada lanzada por `Reportes` podría consumir recursos de la base de datos (CPU, I/O) y afectar negativamente el rendimiento del servicio `Almacén`, que gestiona operaciones transaccionales críticas (OLTP).
-   **Solución:**
    -   **Creación de una Réplica de Lectura (Read Replica):** En lugar de apuntar a la base de datos maestra de `Almacén`, el servicio `Reportes` fue configurado para conectarse a una **réplica de solo lectura** de dicha base de datos.
    -   **Beneficios:** Esta solución desacopla la carga de trabajo. Las operaciones de escritura de `Almacén` van a la base de datos principal, mientras que las operaciones de lectura intensiva de `Reportes` se dirigen a la réplica. Esto aísla el rendimiento y permite que el equipo de `Almacén` evolucione su esquema con mayor seguridad, siempre que mantenga la estructura necesaria para los reportes en la réplica o coordine los cambios.

### 5.3. Reto: Gestión Segura de Secretos Compartidos

-   **Problema:** La seguridad basada en JWT requiere que múltiples servicios (`Almacén`, `Tienda`, `Reportes`) compartan una clave secreta para verificar la firma de los tokens. Almacenar esta clave en archivos de configuración o variables de entorno en texto plano es una mala práctica de seguridad.
-   **Impacto:** Una filtración del código fuente o un acceso no autorizado al entorno de ejecución podría exponer la clave secreta, permitiendo a un atacante falsificar tokens válidos para cualquier usuario y acceder a toda la API.
-   **Solución:**
    -   **Integración con un Gestor de Secretos:** Se decidió no almacenar los secretos directamente en el código o en las variables de entorno de Docker Compose. En su lugar, los servicios se configuraron para obtener los secretos en tiempo de ejecución desde un sistema centralizado de gestión de secretos.
    -   **Implementación:** Para el entorno de desarrollo local con Docker, se utilizó **Docker Secrets**. Para un entorno de producción, la arquitectura está preparada para integrarse con servicios como **AWS Secrets Manager**, **Google Secret Manager** o **HashiCorp Vault**. Al arrancar, cada servicio se autentica con el gestor de secretos y obtiene la clave JWT, que se mantiene únicamente en memoria.

### 5.4. Reto: Trazabilidad y Depuración en un Entorno Distribuido

-   **Problema:** Cuando una petición de un usuario falla (p. ej., al crear un pedido), el error puede haberse originado en el servicio `Tienda` o en la llamada subsecuente al servicio `Almacén`. Rastrear el flujo completo de una petición a través de múltiples logs de servicios es complejo y propenso a errores.
-   **Impacto:** Aumenta significativamente el Tiempo Medio de Resolución (MTTR) de incidentes, ya que los desarrolladores deben correlacionar manualmente timestamps y logs de diferentes fuentes para entender el origen de un problema.
-   **Solución:**
    -   **Implementación de Trazabilidad Distribuida (Distributed Tracing):** Se adoptó el estándar **OpenTelemetry**.
    1.  **ID de Correlación:** Cuando una petición llega al primer servicio (el API Gateway o directamente a `Tienda`), se genera un `Trace ID` único.
    2.  **Propagación de Contexto:** Este `Trace ID` se propaga en los encabezados HTTP (`traceparent`) en todas las llamadas subsiguientes entre servicios (`Tienda` -> `Almacén`).
    3.  **Logging Estructurado:** Todos los logs generados por cada servicio incluyen este `Trace ID`.
    -   **Resultado:** Con una herramienta de observabilidad como **Jaeger**, **Datadog** o **Grafana Tempo**, ahora es posible visualizar la traza completa de una petición como un diagrama de Gantt, mostrando el tiempo que pasó en cada servicio y permitiendo identificar cuellos de botella o el punto exacto del fallo con un solo clic.