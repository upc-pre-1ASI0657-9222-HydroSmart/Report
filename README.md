# Capítulo IV: Product Architecture Design

## 4.1 Design Concepts, ViewPoints & ER Diagrams

### 4.1.1 Principles Statements

A partir de la visión de negocio de HydroSmart (transformar el consumo de agua doméstico en información accionable en tiempo real que permita a los propietarios, arrendadores y estudiantes prevenir fugas, optimizar el riego y ahorrar dinero) y de la visión arquitectónica de construir una plataforma escalable, segura y de baja latencia capaz de procesar datos provenientes de sensores IoT de forma continua, se definen los siguientes principios generales que guían todas las decisiones de diseño y evolución del sistema a largo plazo:

*   **Comunicación asincrónica sobre sincrónica:** La ingesta de lecturas de consumo (caudal de agua) y la generación de alertas se basan en mecanismos no bloqueantes y orientados a eventos. Se emplea un Message Broker para desacoplar la recepción de telemetría de los sensores del procesamiento de anomalías y la actualización de analíticas, evitando que picos de lecturas (por ejemplo, en horas punta de riego) degraden la experiencia del usuario.
*   **Uso de bibliotecas y frameworks con soporte comercial o comunidad activa:** Se priorizan tecnologías con ciclos de actualización claros y ecosistemas maduros. La autenticación se delega a Firebase Authentication; las notificaciones push se gestionan mediante Firebase Cloud Messaging; el procesamiento de pagos de suscripciones se delega a una pasarela como Stripe o Culqi y la documentación de API se estandariza bajo OpenAPI 3.1. Este principio garantiza la sostenibilidad técnica del proyecto a largo plazo.
*   **Monitoreo en tiempo real como eje central del diseño:** La lectura y el procesamiento del consumo hídrico constituyen el activo principal de la plataforma. Todos los servicios que involucran visualización de consumo, detección de fugas y generación de recomendaciones se diseñan priorizando la baja latencia entre la lectura del sensor y la notificación al usuario, por encima de cualquier otra consideración de rendimiento.
*   **Seguridad como principio transversal:** La seguridad se aplica en cada componente del sistema, no como una capa adicional. Se delega la gestión de identidad a Firebase Authentication, se valida el rol del usuario (propietario, arrendador, inquilino o administrador) en cada petición, y el acceso a los datos de consumo de una unidad o vivienda queda restringido exclusivamente a su propietario, arrendador asociado o administrador autorizado.
*   **Dominio sobre implementación técnica (DDD):** El diseño parte del modelo de negocio y no de los detalles técnicos. Se aplica Domain-Driven Design para organizar el sistema en Bounded Contexts claros: Gestión de Identidad (IAM), Propiedades y Unidades, Consumo y Telemetría, Alertas y Notificaciones, Ahorro y Recomendaciones, Analíticas y Reportes, y Suscripciones. Cada contexto evoluciona de forma independiente sin comprometer la coherencia del sistema.
*   **Integridad y trazabilidad de las lecturas de consumo:** La lectura de consumo es el invariante más crítico de la plataforma: una pérdida o duplicación de datos afecta directamente la confianza del usuario en las alertas y en los reportes de ahorro. Toda lectura recibida desde un sensor se procesa de forma idempotente (evitando conteos duplicados ante reenvíos de red) y se persiste de manera transaccional antes de publicar cualquier evento derivado.
*   **Separación de responsabilidades mediante arquitectura en capas:** El sistema se estructura en capas claras: presentación, aplicación, dominio e infraestructura, separando la lógica de negocio de los frameworks, los proveedores externos y los mecanismos de persistencia. El patrón Repository abstrae el acceso a datos de las reglas del dominio, facilitando cambios en la capa de persistencia sin afectar la lógica de negocio.
*   **Escalabilidad progresiva:** La arquitectura de microservicios y el enfoque orientado a eventos se implementan de forma incremental, priorizando primero los flujos más críticos (registro de consumo y alertas de fuga). Se evita introducir complejidad innecesaria en etapas tempranas del desarrollo, optando por soluciones simples y mantenibles que puedan escalar cuando el número de sensores y usuarios lo justifique.
*   **Frontend desacoplado con contrato de API explícito:** La aplicación web y móvil consumen APIs RESTful expuestas por los servicios backend mediante un único punto de entrada centralizado en el API Gateway. Este principio desacopla el frontend de la implementación interna de los servicios y facilita la futura integración con aplicaciones móviles nativas o con dispositivos adicionales.
*   **Delegación de responsabilidades no esenciales a servicios externos:** Funcionalidades que no forman parte del núcleo diferencial de HydroSmart (como autenticación, envío de notificaciones push, procesamiento de pagos y almacenamiento de imágenes de perfil) se delegan a proveedores externos confiables (Firebase, pasarela de pagos, servicio de almacenamiento en la nube). Esto reduce la complejidad interna del sistema y permite al equipo enfocarse en los subdominios de mayor valor: el monitoreo hídrico y la detección temprana de anomalías.

### 4.1.2 Approaches Statements Architectural Styles & Patterns

Para el desarrollo de HydroSmart se adopta un enfoque arquitectónico orientado a la escalabilidad, el procesamiento en tiempo real y la alta disponibilidad, debido a la naturaleza continua de los datos generados por los sensores de consumo de agua y a la necesidad de notificar anomalías (fugas, consumos excesivos) de forma casi inmediata.

Como principio rector, se emplea una arquitectura basada en **Domain-Driven Design (DDD)**, permitiendo modelar de forma precisa los subdominios clave del sistema: Gestión de Identidad, Propiedades y Unidades, Consumo y Telemetría, Alertas y Notificaciones, Ahorro y Recomendaciones, Analíticas y Reportes, y Suscripciones. Este enfoque facilita la alineación entre la lógica de negocio y la implementación técnica, promoviendo un lenguaje ubicuo entre los distintos perfiles de usuario (propietarios, arrendadores, inquilinos) y el equipo de desarrollo.

El sistema se organiza mediante **Bounded Contexts**, delimitando claramente las responsabilidades de cada subdominio. Esto permite desacoplar funcionalidades críticas como la ingesta de telemetría, el cálculo de alertas y la generación de recomendaciones, favoreciendo la mantenibilidad y la evolución independiente de cada componente.

**Estilos arquitectónicos**

El estilo arquitectónico predominante es una **arquitectura basada en microservicios**, complementada con un enfoque **orientado a eventos (Event-Driven Architecture)** para el procesamiento de la telemetría en tiempo real, sin introducir complejidad innecesaria en las primeras iteraciones.

- Cada microservicio es responsable de un dominio específico (identidad, propiedades, consumo, alertas, ahorro, analíticas, suscripciones), permitiendo escalar de forma independiente según la demanda.
- El uso de eventos permite gestionar de manera eficiente acciones críticas como el registro de una nueva lectura de consumo, la detección de una anomalía y la actualización de las métricas del dashboard, sin acoplar estos procesos entre sí.
- Se emplea un **Message Broker** (por ejemplo, RabbitMQ) para la comunicación asíncrona entre servicios, facilitando el manejo de eventos como `ConsumptionRecorded` y `LeakDetected`.
- Los sensores/medidores inteligentes envían sus lecturas mediante un protocolo ligero orientado a IoT (por ejemplo, MQTT) hacia un gateway de ingesta, el cual traduce y reenvía la información al microservicio de Consumo y Telemetría a través de eventos internos.
- Para la gestión de autenticación y seguridad, se integra un servicio externo como Firebase Authentication, delegando la responsabilidad de identidad y acceso.

Adicionalmente, para optimizar la interacción con el usuario final, se emplea un enfoque de **cliente-servidor con frontend desacoplado**, donde la aplicación web y móvil consumen APIs RESTful expuestas por los servicios backend a través de un único API Gateway.

**Patrones de diseño y procesamiento**

Para garantizar el cumplimiento de los atributos de calidad del sistema sin introducir complejidad innecesaria, se adoptan los siguientes patrones de diseño:

- **API Gateway Pattern:** centraliza el acceso al backend mediante un único punto de entrada, encargado de manejar autenticación, enrutamiento y validaciones básicas antes de que la petición llegue a los microservicios internos.
- **MVC (Model-View-Controller):** utilizado en la organización del backend para separar la lógica de negocio (Model), la gestión de solicitudes (Controller) y la representación de datos (View/API Response).
- **Repository Pattern:** permite abstraer el acceso a la base de datos, evitando el acoplamiento directo con la lógica de negocio y facilitando cambios en la capa de persistencia (por ejemplo, si se migra el almacenamiento de lecturas a una base optimizada para series de tiempo).
- **Adapter Pattern:** aplicado para integrar servicios externos como Firebase (autenticación y notificaciones push), la pasarela de pagos y el gateway de sensores IoT, traduciendo sus formatos particulares al modelo interno de HydroSmart.
- **Observer Pattern (Eventos de Dominio):** implementa el mecanismo para reaccionar a cambios críticos. Cuando se registra una nueva lectura de consumo, este evento "notifica" a los componentes de Alertas y de Analíticas para que actualicen el estado del sistema en paralelo, manteniendo un bajo acoplamiento entre módulos.
- **Idempotent Consumer:** aplicado en el servicio de Consumo y Telemetría para evitar que reenvíos de un mismo sensor (por fallas de red) dupliquen una lectura y distorsionen el consumo real o disparen alertas falsas.
- **CQRS (Command Query Responsibility Segregation):** separa las operaciones que alteran el estado (por ejemplo, `RegisterConsumptionCommand`) de las que solo consultan información (por ejemplo, `GetDashboardSummaryQuery`), optimizando las consultas masivas del historial y el dashboard sin ser afectadas por la escritura constante de nuevas lecturas.
- **Validación en Backend (Fail Fast):** se validan los datos en cada solicitud (rol de usuario, rangos de fechas, umbrales de alerta), evitando estados inconsistentes.
- **RESTful API Design:** se adoptan buenas prácticas en el diseño de APIs REST, con endpoints claros, métodos HTTP adecuados y estructuras de respuesta consistentes, facilitando la interoperabilidad con el frontend y con futuras integraciones (por ejemplo, con proveedores de agua como SEDAPAL).

