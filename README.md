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

El propósito principal del diseño de HydroSmart es construir una plataforma escalable, segura y de baja latencia capaz de transformar las lecturas continuas de sensores IoT en información accionable para distintos perfiles de usuario (propietarios de viviendas, arrendadores e inquilinos), permitiéndoles detectar fugas, optimizar el consumo de agua y proyectar su gasto de forma anticipada.

Desde la perspectiva de negocio, el sistema busca proveer una experiencia diferencial frente a soluciones tradicionales de medición: en lugar de entregar únicamente lecturas de volumen, HydroSmart interpreta esos datos mediante algoritmos de detección de anomalías y proyecciones financieras, agregando valor directo al usuario final. El modelo de monetización basado en suscripciones escalonadas (Freemium, Pro, Smart) refuerza la necesidad de garantizar alta disponibilidad y confiabilidad, ya que una interrupción en el servicio afecta directamente la percepción de valor del producto.

Desde la perspectiva técnica, el diseño arquitectónico está orientado a satisfacer tres objetivos fundamentales:

1. **Procesamiento continuo y en tiempo real:** Las lecturas de caudal generadas por los sensores IoT deben ser ingestadas, validadas, procesadas y reflejadas en el dashboard del usuario con la menor latencia posible, garantizando que las alertas de fuga lleguen de forma oportuna.
2. **Escalabilidad progresiva y mantenibilidad:** La arquitectura de microservicios organizada en Bounded Contexts (IAM, Consumo y Telemetría, Alertas y Notificaciones, Analíticas y Reportes, Suscripciones, entre otros) permite que cada dominio escale y evolucione de forma independiente, sin comprometer la integridad del sistema completo.
3. **Seguridad y trazabilidad de los datos:** Los datos de consumo hídrico constituyen el activo más crítico del sistema. La plataforma garantiza su integridad mediante procesamiento idempotente (para evitar lecturas duplicadas), validación por rol de usuario en cada petición y delegación de la gestión de identidad a Firebase Authentication.

En síntesis, el diseño de HydroSmart no busca únicamente conectar sensores a una interfaz visual, sino construir un ecosistema de datos hídricos que sea confiable, seguro y capaz de crecer junto con la base de usuarios sin sacrificar la experiencia ni la precisión de la información entregada.

### 4.2.2 Primary Functionality (Primary User Stories)

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