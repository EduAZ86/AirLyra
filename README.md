# AirLyra

> **Estado:** proyecto experimental y documento de visión técnica. Las arquitecturas y mejoras de este README son **propuestas**, no funcionalidades implementadas ni resultados medidos.

AirLyra (también denominada *Aiflyra* en las conversaciones de diseño) es un experimento para explorar cuánto se puede ampliar la capacidad efectiva de un asistente conversacional local construido alrededor de un **modelo de lenguaje pequeño (SLM), de hasta 4 mil millones de parámetros**. El modelo generativo de referencia es **Gemma 4B**, ejecutado localmente en una Mac con Apple Silicon y 16 GB de memoria unificada.

La hipótesis central no es que el modelo deba resolverlo todo en una única inferencia: **el sistema completo —modelo, memoria, herramientas, verificadores y orquestación— debe resolver los problemas con la mayor corrección posible y un costo de recursos predecible**.

## Objetivos

1. Maximizar la **comprensión, el razonamiento y la corrección real**, no solamente la apariencia de inteligencia.
2. Mantener conversaciones largas y coherentes con **memoria persistente**, sin saturar la ventana de contexto.
3. Responder rápido a tareas simples y **dedicar más cómputo solo cuando haga falta**.
4. Ejecutar todo localmente, dentro de límites realistas de RAM, latencia y uso de swap.
5. Medir las mejoras frente a una línea base, evitando asumir que más pasos implican mejores resultados.

**Principio de diseño:** antes de ajustar los parámetros del modelo (*fine-tuning*), medir cuánto puede mejorarse mediante ingeniería del *harness* o arnés.

## Problemas observados y motivación

En un prototipo anterior, el historial conversacional se compactaba al acercarse aproximadamente al 75 % de la ventana de contexto: se generaba un resumen, se descartaba parte del historial y se continuaba con él. Las compactaciones sucesivas podían perder detalles y coherencia. Además, contextos grandes incrementaban la presión sobre la memoria unificada, el uso de swap y los tiempos de procesamiento.

También se observó que un SLM puede resolver con rapidez tareas **bien delimitadas y formuladas**, pero falla con mayor frecuencia cuando debe interpretar, planificar, verificar y sintetizar todo de una sola vez.

Estos puntos motivan separar responsabilidades entre el modelo y el software que lo rodea.

## Arquitectura conceptual propuesta

```text
Usuario
  │
  ▼
Orquestador / router de complejidad
  ├── Respuesta directa o herramienta determinista (casos simples)
  └── Flujo asistido (cuando aporta valor)
        │
        ├── Comprender intención y restricciones
        ├── Recuperar contexto y recuerdos relevantes
        ├── Planificar subtareas acotadas
        ├── Resolver con Gemma y/o herramientas
        ├── Verificadores rápidos y específicos
        │     ├── Reglas y esquemas
        │     ├── Contraste con datos estructurados
        │     └── Clasificadores livianos opcionales
        ├── ¿Hay evidencia o riesgo suficiente para revisar?
        │     ├── No → entregar
        │     └── Sí → crítica focalizada → corrección → reverificación
        └── Sintetizar respuesta y registrar trazas
  │
  ▼
Usuario
```

El mismo modelo Gemma puede ejecutar distintos roles mediante **inferencias secuenciales e instrucciones específicas**, sin cargar varias copias del modelo. Esto no multiplica necesariamente la memoria ocupada por los pesos, aunque sí aumenta el tiempo y el trabajo de inferencia; cada ejecución sigue consumiendo contexto y caché KV.

### 1. Razonamiento adaptativo

- **Ruta rápida:** consultas simples, operaciones deterministas y respuestas de bajo riesgo.
- **Ruta asistida:** tareas que necesitan descomposición, recuperación de información o revisión.
- **Ruta verificada:** problemas con pruebas ejecutables, restricciones estrictas o evidencia de errores.