La combinación de Domain-Driven Design (DDD), arquitectura de microservicios y un enfoque de Event-Driven Architecture (EDA) permite que HydroSmart procese de forma eficiente el flujo continuo de lecturas de consumo, detecte anomalías con baja latencia y mantenga la consistencia en operaciones críticas como el cálculo de alertas y proyecciones de gasto, sin introducir complejidad innecesaria en las primeras iteraciones del proyecto.


### 4.1.3 Context Diagram

El diagrama de contexto de HydroSmart representa la vista más general del sistema, permitiendo identificar claramente los límites que lo separan de su entorno externo. Este diagrama describe las interacciones entre los actores principales y los servicios externos que se integran con la plataforma, proporcionando una visión global de los flujos de información y los puntos de integración tecnológica.

En primer lugar, se identifican los actores principales que interactúan directamente con el sistema. Los **propietarios de viviendas con áreas verdes** utilizan la plataforma para monitorear su consumo en tiempo real, optimizar el riego de sus jardines y recibir alertas ante posibles fugas. Los **arrendadores** (dueños de departamentos con servicios incluidos) emplean HydroSmart para administrar sus unidades y supervisar el consumo individual de sus inquilinos, protegiendo así su rentabilidad. Los **estudiantes y jóvenes arrendatarios** usan la plataforma para controlar su gasto diario, establecer metas de ahorro adaptadas a su presupuesto y evitar cobros inesperados. Finalmente, los **administradores del sistema** cumplen un rol de supervisión operativa y soporte de la plataforma.

Asimismo, HydroSmart interactúa con servicios externos especializados que complementan su funcionalidad principal. Se integra **Firebase Authentication**, el cual gestiona el registro e inicio de sesión de los usuarios mediante mecanismos seguros de autenticación, delegando la gestión de credenciales y reduciendo la complejidad interna del sistema. Los **sensores IoT / medidores inteligentes de caudal** instalados en el punto de suministro de agua de cada vivienda o unidad envían de forma continua sus lecturas hacia la plataforma, constituyendo la fuente primaria de datos del sistema. Para notificar al usuario ante consumos inusuales o fugas detectadas, el sistema se apoya en un **servicio de notificaciones push (Firebase Cloud Messaging)**. Por otro lado, para el modelo de monetización mediante planes de suscripción (Freemium, Premium), HydroSmart se integra con una **pasarela de pagos** (Stripe o Culqi) que procesa las transacciones de forma segura. Finalmente, se considera un **servicio de almacenamiento en la nube** para la gestión de imágenes de perfil de los usuarios.

La incorporación de estos servicios externos responde a principios arquitectónicos de desacoplamiento y especialización, delegando funcionalidades no críticas a proveedores externos confiables. Esto permite reducir la complejidad interna del sistema, mejorar la seguridad (especialmente en la gestión de autenticación y pagos) y optimizar el rendimiento general de la plataforma.

En conjunto, el diagrama de contexto muestra que HydroSmart actúa como el núcleo central que traduce las lecturas de consumo de agua en información accionable para sus tres segmentos de usuario, mientras delega funciones específicas como autenticación, notificaciones y pagos a servicios externos especializados.

![alt text](images/ContextoHydroSmart-key.png)

![alt text](images/ContextoHydroSmart.png)

### 4.1.4 Approach driven ViewPoints Diagrams
Se presenta el diagrama de secuencia que describe el flujo principal cuando un sensor reporta una lectura de consumo y el sistema detecta una posible anomalía (fuga o consumo excesivo), notificando al usuario en tiempo real. El diagrama organiza las acciones en función de los bounded contexts más relevantes del sistema: **Identity & Access Management (IAM)**, **Consumption & Telemetry**, **Alerts & Notifications** y **Analytics & Reports**.

El flujo se inicia en el bounded context **IAM**, donde el usuario accede a la plataforma web o móvil y realiza el proceso de autenticación mediante correo y contraseña utilizando Firebase Authentication. Una vez autenticado, el sistema valida el rol del usuario (propietario, arrendador o inquilino) y concede acceso a las funcionalidades correspondientes a su vivienda o unidad.

De manera independiente y continua, el bounded context **Consumption & Telemetry** recibe las lecturas de caudal enviadas por el sensor IoT instalado en el punto de suministro. El servicio valida la lectura, la procesa de forma idempotente para evitar duplicados y la persiste, publicando un evento de dominio `ConsumptionRecorded` hacia el Message Broker.

A continuación, el flujo se ramifica hacia dos consumidores del evento. Por un lado, el bounded context **Alerts & Notifications** evalúa la lectura recibida contra los umbrales configurados por el usuario (o contra un patrón de consumo continuo fuera de lo habitual, indicativo de una fuga). Si detecta una anomalía, genera un evento `LeakDetected` y solicita al servicio externo de notificaciones push que envíe una alerta inmediata al usuario, indicando la posible causa y el punto donde ocurre. Por otro lado, el bounded context **Analytics & Reports** consume el mismo evento para actualizar en tiempo real el dashboard del usuario, recalcular la proyección de gasto mensual y verificar el avance respecto a su meta de ahorro.

Finalmente, el usuario visualiza en su dashboard tanto la alerta recibida como el consumo actualizado, pudiendo ajustar su comportamiento (por ejemplo, revisar el riego o reportar la fuga) o modificar su meta de ahorro para el siguiente periodo.

![alt text](images/detecciondefuga.drawio.png)

### 4.1.5 Relational/Non Relational Database Diagram

### 4.1.6 Design Patterns

### 4.1.7 Tactics

## 4.2 Architectural Drivers

### 4.2.1 Design Purpose
 
El propósito del diseño arquitectónico de HydroSmart, producto de la startup AquaPulse, es construir una plataforma web y móvil que permita a los usuarios residenciales monitorear en tiempo real su consumo de agua, detectar fugas de manera temprana y traducir cada litro consumido en su equivalente económico, a partir de las lecturas enviadas por sensores IoT y medidores inteligentes de terceros. La arquitectura debe garantizar que cada decisión de diseño esté justificada por el valor que aporta a los segmentos objetivo del sistema: los propietarios de viviendas con áreas verdes, los arrendadores de departamentos con servicios incluidos y los estudiantes y jóvenes arrendatarios con presupuesto limitado.
 
**Coherencia con el Dominio del Negocio**
 
*   Implementar un sistema que refleje con precisión los procesos reales de gestión del agua en el hogar: registro de propiedades y unidades, vinculación de medidores, captura de lecturas, emisión de alertas y seguimiento de metas de ahorro.
*   Asegurar que las abstracciones arquitectónicas correspondan con los bounded contexts definidos mediante DDD: Gestión de Identidad (IAM), Propiedades y Unidades, Consumo y Telemetría, Alertas y Notificaciones, Ahorro y Recomendaciones, Analíticas y Reportes, y Suscripciones.
*   Mantener trazabilidad entre las épicas EP01 a EP09, las User Stories priorizadas en el Product Backlog y los microservicios que las implementan.
**Monitoreo en Tiempo Real como Eje del Diseño**
 
*   Diseñar el flujo de ingesta de lecturas (sensor IoT → gateway de ingesta MQTT → microservicio de Consumo y Telemetría) como el camino crítico del sistema, ya que la propuesta de valor depende de que el usuario conozca su consumo antes de recibir el recibo físico.
*   Priorizar la frescura de los datos mostrados en el dashboard, indicando siempre la fecha y hora de la última lectura recibida y el estado de conexión de cada medidor.
*   Garantizar que la plataforma funcione con sensores y medidores de distintos fabricantes, dado que AquaPulse no fabrica ni comercializa hardware propio.
**Detección Temprana y Alertas Oportunas**
 
*   Diseñar un mecanismo de evaluación continua de las lecturas que identifique consumos que superan el umbral configurado y patrones de flujo continuo fuera del horario habitual, característicos de una posible fuga.
*   Garantizar que las alertas lleguen al usuario en segundos mediante notificaciones push (Firebase Cloud Messaging), con un canal alternativo por correo electrónico si el canal principal falla.
*   Evitar la saturación de notificaciones agrupando alertas de una misma anomalía y respetando las preferencias configuradas por cada usuario.
**Traducción del Consumo a Impacto Económico**
 
