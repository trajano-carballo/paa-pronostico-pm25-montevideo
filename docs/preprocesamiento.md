# Guía de preprocesamiento

Decisiones metodológicas para transformar los datos crudos de `data/raw/` en el
conjunto de entrenamiento del modelo de pronóstico de PM2.5, guardado en
`data/processed/`.

---

## Punto de partida

Los archivos de `data/raw/` vienen del warehouse **sin procesar**: sin agregar, sin
imputar, sin grilla completa y sin conversión de unidades.

| Archivo | Contenido | Resolución |
|---|---|---|
| `pm25_original.csv.gz` | PM2.5 de Curva de Maroñas y Museo Romántico | por minuto |
| `meteorologia_original.csv` | Meteorología de SUMU (Carrasco) | horaria |

Van separados a propósito: unirlos antes de preprocesar generaría NaN
artificiales y desalineaciones que contaminarían el diagnóstico de faltantes.

```python
import pandas as pd

pm = pd.read_csv('data/raw/pm25_original.csv.gz', parse_dates=['timestamp'])
me = pd.read_csv('data/raw/meteorologia_original.csv', parse_dates=['timestamp'])
```

Los timestamps llevan offset explícito `-03` (hora local de Montevideo, sin
horario de verano). Las dos fuentes ya están alineadas al mismo huso.

## Orden de trabajo

El preprocesamiento va **antes** de la agregación, no después:

1. Diagnóstico de faltantes en resolución original
2. Outliers de validez en resolución original
3. Agregación a horario con umbral de completitud
4. Outliers contextuales sobre la serie horaria
5. Partición cronológica
6. Imputación

### Por qué este orden

**Un pico espurio se vuelve invisible al promediar.** El caso real del
2025-03-20 en Curva de Maroñas:

| Minuto | PM2.5 |
|---|---|
| 02:28 | 429 |
| 02:29 | 428 |
| **02:30** | **3.968** |
| **02:31** | **3.104** |
| **02:32** | **2.481** |
| **02:33** | **2.903** |
| **02:34** | **2.499** |
| 02:35 | 996 |
| 02:36 | 167 |
| 02:39 | 27 |

Cinco minutos por encima de 2.400 µg/m³ y un colapso a 27 cuatro minutos
después. En resolución original salta a la vista; promediado a la hora se diluye
en un valor plausible y ya no hay forma de detectarlo ni de decidir qué hacer con
él.

**El umbral de completitud debe contar solo mediciones válidas.** Si se agrega
primero, una hora llena de valores erróneos cuenta como completa.

**Los inválidos pasan a NaN y se tratan junto con los faltantes reales**, así que
hay que verlos en su resolución original, sin que la agregación los tape.

---

# Tratamiento de la dirección del viento

Esta sección describe una transformación **pendiente**: los datos de `data/`
traen la dirección en grados, sin tocar. La conversión se hace en el
preprocesamiento.

## La unidad de origen: grados

El archivo trae dos columnas separadas:

| Columna | Unidad | Qué mide |
|---|---|---|
| `direccion_viento_grados` | grados sexagesimales, 0 a 360 | ángulo de dónde **viene** el viento |
| `velocidad_viento_nudos` | nudos | qué tan fuerte sopla |

La dirección se cuenta en sentido horario desde el norte, y por convención
meteorológica indica el origen del viento, no su destino: 0° es viento del
norte, 90° del este, 180° del sur, 270° del oeste.

## Por qué hay que cambiar de unidad

**Razón 1 — los grados mienten sobre las distancias.** Un modelo lee una columna
numérica como posiciones sobre una recta: cuanto más separados los valores, más
distintos los casos. Pero la escala de grados **se cierra sobre sí misma**,
después de 359 viene 0. Entonces 359° y 1° —dos vientos del norte separados por
apenas 2 grados— aparecen numéricamente a 358 unidades de distancia. Para
cualquier algoritmo que mida distancias o normalice la variable, son los dos
valores más alejados de toda la columna, cuando en realidad son vecinos.

**Razón 2 — ningún modelo aprovecha bien un ángulo crudo.** Un árbol de decisión
solo pregunta «¿es mayor o menor que X?»: un corte en 355° manda 350° a un lado
y 10° al otro, partiendo dos direcciones casi idénticas. Puede aproximar el
comportamiento circular acumulando muchos cortes, pero gasta capacidad
compensando la codificación en vez de aprender del fenómeno. Un modelo lineal
está peor: con un solo coeficiente, la variable únicamente puede aportar «a más
grados, más PM2.5», que sobre una circunferencia no significa nada — y arrastra
un salto abrupto entre 359 y 0 donde la predicción cambia de golpe sin que el
viento haya cambiado.

