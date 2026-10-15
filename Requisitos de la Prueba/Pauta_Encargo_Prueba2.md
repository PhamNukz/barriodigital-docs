# Evaluación Parcial N° 3
## Encargo | Estudiante

| Sigla | Nombre Asignatura | Tiempo Asignado | % Ponderación | Semana inicio |
| :--- | :--- | :--- | :--- | :--- |
| **DSY1107** | Desarrollo Cloud Native I | 5 horas pedagógicas | 12% | EFT |

> Transcripción de [`EP3_DSY1107_Estudiante_encargo.pdf`](EP3_DSY1107_Estudiante_encargo.pdf).
> Los **indicadores 1 al 8 son idénticos** a los de la [pauta de presentación](Pauta_Presentacion_Prueba2.md);
> solo cambia el indicador 9: aquí puntúa lo **implementado**, en EP4 lo **demostrado en vivo**.

---

## 1. Instrucciones generales

### Descripción
En esta etapa deberás incorporar **RabbitMQ** al sistema desarrollado previamente, implementando **colas, exchanges y DLQs** para gestionar tareas asíncronas como notificaciones, generación de tickets o documentos.

Tu desafío es demostrar que la mensajería desacoplada funciona correctamente, **sin afectar la lógica existente del sistema**.

La distribución de los porcentajes en esta evaluación es la siguiente:

| Evaluación | Tipo de situación evaluativa | Distribución de porcentajes | Individual/Grupal |
| :--- | :--- | :--- | :--- |
| **Evaluación Parcial N°3** (12%) | Encargo | 100% | **Parejas** |
| **Total** | | 100% | |

* El encargo corresponde al código de los componentes frontend y backend del sistema, entregado vía GitHub.
* El tiempo asignado para desarrollar esta evaluación es de 2 semanas para la elaboración del encargo y de la presentación.
* Las/los estudiantes inician el trabajo del encargo en el Taller de Proyectos (TAITE 7), pero deben finalizarlo junto a presentación en su tiempo de trabajo personal.

### Instrucciones específicas de la Evaluación:
* Los componentes desarrollados, frontend y backend se entregan mediante el envío a el/la docente del código fuente.
* El código de todos los componentes del backend debe compilar, seguir buenas prácticas y responder a pruebas básicas.
* El código del frontend debe estar completo, modular, sin errores de compilación y con vistas funcionales.
* La integración del backend con la base de datos cloud debe estar configurada correctamente mediante entidades, repositorios y propiedades de conexión.
* El backend debe incluir filtros que validen el JWT recibido desde el IDaaS para autorizar las peticiones.
* El frontend debe implementar el flujo de login con IDaaS y utilizar el JWT en las llamadas al backend.
* Para ambos contextos, front y back, solo debe subirse a los repositorios lo que corresponda a la tecnología utilizada, configurando los archivos `.gitignore` necesarios para lograr esto.
* **Los microservicios deben incluir en su código productores y consumidores que se conecten a colas RabbitMQ.**
* **Los microservicios deben manejar mensajes no entregados enviándolos a DLQ y registrándolos en logs.**
* **El microservicio administrador de RabbitMQ debe permitir la creación y eliminación de colas, exchanges y bindings.**

### Aspectos formales:
* **Formato**: el backend debe corresponder a varios microservicios construidos en Java con Spring Boot, el frontend debe ser un componente Angular. Ambos tipos de componentes deben ser entregados como enlaces a GitHub.
* **Medio de entrega**: Copiar los enlaces de los repositorios de GitHub a AVA y enviar copia al correo del docente, en el plazo de entrega establecido.

### Materiales, herramientas o insumos requeridos:
* Materiales y contenidos académicos proporcionados durante el curso, incluyendo especificaciones técnicas, análisis previos y guías de referencia.
* Computadores disponibles en taller de proyectos (TAITE 7).