*   Convertir el consumo registrado en litros y metros cúbicos a soles, aplicando la estructura tarifaria vigente de la empresa prestadora del servicio (por ejemplo, SEDAPAL en Lima) aprobada por SUNASS.
*   Proveer proyecciones del monto mensual del recibo y comparativos entre periodos que permitan al usuario anticiparse y ajustar sus hábitos de consumo.
*   Permitir que las metas de ahorro se definan a partir del presupuesto mensual del usuario, calculando automáticamente su equivalente en litros.
**Experiencia Simple y Mobile-First**
 
*   Diseñar la aplicación móvil como el canal principal de interacción, dado que los segmentos consultan su consumo y reciben alertas desde el celular.
*   Presentar los datos de consumo con etiquetas, colores e indicadores visuales comprensibles para usuarios sin conocimientos técnicos, tal como lo solicitaron los entrevistados.
*   Ofrecer al arrendador una vista web de gestión por unidades que le permita supervisar varias propiedades desde un solo panel.
**Seguridad, Privacidad y Control de Acceso por Roles**
 
*   Delegar la autenticación a Firebase Authentication y aplicar control de acceso basado en roles (Propietario, Arrendador, Inquilino y Administrador), diferenciando con claridad las capacidades de cada perfil en cada operación del sistema.
*   Restringir el acceso a los datos de consumo de cada unidad exclusivamente a su propietario o arrendador asociado, al inquilino asignado y al administrador autorizado, dado que los patrones de consumo revelan hábitos y horarios del hogar.
*   Cumplir con la Ley N.° 29733 de Protección de Datos Personales, informando al usuario cómo se almacenan y protegen sus datos y permitiéndole gestionarlos.
La arquitectura debe actuar como un puente coherente entre las necesidades reales de los hogares y las capacidades tecnológicas de la plataforma, asegurando que cada componente del sistema aporte valor directo al objetivo de negocio: transformar el consumo pasivo de agua en una gestión preventiva, inteligente y económica, que permita actuar antes de que el gasto se convierta en un problema.

El propósito principal del diseño de HydroSmart es construir una plataforma escalable, segura y de baja latencia capaz de transformar las lecturas continuas de sensores IoT en información accionable para distintos perfiles de usuario (propietarios de viviendas, arrendadores e inquilinos), permitiéndoles detectar fugas, optimizar el consumo de agua y proyectar su gasto de forma anticipada.

Desde la perspectiva de negocio, el sistema busca proveer una experiencia diferencial frente a soluciones tradicionales de medición: en lugar de entregar únicamente lecturas de volumen, HydroSmart interpreta esos datos mediante algoritmos de detección de anomalías y proyecciones financieras, agregando valor directo al usuario final. El modelo de monetización basado en suscripciones escalonadas (Freemium, Pro, Smart) refuerza la necesidad de garantizar alta disponibilidad y confiabilidad, ya que una interrupción en el servicio afecta directamente la percepción de valor del producto.

Desde la perspectiva técnica, el diseño arquitectónico está orientado a satisfacer tres objetivos fundamentales:

1. **Procesamiento continuo y en tiempo real:** Las lecturas de caudal generadas por los sensores IoT deben ser ingestadas, validadas, procesadas y reflejadas en el dashboard del usuario con la menor latencia posible, garantizando que las alertas de fuga lleguen de forma oportuna.
2. **Escalabilidad progresiva y mantenibilidad:** La arquitectura de microservicios organizada en Bounded Contexts (IAM, Consumo y Telemetría, Alertas y Notificaciones, Analíticas y Reportes, Suscripciones, entre otros) permite que cada dominio escale y evolucione de forma independiente, sin comprometer la integridad del sistema completo.
3. **Seguridad y trazabilidad de los datos:** Los datos de consumo hídrico constituyen el activo más crítico del sistema. La plataforma garantiza su integridad mediante procesamiento idempotente (para evitar lecturas duplicadas), validación por rol de usuario en cada petición y delegación de la gestión de identidad a Firebase Authentication.

En síntesis, el diseño de HydroSmart no busca únicamente conectar sensores a una interfaz visual, sino construir un ecosistema de datos hídricos que sea confiable, seguro y capaz de crecer junto con la base de usuarios sin sacrificar la experiencia ni la precisión de la información entregada.

### 4.2.2 Primary Functionality (Primary User Stories)
 
Las siguientes User Stories representan la funcionalidad primaria que define la estructura arquitectónica central del sistema. Se agrupan por área funcional y se detalla el impacto arquitectónico que cada una genera.
 
#### Funcionalidad Core - Registro y autenticación
 
**US01: Registro de usuario**
 
Como usuario nuevo, quiero registrarme indicando mi tipo de perfil para acceder a las funcionalidades adaptadas a mis necesidades de consumo de agua.
 
**Impacto Arquitectónico:** Establece el microservicio IAM con integración a Firebase Authentication, delegando completamente la gestión de credenciales al proveedor externo; la entidad `users` almacena únicamente el `firebase_uid` como referencia, sin persistir contraseñas localmente. Define la relación uno a uno entre `users` y `profiles`, y la asignación del tipo de perfil mediante la tabla intermedia `user_roles` (PROPIETARIO, ARRENDADOR, INQUILINO, ADMINISTRADOR) en un modelo muchos a muchos. Al completarse el registro, IAM publica el evento de dominio `UserRegistered` para que los demás bounded contexts inicialicen la información del usuario sin acoplarse a IAM.
 
**US02: Inicio de sesión**
 
Como usuario registrado, quiero iniciar sesión de forma segura para acceder a mi información de consumo sin riesgo a que otros accedan a mis datos.
 
**Impacto Arquitectónico:** Define que el inicio de sesión se realiza contra Firebase Authentication, que emite un token JWT (TS02) con el identificador del usuario. La validación de la firma y expiración del token se centraliza en el API Gateway, de modo que los microservicios internos reciben únicamente peticiones autenticadas y aplican el control de acceso según el rol registrado en IAM.
 
#### Funcionalidad Core - Monitoreo de Consumo
 
**US04: Visualización de consumo en tiempo real**
 
Como usuario, quiero visualizar mi consumo de agua en tiempo real para reducir la incertidumbre sobre mi gasto y poder tomar decisiones inmediatas.
 
**Impacto Arquitectónico:** Define la operación más crítica del sistema y el microservicio de Consumo y Telemetría. Los sensores IoT publican sus lecturas mediante MQTT hacia el gateway de ingesta, que las traduce y reenvía al microservicio, donde se almacenan en la entidad `consumption_readings`, vinculada a `meters` mediante `meter_id`. Cada `Meter` registra su zona (RIEGO, COCINA, BAÑO, GENERAL) y su estado de conexión (ONLINE, OFFLINE) según la fecha de su última lectura, lo que permite mostrar el mensaje de datos no disponibles cuando el medidor pierde conexión (escenario 2 de US04). El consumo se expresa en litros y en soles aplicando la entidad `tariffs`, y cada lectura persistida publica el evento `ConsumptionRecorded`.
 
**US07: Proyección de gasto mensual**
 
Como usuario, quiero ver una proyección de mi gasto mensual para anticiparme al monto del recibo y planificar mejor mi presupuesto.
 
**Impacto Arquitectónico:** Requiere que el microservicio de Analíticas y Reportes consuma el evento `ConsumptionRecorded` y mantenga agregados precalculados en la entidad `daily_consumption`, a partir de los cuales se calcula la proyección del mes. La conversión a soles aplica la estructura tarifaria por rangos de consumo definida en `tariffs` y `tariff_ranges`, parametrizada por empresa prestadora y categoría. La regla de negocio que exige al menos una semana de datos para proyectar se implementa como validación del dominio.
 
#### Funcionalidad Core - Alertas y Notificaciones
 
**US08: Alerta de consumo inusual**
 
Como usuario, quiero recibir alertas cuando mi consumo sea inusual para poder actuar a tiempo y evitar gastos excesivos.
 
**Impacto Arquitectónico:** Define el microservicio de Alertas y Notificaciones, que se suscribe al evento `ConsumptionRecorded` mediante el patrón Observer (publicación/suscripción sobre RabbitMQ), manteniendo desacoplados ambos bounded contexts. La entidad `alert_rules` almacena el umbral configurado por el usuario y la entidad `alerts` registra cada anomalía con su tipo (CONSUMO_INUSUAL, POSIBLE_FUGA, META_PROXIMA), severidad y estado (ACTIVA, ATENDIDA). El envío de la notificación push se delega a Firebase Cloud Messaging.
 
**US09: Alerta de posible fuga**
 
Como propietario, quiero recibir una alerta cuando el sistema detecte una posible fuga para tomar acción antes de que el desperdicio sea irreversible.
 
**Impacto Arquitectónico:** Requiere el patrón Strategy en el microservicio de Alertas y Notificaciones para encapsular los distintos algoritmos de detección (umbral fijo, flujo continuo fuera del horario habitual, desviación respecto al promedio histórico) bajo una interfaz común, facilitando la incorporación de nuevas reglas sin alterar la lógica existente. Al confirmarse la anomalía se publica el evento `LeakDetected`, y la ubicación aproximada de la fuga se obtiene a partir de la zona del medidor que originó la lectura.
 
