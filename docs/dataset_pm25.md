# Dataset de PM2.5 horario — reporte general

Datos de entrada para el pronóstico horario de PM2.5 en Montevideo,
horizontes t+1 a t+24.

---

## Qué contiene

Dos archivos en `data/`, uno por estación de monitoreo, con **grilla horaria
completa y continua**: todas las horas del período aparecen como fila, tengan o
no dato. Las horas sin medición quedan con `pm25` nulo.

| Archivo | Estación | Período | Filas | Cobertura PM2.5 |
|---|---|---|---|---|
| `pm25_maronas.csv` | Curva de Maroñas | 2024-01-01 → 2026-05-25 | 21.024 | 94,29 % |
| `pm25_museo.csv` | Museo Romántico (Ciudad Vieja) | 2026-01-01 → 2026-05-25 | 3.480 | 99,74 % |

Cada CSV va acompañado de un reporte técnico (`*_reporte.md`) con el diccionario
de datos, la completitud detallada y las rachas de horas vacías.

---

## Origen

Los datos provienen del data warehouse PostgreSQL del proyecto de ingeniería de
datos (esquema `aire`), que integra dos fuentes públicas:

| Variable | Fuente original | Estación |
|---|---|---|
| PM2.5 | Portal de datos abiertos de la Intendencia de Montevideo (CKAN) | Curva de Maroñas · Museo Romántico |
| Meteorología | Iowa Environmental Mesonet — red ASOS | SUMU (Aeropuerto de Carrasco) |

---

## Variables

| Columna | Unidad | Descripción |
|---|---|---|
| `fecha_hora` | timestamp | Inicio de la ventana horaria `[HH:00:00, HH:59:59]` |
| `pm25` | µg/m³ | Promedio simple de los registros válidos de la hora |
| `n_registros` | conteo | Registros válidos usados en el promedio |
| `n_esperados` | conteo | Registros teóricos según la cadencia detectada |
| `pm25_imputado` | booleano | `True` si la hora quedó sin ningún registro |
| `temperatura` | °C | Estación SUMU |
| `humedad_relativa` | % | Estación SUMU |
| `velocidad_viento` | nudos | Estación SUMU, sin convertir a m/s |
| `visibilidad` | millas terrestres (SM) | Estación SUMU, sin convertir a km |
| `viento_u` | nudos | Componente este-oeste, positivo hacia el este |
| `viento_v` | nudos | Componente norte-sur, positivo hacia el norte |

---

# Procedimientos aplicados

## 1. Tratamiento de la dirección del viento

### La unidad de origen: grados

En el warehouse el viento se guarda en **dos columnas separadas**:

| Columna | Unidad | Qué mide |
|---|---|---|
| `direccion_viento` | grados sexagesimales, 0 a 360 | ángulo de dónde **viene** el viento |
| `velocidad_viento` | nudos | qué tan fuerte sopla |

La dirección se cuenta en sentido horario desde el norte, y por convención
meteorológica indica el origen del viento, no su destino: 0° es viento del
norte, 90° del este, 180° del sur, 270° del oeste.

### Por qué se cambia de unidad

**Razón 1 — los grados mienten sobre las distancias.** Un modelo lee una columna
numérica como posiciones sobre una recta: cuanto más separados los valores, más
distintos los casos. Pero la escala de grados **se cierra sobre sí misma**,
después de 359 viene 0. Entonces 359° y 1° —dos vientos del norte separados por
apenas 2 grados— aparecen numéricamente a 358 unidades de distancia. Para
cualquier algoritmo que mida distancias o normalice la variable, son los dos
valores más alejados de toda la columna, cuando en realidad son vecinos.

**Razón 2 — ningún modelo puede aprovechar bien un ángulo crudo.** Un árbol de
decisión solo pregunta «¿es mayor o menor que X?»: un corte en 355° manda 350° a
un lado y 10° al otro, partiendo dos direcciones casi idénticas. Puede aproximar
el comportamiento circular acumulando muchos cortes, pero gasta capacidad
compensando la codificación en vez de aprender del fenómeno. Un modelo lineal
está peor: con un solo coeficiente, la variable únicamente puede aportar «a más
grados, más PM2.5», que sobre una circunferencia no significa nada — y arrastra
un salto abrupto entre 359 y 0 donde la predicción cambia de golpe sin que el
viento haya cambiado.

