# Datos en resolucion original — reporte de extraccion

Generado por `scripts/exportar_datos_originales.py` (solo lectura sobre el warehouse).

Dos conjuntos, sin agregar, sin imputar y sin grilla completa. Los registros con valor faltante **se conservan**: la columna de valor va vacia y el flag `indicador_valor_faltante` queda en `true`.

## Alcance

- Periodo solicitado: **todo el disponible (sin recorte)**
- Estaciones de PM2.5: **Curva de Maroñas, Museo Romántico**
- Zona horaria de los timestamps: **America/Montevideo (UTC-3)**, con offset explicito en el archivo

## Conjunto 1 — PM2.5 (formato largo)

Archivo: `pm25_original.csv` — **1943957 filas**

| Columna | Contenido |
|---|---|
| `timestamp` | instante de la medicion, con offset `-03` |
| `estacion` | nombre de la estacion |
| `valor` | concentracion medida; vacio si el registro esta marcado como faltante |
| `unidad` | unidad del valor (constante: `ug/m3`) |
| `indicador_valor_faltante` | flag: la fila existe pero el valor no es utilizable |

**Resolucion: por minuto**, tal como la entrega el sensor. El warehouse la conserva intacta.

| Estacion | Filas | Faltantes | Desde | Hasta | Valor min | Valor max |
|---|---|---|---|---|---|---|
| Curva de Maroñas | 1320820 | 53297 | 2024-01-01 00:00:00 | 2026-08-07 10:59:00 | 1.0000 | 3968.0000 |
| Museo Romántico | 623137 | 12369 | 2024-01-01 00:01:18 | 2026-08-12 23:58:29 | 3.0000 | 643.0000 |

## Conjunto 2 — Meteorologia SUMU (formato ancho)

Archivo: `meteorologia_original.csv` — **22893 filas**

| Columna | Unidad | Contenido |
|---|---|---|
| `timestamp` | — | instante de la observacion, con offset `-03` |
| `temperatura_c` | grados Celsius | |
| `humedad_relativa_pct` | porcentaje | |
| `direccion_viento_grados` | grados sexagesimales | **sin transformar** (ver abajo) |
| `velocidad_viento_nudos` | nudos | sin convertir a m/s |
| `visibilidad_millas` | millas terrestres (SM) | sin convertir a km |
| `indicador_valor_faltante` | — | flag de fila: marca la observacion como no utilizable |

| Metrica | Valor |
|---|---|
| Filas | 22893 |
| Rango | 2024-01-01 00:00:00 a 2026-08-13 20:00:00 |
| Filas con flag de faltante | 70 |
| Con temperatura | 22892 |
| Con humedad relativa | 22891 |
| Con direccion de viento | 22825 |
| Con velocidad de viento | 22892 |
| Con visibilidad | 22893 |

### Como esta la direccion del viento

Se exporta **exactamente como esta en el warehouse**, sin ninguna transformacion: sin conversion de unidades, sin descomposicion en seno y coseno, y sin tratamiento especial de casos particulares.

- **Unidad:** grados sexagesimales, dominio 0 a 360 (hay un CHECK en el esquema que lo garantiza).
- **Convencion:** meteorologica — el angulo indica de **donde viene** el viento, no hacia donde va. 0 es viento del norte, 90 del este, 180 del sur, 270 del oeste.
- **Viento en calma:** llega como `0` con `velocidad_viento_nudos` en `0`. No hay un codigo aparte que lo distinga de viento del norte: la unica forma de identificarlo es mirando la velocidad.
- **Direccion variable (VRB):** el METAR original usa ese codigo cuando la direccion es inestable. En el warehouse **no aparece como texto**: el ETL convierte a numerico y lo que no parsea queda en NULL, o sea columna vacia en el CSV. No es posible distinguir un VRB de un dato ausente por otra causa.
- **Nulos:** la columna puede venir vacia de forma independiente del flag `indicador_valor_faltante`. En este export hay 68 filas sin direccion.

### Como se resuelven las horas con varios reportes METAR

Una parte de las horas trae mas de un reporte: ademas del METAR de rutina emitido en punto, la estacion publica SPECI cuando hay un cambio brusco. Como el modelo admite una sola fila por hora, hay que elegir.

**Se conserva el reporte de rutina (minuto 0) y se descartan los SPECI.** Es la convencion habitual para series horarias de METAR: los SPECI se emiten por evento, no son muestras representativas de la hora. Cuando una hora no tiene reporte en punto se toma el mas cercano al comienzo de la hora.

La fila resultante corresponde entonces a **una observacion real y coherente**: todas las variables provienen del mismo instante.

En este export, `direccion_viento_grados` tiene **0 valores que no son multiplos de 10**. METAR reporta siempre en multiplos de 10, asi que ese numero debe ser 0: cualquier valor distinto indicaria que la direccion paso por un promedio y estaria corrompida.

> **Antecedente.** Hasta el 2026-09-10 el ETL resolvia estas colisiones con `AVG()` sobre todas las columnas, incluida la direccion. Promediar grados es invalido porque la direccion es circular: 330 y 30 promediaban 180, el sentido opuesto al real. Habia 844 direcciones corrompidas y 171 horas con desvios mayores a 20 grados, varias de 180. Se corrigio cambiando el criterio a conservar el reporte de rutina, y se recargo la tabla climatica; la verificacion contra el CSV crudo dio 22.873 de 22.873 horas coincidentes.

### Resolucion temporal del conjunto 2

**Horaria.** Es la granularidad con la que el clima esta guardado en el warehouse, no la de la fuente original.

El ETL trunca el timestamp a la hora al cargar, de modo que los reportes METAR especiales (SPECI, emitidos fuera de la hora en punto) quedaron colapsados aguas arriba. En el CSV crudo descargado hay 25.978 filas, de las cuales 3.105 (12 %) caen fuera de la hora en punto; en el warehouse quedan 22.893, una por hora.

> Consecuencia para el preprocesamiento: en meteorologia **no es posible revisar outliers antes de la agregacion**, porque la agregacion ya ocurrio. Recuperar la resolucion sub-horaria exige volver a descargar de IEM, fuera del alcance de este script.

## Que NO hace este script

- No agrega a ninguna resolucion.
- No imputa ni interpola.
- No elimina ni marca outliers.
- No construye grilla temporal completa: solo salen los instantes que existen en el warehouse.
- No convierte unidades.
- No une los dos conjuntos: se entregan separados porque tienen resolucion distinta y unirlos aca generaria NaN artificiales o desalineaciones.

## Variables no disponibles

El modelo no tiene **punto de rocio** ni **precipitacion**: la descarga de `etl/download.py` solo pide temperatura, humedad, direccion, velocidad y visibilidad, asi que esas dos variables nunca entraron al warehouse.