#### Funcionalidad Core - Gestión de Propiedades e Inquilinos
 
**US11: Registro de unidades**
 
Como arrendador, quiero registrar las unidades de mi inmueble en la plataforma para gestionar el consumo de cada una de forma independiente.
 
**Impacto Arquitectónico:** Define el microservicio de Propiedades y Unidades con la entidad `Property` vinculada al arrendador mediante `owner_id` y una relación uno a muchos con `Unit`. La asignación de inquilinos se registra en la tabla `unit_tenants` y la de medidores en la relación entre `Unit` y `Meter`. Para verificar la existencia y el rol del arrendador sin acoplarse a la implementación interna de IAM, se utiliza el patrón Facade (Anti-Corruption Layer) entre ambos contextos.
 
**US12: Monitoreo por unidad**
 
Como arrendador, quiero monitorear el consumo de agua de cada unidad de mi inmueble para identificar inquilinos con consumo excesivo.
 
**Impacto Arquitectónico:** Establece la comunicación síncrona entre Propiedades y Unidades y Consumo y Telemetría: Propiedades resuelve qué medidores pertenecen a cada unidad del arrendador y Consumo y Telemetría devuelve el consumo individual en litros y soles. Exige que cada consulta valide que la unidad pertenezca al arrendador autenticado, evitando que un usuario acceda al consumo de unidades ajenas.
 
#### Funcionalidad Core - Ahorro y Metas
 
**US14: Establecer meta de ahorro**
 
Como inquilino, quiero establecer una meta de consumo mensual para controlar mi gasto y evitar exceder mi presupuesto.
 
**Impacto Arquitectónico:** Define el microservicio de Ahorro y Recomendaciones con la entidad `SavingGoal`, que almacena el presupuesto mensual en soles, su equivalente en litros calculado con la tarifa vigente, el periodo y el estado (ACTIVA, CUMPLIDA, EXCEDIDA). La invariante de negocio establece una sola meta activa por usuario y periodo. El microservicio consume el evento `ConsumptionRecorded` para actualizar el avance y, al alcanzar el 80 % de la meta, publica el evento `GoalThresholdReached`, que Alertas y Notificaciones convierte en notificación.
 
#### Funcionalidad Core - Historial y Reportes
 
**US06: Historial de consumo**
 
Como usuario, quiero revisar mi historial de consumo para identificar patrones y entender cómo varía mi gasto en el tiempo.
 
**Impacto Arquitectónico:** Establece la aplicación del patrón CQRS en el microservicio de Analíticas y Reportes: las consultas de historial (por ejemplo, `GetConsumptionHistoryQuery`) se resuelven sobre agregados diarios, semanales y mensuales, separadas de la escritura continua de lecturas en Consumo y Telemetría. Esto permite responder rápidamente a los gráficos del historial sin afectar el procesamiento en tiempo real.
 
**US13: Reporte de consumo por unidad**
 
Como arrendador, quiero generar reportes de consumo por unidad para tener evidencia documentada ante disputas con inquilinos.
 
**Impacto Arquitectónico:** Define que el microservicio de Analíticas y Reportes genere el documento descargable de forma asíncrona a partir de los agregados de consumo de la unidad y el periodo seleccionados, lo almacene en el servicio de almacenamiento en la nube (AWS S3) y registre su referencia en la entidad `reports`. Las lecturas utilizadas como evidencia no pueden modificarse después de registradas, lo que garantiza la validez del reporte ante una disputa.

Las siguientes User Stories representan la funcionalidad primaria que define la estructura arquitectónica central del sistema. Se agrupan por área funcional y se detalla el impacto arquitectónico que cada una genera.

**Funcionalidad Core – Registro y Autenticación**

US-06: Registro e inicio de sesión seguro

Como usuario, quiero registrarme e iniciar sesión de forma segura con mi correo electrónico para acceder únicamente a los datos de mis viviendas o unidades.

Impacto Arquitectónico: Establece el microservicio de Gestión de Identidad (IAM) con integración a Firebase Authentication, delegando completamente la gestión de credenciales al proveedor externo. La entidad User almacena únicamente el firebase_uid como referencia, sin persistir contraseñas localmente. Define la relación uno a uno entre User y Profile, y la asignación de roles mediante la enumeración UserType (OWNER, TENANT, ADMIN). Esta separación establece la validación de acceso en el API Gateway, donde cada petición es verificada contra el token JWT y el rol del usuario antes de ser enrutada al microservicio correspondiente.

US-05: Administración de unidades (arrendador)

Como arrendador, quiero registrar mis unidades habitacionales y asociar a cada una sus inquilinos y sensores, para supervisar el consumo individual y evitar cobros incorrectos.

Impacto Arquitectónico: Define la separación entre el microservicio IAM (identidad) y el microservicio de Propiedades y Unidades (gestión de inmuebles). El IAM crea el usuario con firebase_uid y asigna el rol de arrendador, mientras que el microservicio de Propiedades registra las unidades habitacionales vinculadas al ownerId. Esta separación establece la comunicación entre bounded contexts mediante el patrón Facade/ACL, validando que solo el propietario registrado pueda asociar inquilinos y sensores a sus unidades. La relación propietario–unidad–inquilino–sensor constituye el modelo de dominio central para la supervisión del consumo individual.

**Funcionalidad Core – Monitoreo y Telemetría en Tiempo Real**

US-01: Monitoreo de consumo en tiempo real

Como propietario, quiero visualizar en mi dashboard el caudal de agua de mi vivienda en tiempo real para identificar variaciones inusuales de inmediato.

Impacto Arquitectónico: Establece el pipeline de ingesta IoT como el componente más crítico del sistema. Los sensores/medidores inteligentes envían lecturas de caudal mediante un protocolo ligero (MQTT) hacia el gateway de ingesta, el cual traduce y reenvía la información al microservicio de Consumo y Telemetría a través de eventos internos. La entidad ConsumptionRecord registra cada lectura con volumeLiters, instantFlowRate y timestamp. Se aplica el patrón Idempotent Consumer para evitar que reenvíos del sensor (por fallas de red) dupliquen una lectura y distorsionen el consumo real. El evento de dominio `ConsumptionRecorded` se publica al Message Broker (RabbitMQ) para ser consumido de forma asíncrona por los bounded contexts de Alertas y Analíticas.

US-02: Detección y alerta de fugas

Como propietario o arrendador, quiero recibir una notificación push inmediata cuando el sistema detecte un patrón de consumo continuo o anómalo que indique una posible fuga.

Impacto Arquitectónico: Define el bounded context de Alertas y Notificaciones como consumidor del evento `ConsumptionRecorded`. El módulo AnomalyDetector evalúa cada lectura contra los umbrales configurados (sensitivityThreshold) y contra patrones de consumo continuo fuera de lo habitual. Si detecta una anomalía, genera el evento `LeakDetected` y solicita al servicio externo de notificaciones push (Firebase Cloud Messaging) que envíe una alerta inmediata al dispositivo del usuario, indicando la severidad (INFO, WARNING, CRITICAL) y el punto donde ocurre. La latencia entre la recepción de la lectura anómala y la entrega de la notificación push no debe superar los 15 segundos en el percentil 95.

**Funcionalidad Core – Analíticas y Proyecciones**

US-03: Proyección de gasto mensual

Como usuario, quiero ver una proyección del gasto en agua de mi vivienda para el cierre del mes en curso, basada en mi consumo histórico y en tiempo real.

Impacto Arquitectónico: Requiere el bounded context de Analíticas y Reportes con acceso a series temporales de consumo. El servicio CostCalculator aplica el tarifario vigente de SEDAPAL (pricePerCubicMeter, fixedCharge, taxRate) para transformar el volumen consumido en un costo proyectado en soles. Se implementa el patrón CQRS para separar las consultas de proyección (GetDashboardSummaryQuery) de las operaciones de escritura de nuevas lecturas (RegisterConsumptionCommand), optimizando las consultas masivas del historial y el dashboard sin afectar la ingesta continua de datos.

US-07: Historial de consumo y reportes exportables

Como propietario o arrendador, quiero consultar el historial de consumo de mis unidades por rango de fechas y exportarlo en formato PDF o CSV para presentarlo ante la junta de propietarios o ante mi inquilino.

Impacto Arquitectónico: Establece que el microservicio de Analíticas y Reportes debe exponer endpoints de consulta y exportación que lean los datos consolidados de las series temporales de consumo y los transformen en los formatos requeridos (PDF, CSV). El acceso a estos reportes queda restringido exclusivamente a usuarios con el rol verificado de propietario o arrendador de la unidad consultada, aplicando validación de rol en cada petición.

**Funcionalidad Core – Ahorro y Recomendaciones**

US-04: Gestión de metas de ahorro

Como usuario, quiero establecer una meta de consumo mensual en m³ o en soles para recibir recomendaciones y alertas cuando me acerque al límite.

