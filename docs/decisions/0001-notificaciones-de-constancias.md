---
status: "proposed"
date: 2026-11-03 # revisado el 2026-11-03 (Revisión 1)
---

# Procesamiento asíncrono mediante eventos persistidos para notificar constancias

## Context and Problem Statement

ConstanciaUV debe permitir que un estudiante autorizado solicite una constancia digital, validando su identidad institucional (R1), generando y almacenando el documento (R2), conservando el historial de constancias emitidas (R3) y notificando al estudiante cuando la constancia esté disponible (R4).

El envío del correo depende de un proveedor externo al sistema (RT1). Ese proveedor puede estar indisponible, y cuando eso ocurra la generación y el almacenamiento de constancias deben continuar funcionando (RT2), las notificaciones pendientes deben conservarse para intentar su envío posteriormente (RT3) y el sistema debe prevenir solicitudes de notificación duplicadas para una misma constancia, sin presuponer que el proveedor garantice por sí mismo una entrega única (RT4).

El problema a decidir es cómo organizar la comunicación entre el componente que genera y almacena la constancia y el servicio externo de notificaciones: en qué momento se intenta el envío y dónde residen las responsabilidades de reintento, persistencia de pendientes y control de duplicados. Esta decisión no define la arquitectura completa de ConstanciaUV, únicamente el acoplamiento con el servicio de notificaciones.

## Decision Drivers

* **D1. Continuidad de la emisión frente a la indisponibilidad del proveedor externo.** La generación y el almacenamiento de constancias (R2, R3) no deben degradarse ni fallar porque un servicio externo (RT1) no responda (RT2). Esto exige que el resultado de la operación del estudiante no dependa del éxito de una llamada remota.
* **D2. Confiabilidad y unicidad de la notificación bajo control de ConstanciaUV.** Ninguna notificación pendiente puede perderse (RT3) y una misma constancia no debe generar solicitudes de envío duplicadas (RT4). Como no puede presuponerse que el proveedor deduplique, el estado de la notificación y su integración deben quedar en un punto único y localizado dentro del sistema.

## Considered Options

* **Alternativa A. Comunicación directa con recuperación de fallos.** Después de almacenar la constancia, la aplicación intenta solicitar directamente el envío de la notificación al proveedor. Si el proveedor no responde, registra el envío pendiente para reintentarlo posteriormente.
* **Alternativa B. Procesamiento asíncrono mediante eventos persistidos.** Después de almacenar la constancia, la aplicación registra una notificación pendiente como parte de la misma operación. Un componente independiente procesa posteriormente esos registros y se comunica con el proveedor externo.

## Decision Outcome

Chosen option: "Alternativa B. Procesamiento asíncrono mediante eventos persistidos", porque desacopla el momento de la emisión del momento del envío y satisface los tres drivers restrictivos del caso de manera uniforme, no solo en la ruta de error.

En la alternativa B la operación que atiende al estudiante termina cuando la constancia está almacenada y la notificación pendiente ha quedado registrada en el mismo límite transaccional. La indisponibilidad del proveedor nunca forma parte de esa operación, por lo que la emisión continúa funcionando por diseño y no por un mecanismo de recuperación añadido (D1, RT2). El registro de notificaciones pendientes es el camino normal y no una excepción, de modo que la conservación de pendientes (RT3) se cumple siempre con el mismo código, y no solo cuando se detectó correctamente un fallo. Al existir un único registro por constancia con un identificador propio y un estado explícito, el control de duplicados (RT4) se resuelve dentro de ConstanciaUV mediante la unicidad de ese registro y la transición de estados, sin depender del comportamiento del proveedor. Además, toda la comunicación con el proveedor queda concentrada en el componente procesador, lo que localiza el impacto de cualquier cambio externo (D2).

La alternativa A se descarta porque hace que el camino feliz dependa de un servicio externo y porque el cumplimiento de RT2, RT3 y RT4 recae en la ruta de error: si el fallo no se detecta correctamente —por ejemplo, una respuesta ambigua o un tiempo de espera agotado tras el cual el proveedor sí envió el correo— el pendiente puede no registrarse (violando RT3) o registrarse cuando el envío ya ocurrió (violando RT4).

El estado de esta decisión es `proposed`: se trata de una propuesta técnica para revisión, no de una decisión institucional aprobada.

### Consequences