**Razón 3 — la dirección sola no dice nada del efecto.** Viento del sur a 2
nudos y viento del sur a 25 comparten el mismo valor de `direccion_viento`, pero
su capacidad de dispersar contaminantes no tiene comparación. Separadas en dos
columnas, el modelo tiene que reconstruir por su cuenta que actúan juntas.

Existe además un cuarto problema conocido —**los grados tampoco se pueden
promediar**: el promedio de 350° y 10°, dos vientos del norte, da 180°, o sea
viento del sur— pero **no aplica a este dataset**. Los datos meteorológicos ya
llegan con granularidad horaria desde la fuente, un registro por hora, así que
en la extracción no se promedia ninguna dirección. La aclaración importa si más
adelante el análisis reagrupa a resolución diaria o semanal: ahí sí habría que
promediar, y las componentes ya lo resuelven de antemano.

### La unidad de destino: nudos

La dirección se combina con la velocidad y el vector resultante se proyecta
sobre dos ejes perpendiculares:

```
viento_u = -velocidad · sin(radianes(dirección))     eje este-oeste
viento_v = -velocidad · cos(radianes(dirección))     eje norte-sur
```

**El cambio de unidad es el punto central: se pasa de grados a nudos.**
`viento_u` y `viento_v` ya no son ángulos — son **velocidades**. Cada una mide
qué parte de la rapidez del viento empuja el aire hacia el este y hacia el
norte respectivamente.

| | Antes | Después |
|---|---|---|
| Columnas | 2 (`direccion_viento`, `velocidad_viento`) | 2 (`viento_u`, `viento_v`) |
| Unidad de la dirección | grados, 0 a 360 | nudos, de −velocidad a +velocidad |
| Tipo de escala | circular, con salto entre 359 y 0 | continua, sin discontinuidad |
| Qué codifica cada columna | orientación e intensidad por separado | ambas juntas, proyectadas por eje |

`velocidad_viento` se conserva en el dataset como columna propia, porque la
magnitud total del viento sigue siendo informativa por sí misma.

### Ejemplo real: antes y después

Seis horas consecutivas del 10 de marzo de 2026, tal como están en el warehouse
y tal como quedaron en el dataset.

**Antes** — así viene del warehouse:

| `fecha_hora` | `direccion_viento` (grados) | `velocidad_viento` (nudos) |
|---|---|---|
| 2026-03-10 06:00 | 0,0 | 0,00 |
| 2026-03-10 07:00 | 0,0 | 0,00 |
| 2026-03-10 08:00 | 80,0 | 1,00 |
| 2026-03-10 09:00 | 90,0 | 7,00 |
| 2026-03-10 10:00 | 100,0 | 7,00 |
| 2026-03-10 11:00 | 110,0 | 9,00 |

**Después** — así queda en el dataset:

| `fecha_hora` | `velocidad_viento` (nudos) | `viento_u` (nudos) | `viento_v` (nudos) |
|---|---|---|---|
| 2026-03-10 06:00 | 0,00 | 0,00 | 0,00 |
| 2026-03-10 07:00 | 0,00 | 0,00 | 0,00 |
| 2026-03-10 08:00 | 1,00 | −0,98 | −0,17 |
| 2026-03-10 09:00 | 7,00 | −7,00 | 0,00 |
| 2026-03-10 10:00 | 7,00 | −6,89 | +1,22 |
| 2026-03-10 11:00 | 9,00 | −8,46 | +3,08 |

Cómo se lee: el viento rota de 80° a 110°, es decir de este a este-sureste, y se
intensifica de 1 a 9 nudos. En las componentes eso aparece como un flujo hacia
el oeste que se hace más fuerte (`viento_u` cada vez más negativo) y que empieza
a inclinarse hacia el norte (`viento_v` pasa de −0,17 a +3,08).

Las dos primeras filas muestran el caso de viento en calma: con velocidad 0, las
dos componentes valen 0 sin importar qué diga el ángulo. En grados esas horas
figuran como 0°, que un modelo podría confundir con viento del norte; en
componentes queda correctamente registrado que no hay desplazamiento de aire.