Impacto Arquitectónico: Introduce la lógica de dominio del bounded context de Ahorro y Recomendaciones. La entidad SavingGoal persiste los objetivos del usuario (targetVolume, targetCost, startDate, endDate, currentProgress) y el sistema verifica periódicamente el avance respecto a la meta. La entidad Recommendation genera sugerencias automáticas para optimizar el consumo basándose en el comportamiento histórico. Cuando el consumo acumulado supera un umbral porcentual de la meta, se dispara una alerta preventiva a través del bounded context de Alertas y Notificaciones.

**Funcionalidad Core – Monetización y Suscripciones**

US-08: Gestión de suscripción y pagos

Como usuario, quiero contratar o cambiar mi plan de suscripción (Freemium, Pro, Smart) y realizar el pago de forma segura para desbloquear las funcionalidades correspondientes a mi nivel.

Impacto Arquitectónico: Define el bounded context de Suscripciones, que gestiona el ciclo de vida completo de la suscripción del usuario: plan_type (Freemium, Pro, Smart), estado activo/inactivo y fecha de renovación. Se integra con una pasarela de pagos externa (Stripe o Culqi) mediante el Adapter Pattern para procesar las transacciones de forma segura. El estado de la suscripción debe sincronizarse con las capacidades de cada plan, controlando qué funcionalidades (alertas avanzadas, reportes exportables, metas personalizadas) están disponibles para cada nivel.

### 4.2.3 Quality Attribute Scenarios
 
En esta sección se definen los Escenarios de Atributos de Calidad (QAS) para la arquitectura de la plataforma HydroSmart. Estos escenarios constituyen una herramienta fundamental de diseño y validación, ya que permiten operativizar y hacer completamente medibles los requerimientos no funcionales del sistema, tales como el rendimiento, la disponibilidad, la seguridad, la escalabilidad y la usabilidad. Al desglosar cada situación en componentes específicos (fuente, estímulo, entorno, artefacto, respuesta y medida de la respuesta), se establecen criterios de prueba claros y verificables. De esta manera, se asegura que las decisiones arquitectónicas implementadas soporten las exigencias operativas del monitoreo del consumo de agua y estén directamente alineadas con el cumplimiento de las User Stories críticas del proyecto.
 
**Escenario 1: Disponibilidad - Falla de un nodo de Consumo y Telemetría**
 
| Elemento | Descripción |
|---|---|
| **Fuente del estímulo** | Infraestructura de red o fallo de hardware interno. |
| **Estímulo** | Un nodo que aloja una instancia del microservicio de Consumo y Telemetría deja de responder repentinamente mientras se reciben lecturas de los sensores. |
| **Entorno** | Tiempo de ejecución, bajo operación normal con usuarios consultando su dashboard. |
| **Artefacto** | Infraestructura de servidores y microservicio de Consumo y Telemetría. |
| **Respuesta** | El sistema emplea Redundancia Activa: el balanceador de carga detecta la falla mediante health checks y redirige las peticiones hacia las instancias activas, mientras las lecturas pendientes permanecen en las colas durables de RabbitMQ hasta ser consumidas por otra instancia. |
| **Medida de la respuesta** | La conmutación hacia las instancias de respaldo se realiza en menos de 5 segundos, sin pérdida de lecturas, garantizando una disponibilidad del 99.9 % del servicio de monitoreo. |
 
**Escenario 2: Disponibilidad - Pérdida de conexión de un medidor**
 
| Elemento | Descripción |
|---|---|
| **Fuente del estímulo** | Sensor IoT o medidor inteligente de terceros. |
| **Estímulo** | El medidor de una vivienda deja de enviar lecturas por un corte de energía o pérdida de conectividad WiFi. |
| **Entorno** | Tiempo de ejecución, operación normal. |
| **Artefacto** | Microservicio de Consumo y Telemetría (entidad `meters`) y dashboard del usuario. |
| **Respuesta** | El sistema aplica la táctica Heartbeat: cada medidor debe reportar al menos una señal periódica; si no se recibe, el medidor pasa al estado OFFLINE, el dashboard muestra el mensaje de datos no disponibles junto con la última lectura válida y se notifica al usuario. |
| **Medida de la respuesta** | La desconexión se detecta y se refleja en el dashboard en un máximo de 5 minutos desde la última señal recibida, en el 100 % de los casos. |
 
**Escenario 3: Rendimiento - Alerta de consumo inusual**
 
| Elemento | Descripción |
|---|---|
| **Fuente del estímulo** | Sensor IoT del usuario. |
| **Estímulo** | Llega una lectura cuyo consumo supera el umbral configurado por el usuario. |
| **Entorno** | Tiempo de ejecución, carga normal de lecturas. |
| **Artefacto** | Microservicio de Alertas y Notificaciones y servicio externo Firebase Cloud Messaging. |
| **Respuesta** | El sistema procesa el evento `ConsumptionRecorded` de forma asíncrona, evalúa las reglas del usuario, registra la alerta y envía la notificación push con el detalle de la anomalía. |
| **Medida de la respuesta** | La notificación push llega al dispositivo del usuario en menos de 10 segundos desde la recepción de la lectura en el 95 % de los casos. |
 
**Escenario 4: Rendimiento / Fiabilidad - Detección de posible fuga**
 
| Elemento | Descripción |
|---|---|
| **Fuente del estímulo** | Sensor IoT del usuario. |
| **Estímulo** | Se registra un flujo de agua continuo durante un periodo prolongado fuera del horario habitual del hogar (por ejemplo, un inodoro malogrado durante la madrugada). |
| **Entorno** | Tiempo de ejecución, horario nocturno con bajo consumo esperado. |
| **Artefacto** | Microservicio de Alertas y Notificaciones (estrategia de detección de fugas). |
| **Respuesta** | El sistema compara el patrón de consumo con la línea base histórica del usuario por franja horaria y, al confirmar el flujo continuo durante el periodo configurado (30 minutos por defecto), publica el evento `LeakDetected` y genera una alerta de tipo POSIBLE_FUGA indicando la zona del medidor. |
| **Medida de la respuesta** | La alerta se emite en menos de 1 minuto después de cumplido el periodo configurado, con una tasa de falsos positivos menor al 5 % en las pruebas de validación. |
 
**Escenario 5: Escalabilidad - Ingesta masiva de lecturas**
 
| Elemento | Descripción |
|---|---|
| **Fuente del estímulo** | Sensores IoT de múltiples usuarios. |
| **Estímulo** | Al crecer la base de usuarios hacia la meta de 800 usuarios activos, miles de sensores envían lecturas de forma simultánea cada 15 segundos. |
| **Entorno** | Tiempo de ejecución, pico de tráfico de lecturas en horas de mayor consumo (mañana, noche y horarios de riego). |
| **Artefacto** | Gateway de ingesta, RabbitMQ y microservicio de Consumo y Telemetría. |
| **Respuesta** | El sistema utiliza Comunicación Asíncrona mediante colas de mensajes: las lecturas se encolan en RabbitMQ y son procesadas por consumidores competidores que escalan horizontalmente según el tamaño de la cola. |
| **Medida de la respuesta** | El sistema soporta la ingesta de hasta 20,000 lecturas por minuto sin pérdida de mensajes y con un retraso de procesamiento menor a 5 segundos. |
 
**Escenario 6: Rendimiento - Carga del dashboard de consumo**
 
| Elemento | Descripción |
|---|---|
| **Fuente del estímulo** | Usuario (propietario, arrendador o inquilino). |
| **Estímulo** | El usuario abre la aplicación móvil para revisar su consumo del día en litros y soles. |
| **Entorno** | Tiempo de ejecución, bajo condiciones normales de operación. |
| **Artefacto** | Aplicación móvil y microservicios de Consumo y Telemetría y Analíticas y Reportes. |
| **Respuesta** | El sistema emplea la táctica de Mantener múltiples copias de datos: la última lectura de cada medidor y los agregados del día se mantienen precalculados (modelo de lectura CQRS), evitando recalcular el consumo a partir de las lecturas crudas en cada consulta. |
| **Medida de la respuesta** | El dashboard se carga con el consumo actual, su equivalente en soles y el avance de la meta en menos de 2 segundos. |
 
**Escenario 7: Rendimiento - Cálculo de la proyección mensual**
 
| Elemento | Descripción |
|---|---|
| **Fuente del estímulo** | Usuario. |
| **Estímulo** | El usuario solicita la proyección estimada del monto a pagar al cierre del mes. |
| **Entorno** | Tiempo de ejecución, usuario con al menos una semana de consumo registrado. |
| **Artefacto** | Microservicio de Analíticas y Reportes (entidad `daily_consumption`) y tarifas de Consumo y Telemetría. |
| **Respuesta** | El sistema calcula la proyección a partir de los agregados diarios ya consolidados y aplica la estructura tarifaria por rangos vigente, sin procesar el historial completo de lecturas. |
| **Medida de la respuesta** | La proyección en soles se muestra en menos de 3 segundos, con una desviación menor al 10 % respecto al recibo real al cierre del periodo. |
 
**Escenario 8: Seguridad - Acceso con token expirado**
 
