# ADR-001: Optimización de la consulta de resumen de incidentes activos

-   **Estado**: Aceptado
-   **Fecha**: 2026-11-25
-   **Autor(es)**: Equipo de Ingeniería de IncidentOps
-   **Componentes Afectados**: \[`src/incident_repository.py`,
    `src/models.py`, `src/sla_calculator.py` (contrato conservado)\]

------------------------------------------------------------------------

## 1. Contexto y Problema

El método
`get_active_incidents_summary(component_filter, reference_time)` de
`src/incident_repository.py` obtiene actualmente todos los registros de
la tabla `incidents` mediante `SELECT * FROM incidents` y realiza en
Python el filtrado por componente y por estados activos (`OPEN`,
`IN_PROGRESS`).

Este enfoque presenta dos problemas principales:

-   **Lectura y procesamiento innecesarios**: recupera hasta 10.000
    filas, aunque el resultado solo necesita los incidentes activos de
    un componente específico. Esto incrementa el trabajo de SQLite, el
    volumen transferido al proceso y el consumo de memoria.
-   **Consulta N+1**: por cada incidente que supera los filtros se
    ejecuta otra consulta a `sla_policies` para obtener
    `max_resolution_minutes`. El valor recuperado no se utiliza, por lo
    que estas consultas agregan latencia y carga sin aportar al
    resultado.

Después, el método construye un objeto `Incident`, llama a
`evaluate_sla_status` con el `reference_time` recibido y genera una
lista de diccionarios con `incident_id`, `severity`, `status`,
`elapsed_minutes` e `is_breached`.

La optimización debe conservar ese comportamiento observable y las
reglas de negocio existentes. En particular, `evaluate_sla_status`
seguirá siendo responsable de la evaluación SLA y su firma no se
modificará. También se deben mantener la semántica temporal, los límites
de cálculo y la estructura de resultados.

**Restricciones técnicas y de seguridad:**

-   Utilizar exclusivamente `sqlite3` de la biblioteca estándar; no
    incorporar ORM ni dependencias externas.
-   Aplicar consultas parametrizadas con marcadores `?` para todos los
    valores variables.
-   Filtrar en SQLite por `component` y por los estados `OPEN` e
    `IN_PROGRESS`.
-   Recuperar únicamente las columnas necesarias para construir
    `Incident` y producir el resumen.
-   No duplicar en SQL ni en el repositorio las reglas de evaluación
    SLA.

------------------------------------------------------------------------

## 2. Opciones Consideradas

1.  **Opción A --- Mantener la consulta completa y el filtrado en
    Python**:
    -   *Ventajas*: requiere pocos cambios y conserva la estructura
        actual del método.
    -   *Desventajas*: lee y procesa registros que no forman parte del
        resultado; mantiene el costo de memoria y CPU asociado a
        recorrer toda la tabla. No resuelve la consulta N+1.
2.  **Opción B --- Filtrar en SQLite y conservar la evaluación SLA en
    Python (seleccionada)**:
    -   *Ventajas*: reduce las filas leídas mediante filtros por
        componente y estado; permite seleccionar solo las columnas
        requeridas; elimina la consulta innecesaria a `sla_policies`;
        conserva `Incident`, `evaluate_sla_status` y el formato de
        salida.
    -   *Desventajas*: requiere ajustar la consulta y comprobar con el
        plan de ejecución si conviene crear un índice compuesto o
        parcial. La evaluación SLA continúa ejecutándose por cada
        incidente devuelto, porque es lógica de negocio que debe
        preservarse.
3.  **Opción C --- Trasladar la evaluación SLA a una consulta SQL o
    introducir un ORM**:
    -   *Ventajas*: podría concentrar más operaciones en la capa de
        persistencia o proporcionar abstracciones adicionales.
    -   *Desventajas*: duplica o desplaza reglas de negocio que hoy
        pertenecen a `evaluate_sla_status`, aumenta el riesgo de
        divergencia en los cálculos temporales y añade complejidad o
        dependencias innecesarias. No se justifica para resolver los
        problemas identificados.

También se descarta conservar la consulta N+1 a `sla_policies`: el valor
consultado no participa en el resultado y la política SLA ya se resuelve
mediante la lógica existente.

------------------------------------------------------------------------

## 3. Decisión Arquitectónica

Se adopta la **Opción B**. `get_active_incidents_summary` consultará
SQLite con una única sentencia parametrizada que filtre por el
componente solicitado y limite los registros a los estados activos. La
consulta proyectará solo las columnas necesarias para construir cada
objeto `Incident`.

La forma lógica de la consulta será:

``` sql
SELECT incident_id, title, component, severity, status, created_at, resolved_at
FROM incidents
WHERE component = ?
  AND status IN (?, ?)
```