### Tabla de referencia

Las ocho direcciones cardinales, todas con velocidad de 10 nudos:

| `direccion_viento` | Viene del | `viento_u` | `viento_v` | El aire se desplaza hacia |
|---|---|---|---|---|
| 0° | norte | 0,0 | −10,0 | sur |
| 45° | noreste | −7,1 | −7,1 | suroeste |
| 90° | este | −10,0 | 0,0 | oeste |
| 135° | sureste | −7,1 | +7,1 | noroeste |
| 180° | sur | 0,0 | +10,0 | norte |
| 225° | suroeste | +7,1 | +7,1 | noreste |
| 270° | oeste | +10,0 | 0,0 | este |
| 315° | noroeste | +7,1 | −7,1 | sureste |

**`viento_u` positivo significa que el aire se desplaza hacia el este;
`viento_v` positivo, hacia el norte.** Un viento diagonal reparte su velocidad
entre ambos ejes: 10 nudos del noreste aportan 7,1 a cada componente.

### Qué se resuelve con el cambio

Los mismos 350° y 10° que en grados distaban 340 unidades quedan así:

| Dirección | `viento_u` | `viento_v` |
|---|---|---|
| 350° | +1,74 | −9,85 |
| 10° | −1,74 | −9,85 |

Ahora los separan menos de 3,5 nudos, que es lo que corresponde a dos vientos
casi idénticos. Desaparece la discontinuidad entre 359 y 0, los cortes de un
árbol dejan de partir direcciones contiguas, un modelo lineal puede asignarle a
cada eje un coeficiente con sentido físico, y dirección e intensidad llegan
juntas en vez de separadas.

### El signo negativo

Sale de la convención meteorológica. La dirección indica de dónde **viene** el
viento, mientras que `viento_u` y `viento_v` describen hacia dónde **va** el
aire, que es lo relevante para el transporte de material particulado. El signo
negativo hace esa inversión: un viento de 0° es viento *del* norte, o sea que
sopla *hacia* el sur, y por eso su `viento_v` es negativo.

### Qué pasa con el ángulo original

**Se descarta**: no aparece en el dataset. Conservarlo invitaría a usarlo por
error, y toda su información ya está contenida en el par `viento_u` /
`viento_v`, del que puede recuperarse si hiciera falta.

La descomposición se aplica **a cada registro individual**, antes de cualquier
agregación. En este dataset la distinción es irrelevante porque hay un solo
registro meteorológico por hora, pero es la práctica correcta y evita el error
si en algún momento la fuente entregara datos sub-horarios.

---

## 2. Agregación de minutos a horas

**Esto aplica únicamente al PM2.5.** Los sensores de material particulado
registran a resolución de minuto y el dataset es horario, así que hay que
agregar. Las variables meteorológicas no pasan por este paso: la red ASOS
entrega una observación por hora, de modo que llegan al dataset con la
granularidad final y se unen por marca temporal sin transformación.

**Ventana cerrada a izquierda.** Cada hora comprende el intervalo
`[HH:00:00, HH:59:59]` y se etiqueta con su inicio, `HH:00`. La fila de las
14:00 resume lo ocurrido entre las 14:00:00 y las 14:59:59, nunca después. Esto
evita que una fila contenga información posterior a su propia etiqueta, que en
un problema de pronóstico sería fuga de información hacia el pasado.

**Promedio simple** de los registros válidos de la ventana, sin ponderaciones.

**Sin umbral de completitud.** Una hora con 12 registros de los 60 esperados
conserva su promedio en lugar de anularse. La razón es que descartar en la
extracción es irreversible: si más adelante el análisis concluye que el umbral
adecuado es 50 % y no 75 %, habría que regenerar todo. En cambio, arrastrar
`n_registros` y `n_esperados` permite aplicar cualquier criterio después:

```python
# el umbral se decide en el analisis, no en la extraccion
confiables = df[df.n_registros >= 0.75 * df.n_esperados]
```