| Elemento | Descripción |
|---|---|
| **Fuente del estímulo** | Consumidor del API (usuario, sistema externo o atacante). |
| **Estímulo** | Se realiza una petición hacia un endpoint protegido (por ejemplo, consulta del consumo de una unidad) utilizando un token JWT caducado o alterado. |
| **Entorno** | Tiempo de ejecución, intento de consumo del servicio. |
| **Artefacto** | API Gateway y microservicio IAM. |
| **Respuesta** | El sistema aplica las tácticas de Autenticar actores y Limitar el acceso, validando criptográficamente la firma y la fecha de expiración del JWT emitido por Firebase Authentication en el API Gateway. La petición es rechazada antes de llegar a los microservicios internos. |
| **Medida de la respuesta** | El sistema otorga 0 % de acceso a las funcionalidades protegidas cuando el token es inválido o ha expirado, respondiendo con estado 401. |
 
**Escenario 9: Seguridad / Privacidad - Acceso a datos de otra unidad**
 
| Elemento | Descripción |
|---|---|
| **Fuente del estímulo** | Usuario autenticado (arrendador o inquilino). |
| **Estímulo** | Un usuario intenta consultar el consumo o los reportes de una unidad que no le pertenece o a la que no está asignado, modificando el identificador en la petición. |
| **Entorno** | Tiempo de ejecución, operación normal. |
| **Artefacto** | Microservicios de Propiedades y Unidades, Consumo y Telemetría, y Analíticas y Reportes. |
| **Respuesta** | El sistema aplica la táctica de Autorizar actores, verificando en cada operación la relación de propiedad (`owner_id`) o de asignación (`unit_tenants`) entre el usuario y la unidad, y registra el intento en el log de auditoría. |
| **Medida de la respuesta** | El 100 % de los intentos de acceso a unidades ajenas es rechazado con estado 403 y queda registrado, sin exponer datos de consumo de terceros. |
 
**Escenario 10: Seguridad - Suplantación de un sensor**
 
| Elemento | Descripción |
|---|---|
| **Fuente del estímulo** | Agente externo no autorizado. |
| **Estímulo** | Un dispositivo no registrado intenta publicar lecturas falsas en el tópico MQTT de un medidor existente. |
| **Entorno** | Tiempo de ejecución, ingesta de lecturas. |
| **Artefacto** | Gateway de ingesta MQTT y microservicio de Consumo y Telemetría. |
| **Respuesta** | El sistema aplica la táctica de Autenticar actores a nivel de dispositivo: cada sensor se conecta mediante MQTT sobre TLS con credenciales únicas emitidas al vincularlo, y el gateway solo le permite publicar en su propio tópico. |
| **Medida de la respuesta** | El 100 % de las conexiones con credenciales inválidas es rechazado y ninguna lectura falsa llega a ser registrada en `consumption_readings`. |
 
**Escenario 11: Disponibilidad - Falla del proveedor de notificaciones push**
 
| Elemento | Descripción |
|---|---|
| **Fuente del estímulo** | Servicio externo Firebase Cloud Messaging. |
| **Estímulo** | El proveedor de notificaciones push no responde o devuelve error al enviar una alerta de posible fuga. |
| **Entorno** | Tiempo de ejecución, envío de alertas críticas. |
| **Artefacto** | Microservicio de Alertas y Notificaciones. |
| **Respuesta** | El sistema aplica reintentos con espera exponencial y, si el envío continúa fallando, utiliza el canal alternativo de correo electrónico mediante SendGrid, manteniendo la alerta visible en la aplicación. |
| **Medida de la respuesta** | El 99 % de las alertas críticas llega al usuario por al menos un canal en menos de 5 minutos, aun con el proveedor push no disponible. |
 
**Escenario 12: Usabilidad - Configuración de una meta de ahorro**
 
| Elemento | Descripción |
|---|---|
| **Fuente del estímulo** | Inquilino (estudiante o joven arrendatario). |
| **Estímulo** | El usuario desea establecer su meta de consumo mensual ingresando únicamente su presupuesto disponible en soles. |
| **Entorno** | Tiempo de ejecución, primer uso de la sección de metas. |
| **Artefacto** | Aplicación móvil y microservicio de Ahorro y Recomendaciones. |
| **Respuesta** | El sistema calcula automáticamente la meta equivalente en litros con la tarifa vigente y muestra el resultado con indicadores visuales, sin que el usuario necesite conocer conceptos técnicos como metros cúbicos o rangos tarifarios. |
| **Medida de la respuesta** | El 90 % de los usuarios de prueba configura su meta en menos de 1 minuto y en no más de 3 pasos, sin solicitar ayuda. |
 
**Escenario 13: Modificabilidad - Actualización de tarifas de agua**
 
| Elemento | Descripción |
|---|---|
| **Fuente del estímulo** | Administrador de la plataforma. |
| **Estímulo** | SUNASS aprueba un reajuste tarifario para SEDAPAL o se incorpora una nueva empresa prestadora en otra región del país. |
| **Entorno** | Tiempo de ejecución, operación normal. |
| **Artefacto** | Microservicio de Consumo y Telemetría (entidades `tariffs` y `tariff_ranges`). |
| **Respuesta** | Las tarifas se gestionan como datos parametrizados y versionados por fecha de vigencia, no como lógica codificada, de modo que el administrador registra la nueva estructura sin modificar ni volver a desplegar los microservicios. |
| **Medida de la respuesta** | La nueva tarifa se aplica a los cálculos en soles en menos de 1 día hábil, con 0 líneas de código modificadas. |
 
**Escenario 14: Interoperabilidad - Integración de un nuevo modelo de sensor**
 
| Elemento | Descripción |
|---|---|
| **Fuente del estímulo** | Equipo de desarrollo. |
| **Estímulo** | Se requiere dar soporte a un nuevo fabricante de medidores inteligentes cuyo formato de lectura es distinto al de los medidores ya integrados. |
| **Entorno** | Tiempo de diseño y desarrollo. |
| **Artefacto** | Gateway de ingesta y microservicio de Consumo y Telemetría. |
| **Respuesta** | El sistema aplica el patrón Adapter y la táctica de Definiciones de interfaces compartidas: cada fabricante cuenta con un adaptador que traduce su formato al contrato canónico de lectura (`meterId`, `timestamp`, litros acumulados), sin modificar la lógica de dominio. |
| **Medida de la respuesta** | La integración del nuevo fabricante se completa en un máximo de 3 días de desarrollo, modificando únicamente el nuevo adaptador. |
 
**Escenario 15: Rendimiento - Generación de reporte por unidad**
 
| Elemento | Descripción |
|---|---|
| **Fuente del estímulo** | Arrendador. |
| **Estímulo** | El arrendador solicita el reporte descargable del consumo de una unidad para un periodo de tres meses. |
| **Entorno** | Tiempo de ejecución, operación normal. |
| **Artefacto** | Microservicio de Analíticas y Reportes y servicio externo AWS S3. |
| **Respuesta** | El sistema procesa la solicitud de forma asíncrona: registra el reporte en estado EN_PROCESO, lo genera a partir de los agregados de consumo, lo almacena en AWS S3 y notifica al arrendador cuando está disponible, sin bloquear la interfaz. |
| **Medida de la respuesta** | El reporte queda disponible para descarga en menos de 30 segundos y la solicitud inicial se confirma en menos de 1 segundo. |

En esta sección se definen los Escenarios de Atributos de Calidad (QAS) para la arquitectura de la plataforma HydroSmart. Estos escenarios constituyen una herramienta fundamental de diseño y validación, ya que permiten operativizar y hacer completamente medibles los requerimientos no funcionales (RNF) del sistema, tales como el rendimiento, la seguridad, la disponibilidad y la escalabilidad. Al desglosar cada atributo en términos de fuente, estímulo, artefacto, entorno, respuesta y medida, se garantiza que cada decisión arquitectónica pueda ser verificada objetivamente.

Escenario 1: Disponibilidad – Ingesta continua de lecturas IoT

| Elemento | Descripción |
| :--- | :--- |
| **Fuente del estímulo** | Sensor IoT instalado en el punto de suministro de agua de una vivienda. |
| **Estímulo** | El sensor envía lecturas de caudal de forma continua cada 30 segundos durante las 24 horas del día. |
| **Entorno** | Tiempo de ejecución, bajo condiciones de operación normal en producción con múltiples sensores activos simultáneamente. |
| **Artefacto** | Microservicio de Consumo y Telemetría + Message Broker (RabbitMQ). |
| **Respuesta** | El sistema procesa e ingesta cada lectura sin pérdida de datos. Ante una falla del microservicio, el Message Broker retiene los eventos en cola y el servicio los procesa al recuperarse. |
| **Medida de la respuesta** | El servicio debe estar disponible el **99.5 %** del tiempo mensual, garantizando cero pérdida de lecturas incluso ante reinicios o fallas transitorias del nodo. |

Escenario 2: Rendimiento – Notificación de fuga en tiempo real