* Good, because la generación y el almacenamiento de constancias dejan de depender de la disponibilidad del proveedor externo, cumpliendo RT2 por construcción y no por recuperación.
* Good, because los pendientes quedan persistidos siempre (RT3) y la deduplicación es responsabilidad explícita y verificable de ConstanciaUV (RT4).
* Good, because la integración con el proveedor queda aislada en un solo componente, por lo que un cambio externo tiene un radio de impacto acotado.
* Bad, because introduce un componente adicional que debe desplegarse, planificarse y supervisarse, aumentando la complejidad operativa y la superficie de fallos propios (por ejemplo, que el procesador se detenga y los pendientes se acumulen sin que nadie lo note).
* Bad, because la notificación deja de ser inmediata: entre la disponibilidad de la constancia y la llegada del correo existe una latencia que depende de la frecuencia del procesador, lo que debe comunicarse al estudiante en la interfaz.
* Trade-off: se acepta mayor complejidad operativa y una notificación eventual en lugar de inmediata, a cambio de que la emisión de constancias sea resistente a fallos externos y de que la unicidad de la notificación sea controlable y auditable dentro del sistema. La alternativa A ofrecería menos piezas y menor latencia, pero al costo de acoplar el camino principal al proveedor y de concentrar el cumplimiento de las restricciones en la ruta menos ejercitada del código.

### Confirmation

Las siguientes comprobaciones se proponen para una implementación futura; no han sido ejecutadas.

* **Continuidad ante fallo del proveedor (RT2).** Prueba de integración con un doble del proveedor configurado para no responder o devolver error. Se solicita una constancia y se verifica que la operación concluya con éxito, que la constancia quede almacenada y aparezca en el historial (R2, R3), y que exista exactamente un registro de notificación en estado pendiente. Repetir el escenario con tiempo de espera agotado, no solo con error inmediato.
* **Conservación de pendientes (RT3).** Prueba en la que el proveedor falla en el primer procesamiento y responde correctamente en uno posterior: se comprueba que el registro permanezca pendiente tras el fallo, que conserve su información de reintento y que acabe en estado enviado sin intervención manual. Comprobar también que el reinicio del procesador no pierde registros pendientes.
* **Prevención de duplicados (RT4).** Prueba que solicite dos veces la notificación de la misma constancia y verifique que solo exista un registro, por restricción de unicidad sobre el identificador de la constancia. Prueba adicional que ejecute dos instancias del procesador en paralelo sobre el mismo registro y verifique una sola llamada efectiva al proveedor; y prueba en la que el proveedor responde tras un tiempo de espera agotado, verificando que el reintento no produzca un segundo envío confirmado.
* **Revisión de diseño.** Revisión por pares de que ninguna ruta del código llame al proveedor fuera del componente procesador, y de que el registro de la notificación pendiente ocurra en el mismo límite transaccional que el almacenamiento de la constancia (si la constancia se guarda, el pendiente también; si no, ninguno).
* **Observabilidad.** Verificar que existan métricas de pendientes por antigüedad y de reintentos fallidos, de modo que la acumulación silenciosa de pendientes sea detectable.
* **Clasificación de errores del proveedor (añadida en la Revisión 1).** Prueba con un doble del proveedor que reproduzca su contrato de errores vigente —tanto los fallos transitorios como los permanentes y las respuestas ambiguas— verificando que el procesador reintente solo los transitorios, marque como fallo definitivo los permanentes y no genere un segundo envío ante una respuesta ambigua. Esta prueba debe ejecutarse contra el contrato del proveedor y volver a revisarse cada vez que ese contrato cambie.

## Pros and Cons of the Options

### Alternativa A. Comunicación directa con recuperación de fallos

La aplicación llama al proveedor inmediatamente después de almacenar la constancia y, ante un fallo, registra el pendiente para reintentarlo.

* Good, because la notificación llega con la mínima latencia posible cuando el proveedor está disponible, que es el caso mayoritario.
* Good, because requiere menos piezas desplegables y resulta más simple de implementar y de seguir en el código.
* Neutral, because igualmente obliga a implementar persistencia de pendientes y reintentos, por lo que no elimina la complejidad de RT3, solo la traslada a la ruta de error.
* Bad, because acopla el camino principal de emisión a un servicio externo: una latencia alta del proveedor degrada directamente el tiempo de respuesta que percibe el estudiante y tensiona RT2.
* Bad, because el cumplimiento de RT3 y RT4 depende de clasificar correctamente el fallo; ante respuestas ambiguas o tiempos de espera agotados puede perderse un pendiente o duplicarse un envío, y esa ruta se ejercita poco en producción.

### Alternativa B. Procesamiento asíncrono mediante eventos persistidos

La aplicación registra la notificación pendiente junto con el almacenamiento de la constancia; un componente independiente la procesa después y habla con el proveedor.