Evitar una cadena fija de múltiples inferencias para toda consulta. Definir límites de iteraciones, tiempo y tokens. Si la revisión no aporta evidencia nueva, detenerla; un modelo pequeño también puede empeorar una respuesta correcta.

### 2. Memoria en capas

- **Contexto de trabajo:** mensajes recientes y estado mínimo de la tarea actual.
- **Resumen de continuidad:** breve, actualizable, sin tratarlo como fuente infalible.
- **Memoria persistente:** recuerdos en texto con metadatos (fecha, fuente, vigencia, confianza, relaciones).
- **Archivo de conversaciones:** fuente original consultable para verificar resúmenes o recuerdos.

Propuesta técnica: **SQLite** para textos y metadatos, más **embeddings** y búsqueda vectorial/semántica para recuperar recuerdos relevantes. Un embedding **no es el recuerdo**: sirve para localizar el texto que después se entrega al modelo. Considerar búsqueda híbrida (semántica + palabras clave), deduplicación, contradicciones temporales y políticas de actualización de recuerdos.

La recuperación debe ser selectiva: no volcar toda la base de memoria en cada prompt. Priorizar índices y consultas en CPU, con generación de embeddings fuera de la ruta crítica cuando sea posible.

### 3. Verificación por capas: barata primero, costosa después

**Pregunta de investigación:** ¿qué señales permiten decidir, mucho más rápido que una nueva inferencia de Gemma, si una respuesta merece revisión?

| Capa | Técnica | Qué puede detectar | Limitación |
| --- | --- | --- | --- |
| Determinista | Validación de JSON/esquemas, reglas, restricciones, cálculos y tests | Errores comprobables y violaciones de contrato | No entiende lógica abierta |
| Contextual estructurada | Comparar afirmaciones con fechas, estados y datos confirmados | Discrepancias verificables | Depende de extraer bien las afirmaciones |
| Lingüística liviana | Correctores basados en reglas, diccionarios y análisis gramatical | Errores ortográficos o gramaticales | Gramática correcta no implica verdad |
| Semántica liviana (opcional) | Clasificadores NLI de implicación/contradicción | Posibles contradicciones entre pares de enunciados | No valida razonamiento arbitrario; tiene falsos positivos y negativos |
| Revisión con Gemma | Crítica focalizada sobre una dimensión concreta | Omisiones, inconsistencias o problemas que requieran interpretación | Puede repetir sus propios errores |
| Verificación externa | Tests ejecutables, consultas a fuentes y herramientas | Evidencia independiente | No siempre existe un oráculo verificable |

Un **clasificador NLI** puede estimar si una hipótesis contradice una premisa; no es un detector universal de falsedad ni de razonamiento inválido. Su costo y velocidad deben medirse en el hardware real antes de asumir ventajas.

Las revisiones del modelo deben formularse como tareas acotadas, por ejemplo: «identificá las dos afirmaciones contradictorias y explicá la contradicción», en lugar de «revisá todo y decime si está bien». La ausencia de errores detectados **no prueba** que la respuesta sea correcta.

### 4. Ciclo generar → verificar → corregir

Ejemplo con código:

1. Gemma genera una función y sus supuestos.
2. El arnés ejecuta tests reales y validaciones estáticas.
3. Si fallan, devuelve **entradas, salida obtenida, salida esperada y error**.
4. Gemma propone una corrección focalizada.
5. El arnés vuelve a ejecutar las pruebas, conserva la mejor versión **verificada** y corta tras un número limitado de intentos.

En conversación abierta, donde no siempre hay tests deterministas, evaluar por separado: **coherencia interna**, **fidelidad al contexto** y **corrección factual**. No confundir fluidez textual con validez lógica.

### 5. Rendimiento local