Los valores se suministrarán como parámetros (`component_filter`,
`"OPEN"` y `"IN_PROGRESS"`); no se interpolarán directamente en la
cadena SQL.

Por cada fila resultante, el repositorio continuará construyendo
`Incident` y llamando a
`evaluate_sla_status(incident, reference_time=reference_time)`. Se
eliminará por completo la consulta a `sla_policies`, ya que su resultado
no se utiliza. Se conservarán la firma del método, la firma de
`evaluate_sla_status` y las claves y tipos de datos del resumen
retornado.

**Índices:** se evaluarán un índice compuesto sobre
`(component, status)` y un índice parcial limitado a los estados
activos. La elección se realizará con `EXPLAIN QUERY PLAN` y mediciones
sobre los datos representativos del proyecto; este ADR no presupone que
uno de ellos sea siempre superior ni exige crear ambos. Se evitarán
índices redundantes si el plan y las mediciones no demuestran un
beneficio.

### Diagrama de contraste: flujo anterior y flujo optimizado

``` mermaid
flowchart LR
    subgraph OLD["Flujo anterior — consulta completa + N+1"]
        A["Solicitud: componente y reference_time"] --> B["SELECT * FROM incidents"]
        B --> C["Traer todas las filas"]
        C --> D["Filtrar OPEN / IN_PROGRESS y componente en Python"]
        D --> E["Por cada incidente coincidente"]
        E --> F["SELECT a sla_policies por severidad"]
        F --> G["Resultado de política no utilizado"]
        G --> H["Construir Incident"]
        H --> I["evaluate_sla_status"]
        I --> J["Agregar resumen"]
    end

    subgraph NEW["Flujo nuevo — filtrado en SQLite"]
        A2["Solicitud: componente y reference_time"] --> B2["SELECT de columnas necesarias"]
        B2 --> C2["WHERE component = ? AND status IN (?, ?)"]
        C2 --> D2["Solo filas activas del componente"]
        D2 --> E2["Construir Incident"]
        E2 --> F2["evaluate_sla_status"]
        F2 --> G2["Agregar resumen"]
    end
```

------------------------------------------------------------------------

## 4. Consecuencias e Impacto

-   **Positivas**:
    -   Se evita recuperar y recorrer en Python toda la tabla de
        incidentes cuando solo se requiere un componente y dos estados.
    -   Se elimina la consulta N+1 innecesaria a `sla_policies`; la
        consulta principal pasa a ser la única operación de lectura por
        invocación del método.
    -   Se reduce el volumen de columnas y filas transferidas al
        proceso.
    -   Se conserva la separación de responsabilidades: SQLite filtra
        registros y `evaluate_sla_status` aplica las reglas SLA.
    -   Se mantienen la firma pública del método, la construcción de
        `Incident` y la estructura de los resultados.
-   **Negativas / Trade-offs**:
    -   El beneficio depende de la distribución de incidentes y de que
        el plan de SQLite aproveche un índice adecuado; por eso la
        selección del índice se validará empíricamente.
    -   La evaluación SLA sigue siendo individual por incidente,
        deliberadamente, para no duplicar lógica de negocio.
    -   La creación de índices añade costo de almacenamiento y de
        mantenimiento en escrituras. Solo se conservará el índice que
        demuestre una mejora relevante.
    -   La optimización no modifica ni redefine las reglas de
        calendario, los límites temporales o la política de SLA.
-   **Verificación**:
    1.  Ejecutar pruebas de regresión que comparen los resultados del
        método antes y después para los mismos datos y el mismo
        `reference_time`.
    2.  Cubrir incidentes `OPEN` e `IN_PROGRESS`, otros estados,
        componentes distintos, conjuntos vacíos y valores de fecha
        válidos, incluyendo `resolved_at` nulo.
    3.  Verificar que `elapsed_minutes` e `is_breached` sigan siendo
        determinados por `evaluate_sla_status` y que se mantengan sus
        límites y reglas SLA.
    4.  Confirmar mediante inspección o pruebas que la consulta usa
        parámetros `?`, selecciona solo las columnas necesarias y no
        ejecuta consultas por incidente a `sla_policies`.
    5.  Comparar `EXPLAIN QUERY PLAN` para las alternativas de índice
        `(component, status)` y parcial sobre estados activos. Elegir
        según el plan observado y las mediciones.
    6.  Medir tiempo de ejecución y recursos antes y después con un
        conjunto representativo (incluyendo el escenario de 10.000
        incidentes), manteniendo iguales los datos y parámetros.
        Registrar cantidad de filas examinadas/devueltas cuando sea
        posible y número de consultas ejecutadas.
    7.  Aceptar la implementación si conserva la equivalencia funcional
        y demuestra una reducción del trabajo de consulta y del tiempo
        de ejecución sin introducir dependencias externas.