* Good, because la emisión de constancias es independiente de la disponibilidad y del tiempo de respuesta del proveedor, cumpliendo RT2 por diseño.
* Good, because el pendiente persistido es el camino normal, lo que hace que RT3 se cumpla con el mismo código en todos los casos y que RT4 se apoye en una restricción de unicidad propia del sistema.
* Good, because concentra la integración externa en un único componente, acotando el impacto de cambios en el proveedor.
* Neutral, because la notificación se vuelve eventual: aceptable para un aviso de disponibilidad, pero debe quedar explícito ante los usuarios.
* Bad, because añade un componente que hay que operar y supervisar; si se detiene, las constancias se siguen emitiendo pero nadie recibe aviso, y el fallo es silencioso.
* Bad, because dificulta el diagnóstico de un solo caso, ya que el envío ya no ocurre en la misma traza de la solicitud del estudiante.
* Bad, because la correcta clasificación de las respuestas del proveedor sigue siendo necesaria dentro del procesador: aislar la integración acota dónde se hace, pero no elimina la dependencia de la semántica de errores del proveedor (véase *Revisión 1*).

## More Information

* Requisitos y restricciones del caso ConstanciaUV: R1–R4, RT1–RT4.
* Plantilla utilizada: MADR 4.0.0 (`adr-template.md`), de Architectural Decision Records (adr.github.io).
* Esta decisión se limita a la comunicación entre la emisión de constancias y el servicio externo de notificaciones. Quedan fuera de su alcance la validación de identidad institucional (R1), el formato y la firma del documento y el diseño del historial (R3).
* Decisión pendiente de revisión: el estado `proposed` se mantendrá hasta que se valide con las comprobaciones descritas en *Confirmation*.

### Revisión 1. Cambio en el proveedor de notificaciones

**Cambio recibido.** El proveedor externo de notificaciones modificó su interfaz de comunicación y cambió la forma en que reporta los errores de envío. No cambiaron los requisitos ni las restricciones del caso: sigue siendo un servicio externo (RT1) que puede estar indisponible (RT2), los pendientes siguen debiendo conservarse (RT3) y la unicidad de la solicitud sigue sin poder presuponerse del proveedor (RT4).

**Impacto sobre la alternativa seleccionada.** El cambio afecta únicamente al componente procesador de notificaciones pendientes, que es el único que se comunica con el proveedor: su cliente de integración y, sobre todo, la lógica que traduce la respuesta del proveedor a un resultado interno (enviado, fallo transitorio reintentable, fallo permanente). No afecta a la generación ni al almacenamiento de constancias (R2, R3), ni al registro de la notificación pendiente, ni al mecanismo de unicidad que previene duplicados, porque ninguno de ellos conoce al proveedor. El aislamiento previsto en la decisión permitió localizar el cambio: el radio de impacto es un componente y su traducción de errores, no la ruta de emisión.

Sí aparece un impacto real sobre el comportamiento: si la nueva forma de reportar errores clasifica como permanente algo que antes era transitorio (o al revés), el procesador podría dejar de reintentar envíos recuperables —poniendo en riesgo RT3— o reintentar envíos que el proveedor ya aceptó —poniendo en riesgo RT4 si la respuesta es ambigua. Es decir, el cambio no invalida la estructura, pero invalida supuestos de la lógica de reintento.

**Estado de la decisión.** La decisión se mantiene. El cambio confirma el driver D2 en lugar de contradecirlo: precisamente porque la integración está concentrada en un componente, un cambio de interfaz externa se resuelve en un punto conocido y sin tocar la emisión de constancias. La justificación original no se modifica; lo que se agrega es la información nueva de que la semántica de errores del proveedor es un punto de variación que debe tratarse como contrato explícito y versionable, y no como un detalle del cliente. Si en el futuro el proveedor cambiara su modelo de entrega de forma que ya no fuera posible distinguir envíos confirmados de ambiguos, la prevención de duplicados dentro de ConstanciaUV tendría que reevaluarse; esa evaluación posterior queda registrada como pendiente, pero no es necesaria hoy.

**Consecuencia que debe revisarse.** Se refuerza la consecuencia desfavorable sobre la dependencia de la semántica de errores del proveedor, añadida en *Consequences*: el aislamiento acota el lugar del cambio, no su necesidad, y cada cambio de contrato externo implica trabajo de mantenimiento en el procesador.

**Comprobación que debe actualizarse.** Se incorpora en *Confirmation* la prueba de clasificación de errores del proveedor, y se actualiza la comprobación de prevención de duplicados (RT4) para incluir explícitamente el escenario de respuesta ambigua bajo el nuevo contrato de errores. Ambas se describen como pruebas a realizar en una implementación futura; no se han ejecutado.