### Sistematización del proyecto:
Las/los estudiantes deben incluir en el repositorio (README o documento en la carpeta `/docs`) una **matriz de funcionalidades** con el listado completo de las funcionalidades del alcance del proyecto. Para cada funcionalidad se debe indicar su **estado** (implementada, parcial o no implementada), el **microservicio o componente responsable** y una **evidencia verificable** (endpoint, commit o Pull Request, prueba o captura). El logro se calcula sobre las funcionalidades implementadas y operativas; el puntaje máximo se obtiene con un **90% o más** del alcance implementado.

---

## 2. Pauta de Evaluación

### Niveles de Desempeño
* **Muy buen desempeño (100%)**: Demuestra un desempeño destacado, evidenciando el logro de todos los aspectos evaluados en el indicador.
* **Buen desempeño (80%)**: Demuestra un alto desempeño del indicador, presentando pequeñas omisiones, dificultades y/o errores.
* **Desempeño aceptable (60%)**: Demuestra un desempeño competente, evidenciando el logro de los elementos básicos del indicador, pero con omisiones, dificultades o errores.
* **Desempeño incipiente (30%)**: Presenta importantes omisiones, dificultades o errores en el desempeño, que no permiten evidenciar los elementos básicos del logro del indicador, por lo que no puede ser considerado competente.
* **Desempeño no logrado (0%)**: Presenta ausencia o incorrecto desempeño.

### Resumen de ponderaciones

| # | Indicador | Ponderación |
|---|---|---|
| 1 | Nombres de colas, exchanges y bindings centralizados en `application.yml` o `@Configuration` | 10% |
| 2 | Beans `Queue`, `Exchange` y `Binding` declarados para cada caso de uso | 10% |
| 3 | El código no mezcla lógica de negocio con configuración de mensajería | 8% |
| 4 | Consumidores con `@RabbitListener`, agrupados por dominio funcional | 12% |
| 5 | **Confirmación de mensajes: uso de ACK y manejo explícito de errores** | **16%** |
| 6 | El microservicio administrador expone endpoints REST bien definidos | 10% |
| 7 | La lógica de crear colas y exchanges encapsulada en un servicio | 8% |
| 8 | La API valida los parámetros de entrada | 6% |
| 9 | **Sistematización mediante matriz de funcionalidades** | **20%** |
| | **Total** | **100%** |

---

### Indicadores de Evaluación (Ponderación Total: 100%)

1. **Define en `application.yml` o clase `@Configuration` los nombres de colas, exchanges y bindings de forma centralizada (Ponderación: 10%)**
   * **100%**: Todos los nombres de colas, exchanges y bindings están definidos de manera centralizada en `application.yml` o en una clase de configuración única. No hay valores sueltos en el código. La organización es clara y consistente para todos los microservicios.
   * **80%**: La mayoría de los nombres está centralizada en el archivo de configuración o en la clase `@Configuration`. Existen pocos valores aislados en el código que no afectan de forma importante el mantenimiento.
   * **60%**: Existe una mezcla evidente entre nombres centralizados y nombres escritos directamente en código. La configuración es usable pero dificulta el mantenimiento y la lectura.
   * **30%**: La mayoría de los nombres de colas y exchanges está escrita directamente en las clases de negocio. El archivo de configuración se usa poco o nada.
   * **0%**: No se utiliza configuración centralizada. Todos los nombres están dispersos en el código de forma manual.

2. **Declara beans de configuración de RabbitMQ con `Queue`, `Exchange` y `Binding` para cada caso de uso (Ponderación: 10%)**
   * **100%**: Existen clases de configuración que declaran beans de `Queue`, `Exchange` y `Binding` para cada caso de uso definido en el sistema. Cada ruta de mensajería está claramente identificada y documentada en código.
   * **80%**: La mayoría de los casos de uso tiene sus beans de `Queue`, `Exchange` y `Binding` definidos. Faltan algunos escenarios poco utilizados o secundarios.
   * **60%**: Solo los casos principales están configurados con beans explícitos. Otros usos dependen de configuraciones implícitas o incompletas.
   * **30%**: Hay muy pocos beans de configuración. La mayor parte de la interacción con RabbitMQ se realiza desde el código de negocio sin una configuración clara.
   * **0%**: No hay clases de configuración que declaren beans de `Queue`, `Exchange` o `Binding`.

