# Evaluación Parcial N° 4
## Presentación | Instrucciones y pauta de evaluación | Estudiante

| Sigla | Nombre Asignatura | Tiempo Asignado | % Ponderación | Semana inicio |
| :--- | :--- | :--- | :--- | :--- |
| **DSY1107** | Desarrollo Cloud Native I | 5 horas pedagógicas | 18% | EFT |

> Transcripción de [`EP4_DSY1107_Estudiante_presentacion.pdf`](EP4_DSY1107_Estudiante_presentacion.pdf).
> Los **indicadores 1 al 8 son idénticos** a los del [encargo](Pauta_Encargo_Prueba2.md);
> solo cambia el indicador 9: allá puntúa lo **implementado**, aquí lo **demostrado en vivo**.

---

## 1. Instrucciones generales

### Descripción
En esta etapa deberás incorporar **RabbitMQ** al sistema desarrollado previamente, implementando **colas, exchanges y DLQs** para gestionar tareas asíncronas como notificaciones, generación de tickets o documentos.

Tu desafío es demostrar que la mensajería desacoplada funciona correctamente, **sin afectar la lógica existente del sistema**.

La distribución de los porcentajes en esta evaluación es la siguiente:

| Evaluación | Tipo de situación evaluativa | Distribución de porcentajes | Individual/Grupal |
| :--- | :--- | :--- | :--- |
| **Evaluación Parcial N°4** (18%) | Presentación | 100% | **Individual** |
| **Total** | | 100% | |

* El encargo corresponde al código de los componentes frontend y backend del sistema, entregado vía GitHub.
* El tiempo asignado para desarrollar esta evaluación es de 2 semanas para la elaboración del encargo y de la presentación.
* Las/los estudiantes inician el trabajo del encargo en el Taller de Proyectos (TAITE 7), pero deben finalizarlo junto a presentación en su tiempo de trabajo personal.

### Instrucciones específicas de la Evaluación:
La presentación del proyecto se realiza de **manera grupal**, respondiendo dudas que el docente realice. **Cada estudiante fundamenta la propuesta desarrollada**, justificando el diseño, tecnologías utilizadas en el sistema y mostrando su funcionamiento.

La presentación debe incluir los siguientes elementos:

| # | El estudiante debe… |
|---|---|
| 1 | mostrar en la plataforma cloud la **creación de la instancia de API Manager** en funcionamiento. |
| 2 | mostrar la **configuración del API Manager** que permite llamar a los endpoints del backend. |
| 3 | demostrar que el **frontend consume los endpoints a través del API Manager** correctamente configurado. |
| 4 | mostrar que el **API Manager valida JWT**, rechazando peticiones inválidas y aceptando las correctas. |
| 5 | mostrar la **creación del tenant en IDaaS** y la existencia de usuarios registrados. |
| 6 | demostrar que el frontend utiliza **OAuth 2.0 / OpenID Connect** para iniciar sesión y obtener un JWT válido. |
| 7 | mostrar en la nube que el **backend y el frontend están desplegados, activos e integrados**. |
| 8 | mostrar la existencia de **dos nodos RabbitMQ** configurados y su **integración en un clúster**. |
| 9 | demostrar que existen **tres colas de mensajes con sus respectivas DLQ** funcionando. |
| 10 | mostrar la existencia de **exchanges de tipo direct y topic** correctamente configurados en RabbitMQ. |
| 11 | demostrar en la nube la ejecución de **docker-compose levantando servicios de RabbitMQ**. |

### Aspectos formales:
* **La presentación debe tener entre 5 y 10 minutos de duración.**
* La información presentada debe tener un **orden e hilo conductor**, haciendo uso de un **lenguaje técnico**.

### Materiales, herramientas o insumos requeridos:
* Materiales y contenidos académicos proporcionados durante el curso, incluyendo especificaciones técnicas, análisis previos y guías de referencia.
* Computadores disponibles en taller de proyectos (TAITE 7).

### Sistematización del proyecto:
El estudiante debe presentar la **matriz de funcionalidades** del proyecto, con el listado completo del alcance, el **estado** de cada funcionalidad y el **componente responsable**, y **demostrar en vivo** las funcionalidades declaradas como implementadas. El logro se calcula sobre las funcionalidades **demostradas en funcionamiento**; el puntaje máximo se obtiene con un **90% o más** del alcance demostrado.

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
| 9 | **Presenta la sistematización, demostrando en funcionamiento** | **20%** |
| | **Total** | **100%** |

