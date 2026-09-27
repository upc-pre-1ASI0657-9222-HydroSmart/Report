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


### 4.1.4 Approach driven ViewPoints Diagrams



### 4.1.5 Relational/Non Relational Database Diagram

### 4.1.6 Design Patterns

### 4.1.7 Tactics

## 4.2 Architectural Drivers

### 4.2.1 Design Purpose

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