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

### 4.2.3 Quality Attribute Scenarios

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
