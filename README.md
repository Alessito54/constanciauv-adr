# ConstanciaUV — Architecture Decision Records

Documentación de una decisión arquitectónica del caso ConstanciaUV con la plantilla
**MADR 4.0.0**, conservando su evolución mediante Git.

## Decisiones

- [0001 — Notificaciones de constancias](docs/decisions/0001-notificaciones-de-constancias.md) · estado: `proposed`

Historial del ADR:

1. `docs: registrar propuesta inicial de ADR`
2. `docs: registrar impacto del cambio de proveedor`

---

## Indicaciones de la actividad

**Tema:** Documentación de la arquitectura y del diseño del software · **Modalidad:** Individual

**Objetivo de aprendizaje.** Utilizar una plantilla real de Architecture Decision Record (ADR)
para documentar una decisión arquitectónica, justificar las alternativas consideradas,
establecer cómo podría verificarse y conservar su evolución mediante Git.

**Pregunta guía.** ¿Cómo documentamos una decisión arquitectónica para que otro ingeniero
comprenda qué decidimos, por qué y qué ocurrió cuando cambiaron las condiciones originales?

### Caso práctico: ConstanciaUV

La Universidad necesita un sistema que permita a estudiantes autorizados solicitar y obtener
constancias digitales.

**Requisitos**

| ID | Requisito |
|----|-----------|
| R1 | Validar la identidad institucional del solicitante. |
| R2 | Generar y almacenar la constancia digital. |
| R3 | Mantener un historial de constancias generadas. |
| R4 | Notificar al estudiante cuando su constancia esté disponible. |

**Restricciones**

| ID | Restricción |
|----|-------------|
| RT1 | El servicio de notificaciones por correo electrónico es externo a ConstanciaUV. |
| RT2 | Si el proveedor de correo no está disponible, la generación y el almacenamiento de constancias deberán continuar. |
| RT3 | Las notificaciones pendientes deberán conservarse para intentar su envío posteriormente. |
| RT4 | El sistema deberá prevenir solicitudes de notificación duplicadas para una misma constancia. No se presupone que el proveedor externo garantice por sí mismo una entrega única. |

**Problema arquitectónico.** Decidir cómo organizar la comunicación entre la generación de
constancias y el servicio externo de notificaciones.

- **Alternativa A. Comunicación directa con recuperación de fallos.** Después de almacenar una
  constancia, la aplicación intenta solicitar directamente el envío de la notificación. Si el
  proveedor no responde, registra el envío pendiente para reintentarlo posteriormente.
- **Alternativa B. Procesamiento asíncrono mediante eventos persistidos.** Después de almacenar
  una constancia, la aplicación registra una notificación pendiente. Un componente independiente
  procesa posteriormente esos registros y se comunica con el proveedor externo.

Ambas alternativas requieren controlar fallos, pendientes y duplicados. Su diferencia principal
es cuándo se intenta enviar la notificación y cómo se distribuyen las responsabilidades.

### Partes de la práctica

| Parte | Contenido |
|-------|-----------|
| 1 | Conocer la plantilla MADR 4.0.0 (`adr-template.md`). |
| 2 | Análisis manuscrito previo (problema, drivers, ventajas y desventajas, propuesta, comprobaciones). |
| 3 | Elaborar el ADR en `docs/decisions/0001-notificaciones-de-constancias.md` y registrar el primer commit. |
| 4 | Revisar la decisión ante el cambio de interfaz del proveedor y registrar el segundo commit. |
| 5 | Verificación. |
| 6 | Entrega en Eminus. |

### Secciones obligatorias de MADR

| Sección | Qué se documenta |
|---------|------------------|
| Context and Problem Statement | Problema y restricciones que obligan a decidir |
| Decision Drivers | Dos drivers que influyen en la decisión |
| Considered Options | Las alternativas A y B |
| Decision Outcome | Alternativa propuesta y justificación |
| Consequences | Beneficio, costo o riesgo y trade-off |
| Confirmation | Cómo comprobar posteriormente que la solución cumple la decisión |
| Pros and Cons of the Options | Al menos una ventaja y una desventaja de cada alternativa |
| More Information | Registro de la revisión solicitada en la Parte 4 |

Metadatos: `status: "proposed"` y `date` con la fecha de elaboración o última revisión.

### Reglas técnicas

- Documentar una decisión arquitectónica, no toda la arquitectura de ConstanciaUV.
- Comparar ambas alternativas antes de seleccionar una.
- Relacionar la justificación con RT2, RT3 y RT4.
- Reconocer tanto beneficios como consecuencias desfavorables.
- En *Confirmation*, describir pruebas o revisiones realizables en una implementación futura,
  sin afirmar haberlas ejecutado.
- Mantener el estado `proposed`: es una propuesta académica, no una decisión institucional
  aprobada.

### Criterios de evaluación

| Criterio | Puntos |
|----------|--------|
| Análisis manuscrito y razonamiento inicial | 10 |
| Identificación de drivers y comparación de alternativas | 25 |
| Decisión, justificación y consecuencias | 25 |
| Comprobación propuesta en Confirmation | 10 |
| Uso correcto y completo de la plantilla MADR | 10 |
| Dos commits con evolución verificable del ADR | 15 |
| Evidencias y enlaces de entrega | 5 |
| **Total** | **100** |

**Criterio de cierre.** Problema → drivers → alternativas → decisión → consecuencias →
comprobación → revisión → historial Git.

### Referencia

- Plantilla: [MADR 4.0.0 — `adr-template.md`](https://github.com/adr/madr/blob/main/template/adr-template.md)