- Mantener prompts y contexto de trabajo pequeños; recuperar información relevante bajo demanda.
- Medir *prefill*, generación, tamaño de KV cache, memoria pico y swap.
- Reutilizar prefijos estables mediante caché cuando el motor lo permita.
- Mantener el modelo generativo residente y usar CPU para reglas, índices y verificaciones livianas, sujeto a mediciones reales.
- Diferir indexación y consolidación de recuerdos a momentos de baja actividad.
- Presupuestar latencia y tokens por tarea, no solo por llamada.

## Propuesta de experimentos

### Línea base

Crear un conjunto fijo de tareas representativas de AirLyra: conversación con referencias a turnos anteriores, seguimiento de decisiones, lógica básica, planificación, recuperación de hechos, generación de código y respuestas con restricciones explícitas. Preparar respuestas esperadas, pruebas o rúbricas **independientes** del modelo evaluado cuando sea posible.

### Comparación controlada

- **A — Gemma directo:** una sola inferencia, sin arnés de razonamiento.
- **B — Autocrítica:** generar → crítica focalizada → revisar.
- **C — Verificación externa:** generar → tests/reglas/fuentes → corregir.
- **D — Verificación adaptativa:** filtros baratos → escalar a revisión solo ante señales de riesgo.
- **E — Memoria recuperada:** comparar continuidad y precisión con/sin recuperación selectiva.

Usar el mismo modelo, configuración, conjunto de tareas y presupuesto comparable. Ejecutar varias repeticiones cuando exista variabilidad.

### Métricas

- **Calidad:** exactitud o tasa de tareas resueltas, cumplimiento de instrucciones, consistencia y errores factuales.
- **Evaluador rápido:** *recall* de errores, precisión de alertas, falsos negativos y falsos positivos.
- **Costo:** latencia mediana y percentil 95, tokens de entrada/salida, número de inferencias, RAM pico, swap y CPU/GPU.
- **Eficiencia:** respuestas correctas por segundo y costo promedio por respuesta correcta.
- **Memoria:** recuperación correcta, pérdida de detalles tras muchas conversaciones, contradicciones y atribución a fuentes.

Evitar que Gemma sea el único juez de Gemma: usar casos etiquetados, pruebas deterministas y evaluación humana cuando corresponda.

## Hoja de ruta sugerida

1. **Instrumentación:** registrar tiempos, tokens, memoria y resultados del prototipo actual.
2. **Banco de pruebas:** reunir casos correctos/incorrectos y establecer línea base.
3. **Verificadores deterministas:** restricciones, esquemas, operaciones y pruebas externas.
4. **Router adaptativo:** reglas de escalamiento y presupuestos de revisión.
5. **Autocrítica focalizada:** comparar contra la línea base y detectar regresiones.
6. **Memoria persistente recuperable:** SQLite, embeddings, fuentes y control de vigencia.
7. **Clasificadores livianos:** incorporar NLI u otros solo si mejoran métricas netas.
8. **Optimización:** caché, procesamiento asíncrono, compresión y uso de recursos.
9. **Fine-tuning opcional:** evaluar LoRA u otras técnicas solo después de identificar límites persistentes del arnés.

## Decisiones abiertas

- ¿Qué criterios concretos activan una segunda inferencia y con qué umbrales?
- ¿Qué verificaciones semánticas aportan valor medible en español?
- ¿Cuánto contexto reciente conviene mantener antes de recuperar memoria externa?
- ¿Cómo distinguir recuerdos confirmados, inferidos, corregidos y caducos?
- ¿Cómo evitar que la autocrítica empeore respuestas ya correctas?
- ¿Cuál es el presupuesto máximo aceptable de latencia para una conversación fluida?

## Alcance actual

Este README documenta **ideas discutidas y propuestas de mejora**. No afirma que ya existan en el repositorio un router, RAG, clasificadores NLI, tests de razonamiento ni un ciclo de autocorrección. La implementación se decidirá después de inspeccionar el código vigente y medir una línea base.

---

**Idea rectora:** *No hacer que Gemma piense más veces por defecto, sino darle un entorno que permita pensar mejor cuando realmente haga falta.*
