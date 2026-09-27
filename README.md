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

### 4.2.4 Constraints

### 4.2.5 Architectural Concerns

## 4.3 ADD Iterations

### 4.3.X Iteration N: <Iteration Name>

#### 4.3.X.1 Architectural Design Backlog N

#### 4.3.X.2 Establish Iteration Goal by Selecting Drivers

#### 4.3.X.3 Choose One or More Elements of the System to Refine

#### 4.3.X.4 Choose One or More Design Concepts That Satisfy the Selected Drivers

#### 4.3.X.5 Instantiate Architectural Elements, Allocate Responsibilities, and Define Interfaces

#### 4.3.X.6 Sketch Views (C4 & UML) and Record Design Decisions

#### 4.3.X.7 Analysis of Current Design and Review Iteration Goal (Kanban Board)