**Sin imputación ni interpolación.** El dataset no inventa ningún valor. La
columna `pm25_imputado` marca las horas que quedaron sin ningún registro, para
que sean identificables sin tener que compararlas contra la grilla.

---

## 3. Detección de la cadencia

Las dos estaciones no muestrean a la misma frecuencia, así que `n_esperados` no
puede ser una constante única.

**La cadencia no está hardcodeada**: se deriva de los propios datos calculando
la **moda del intervalo entre timestamps consecutivos**, mes a mes.

Se usa la moda y no el promedio porque el promedio queda contaminado por los
huecos: si un sensor de 60 segundos estuvo caído cuatro horas, ese salto entra
en el cálculo y arrastra la media hacia arriba, sugiriendo una cadencia más lenta
que la real. La moda es inmune, porque el intervalo nominal es de lejos el más
frecuente.

El cálculo se hace por mes para detectar cambios de equipamiento a mitad del
período. Resultado:

| Estación | Cadencia | Registros esperados/hora | ¿Cambia en el período? |
|---|---|---|---|
| Curva de Maroñas | 60 s | 60 | No, estable en los 29 meses |
| Museo Romántico | 120 s | 30 | No, estable en los 29 meses |

Museo Romántico muestrea a la mitad de la frecuencia. Aplicarle el mismo
denominador que a Maroñas daría una completitud máxima de 0,5 y descartaría la
estación entera por un criterio mal calibrado.

---

## 4. Filtrado de calidad

Se descartan antes de promediar:

- Registros marcados como faltantes en el warehouse.
- Registros marcados como atípicos (hoy 0 filas; se filtra por robustez).
- Valores negativos (el esquema ya los impide; se filtra por robustez).

El campo de calidad del registro **no se usa porque está 100 % vacío**.

**No se detectaron valores sentinela.** Se revisó la distribución de valores
buscando códigos de error disfrazados de medición (el clásico −999, o un valor
alto repetido muchas veces). Los valores altos aparecen dispersos y sin
repeticiones sospechosas, así que no hay nada que filtrar por ese lado.

### Transmitir y medir no son lo mismo

El warehouse distingue entre una fila que no existe y una fila que existe pero
cuyo valor es inutilizable. La diferencia parece técnica pero cambia el
diagnóstico: en 2026, Curva de Maroñas estuvo **16 días transmitiendo con
cadencia completa** —unos 1.439 registros diarios, las 24 horas cubiertas— con
el 100 % de los valores marcados como no utilizables. Visto solo desde el
resultado final parece una interrupción de servicio; en realidad el equipo
funcionaba y medía mal.

Para el modelo el efecto es idéntico (no hay dato aprovechable), pero para
interpretar la serie y para diagnosticar el sensor es una distinción importante.

---

## 5. Grilla horaria completa

Las horas sin medición **aparecen como filas con valores nulos**, no se omiten.

Esto no es una comodidad de formato: es un requisito. Los modelos de series
temporales construyen sus predictores a partir de rezagos —el valor de hace una
hora, de hace dos, de hace veinticuatro—. Si las filas faltantes simplemente no
estuvieran, la fila siguiente a un hueco de tres días quedaría pegada a la
anterior y el modelo tomaría como «hora previa» un valor de setenta y dos horas
antes, sin ninguna señal de que algo se saltó.

Con la grilla completa, cada rezago apunta siempre a la posición temporal
correcta. Los nulos se propagan de forma explícita y son visibles, que es
justamente lo que se busca.

---

## 6. Zona horaria

Todo está en **hora local de Montevideo (UTC-3)**. Uruguay no aplica horario de
verano desde 2015, así que el offset es constante y no hay saltos.

No se aplicó ninguna conversión, porque las dos fuentes ya vienen alineadas:

- **PM2.5**: los CSV de CKAN traen la fecha en hora local y el ETL la lee sin
  asignarle huso horario.
- **Meteorología**: la serie se descarga de IEM pidiendo explícitamente
  `tz=America/Montevideo`, de modo que la conversión la resuelve el proveedor.

Esto se verificó descargando el mismo tramo con ambos husos y comparando: la
hora UTC 03:00 corresponde a la local 00:00 con valores idénticos de
temperatura, humedad, dirección y velocidad. El desfasaje es constante de −3 h.

