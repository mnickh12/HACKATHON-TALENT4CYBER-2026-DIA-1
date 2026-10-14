# Decisión de Aislamiento: Justificación — PIR-08 (parte escrita)

## 1. Datos del reto

| Campo | Valor |
|-------|-------|
| **Nombre** | PIR-08. Decisión de Aislamiento: Justificación |
| **Categoría** | Contención (parte escrita) |
| **Objetivo** | Justificar la decisión de contención parcial frente al aislamiento total del servidor |
| **Puntos** | 0-8 pts (evaluado por el comité técnico) |
| **Extensión** | Máximo 150 palabras |

---

## 2. Contexto

El reto PIR-08 pedía decidir entre:

- **Aislar por completo** el servidor de planificación (máxima seguridad, pero pérdida de servicio).
- **Contener parcialmente** el implante sin desconectar el servicio (equilibrio seguridad/servicio).

La decisión debía estar **justificada**, no ser automática.

---

## 3. Justificación escrita (entrega final)

**Decisión: contención parcial sin desconectar el servicio.**

Se opta por contener el implante sin aislar por completo el servidor de planificación. La evidencia técnica muestra que es posible neutralizar la amenaza manteniendo operativo el servicio legítimo `naval-sync`, del que depende el movimiento de la unidad desplegada. La contención se ha ejecutado en tres pasos:

1. Desactivar la persistencia del implante mediante la máscara `naval-syncd.timer.masked`.
2. Terminar el proceso `naval-syncd` una vez neutralizado su relanzamiento automático.
3. Verificar que `naval-sync` sigue en ejecución.

Aislar por completo el host habría detenido la planificación y generado un impacto operativo innecesario, dado que el implante ha quedado contenido sin propagación ni persistencia. La decisión queda documentada y es reversible si aparecieran indicios de reinfección o movimiento lateral. Esta opción equilibra seguridad y continuidad del servicio, criterio central de la misión.

---

## 4. Razonamiento técnico

### 4.1. Por qué NO se aisló por completo

- El servicio `naval-sync` es **crítico** para la planificación y el movimiento de la unidad.
- Aislar el host habría detenido la planificación sin necesidad.
- El implante `naval-syncd` quedó **contenido** (sin propagación ni persistencia).

### 4.2. Por qué SÍ se contuvo parcialmente

- Se eliminó la **persistencia** (timer y máscara).
- Se terminó el **proceso malicioso**.
- Se mantuvo el **servicio legítimo** operativo.
- La decisión es **reversible** si aparecen indicios de reinfección o movimiento lateral.

### 4.3. Proporcionalidad

La respuesta debe ser **proporcional** al daño. En este caso:

- El implante estaba **contenido**.
- El servicio legítimo era **necesario**.
- Aislar todo habría sido **desproporcionado**.

---

## 5. Lecciones aprendidas

- **La contención no es apagar todo.** Hay que equilibrar seguridad y servicio.
- **La decisión debe estar justificada**, no ser automática.
- **La proporcionalidad** es un criterio central en la respuesta a incidentes.
- **Documentar la decisión** permite revertirla si cambian las condiciones.
