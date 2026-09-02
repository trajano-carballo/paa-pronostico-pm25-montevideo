# Dataset PM2.5 horario - Museo Romántico

Generado por `scripts/extraer_dataset_pm25.py` (solo lectura sobre el warehouse).

## Alcance

- Filas (grilla horaria completa): **3480**
- Rango: **2026-01-01 00:00:00** a **2026-05-25 23:00:00**
- Zona horaria: **America/Montevideo (UTC-3)**, sin conversion aplicada (contaminantes y meteorologia ya vienen alineados en hora local).
- Cobertura de PM2.5: **99.74 %** de las horas con al menos un registro.

## Cadencia detectada

Derivada de la moda del delta entre timestamps consecutivos, por mes.

Estable en **120 s** (30 registros/hora) en los 5 meses del periodo. Sin cambios de cadencia.

## Completitud horaria

| Categoria | Criterio | Horas | % |
|---|---|---|---|
| Completas | n_registros >= 75 % de n_esperados | 3451 | 99.17 % |
| Parciales | entre 1 y 74 % | 20 | 0.57 % |
| Vacias | n_registros = 0 | 9 | 0.26 % |

> El dataset **no aplica** este umbral: las horas parciales conservan su promedio y `n_registros` permite filtrarlas en el analisis exploratorio.

## Rachas de horas vacias

Total de rachas: **1** — 9 horas vacias — racha maxima **9 h**.

| Largo de racha | Cantidad |
|---|---|
| 1h | 0 |
| 2h | 0 |
| 3h | 0 |
| 4-6h | 0 |
| 7-24h | 1 |
| >24h | 0 |

## Faltantes por columna

| Columna | Nulos | % |
|---|---|---|
| `fecha_hora` | 0 | 0.00 % |
| `pm25` | 9 | 0.26 % |
| `n_registros` | 0 | 0.00 % |
| `n_esperados` | 0 | 0.00 % |
| `pm25_imputado` | 0 | 0.00 % |
| `temperatura` | 8 | 0.23 % |
| `humedad_relativa` | 8 | 0.23 % |
| `velocidad_viento` | 8 | 0.23 % |
| `visibilidad` | 8 | 0.23 % |
| `viento_u` | 8 | 0.23 % |
| `viento_v` | 8 | 0.23 % |

## Diccionario de datos

| Columna | Unidad | Fuente | Origen |
|---|---|---|---|
| `fecha_hora` | timestamp (America/Montevideo, UTC-3) | `aire.dim_tiempo.fecha_hora` | Inicio de la ventana horaria [HH:00:00, HH:59:59]. Grilla completa y continua. |
| `pm25` | ug/m3 | `aire.fact_medicion_aire.valor_normalizado (dim_contaminante = PM25)` | Promedio simple de los registros validos de la hora. NULL si no hubo ninguno. |
| `n_registros` | conteo | `aire.fact_medicion_aire` | Registros validos usados en el promedio (excluye faltante/atipico/negativo). |
| `n_esperados` | conteo | `derivado: 3600 / cadencia modal del mes` | Registros teoricos de la hora segun la cadencia detectada en los datos. |
| `pm25_imputado` | booleano | `derivado` | True si la fila de la grilla quedo sin ningun registro. No se imputo nada. |
| `temperatura` | grados Celsius | `aire.fact_medicion_climatica.temperatura` | Estacion SUMU (Aeropuerto de Carrasco), fuente IEM ASOS. Horaria. |
| `humedad_relativa` | porcentaje | `aire.fact_medicion_climatica.humedad_relativa` | Estacion SUMU, fuente IEM ASOS. Horaria. |
| `velocidad_viento` | nudos | `aire.fact_medicion_climatica.velocidad_viento` | Estacion SUMU, fuente IEM ASOS. Sin convertir a m/s. |
| `visibilidad` | millas terrestres (SM) | `aire.fact_medicion_climatica.visibilidad` | Estacion SUMU, fuente IEM ASOS. Sin convertir a km. |
| `viento_u` | nudos | `derivado de velocidad_viento y direccion_viento` | Componente este-oeste: -velocidad * sin(radianes(direccion)). Positivo hacia el este. |
| `viento_v` | nudos | `derivado de velocidad_viento y direccion_viento` | Componente norte-sur: -velocidad * cos(radianes(direccion)). Positivo hacia el norte. |

## Criterios de calidad aplicados

Se descartan antes de promediar: `indicador_valor_faltante`, `indicador_valor_atipico` y valores negativos. `calidad_registro` no se usa porque esta 100 % NULL en el warehouse. No se detectaron valores sentinela.

Sin imputacion ni interpolacion. `pm25_imputado` marca las horas de la grilla que quedaron sin ningun registro.
