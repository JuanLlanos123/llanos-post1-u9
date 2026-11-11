# Recordatorios Citas Service

![CI](https://github.com/JuanLlanos123/llanos-post1-u9/actions/workflows/ci.yml/badge.svg)

## Descripción

Repositorio del post-contenido de la Unidad 9 de Patrones de Diseño de Software — Sexto Semestre. Un único proyecto Spring Boot que envía recordatorios de citas por el canal preferido del paciente, construido desde el inicio con Strategy (`ReminderChannel`) y Factory Method (`ReminderChannelFactory`), ambos utilizados mediante inyección por constructor.

## Parte 1 — Pruebas Unitarias con JUnit 5 y Mockito

`CitaReminderServiceImplTest` aísla `CitaReminderServiceImpl` de sus colaboradores mediante Test Doubles. Se utilizan Mocks para el repositorio y los componentes de notificación, mientras que el control del tiempo se realiza mediante `Clock.fixed(...)`. La selección de cada alternativa se fundamenta en el comportamiento que se necesita comprobar y en los criterios de la tabla de Test Doubles de la Sección 3.1 de la guía de contenido de esta unidad. Consultar la sección «Decisiones de diseño» para conocer la justificación completa.

## Parte 2 — Refactor DIP, Integración y CI/CD

En esta parte se aborda el problema de testabilidad de `enviarRecordatorioUrgente()`, cuyo diseño original construía directamente una instancia de `SmsReminderChannel` mediante `new` dentro del cuerpo del método. Esta decisión generaba un acoplamiento con una implementación concreta e impedía sustituir fácilmente esa dependencia por un Test Double desde una prueba unitaria.

Para solucionar el problema, la construcción se trasladó a `config/CanalUrgenteConfig.java`, mediante el método `canalUrgente()`, y se inyectó la abstracción `ReminderChannel` por constructor. De esta manera, el servicio dejó de crear directamente su dependencia y pasó a recibirla desde la configuración de Spring.

Adicionalmente, `CitaRepositoryTest` (`@DataJpaTest`) y `CitaReminderServiceIntegrationTest` (`@SpringBootTest`) comprueban la persistencia y el flujo completo utilizando H2 y un `Clock` fijo. Finalmente, el pipeline `.github/workflows/ci.yml` ejecuta `mvn verify` en cada push para automatizar la ejecución de las pruebas.

## Decisiones de diseño

### 1. ¿Por qué Strategy facilitó las pruebas?

El patrón Strategy facilitó las pruebas porque permitió separar el mecanismo de envío de notificaciones de la lógica principal del servicio. La interfaz `ReminderChannel` establece una abstracción común para los diferentes medios de comunicación, como SMTP o SMS, evitando que sus implementaciones estén escritas directamente dentro de `CitaReminderServiceImpl`.

Sin esta abstracción, la lógica de envío estaría acoplada a implementaciones concretas y probar la selección del canal exigiría interactuar directamente con librerías de bajo nivel, como clientes SMTP o HTTP. En cambio, al trabajar con una interfaz propia que expone el método `enviar()`, es posible utilizar un Mock de `ReminderChannel` para controlar su comportamiento y verificar que recibe el mensaje correspondiente.

De esta manera, las pruebas se concentran en comprobar que el servicio utiliza correctamente la estrategia de notificación, sin necesidad de realizar envíos reales ni depender del funcionamiento de servicios externos. Además, si posteriormente se incorpora otro medio de comunicación, como WhatsApp, se puede agregar una nueva implementación sin modificar la lógica principal del servicio.

### 2. ¿Por qué Factory Method facilitó las pruebas?

El patrón Factory Method facilitó las pruebas porque permitió separar la decisión de selección del canal de comunicación de la lógica de `CitaReminderServiceImpl`. La clase `ReminderChannelFactory` se encarga de proporcionar el canal correspondiente según la preferencia establecida para cada cita.

Sin esta fábrica, el servicio tendría que incluir directamente un `if/else` o un `switch` para decidir qué implementación utilizar. Esto aumentaría el acoplamiento y dificultaría comprobar de manera aislada que se selecciona el canal correcto.

Con la fábrica inyectada por constructor, esta decisión puede verificarse como una interacción explícita mediante Mockito. Por ejemplo, cuando una cita tiene configurada la preferencia EMAIL, se puede comprobar que el servicio solicita precisamente ese canal mediante `verify(reminderChannelFactory).obtenerCanal(EMAIL)`.

Adicionalmente, la fábrica puede configurarse para devolver un Mock de `ReminderChannel`, permitiendo comprobar que el servicio utiliza el canal obtenido y que este recibe el mensaje que debe transmitir. Así, se verifica tanto la selección de la estrategia como su utilización, sin ejecutar las implementaciones reales de comunicación.

### 3. Selección de Test Doubles: Mock, Stub o Fake

La selección de los Test Doubles para `CitaReminderServiceImpl` se realizó teniendo en cuenta la responsabilidad de cada colaborador y el tipo de comportamiento que se necesita comprobar durante las pruebas unitarias. No todos los colaboradores requieren el mismo tratamiento: algunos necesitan controlar las respuestas, otros permiten verificar las interacciones y, en determinados casos, resulta más conveniente utilizar una implementación funcional existente.

Por esta razón, se analizaron las alternativas Mock, Stub y Fake de acuerdo con sus características y con los criterios establecidos en la tabla de Test Doubles de la Sección 3.1 de la guía de contenido de la unidad.

#### 3.1. CitaRepository → Mock

Se selecciona un Mock para `CitaRepository` porque durante las pruebas de `CitaReminderServiceImpl` no es suficiente comprobar que el repositorio devuelve las citas pendientes de notificación. También es necesario verificar que el servicio interactúa correctamente con él al finalizar el proceso.

Por una parte, se necesita configurar el método `findByFechaHoraCitaBetween(...)` para que devuelva una lista de citas de prueba, permitiendo controlar cuáles serán procesadas y bajo qué condiciones. Sin embargo, el aspecto más importante es comprobar que, después de enviar el recordatorio, el servicio ejecuta `save()` exactamente una vez y recibe la cita con su estado actualizado, indicando que el recordatorio ya fue enviado.

Se elige un Mock porque permite tanto configurar respuestas como verificar las interacciones realizadas con el colaborador. Un Stub podría proporcionar la lista de citas, pero no permitiría comprobar directamente que se llamó a `save()` con los argumentos esperados. Por su parte, un Fake requeriría implementar una versión alternativa del repositorio, algo innecesario cuando el objetivo principal es verificar las interacciones entre el servicio y el repositorio.

Además, utilizar un Mock permite comprobar que el servicio no realiza operaciones de persistencia innecesarias. Por ejemplo, si una cita no cumple las condiciones para recibir el recordatorio o si ocurre un error durante el envío, se puede verificar que no se guarde un estado que indique incorrectamente que la notificación fue enviada. Esto permite evaluar no solamente el resultado del proceso, sino también las decisiones que toma el servicio frente a diferentes situaciones.

#### 3.2. ReminderChannelFactory / ReminderChannel → Mock

Se selecciona un Mock para `ReminderChannelFactory` y `ReminderChannel` porque uno de los comportamientos principales que se quiere evaluar es la selección y utilización correcta del canal de comunicación.

El servicio debe consultar la fábrica para obtener el canal correspondiente a la preferencia configurada en cada cita. Por ejemplo, cuando una cita establece que su medio de notificación es EMAIL, se espera que el servicio solicite precisamente ese canal y no otro. Después de obtenerlo, también debe enviar el mensaje mediante el canal seleccionado.

Utilizar Mocks permite verificar ambas interacciones: primero, que `ReminderChannelFactory` recibe la solicitud del canal EMAIL y, segundo, que el `ReminderChannel` correspondiente recibe el mensaje que debe transmitir.

Para conseguirlo, se puede configurar el Mock de la fábrica para que devuelva un Mock del canal de correo electrónico. De esta manera, la prueba controla qué canal se obtiene y puede verificar posteriormente que ese mismo canal recibió el mensaje. Esto permite comprobar que la fábrica y el servicio trabajan correctamente en conjunto, sin necesidad de ejecutar un envío real.

Un Stub permitiría configurar qué canal devuelve la fábrica, pero no sería suficiente para comprobar que el servicio solicitó el canal correcto ni que utilizó efectivamente el canal devuelto. Un Fake implicaría desarrollar implementaciones alternativas de los canales de comunicación, algo innecesario porque la prueba no pretende comprobar el funcionamiento real del correo electrónico, sino verificar que el servicio selecciona y utiliza correctamente la estrategia de notificación.

También es importante aclarar que estas pruebas no buscan comprobar que SMTP o SMS funcionan correctamente, ya que ese comportamiento pertenece a las implementaciones concretas de cada canal. Lo que se pretende evaluar es que el servicio delegue correctamente el envío y respete la preferencia de comunicación definida para cada cita. Así, la prueba permanece aislada de servicios externos y se concentra en la lógica de negocio.

#### 3.3. Clock → Fake

Se selecciona un Fake mediante `Clock.fixed(...)` porque el servicio necesita consultar la fecha y hora actuales para determinar cuáles citas se encuentran dentro del intervalo de tiempo en el que deben enviarse los recordatorios.

En las pruebas unitarias, depender de la hora real del sistema podría generar resultados inconsistentes, ya que una cita podría entrar o salir del intervalo de búsqueda dependiendo del momento exacto en que se ejecute la prueba. Para evitarlo, se utiliza `Clock.fixed(...)`, que permite establecer un instante específico y mantenerlo constante durante toda la ejecución.

Esta alternativa proporciona una implementación funcional de `Clock`, suministrada por el propio JDK, que conserva el comportamiento esperado de un reloj y permite que operaciones como `LocalDateTime.now(clock)` funcionen correctamente. De esta manera, las pruebas pueden definir fechas y horas conocidas, facilitando la comprobación de los límites del intervalo de búsqueda y del envío de recordatorios.

Se prefiere esta alternativa frente a un Mock porque no es necesario verificar interacciones con el reloj ni simular individualmente métodos como `instant()` y `getZone()`. Tampoco se necesita construir una implementación propia desde cero, puesto que `Clock.fixed(...)` ya proporciona el comportamiento requerido de forma sencilla, legible y determinista.

Otra ventaja importante es que permite que las pruebas sean reproducibles. Independientemente del día o la hora en que se ejecuten, el reloj siempre proporcionará el mismo instante establecido. Esto evita errores relacionados con cambios de fecha, diferencias entre ejecuciones o variaciones en el tiempo del sistema. Además, permite probar escenarios específicos, como citas próximas a la hora actual, citas fuera del intervalo de búsqueda o situaciones en las que no existen recordatorios pendientes.

### 4. Relación entre los Test Doubles y el Principio de Inversión de Dependencias (DIP)

Estas decisiones también se relacionan con el Principio de Inversión de Dependencias (DIP), porque `CitaReminderServiceImpl` recibe sus colaboradores mediante el constructor en lugar de crear directamente sus implementaciones dentro de la lógica de negocio. Esto permite sustituir las dependencias originales por objetos de prueba sin modificar el código del servicio.

Gracias a esta estructura, durante las pruebas se puede proporcionar un Mock de `CitaRepository`, controlar la selección y utilización de los canales mediante Mocks de `ReminderChannelFactory` y `ReminderChannel`, e inyectar un reloj fijo mediante `Clock.fixed(...)`. Cada colaborador se reemplaza o configura según la responsabilidad que desempeña y el comportamiento que se necesita evaluar.

La principal ventaja de esta decisión es que las pruebas pueden concentrarse en la lógica de `CitaReminderServiceImpl`, sin depender de una base de datos real, de servicios externos de comunicación ni de la hora del sistema. Además, se reduce el acoplamiento entre el servicio y sus colaboradores, lo que facilita tanto el mantenimiento como la incorporación de nuevas implementaciones.

En conclusión, la selección de los Test Doubles no se realiza de manera arbitraria, sino de acuerdo con el propósito de cada prueba. Los Mocks permiten verificar interacciones importantes para el comportamiento del servicio, mientras que el Fake proporciona una implementación controlada y funcional para manejar el tiempo. Esta combinación permite construir pruebas unitarias más aisladas, deterministas y fáciles de mantener.

### 5. Diagnóstico antes del refactor: ¿por qué enviarRecordatorioUrgente() no era testeable?

Antes de modificar el código, se identificó que el método `enviarRecordatorioUrgente()` tenía un problema de testabilidad debido a que construía directamente una instancia de `SmsReminderChannel` mediante `new` dentro del cuerpo del método. La instrucción responsable se encuentra en la línea **[NÚMERO DE LÍNEA REAL DEL CÓDIGO ORIGINAL]** del código anterior al refactor.

Esta construcción genera un acoplamiento directo con una implementación concreta, porque el propio método decide qué objeto crear en lugar de recibirlo como dependencia. Como consecuencia, desde una prueba unitaria no se puede sustituir fácilmente esa instancia por un Mock para controlar su comportamiento y verificar las interacciones sin involucrar la implementación concreta. Esto dificulta aislar la lógica del servicio y controlar los diferentes escenarios que se necesitan evaluar durante las pruebas.

En contraste, `enviarRecordatoriosPendientes()` sí es testeable desde la Parte 1 porque recibe sus colaboradores mediante el constructor de `CitaReminderServiceImpl`. Esto permite proporcionar Mocks para el repositorio y los componentes de notificación, además de un `Clock.fixed(...)` para controlar el tiempo. Así se pueden aislar las pruebas, controlar las respuestas de las dependencias y verificar las interacciones del servicio sin recurrir a una base de datos real, servicios de comunicación externos ni a la hora del sistema.

La diferencia entre ambos métodos evidencia la importancia de aplicar el Principio de Inversión de Dependencias (DIP): las dependencias deben recibirse desde el exterior y no construirse directamente dentro de los métodos que contienen la lógica de negocio. El diagnóstico permitió identificar que el problema no estaba necesariamente en la lógica de envío del recordatorio urgente, sino en la manera en que se obtenía su dependencia.

### 6. El refactor DIP: solución al problema de testabilidad

Para solucionar el problema identificado, la construcción de `SmsReminderChannel` se trasladó a `config/CanalUrgenteConfig.java`, mediante el método `canalUrgente()`, y se inyectó la abstracción `ReminderChannel` por constructor en `CitaReminderServiceImpl`.

De esta manera, la creación de la implementación concreta queda bajo la responsabilidad de la configuración de Spring, mientras que el servicio depende de una abstracción y no de una clase específica. Esto permite sustituir el canal por un Mock durante las pruebas, controlar su comportamiento y verificar las interacciones sin necesidad de modificar la lógica de negocio.

El refactor no busca cambiar el comportamiento externo del método, sino mejorar su estructura interna para facilitar las pruebas y reducir el acoplamiento. La prueba `enviarRecordatorioUrgente_gatewayFalla_noMarcaComoEnviado` permite comprobar el comportamiento ante un fallo del canal y verificar que la cita no se marca como enviada cuando la operación de notificación no se completa correctamente.

Esta prueba demuestra que la mejora de testabilidad es una consecuencia directa de aplicar el DIP: al depender de abstracciones inyectadas, el servicio puede probarse de forma aislada y sus colaboradores pueden reemplazarse según las necesidades de cada escenario.

## Ejecutar las pruebas

```bash
mvn test                  # Pruebas unitarias
mvn verify                # Pruebas unitarias + integración
```

## Herramientas utilizadas

- Java 21
- Spring Boot 3.3
- Spring Data JPA
- H2
- JUnit 5
- Mockito
- Apache Maven
- GitHub Actions
- Git
- GitHub

## Conclusiones

Durante el desarrollo de las dos partes se comprendió que la testabilidad de un servicio depende en gran medida de cómo se diseñan y administran sus dependencias. La aplicación de Strategy y Factory Method permitió separar la lógica de notificación de los mecanismos de comunicación, facilitando la verificación de la selección y utilización de los canales mediante Test Doubles. Asimismo, el diagnóstico de `enviarRecordatorioUrgente()` evidenció cómo construir dependencias directamente con `new` dentro de la lógica de negocio dificulta las pruebas unitarias, mientras que aplicar el Principio de Inversión de Dependencias mediante inyección por constructor permite sustituirlas y controlarlas. Finalmente, la integración con Spring Boot, las pruebas con H2 y la automatización mediante GitHub Actions permitieron comprobar el funcionamiento del sistema en diferentes niveles, desde las pruebas unitarias hasta la integración continua.