3. **El código del microservicio no mezcla la lógica de negocio con la configuración de mensajería. Hay clases de configuración separadas (Ponderación: 8%)**
   * **100%**: La configuración de mensajería está completamente aislada en clases específicas. Los servicios de negocio solo consumen interfaces o componentes ya configurados. No se observa lógica de configuración en controladores o servicios de dominio.
   * **80%**: En general la separación es clara. Existen pequeños fragmentos de configuración en clases de negocio que no afectan el diseño global.
   * **60%**: La separación es parcial. Parte de la configuración está en clases dedicadas y otra parte se implementa directamente dentro de servicios o controladores.
   * **30%**: Se observa una mezcla importante de lógica de negocio con instrucciones de configuración de colas y exchanges. La estructura dificulta el mantenimiento.
   * **0%**: No existe separación. Toda la configuración de mensajería se encuentra incrustada en la lógica de negocio.

4. **Los consumidores están implementados con `@RabbitListener` u otra anotación equivalente y se agrupan por dominio funcional (Ponderación: 12%)**
   * **100%**: Todos los consumidores están correctamente implementados con `@RabbitListener` u otra anotación adecuada. Se organizan de manera clara por dominio funcional o contexto de negocio. La estructura del código facilita el mantenimiento y refleja una separación coherente de responsabilidades.
   * **80%**: La mayoría de los consumidores está implementada con `@RabbitListener` y se observa una organización por dominio funcional. Existen pocos casos donde la agrupación podría ser más consistente.
   * **60%**: Los consumidores funcionan pero se encuentran mezclados dentro de un mismo paquete o clase sin una separación clara por dominios. La estructura es comprensible pero no óptima.
   * **30%**: Los consumidores están implementados de forma dispersa. No existe un criterio claro de organización por dominio y es difícil identificar a qué parte del sistema pertenece cada consumidor.
   * **0%**: No se utilizan `@RabbitListener` ni anotaciones equivalentes para implementar consumidores o directamente no se implementa ningún consumidor.

5. **En los consumidores se maneja la confirmación de mensajes. Se ve uso de ACK y manejo explícito de errores (Ponderación: 16%)**
   * **100%**: El manejo de confirmación está completamente implementado. **Cada consumidor gestiona ACK de forma explícita.** Los errores se manejan correctamente con **rutas diferenciadas y decisiones claras entre reintentos, NACK y envío a DLQ**. La lógica de error está documentada y es consistente.
   * **80%**: Los consumidores manejan ACK y controlan errores de manera adecuada en la mayoría de los casos. Existe alguna omisión menor en escenarios poco frecuentes que no afecta el funcionamiento general.
   * **60%**: Los consumidores realizan **ACK automáticos o parciales**. El manejo de errores existe pero es limitado y no cubre todos los casos esperados. Puede haber requeue innecesario o falta de control cuando aparece un error.
   * **30%**: El manejo de confirmación es muy básico. Los consumidores no distinguen entre errores recuperables o no recuperables. Se observan respuestas imprecisas que pueden provocar reprocesos o pérdida de mensajes.
   * **0%**: No se implementa ACK ni manejo explícito de errores. Los consumidores procesan mensajes sin control de flujo y sin mecanismos para gestionar fallas.

