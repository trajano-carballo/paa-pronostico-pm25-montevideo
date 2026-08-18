# Registro de uso de Inteligencia Artificial

## Declaración

El equipo **sí utilizó** herramientas de inteligencia artificial generativa durante el desarrollo de este proyecto. Este archivo se actualiza de forma incremental durante todo el proceso, registrando los usos relevantes a medida que ocurren.

No se comparten con estas herramientas datos personales, sensibles, confidenciales ni restringidos. Las consultas al data warehouse se formulan sobre agregados estadísticos (cobertura, conteos, promedios), sin incluir información identificable de personas.

## Herramienta utilizada

- **Claude** (Anthropic)

## Criterio general de uso

Las decisiones metodológicas del proyecto —formulación del problema, selección de variables, diseño de validación, elección de modelos, interpretación de resultados— son tomadas por el equipo y discutidas con el tutor. La IA se usa como apoyo para modificar texto, formular consultas de diagnóstico, revisar código y buscar referencias. Todo resultado se verifica antes de incorporarse al proyecto.

---

## Registro de usos

### 2026-08-17 — Diagnóstico de cobertura de estaciones

**Prompt:**

> Necesito diagnosticar la viabilidad de usar Ciudad Vieja y Tres Cruces como estaciones de evaluación (sin reentrenamiento) para un modelo entrenado en Curva de Maroñas. Solo consultá y reportá, no modifiques nada. Para las tres estaciones y PM2.5: 1\) cadencia real de cada sensor, 2\) cobertura horaria en 2024, 2025, y específicamente en el período de test (2026-01-01 a 2026-05-25), 3\) cortes largos dentro de esa ventana.

**Respuesta (resumen):** la herramienta propuso una consulta SQL parametrizada para calcular completitud horaria por estación y período, y una tabla de cortes largos por estación.

**Uso dado:** la consulta propuesta fue ejecutada por el equipo directamente sobre el warehouse. Los resultados numéricos revelaron que Tres Cruces no registra PM2.5 en ningún período, dato no anticipado por la propuesta original y que obligó a reformular la evaluación de generalización usando únicamente Curva de Maroñas y Museo Romántico (Ciudad Vieja). Los números de cobertura fueron verificados de forma independiente antes de aceptarse.

---

### 2026-08-17 — Redistribución del cronograma de trabajo

**Prompt:**

> Hay que ajustar el cronograma según el comentario del tutor: en agosto la aprobación, planificación y verificación de recursos; en septiembre la preparación de datos, el protocolo de evaluación y las líneas base; en octubre el backtesting de los modelos, la selección y evaluación, y la integración del prototipo; en noviembre las correcciones, la documentación definitiva y la preparación de la defensa. Estas son las fechas oficiales de los entregables del curso, incorporalas a la tabla.

**Respuesta (resumen):** la herramienta propuso una tabla de cronograma con una fila por actividad, marcando el mes correspondiente y agregando en la columna de hito las fechas oficiales de los cuatro entregables y las dos presentaciones intermedias del curso.

---