**Razón 3 — la dirección sola no dice nada del efecto.** Viento del sur a 2
nudos y viento del sur a 25 comparten el mismo valor de dirección, pero su
capacidad de dispersar contaminantes no tiene comparación. Separadas en dos
columnas, el modelo tiene que reconstruir por su cuenta que actúan juntas.

**Razón 4 — los grados no se pueden promediar**, y en el paso 3 hay que agregar.
El promedio de 350° y 10°, dos vientos del norte, da 180°: viento del sur, el
sentido exactamente opuesto.

> Esto no es hipotético. El ETL del warehouse cometía exactamente ese error al
> deduplicar las horas con varios reportes METAR: había 844 direcciones
> corrompidas y 171 horas con desvíos mayores a 20 grados, varias de 180. Se
> corrigió el 2026-09-10 y los datos de `data/` ya salen sanos, pero conviene
> tenerlo presente al reagrupar a resolución diaria o semanal.

## La unidad de destino: nudos

La dirección se combina con la velocidad y el vector resultante se proyecta sobre
dos ejes perpendiculares:

```python
import numpy as np

rad = np.radians(me.direccion_viento_grados)
me['viento_u'] = -me.velocidad_viento_nudos * np.sin(rad)   # eje este-oeste
me['viento_v'] = -me.velocidad_viento_nudos * np.cos(rad)   # eje norte-sur
```

**El cambio de unidad es el punto central: se pasa de grados a nudos.** `viento_u`
y `viento_v` ya no son ángulos — son **velocidades**. Cada una mide qué parte de
la rapidez del viento empuja el aire hacia el este y hacia el norte
respectivamente.

| | Antes | Después |
|---|---|---|
| Columnas | 2 (`direccion_viento_grados`, `velocidad_viento_nudos`) | 2 (`viento_u`, `viento_v`) |
| Unidad de la dirección | grados, 0 a 360 | nudos, de −velocidad a +velocidad |
| Tipo de escala | circular, con salto entre 359 y 0 | continua, sin discontinuidad |
| Qué codifica cada columna | orientación e intensidad por separado | ambas juntas, proyectadas por eje |

Conviene conservar `velocidad_viento_nudos` como columna propia: la magnitud
total del viento sigue siendo informativa por sí misma.

## Cómo queda

Las ocho direcciones cardinales, todas con velocidad de 10 nudos:

| Dirección | Viene del | `viento_u` | `viento_v` | El aire se desplaza hacia |
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

## Qué se resuelve

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

## El signo negativo

Sale de la convención meteorológica. La dirección indica de dónde **viene** el
viento, mientras que `viento_u` y `viento_v` describen hacia dónde **va** el
aire, que es lo relevante para el transporte de material particulado. Un viento
de 0° es viento *del* norte, o sea que sopla *hacia* el sur, y por eso su
`viento_v` es negativo.

## Dos detalles al implementar

**Descomponer antes de agregar, nunca al revés.** Si se promedian los grados de
la hora y después se descompone ese promedio, se reintroduce exactamente el error
que la técnica evita.

**Viento en calma.** Llega como dirección `0` con velocidad `0`. No hay código
que lo distinga de viento del norte: la única forma de identificarlo es mirando
la velocidad. Al descomponer queda correctamente en `u = 0, v = 0`, sin
desplazamiento — otra ventaja de las componentes sobre el ángulo crudo.

---

# Agregación a horario

## Ventana y etiqueta

Ventana cerrada a izquierda: cada hora comprende `[HH:00:00, HH:59:59]` y se
etiqueta con su inicio, `HH:00`. La fila de las 14:00 resume lo ocurrido entre
las 14:00:00 y las 14:59:59, nunca después. Así ninguna fila contiene
información posterior a su propia etiqueta, que en un problema de pronóstico
sería fuga de información.

## Umbral de completitud

Una hora se considera válida si tiene al menos el 75 % de las mediciones
esperadas. **El denominador es propio de cada estación**, porque las cadencias
difieren:

| Estación | Cadencia | Registros esperados/hora | Mínimo al 75 % |
|---|---|---|---|
| Curva de Maroñas | 60 s | 60 | 45 |
| Museo Romántico | 120 s | 30 | 23 |

Aplicar un denominador único daría completitud máxima 0,5 en Museo Romántico y
descartaría la estación entera por un criterio mal calibrado.