La verificación no era opcional. Contaminantes y meteorología provienen de
proveedores distintos y se unen por marca temporal; si uno estuviera en UTC y el
otro en hora local, cada fila cruzaría el PM2.5 de una hora con el clima de otra
y el error sería invisible al mirar el CSV.

---

# Particiones previstas

El dataset **no viene particionado**. La división se hace en el análisis:

| Partición | Origen | Filas | Con dato | Media PM2.5 |
|---|---|---|---|---|
| Entrenamiento | Maroñas 2024-2025 | 17.544 | 16.343 (93,2 %) | 13,41 |
| Validación temporal | Maroñas ene → 25-may 2026 | 3.480 | 3.480 (100 %) | 10,65 |
| Validación espacial | Museo Rom. ene → 25-may 2026 | 3.480 | 3.471 (99,7 %) | 7,70 |

```python
import pandas as pd

m = pd.read_csv('data/pm25_maronas.csv', parse_dates=['fecha_hora'])
u = pd.read_csv('data/pm25_museo.csv',   parse_dates=['fecha_hora'])

train          = m[m.fecha_hora <  '2026-01-01']
valid_temporal = m[m.fecha_hora >= '2026-01-01']
valid_espacial = u
```

La validación espacial evalúa si un modelo entrenado en Maroñas generaliza a
otra ubicación **sin reentrenar**.

### Por qué el corte es el 25 de mayo de 2026

Curva de Maroñas interrumpe el monitoreo útil a partir del **2026-05-26**,
durante 16 días. Es el caso descrito más arriba: el sensor siguió transmitiendo
con cadencia completa, pero ningún valor resultó utilizable.

---

# Advertencias conocidas

Ninguna de estas se corrigió en el dataset: quedan documentadas para decidirlas
en el análisis exploratorio.

**Un dato de viento imposible.** El 2025-12-20 a las 12:00 figura una velocidad
de **162 nudos** (300 km/h). Es un error de la fuente IEM. Arrastra valores
extremos en `viento_u` y `viento_v` que van a distorsionar cualquier escalado.
Es una sola fila y conviene excluirla.

**Valores horarios de PM2.5 muy altos.** Maroñas tiene 54 horas por encima de
200 µg/m³, con un máximo de 706. Los picos de julio de 2024 coinciden entre
ambas estaciones y con episodios de invierno conocidos, así que parecen reales.
El del 2025-03-20 (706 µg/m³ en Maroñas contra 20 en Museo Romántico
simultáneamente) es más dudoso.

**Sesgo estacional en las particiones de 2026.** Cubren solo de enero a mayo, sin
el invierno, que es la estación de mayor contaminación. Por eso su media es más
baja que la del entrenamiento. Al comparar métricas hay que tenerlo presente:
una diferencia entre entrenamiento y validación mezcla corrimiento temporal real
con sesgo de cobertura estacional.

**Las dos estaciones miden niveles distintos.** Sobre los días con buena
cobertura en ambas, Maroñas promedia 3,47 µg/m³ más que Museo Romántico, con
correlación diaria de 0,838. La diferencia es real y no un artefacto de
completitud: se verificó que los días de mayor discrepancia tienen 23-24 horas
válidas en las dos estaciones. Un modelo entrenado en Maroñas va a
**sobrepredecir sistemáticamente** en Ciudad Vieja.

---

# Reproducibilidad

**Los archivos de `data/` son la entrada reproducible del proyecto.** Están
versionados en el repositorio, así que cualquier análisis parte exactamente de
los mismos datos sin depender de acceso al warehouse ni de que las fuentes
originales sigan disponibles o inalteradas.

Los portales públicos republican y corrigen series con el tiempo: una extracción
hecha hoy y otra dentro de seis meses pueden diferir aunque cubran el mismo
período. Fijar el dataset en el repositorio elimina esa variabilidad como fuente
de discrepancias entre resultados.

La extracción desde el warehouse queda documentada en este reporte a efectos de
trazabilidad —para que se sepa de dónde salió cada columna y bajo qué
criterios— pero **no forma parte del pipeline reproducible**.