---

### Indicadores de Evaluación (Ponderación Total: 100%)

> Los indicadores **1 al 8 son textualmente los mismos** del encargo.
> Están transcritos completos en [`Pauta_Encargo_Prueba2.md`](Pauta_Encargo_Prueba2.md#indicadores-de-evaluación-ponderación-total-100)
> y no se repiten aquí para no duplicar la fuente. Abajo solo el indicador 9, que **sí cambia**.

1. **Define en `application.yml` o clase `@Configuration` los nombres de colas, exchanges y bindings de forma centralizada** — 10%
2. **Declara beans de configuración de RabbitMQ con `Queue`, `Exchange` y `Binding` para cada caso de uso** — 10%
3. **El código del microservicio no mezcla la lógica de negocio con la configuración de mensajería** — 8%
4. **Los consumidores están implementados con `@RabbitListener` y se agrupan por dominio funcional** — 12%
5. **En los consumidores se maneja la confirmación de mensajes: ACK y manejo explícito de errores** — 16%
6. **El microservicio administrador expone endpoints REST bien definidos** — 10%
7. **La lógica para crear colas y exchanges está encapsulada en un servicio** — 8%
8. **La API valida los parámetros de entrada** — 6%

9. **Presenta la sistematización del proyecto, demostrando en funcionamiento las funcionalidades implementadas respecto del alcance definido (Ponderación: 20%)**
   * **100%**: Presenta la matriz de funcionalidades y **demuestra en funcionamiento 90% o más** de las funcionalidades del alcance, explicando su estado y el componente responsable de cada una.
   * **80%**: Presenta la matriz de funcionalidades y demuestra en funcionamiento entre **72% y 89%** de las funcionalidades del alcance.
   * **60%**: Presenta la matriz de funcionalidades y demuestra en funcionamiento entre **54% y 71%** de las funcionalidades del alcance.
   * **30%**: Presenta la matriz de funcionalidades y demuestra en funcionamiento entre **27% y 53%** de las funcionalidades del alcance.
   * **0%**: Demuestra en funcionamiento **menos del 27%** de las funcionalidades del alcance, o no presenta la sistematización del proyecto.

> **Nota del indicador de sistematización:** el porcentaje se calcula como
> *(funcionalidades **demostradas en funcionamiento** ÷ total de funcionalidades del alcance) × 100*.
> **Las funcionalidades que no se logran demostrar durante la presentación no se contabilizan.**
> Los tramos son proporcionales al máximo (90%): 80% del puntaje = 72% de funcionalidades; 60% = 54%; 30% = 27%.

---

## 3. Dos diferencias con EP3 que cambian la estrategia

Esto no está en el PDF: es la lectura del equipo sobre qué implican las dos pautas juntas.

| | EP3 — Encargo | EP4 — Presentación |
|---|---|---|
| Ponderación | 12% | **18%** |
| Modalidad | **Parejas** | **Individual** (aunque la presentación sea grupal) |
| Qué puntúa el indicador 9 | Lo **implementado** con evidencia en el repo | Lo **demostrado en vivo** |

1. **La nota es individual pero la presentación es grupal**, y *"cada estudiante fundamenta la propuesta desarrollada"*. El reparto de [`Pendientes.md`](../Pendientes.md) separa AWS (Francisco) de Entra/frontend (Benjamín): para una nota individual **cada uno tiene que poder defender RabbitMQ, el clúster y el ACK**, no solo su mitad.
2. **Una fila marcada `✅` en la matriz que no se pueda demostrar en vivo vale 0 en EP4.** Conviene una matriz más corta y enteramente demostrable que una larga a medias. El guion y el ensayo cronometrado están en [`Guion-Presentacion-Prueba2.md`](../Guion-Presentacion-Prueba2.md).

Con **11 ítems obligatorios en 5–10 minutos** el presupuesto es de **~30 a 50 segundos por ítem**: el tiempo es parte de la pauta, así que el guion no es opcional.