En análisis previos sobre la serie horaria, el umbral resultó **poco sensible**:
entre 50 % y 90 % la cantidad de horas válidas de Maroñas variaba en 0,3 puntos
porcentuales. La distribución es bimodal — una hora está completa o no existe —
así que el 75 % es defendible precisamente porque el resultado no depende de esa
elección.

## Grilla horaria completa

Después de agregar hay que **reindexar sobre una grilla horaria continua**, con
las horas sin dato presentes como filas de valores nulos.

No es una comodidad de formato: los modelos de series temporales construyen sus
predictores a partir de rezagos. Si las filas faltantes no estuvieran, la fila
siguiente a un hueco de tres días quedaría pegada a la anterior y el modelo
tomaría como «hora previa» un valor de setenta y dos horas antes, sin ninguna
señal de que algo se saltó.

```python
grilla = pd.date_range(inicio, fin, freq='h', tz='America/Montevideo')
serie = serie.reindex(grilla)
```

---

# Particiones previstas

| Partición | Origen | Período |
|---|---|---|
| Entrenamiento | Curva de Maroñas | 2024-01-01 → 2025-12-31 |
| Validación temporal | Curva de Maroñas | 2026-01-01 → 2026-05-25 |
| Validación espacial | Museo Romántico | 2026-01-01 → 2026-05-25 |

La validación espacial evalúa si un modelo entrenado en Maroñas generaliza a otra
ubicación **sin reentrenar**.

El corte del 25 de mayo de 2026 no es arbitrario: Curva de Maroñas interrumpe el
monitoreo útil a partir del 2026-05-26 durante 16 días. El sensor no dejó de
transmitir —siguió enviando unos 1.439 registros diarios con cadencia completa—
pero el 100 % de esos valores quedó marcado como no utilizable. Es una falla de
medición, no de transmisión.

---

# Advertencias conocidas

Verificadas sobre los datos actuales de `data/`. Ninguna está corregida: quedan
para decidir en el preprocesamiento.

## Un dato de viento imposible

El **2025-12-20 12:00** figura una velocidad de **162 nudos** (300 km/h). El
segundo valor más alto de toda la serie es 34 nudos, o sea casi cinco veces
menos. Es un error de la fuente. Al descomponer en componentes arrastra valores
extremos que van a distorsionar cualquier escalado. Es una sola fila.

## Picos de PM2.5 en resolución minutal

49 mediciones por minuto superan los 1.000 µg/m³, con un máximo de **3.968**. La
mayoría se concentra en ráfagas de pocos minutos como la del 2025-03-20 mostrada
arriba. Hay que decidir si son eventos locales reales (una fuente de combustión
junto al sensor) o fallas del instrumento — la forma de la ráfaga, con subida y
colapso en minutos, sugiere lo segundo.

## Visibilidad censurada

**El 85 % de las filas vale exactamente 6,21 millas**, que son los 10 km del tope
que reporta METAR. La variable solo varía cuando la visibilidad cae por debajo de
ese techo, así que no es continua sino **censurada por la derecha**. Tratarla como
una variable continua normal sería incorrecto; conviene evaluar si aporta algo o
si conviene binarizarla.

## Sesgo estacional en las particiones de 2026

Cubren solo de enero a mayo, sin el invierno, que es la estación de mayor
contaminación. Por eso su media es más baja que la del entrenamiento. Al comparar
métricas hay que tenerlo presente: una diferencia entre entrenamiento y
validación mezcla corrimiento temporal real con sesgo de cobertura estacional.

## Las dos estaciones miden niveles distintos

Sobre los días con buena cobertura en ambas, Maroñas promedia 3,47 µg/m³ más que
Museo Romántico, con correlación diaria de 0,838. La diferencia es real y no un
artefacto de completitud: los días de mayor discrepancia tienen 23 o 24 horas
válidas en las dos estaciones. Un modelo entrenado en Maroñas va a
**sobrepredecir sistemáticamente** en Ciudad Vieja.

---

# Reproducibilidad

Los archivos de `data/raw/` son la entrada reproducible del proyecto. Están
versionados, así que cualquier análisis parte exactamente de los mismos datos sin
depender de acceso al warehouse ni de que las fuentes originales sigan
disponibles o inalteradas.

Los portales públicos republican y corrigen series con el tiempo: una extracción
hecha hoy y otra dentro de seis meses pueden diferir aunque cubran el mismo
período. Fijar el dataset en el repositorio elimina esa variabilidad como fuente
de discrepancias entre resultados.

El detalle de columnas, unidades y criterios de calidad de cada archivo está en
[`data/raw/datos_originales_reporte.md`](../data/raw/datos_originales_reporte.md).
