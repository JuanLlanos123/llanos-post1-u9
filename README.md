Decisiones de diseño: selección de Test Doubles
1. CitaRepository → Mock
Se selecciona un Mock para CitaRepository porque durante las pruebas de CitaReminderServiceImpl no es suficiente comprobar que el repositorio devuelve las citas pendientes de notificación. También es necesario verificar que el servicio interactúa correctamente con él al finalizar el proceso.

Por una parte, se necesita configurar el método findByFechaHoraCitaBetween(...) para que devuelva una lista de citas de prueba, permitiendo controlar cuáles serán procesadas y bajo qué condiciones. Sin embargo, el aspecto más importante es comprobar que, después de enviar el recordatorio, el servicio ejecuta save() exactamente una vez y recibe la cita con su estado actualizado, indicando que el recordatorio ya fue enviado.

Se elige un Mock porque permite tanto configurar respuestas como verificar las interacciones realizadas con el colaborador. Un Stub podría proporcionar la lista de citas, pero no permitiría comprobar directamente que se llamó a save() con los argumentos esperados. Por su parte, un Fake requeriría implementar una versión alternativa del repositorio, algo innecesario cuando el objetivo principal es verificar las interacciones entre el servicio y el repositorio.

2. ReminderChannelFactory / ReminderChannel → Mock
Se selecciona un Mock para ReminderChannelFactory y ReminderChannel porque uno de los comportamientos principales que se quiere evaluar es la selección y utilización correcta del canal de comunicación.

El servicio debe consultar la fábrica para obtener el canal correspondiente a la preferencia configurada en cada cita. Por ejemplo, cuando una cita establece que su medio de notificación es EMAIL, se espera que el servicio solicite precisamente ese canal y no otro. Después de obtenerlo, también debe enviar el mensaje mediante el canal seleccionado.

Utilizar Mocks permite verificar ambas interacciones: primero, que ReminderChannelFactory recibe la solicitud del canal EMAIL y, segundo, que el ReminderChannel correspondiente recibe el mensaje que debe transmitir.

Un Stub permitiría configurar qué canal devuelve la fábrica, pero no sería suficiente para comprobar que el servicio solicitó el canal correcto ni que utilizó efectivamente el canal devuelto. Un Fake implicaría desarrollar implementaciones alternativas de los canales de comunicación, algo innecesario porque la prueba no pretende comprobar el funcionamiento real del correo electrónico, sino verificar que el servicio selecciona y utiliza correctamente la estrategia de notificación.

3. Clock → Fake
Se selecciona un Fake mediante Clock.fixed(...) porque el servicio necesita consultar la fecha y hora actuales para determinar cuáles citas se encuentran dentro del intervalo de tiempo en el que deben enviarse los recordatorios.

En las pruebas unitarias, depender de la hora real del sistema podría generar resultados inconsistentes, ya que una cita podría entrar o salir del intervalo de búsqueda dependiendo del momento exacto en que se ejecute la prueba. Para evitarlo, se utiliza Clock.fixed(...), que permite establecer un instante específico y mantenerlo constante durante toda la ejecución.

Esta alternativa proporciona una implementación funcional de Clock, suministrada por el propio JDK, que conserva el comportamiento esperado de un reloj y permite que operaciones como LocalDateTime.now(clock) funcionen correctamente. De esta manera, las pruebas pueden definir fechas y horas conocidas, facilitando la comprobación de los límites del intervalo de búsqueda y del envío de recordatorios.

Se prefiere esta alternativa frente a un Mock porque no es necesario verificar interacciones con el reloj ni simular individualmente métodos como instant() y getZone(). Tampoco se necesita construir una implementación propia desde cero, puesto que Clock.fixed(...) ya proporciona el comportamiento requerido de forma sencilla, legible y determinista.