| Elemento | Descripción |
| :--- | :--- |
| **Fuente del estímulo** | Módulo AnomalyDetector del bounded context de Alertas y Notificaciones. |
| **Estímulo** | El algoritmo detecta un patrón de consumo continuo indicativo de fuga (flujo constante durante más de 60 minutos sin interrupción). |
| **Entorno** | Operación normal en producción; el usuario tiene la app instalada con permisos de notificaciones push activos. |
| **Artefacto** | Bounded context de Alertas y Notificaciones + Firebase Cloud Messaging. |
| **Respuesta** | El sistema genera el evento `LeakDetected`, persiste la alerta con severidad CRITICAL y despacha la notificación push al dispositivo del usuario. |
| **Medida de la respuesta** | El tiempo transcurrido desde la recepción de la lectura anómala hasta la entrega de la notificación push no debe superar los **15 segundos** en el percentil 95 de las solicitudes. |

Escenario 3: Seguridad – Acceso no autorizado a datos de consumo

| Elemento | Descripción |
| :--- | :--- |
| **Fuente del estímulo** | Usuario no autenticado o con rol incorrecto (por ejemplo, un inquilino intentando acceder a datos de una unidad que no le pertenece). |
| **Estímulo** | Intento de acceso a los endpoints de consumo, alertas o reportes de una unidad sin autorización. |
| **Entorno** | Cualquier entorno (producción o staging), bajo condiciones normales de operación. |
| **Artefacto** | API Gateway + microservicio de Gestión de Identidad (IAM). |
| **Respuesta** | El sistema valida el token JWT y el rol del usuario; si la validación falla, rechaza la solicitud con un error `403 Forbidden` sin exponer información sensible ni datos de la unidad consultada. |
| **Medida de la respuesta** | El **100 %** de los endpoints protegidos valida token y rol antes de procesar la petición. Ningún dato de consumo es accesible sin autorización explícita del propietario o arrendador correspondiente. |

Escenario 4: Escalabilidad – Crecimiento de sensores activos

| Elemento | Descripción |
| :--- | :--- |
| **Fuente del estímulo** | Crecimiento orgánico de la base de usuarios y sensores registrados en la plataforma. |
| **Estímulo** | El número de sensores activos se incrementa de 500 a 5 000 en un periodo de seis meses. |
| **Entorno** | Crecimiento sostenido en producción, con picos de lecturas simultáneas en horas punta de consumo (mañana y noche). |
| **Artefacto** | Microservicio de Consumo y Telemetría + Message Broker (RabbitMQ). |
| **Respuesta** | El sistema mantiene el procesamiento de lecturas dentro de los umbrales de latencia definidos, escalando horizontalmente las instancias del microservicio sin cambios en la arquitectura ni en los contratos de API. |
| **Medida de la respuesta** | La latencia de procesamiento de una lectura no debe incrementarse más del **20 %** respecto a la línea base cuando la carga de sensores se multiplica por 10. |

Escenario 5: Mantenibilidad – Cambio de proveedor de persistencia

| Elemento | Descripción |
| :--- | :--- |
| **Fuente del estímulo** | Equipo de desarrollo durante una fase de evolución del sistema. |
| **Estímulo** | Se requiere migrar el almacenamiento de lecturas de consumo a una base de datos optimizada para series de tiempo (por ejemplo, InfluxDB o TimescaleDB). |
| **Entorno** | Fase de evolución del sistema (iteración futura), sin afectar la operación en producción. |
| **Artefacto** | Capa de repositorio (Repository Pattern) del microservicio de Consumo y Telemetría. |
| **Respuesta** | El cambio de proveedor de persistencia se realiza modificando únicamente la implementación del repositorio, sin alterar la lógica de dominio, los eventos publicados ni los contratos de API. |
| **Medida de la respuesta** | La migración no requiere modificaciones en ningún otro bounded context ni en el API Gateway. El tiempo de implementación y pruebas no debe superar **tres días** de trabajo del equipo. |

Escenario 6: Interoperabilidad – Integración de nuevo protocolo IoT

| Elemento | Descripción |
| :--- | :--- |
| **Fuente del estímulo** | Fabricante de sensor IoT externo que utiliza un protocolo de comunicación diferente al inicialmente soportado (por ejemplo, CoAP en lugar de MQTT). |
| **Estímulo** | Se integra un nuevo modelo de medidor inteligente con un formato de telemetría distinto. |
| **Entorno** | Expansión del ecosistema de hardware compatible de HydroSmart. |
| **Artefacto** | Gateway de ingesta IoT (Adapter Pattern). |
| **Respuesta** | El sistema incorpora el nuevo protocolo implementando un adaptador en el gateway de ingesta, traduciendo el formato del sensor al modelo interno de ConsumptionRecord sin modificar el microservicio de Consumo y Telemetría ni los bounded contexts internos. |
| **Medida de la respuesta** | La integración de un nuevo protocolo de sensor requiere únicamente la creación del adaptador correspondiente, con **cero cambios** en los microservicios de dominio internos. |

Escenario 7: Rendimiento – Carga del dashboard de consumo

| Elemento | Descripción |
| :--- | :--- |
| **Fuente del estímulo** | Usuario propietario o arrendador accediendo al dashboard principal de la aplicación. |
| **Estímulo** | El usuario solicita la visualización del resumen de consumo actual, la proyección de gasto mensual y el estado de sus metas de ahorro. |
| **Entorno** | Operación normal en producción, con múltiples usuarios consultando el dashboard simultáneamente. |
| **Artefacto** | Bounded context de Analíticas y Reportes + API Gateway. |
| **Respuesta** | El sistema recupera y presenta los datos consolidados del dashboard (consumo actual, proyección, metas) de forma completa y sin errores. |
| **Medida de la respuesta** | El tiempo de carga del dashboard completo no debe superar los **2 segundos** en el percentil 95, incluyendo el consumo en tiempo real, la proyección y el progreso de ahorro. |

Escenario 8: Seguridad – Procesamiento de pagos de suscripción

| Elemento | Descripción |
| :--- | :--- |
| **Fuente del estímulo** | Usuario que contrata o cambia su plan de suscripción (Freemium, Pro, Smart). |
| **Estímulo** | El usuario inicia el flujo de pago para activar o actualizar su suscripción. |
| **Entorno** | Operación normal en producción, con datos financieros sensibles en tránsito. |
| **Artefacto** | Bounded context de Suscripciones + pasarela de pagos externa (Stripe/Culqi). |
| **Respuesta** | El sistema delega el procesamiento del pago a la pasarela externa mediante HTTPS, sin almacenar datos de tarjeta localmente. El estado de la suscripción se actualiza únicamente tras confirmación exitosa de la pasarela. |
| **Medida de la respuesta** | El **100 %** de las transacciones de pago se procesan a través de la pasarela certificada PCI-DSS. Ningún dato financiero del usuario se persiste en la base de datos de HydroSmart. |

### 4.2.4 Constraints
 
Identificamos los factores técnicos, legales o de diseño que limitan y condicionan el desarrollo del proyecto. Estas condiciones establecen el marco obligatorio dentro del cual se debe construir la solución, asegurando su viabilidad dentro del entorno previsto.
 
| ID | Descripción |
|---|---|
| R01 | La autenticación debe delegarse a Firebase Authentication, y el sistema debe validar los permisos mediante tokens JWT y roles de usuario (Propietario, Arrendador, Inquilino y Administrador) para bloquear accesos no autorizados a funcionalidades y datos de consumo. |
| R02 | AquaPulse no fabrica ni comercializa hardware: la captura de datos depende de sensores IoT y medidores inteligentes de terceros compatibles que publiquen sus lecturas mediante el protocolo MQTT. |
| R03 | La plataforma y los servicios web deben seguir el enfoque de diseño DDD (Domain-Driven Design), con una arquitectura de microservicios en la que cada bounded context gestiona su propia base de datos. |
| R04 | El backend debe exponer una API RESTful en formato JSON, documentada con OpenAPI 3.1 / Swagger, a través de un único API Gateway para su consumo desde la aplicación web y la aplicación móvil. |
| R05 | Los microservicios deben desarrollarse en Java 17 o superior con Spring Boot 3. |
| R06 | La aplicación web debe desarrollarse en Angular con TypeScript y la aplicación móvil en Flutter para Android e iOS. |
| R07 | La landing page debe desarrollarse con HTML, CSS y JavaScript, y ser responsive para dispositivos móviles. |
| R08 | Se debe usar MySQL como base de datos de los microservicios transaccionales y MongoDB para el almacenamiento de las lecturas de consumo como series de tiempo. |
| R09 | Las lecturas de los sensores deben recibirse mediante MQTT a través de un gateway de ingesta, y la comunicación asíncrona entre microservicios debe realizarse mediante RabbitMQ. |
| R10 | El intercambio de datos entre clientes y servidor debe realizarse mediante HTTPS, y la conexión de los sensores mediante MQTT sobre TLS. |
| R11 | El tratamiento de los datos personales y de consumo debe cumplir con la Ley N.° 29733, Ley de Protección de Datos Personales, y su reglamento. |
| R12 | Los montos deben expresarse en soles (PEN) y calcularse según la estructura tarifaria de la empresa prestadora del servicio (SEDAPAL en Lima) aprobada por SUNASS. |
| R13 | La plataforma contará con tres planes de suscripción: Freemium, Premium (S/ 15 a S/ 25 mensuales) y Arrendador (S/ 40 a S/ 60 mensuales), cuyos pagos se procesan mediante una pasarela externa (Stripe o Culqi). |
| R14 | Solo puede existir una meta de ahorro activa por usuario y periodo mensual, y un medidor solo puede estar vinculado a una unidad a la vez. |
| R15 | Las notificaciones push deben enviarse mediante Firebase Cloud Messaging y los correos electrónicos mediante SendGrid. |
| R16 | Las imágenes de perfil y los reportes descargables deben almacenarse en AWS S3, y la solución debe desplegarse en la nube de AWS. |
| R17 | La aplicación web debe ser compatible con las dos últimas versiones de Chrome, Firefox, Safari y Edge, y la aplicación móvil con Android 8.0 e iOS 14 o superiores. |
| R18 | La solución debe completarse dentro del periodo académico 2026-20, según el cronograma de entregas del curso. |