6. **El microservicio administrador expone endpoints REST bien definidos, por ejemplo `POST /queues` o `DELETE /queues/{name}` (Ponderación: 10%)**
   * **100%**: El microservicio administrador expone una API REST clara y coherente. Los endpoints tienen rutas, verbos HTTP y nombres de recursos bien definidos. La estructura sigue buenas prácticas de diseño REST y está documentada.
   * **80%**: Los endpoints están bien definidos en su mayoría. Hay detalles menores en nombres o verbos que no afectan el uso de la API.
   * **60%**: La API es funcional pero presenta inconsistencias en nombres de rutas, verbos o estructura de los recursos. La intención se entiende pero no se alinea completamente con un diseño REST cuidadoso.
   * **30%**: Los endpoints son confusos o no siguen una estructura clara. Es difícil deducir qué hace cada ruta solo leyendo el código.
   * **0%**: El microservicio administrador no expone endpoints REST para administrar colas o **no implementa este microservicio**.

7. **La lógica para crear colas y exchanges está encapsulada en un servicio, por ejemplo `RabbitAdminService` (Ponderación: 8%)**
   * **100%**: Toda la lógica de creación, actualización y eliminación de colas y exchanges está encapsulada en un servicio dedicado. **Los controladores llaman a métodos de alto nivel y no conocen detalles internos de RabbitMQ.**
   * **80%**: La mayoría de las operaciones de administración está concentrada en un servicio específico. Existen algunas llamadas directas a la librería de RabbitMQ desde otros puntos del código.
   * **60%**: Parte de la lógica de administración está en un servicio y parte se reparte entre controladores o utilidades. La responsabilidad del servicio no es completamente clara.
   * **30%**: Las operaciones de administración se implementan directamente en controladores o clases no especializadas. El servicio dedicado es inexistente o casi no se utiliza.
   * **0%**: No existe un servicio dedicado para administrar colas y exchanges.

8. **La API valida los parámetros de entrada para evitar crear colas con nombres vacíos o configuraciones inválidas (Ponderación: 6%)**
   * **100%**: Los endpoints del microservicio administrador validan todos los parámetros de entrada. Se usan **DTO con anotaciones de validación** o lógica equivalente. No es posible crear colas con nombres vacíos ni con configuraciones inválidas. **Los mensajes de error son claros.**
   * **80%**: La validación cubre los casos más importantes. Algunos parámetros secundarios no se validan pero no generan fallas críticas.
   * **60%**: La API realiza validaciones básicas. Es posible que se cuelen configuraciones poco claras aunque la mayoría de los errores evidentes se detecta.
   * **30%**: La validación es mínima. Es fácil crear colas con nombres inconsistentes o parámetros poco seguros.
   * **0%**: No se valida la entrada. El backend acepta cualquier dato sin controles previos.

9. **Sistematiza el proyecto mediante una matriz de funcionalidades que evidencia el grado de implementación del sistema respecto del alcance definido (Ponderación: 20%)**
   * **100%**: La sistematización evidencia que **90% o más** de las funcionalidades del alcance están implementadas y operativas. Cada funcionalidad registra **estado, componente responsable y evidencia verificable** en el repositorio.
   * **80%**: La sistematización evidencia que entre **72% y 89%** de las funcionalidades del alcance están implementadas y operativas, con evidencia verificable.
   * **60%**: La sistematización evidencia que entre **54% y 71%** de las funcionalidades del alcance están implementadas y operativas, con evidencia verificable.
   * **30%**: La sistematización evidencia que entre **27% y 53%** de las funcionalidades del alcance están implementadas y operativas, con evidencia verificable.
   * **0%**: Menos del **27%** de las funcionalidades del alcance está implementado y operativo, o no se entrega la sistematización del proyecto.

> **Nota del indicador de sistematización:** el porcentaje se calcula como
> *(funcionalidades implementadas y operativas ÷ total de funcionalidades del alcance) × 100*.
> Las funcionalidades **parciales o sin evidencia verificable no se contabilizan** como implementadas.
> Los tramos son proporcionales al máximo (90%): 80% del puntaje = 72% de funcionalidades; 60% = 54%; 30% = 27%.