### 4.2.5 Architectural Concerns
 
**1. Ingesta Confiable de Lecturas en Tiempo Real**
 
Recibir de forma continua las lecturas de miles de sensores sin perder información es la base de todo el sistema. Se aborda mediante un **gateway de ingesta MQTT** que reenvía las lecturas a **RabbitMQ** con colas durables y confirmación de mensajes (acknowledgement), de modo que una lectura solo se descarta de la cola cuando el microservicio de **Consumo y Telemetría** la ha registrado correctamente.
 
**2. Heterogeneidad de los Sensores de Terceros**
 
Al no contar con hardware propio, HydroSmart debe convivir con sensores y medidores de distintos fabricantes y formatos. Se resuelve con el patrón **Adapter** como capa anticorrupción en el gateway de ingesta, traduciendo cada formato a un contrato canónico de lectura (meterId, timestamp, litros acumulados).
 
**3. Precisión en la Detección de Fugas y Anomalías**
 
Una alerta falsa reduce la confianza del usuario, y una fuga no detectada anula la propuesta de valor. Se aborda con el patrón **Strategy** en el microservicio de **Alertas y Notificaciones**, combinando umbrales configurables con una línea base histórica del consumo de cada hogar por franja horaria.
 
**4. Lecturas Perdidas, Duplicadas o Fuera de Orden**
 
Los cortes de conexión pueden provocar reenvíos o vacíos en los datos. Se aplica el patrón **Idempotent Consumer**, registrando cada lectura con la clave única (meterId, timestamp), y se trabaja con el valor **acumulado** del medidor, lo que permite recalcular el consumo del intervalo aunque se pierdan lecturas intermedias.
 
**5. Conversión Confiable del Consumo a Soles**
 
El valor económico mostrado debe coincidir con lo que el usuario pagará. Las tarifas se modelan como datos **parametrizados y versionados** por empresa prestadora, categoría, rango de consumo y fecha de vigencia, y los montos se presentan siempre como estimaciones referenciales del recibo.
 
**6. Entrega Garantizada y No Invasiva de Alertas**
 
Las alertas deben llegar a tiempo sin saturar al usuario. Se utilizan reintentos con espera exponencial, un canal alternativo por correo mediante **SendGrid** cuando **Firebase Cloud Messaging** falla, y la agrupación de alertas de una misma anomalía respetando las preferencias configuradas por el usuario (US10).
 
**7. Seguridad Perimetral y Validación de Identidad**
 
La protección contra accesos no autorizados se resuelve centralizando la validación de los **JWT de Firebase Authentication** en el **API Gateway**, que actúa como filtro de seguridad antes de que la petición llegue a los microservicios internos.
 
**8. Privacidad de los Datos de Consumo entre Arrendador e Inquilino**
 
Los patrones de consumo revelan hábitos y horarios de ocupación del hogar. Arquitectónicamente, se debe asegurar que el arrendador solo acceda al consumo de sus propias unidades, que el inquilino solo vea la unidad que ocupa y que el tratamiento de los datos cumpla la **Ley N.° 29733**, con consentimiento informado y opciones de gestión de datos (US17).
 
**9. Almacenamiento y Consulta Eficiente de Series de Tiempo**
 
El volumen de lecturas crece de forma constante. Se almacenan las lecturas crudas en **MongoDB** como series de tiempo y se consolidan agregados diarios, semanales y mensuales en **Analíticas y Reportes** (modelo de lectura **CQRS**) para el historial, la proyección y los reportes, aplicando una política de retención de lecturas crudas según el plan del usuario.
 
**10. Disponibilidad del Monitoreo ante Fallas de Servicios Secundarios**
 
El usuario debe seguir viendo su consumo y recibiendo alertas aunque servicios no críticos, como **Analíticas y Reportes** o **Ahorro y Recomendaciones**, estén caídos. Se utiliza **RabbitMQ** para desacoplar los servicios mediante comunicación asíncrona basada en eventos de dominio.
 
**11. Gestión de Planes de Suscripción (Monetización)**
 
Controlar que cada usuario acceda solo a las funcionalidades de su plan: por ejemplo, la gestión de múltiples unidades es exclusiva del plan Arrendador. Esta preocupación es atendida por el microservicio de **Suscripciones**, que procesa los pagos mediante la pasarela externa (**Stripe o Culqi**) y sincroniza el estado del plan con las capacidades habilitadas en **Propiedades y Unidades** y **Ahorro y Recomendaciones**.
 
**12. Trazabilidad y Evidencia del Consumo**
 
Los reportes por unidad sirven como evidencia ante disputas con inquilinos. Las lecturas se registran como datos **inmutables** (solo inserción) con marca de tiempo del medidor y del servidor, y cada reporte generado en **Analíticas y Reportes** conserva el periodo y la fuente de datos utilizada.
 
**13. Mantenibilidad mediante Bounded Contexts**
 
Evitar que un cambio en la gestión de propiedades afecte la detección de fugas. Siguiendo el enfoque **DDD**, cada microservicio tiene su propio contexto delimitado y base de datos independiente, y se comunica con los demás mediante contratos de API o eventos de dominio, facilitando actualizaciones aisladas.

## 4.3 ADD Iterations

### 4.3.1 Iteration 1: <Iteration Name>

#### 4.3.1.1 Architectural Design Backlog N

#### 4.3.1.2 Establish Iteration Goal by Selecting Drivers

#### 4.3.1.3 Choose One or More Elements of the System to Refine

#### 4.3.1.4 Choose One or More Design Concepts That Satisfy the Selected Drivers

Considerando las necesidades funcionales de HydroSmart y la estructura definida previamente para la solución, se seleccionan conceptos arquitectónicos que permitan mantener una comunicación organizada entre la aplicación web, la lógica del sistema y la información almacenada. La propuesta toma como base el uso de DDD, una API REST y una organización por contextos funcionales, de manera que las principales operaciones de la plataforma puedan evolucionar sin afectar todo el sistema.

**Performance:**

- La aplicación web se comunicará con el backend mediante una API REST, permitiendo solicitar únicamente la información necesaria para cada funcionalidad. Esto resulta importante para las vistas que muestran datos de consumo, historial, reportes y comparativos, ya que la información debe llegar de forma clara y sin recargar innecesariamente la interfaz.
- El procesamiento de los datos de consumo se concentrará en el backend, donde se realizan las consultas y operaciones necesarias para alimentar el Dashboard y los reportes. De esta manera, la aplicación web se enfoca principalmente en presentar la información al usuario, mientras que el servidor se encarga de procesarla y obtenerla desde la base de datos.

**Security:**

- El acceso a la plataforma contempla un proceso de autenticación de usuarios, permitiendo validar las credenciales antes de acceder a las funcionalidades privadas de HydroSmart. Esta responsabilidad se encuentra relacionada con el contexto de User Management y con los endpoints destinados al registro e inicio de sesión.
- Las operaciones relacionadas con usuarios, perfiles, dispositivos y notificaciones se gestionan desde el backend mediante endpoints específicos. Esta separación permite centralizar el control de las operaciones y evitar que la aplicación web tenga acceso directo a la base de datos.
- La gestión de preferencias y configuraciones del usuario también se mantiene dentro del backend, permitiendo conservar de forma persistente información como las configuraciones de notificaciones.

**Interoperabilidad y Estructura (Modificabilidad):**

- HydroSmart organiza la lógica del dominio mediante bounded contexts, separando responsabilidades como User Management, Consumption Analytics / Reporting, Consumption Monitoring, Anomaly Detection, Notification y Saving Goals. Esta organización reduce el acoplamiento entre funcionalidades y facilita trabajar sobre una parte específica del sistema.
- El backend se implementa con .NET y utiliza MySQL para la persistencia de la información. La comunicación con la aplicación se realiza mediante una API REST, cuyos endpoints fueron documentados y probados con Swagger. Esto establece una interfaz uniforme entre el frontend y los servicios del sistema.
- La solución también mantiene preparada la gestión de dispositivos para una futura integración con sensores o medidores inteligentes, sin hacer que el funcionamiento inicial dependa de hardware especializado.
#### 4.3.1.5 Instantiate Architectural Elements, Allocate Responsibilities, and Define Interfaces

#### 4.3.1.6 Sketch Views (C4 & UML) and Record Design Decisions

#### 4.3.1.7 Analysis of Current Design and Review Iteration Goal (Kanban Board)
