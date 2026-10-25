# Laboratorio 04: respuestas

**Grupo:** 1  
**Integrantes:** Valeria, Angel, Cindy, Sindy y Luz  

Regla: toda respuesta numérica cita su fuente (CSV y comando, consulta PromQL o captura con hora).

## Parte A: estadística del rendimiento

Horas de las corridas (fuente: primer y último `timestamp` de cada CSV, hora de Guatemala):

| Corrida | Inicio | Fin |
| --- | --- | --- |
| base1 | 15:46:58 | 15:49:05 |
| base2 | 15:54:37 | 15:56:46 |
| escalado |  |  |
| colgada |  |  |
| sin_traefik |  |  |

### 6.3 Estadísticas

Fuente: `python analizar.py resultados/base1.csv` y `python analizar.py resultados/base2.csv`

| Medida | base1 | base2 | Diferencia (%) |
| --- | --- | --- | --- |
| Throughput (lecturas/s) | 21.10 | 24.10 | +14.2 |
| Media (ms) | 474.7 | 409.6 | -13.7 |
| Mediana (ms) | 408.6 | 389.9 | -4.6 |
| p90 (ms) | 614.2 | 586.3 | -4.5 |
| p95 (ms) | 710.2 | 628.3 | -11.5 |
| p99 (ms) | 2395.7 | 2202.8 | -8.1 |
| Máximo (ms) | 2908.8 | 2780.4 | -4.4 |

Diferencia (%) = (base2 - base1) / base1 × 100. Solicitudes: 2701 en base1 y 3133 en base2, 0 fallidas en ambas. Histogramas: `resultados/base1_histograma.png` y `resultados/base2_histograma.png`.

### 6.4 Interpretación

1. p95 de k6 contra p95 de `analizar.py`:

Coinciden. El resumen de k6 de base1 dice `p(95)=710.21ms` y `analizar.py` da 710.2 ms. La diferencia de 0.01 ms es solo redondeo: k6 muestra dos decimales y `analizar.py` uno. Los dos calculan sobre las mismas 2701 solicitudes porque en base1 no hubo fallas. Si hubiera fallas podrían diferir más, porque `http_req_duration` de k6 incluye todas las solicitudes y `analizar.py` solo las exitosas.

2. Forma del histograma de base1, media contra mediana:

No es simétrico. La mayoría de las lecturas está entre 200 y 800 ms, con el pico cerca de 400 ms, y hay una cola larga a la derecha que llega hasta 2908.8 ms: 41 solicitudes tardaron más de 2000 ms. La media (474.7 ms) quedó por encima de la mediana (408.6 ms) porque esas pocas solicitudes muy lentas suman mucho tiempo y jalan el promedio hacia arriba, mientras que la mediana solo depende del valor que queda en el centro. También se ven barras altas separadas cada 100 ms aproximadamente (cerca de 300, 400, 500 y 600 ms). Una posible explicación es el límite `cpus: 0.5` de `api-1`, que le da 50 ms de CPU por cada 100 ms: una solicitud que se queda sin CPU espera al siguiente periodo.

3. "El ___ % de las lecturas fue aceptado en ___ ms o menos; 1 de cada ___ tardó más."

"El **95 %** de las lecturas fue aceptado en **710.2 ms** o menos; 1 de cada **20** tardó más." (fuente: p95 de `python analizar.py resultados/base1.csv`)

4. Ley de Little (L = λ · W):

λ = 21.10 lecturas/s y W = 474.7 ms = 0.4747 s, los dos de `python analizar.py resultados/base1.csv`.

L = 21.10 × 0.4747 = 10.02, prácticamente igual a los 10 VUs de k6.

En teoría L debería salir un poco menor que 10, porque W solo cuenta el tiempo de la solicitud HTTP y cada VU también pasa tiempo fuera de ella: k6 arma el JSON, ejecuta el `check` y arranca la siguiente iteración (y el `sleep(0.5)` cuando una lectura falla, que en base1 no pasó). Ese tiempo del lado del cliente no entra en la latencia. Que salga 10.02 y no un poco menos se explica por el problema de medición de la nota de abajo: algunas latencias salieron infladas.

5. Variabilidad entre base1 y base2:

Entre dos corridas idénticas, el p95 bajó 11.5 % (de 710.2 a 628.3 ms) y el throughput subió 14.2 % (de 21.10 a 24.10 lecturas/s). Entonces, en este ambiente, una "mejora" menor a unos 12 % en el p95 o a unos 14 % en el throughput no está demostrada: puede ser solo ruido entre corridas. La mediana varió mucho menos (4.6 %), así que el centro de la distribución es más estable que las colas.

### Nota sobre la medición


- Las pruebas duran 120 s (`duration: '2m'` en `k6/carga.js`), pero los timestamps del CSV abarcan 128 s en base1 y 130 s en base2. Además faltan segundos completos: en base2 no hay ningún registro en los segundos 22, 23, 55, 86, 87 y 119 de la prueba.
- Una iteración de k6 incluye su solicitud, así que no puede durar menos que ella. Aun así, el máximo de `http_req_duration` es mayor que el de `iteration_duration`: 2908.8 ms contra 2572.8 ms en base1, y 2780.4 ms contra 1024.6 ms en base2.
- Las solicitudes de más de 2 s aparecen en grupos, aproximadamente cada 32 s, justo después de los segundos que faltan.
- Lo más probable es que el reloj de la máquina virtual donde corre Docker se adelante de golpe unos 2 s cada cierto tiempo. Las solicitudes que estaban en curso en ese momento aparecen unos 2 s más lentas, pero las iteraciones no, porque k6 las mide con un reloj que no salta.
- Efecto: el p99 (2395.7 ms) y el máximo de base1 están inflados. Con `iteration_duration`, el p99 de base1 es 887.6 ms. El throughput de `analizar.py` también está subestimado, porque divide entre 128 s y no entre 120 s: con 120 s serían 2701 / 120 = 22.5 lecturas/s.
- Se puede comprobar en la Parte B con el histograma del servidor, que mide con `time.perf_counter()` (un reloj que no salta): en Grafana, el p99 del servidor se mantuvo cerca de 1 s durante casi toda base1.

## Parte B: observabilidad

### 7.1 Las métricas crudas

Fuente: `curl.exe -s http://localhost:8000/metrics`, filtrado con `findstr smartcity_solicitud_duracion_segundos`.

Salen las tres series. `_bucket` es acumulado: cada `le` incluye a los anteriores. En la captura, `le="0.1"` va en 10096 y desde `le="0.5"` hasta `+Inf` todos están en 14409, igual que el `_count`. Entonces 4313 lecturas tardaron más de 100 ms y ninguna pasó de 500 ms. El `_sum` da 1283.92 segundos.

Los `le` son 0.1, 0.5, 1.0, 2.5, 5.0, 10.0 y `+Inf`. En `app.py` los buckets son `(0.1, 0.5, 1.0, 2.5, 5.0, 10.0)`. Coinciden. El `+Inf` lo agrega la librería.

El count (14409) es más alto que las 12280 del CSV porque el proceso no se reinició entre el calentamiento de 20 s y la prueba de 2 min.

### 7.2 Consultas PromQL

Prometheus de esta máquina no tiene la ventana de base1: a las 15:49:05 el contador sigue en 1. La comparación de abajo es de la corrida de las 19:11:33 a las 19:13:33, la que sí quedó scrapeada. Misma prueba (10 VUs, 2 min). CSV: `resultados/obs.csv`.

1. Contador crudo, qué información útil da:

Solo sube: 1960, 4951, 8028, 11108 y 14180, cada 30 s. No dice latencia ni errores. Sirve para ver que el proceso no se reinició. La pendiente se puede leer a ojo: 14180 - 1960 = 12220 en 120 s, unos 102 por segundo. Para eso está `rate()`.

2. Tasa por segundo comparada con el throughput de la Parte A:

`sum(rate(smartcity_solicitud_duracion_segundos_count[2m]))` a las 19:13:33 dio 102.38 lecturas/s. `analizar.py` de esta corrida dio 101.49 y k6 102.32. Quedan juntos. No es el 21.10 de base1: esa corrida fue en la VM, y aquí la misma prueba salió mucho más rápida (p95 del CSV 161.3 ms contra 710.2 ms allá).

3. p95 con ventana de 2 min, evaluado a las 19:13:33: 435.08 ms.

### 7.3 Prometheus contra el cálculo a mano

Fuente del p95 a mano: `python analizar.py resultados/obs.csv` (161.3 ms). k6 dijo `p(95)=161.33ms`. Fuente de Prometheus: `histogram_quantile(0.95, sum by (le) (rate(smartcity_solicitud_duracion_segundos_bucket[2m]))) * 1000` a las 19:13:33.

| Corrida | p95 calculado con `analizar.py` (ms) | p95 estimado por Prometheus (ms) | Diferencia (%) |
| --- | --- | --- | --- |
| base1 | 710.2 | sin datos (contador en 1 a las 15:49) |  |
| 19:11 a 19:13 | 161.3 | 435.1 | +169.7 |

Diferencia = (435.1 - 161.3) / 161.3 × 100.

1. Entre qué límites `le` cae el p95 y qué supone Prometheus dentro del bucket:

435 ms cae entre `le="0.1"` y `le="0.5"`. Prometheus no vio el tiempo de cada solicitud, solo cuántas cayeron en ese rango. Supone que adentro se reparten parejo. Por eso llega a 435 ms aunque en el CSV el máximo fue 204.6 ms: de 200 a 500 el bucket está vacío y la cuenta lineal igual lo recorre.

2. Tiempo que queda fuera de la medición del servidor:

k6 mide desde que manda la lectura hasta que vuelve la respuesta. La API mide adentro del proceso. Afuera queda Traefik, la red y el armado del JSON. En base1 el reloj de la VM saltaba y el p99 del cliente se fue a 2395.7 ms. Aquí no pasó: el máximo de `http_req_duration` es 204.6 ms y el de la iteración 204.7 ms.

3. Cuál p95 reportarían a un gerente y por qué:

El 161.3 ms, el del cliente, porque es lo que esperó el sensor. El 435 de Prometheus no es la latencia real, es el hueco del bucket.

4. Qué cambiarían en `app.py` y qué costo tendría:

Buckets más juntos donde de verdad caen las lecturas, por ejemplo 0.05, 0.1, 0.15, 0.2, 0.3 y 0.5. Cuesta más series en Prometheus: más memoria y un poco más de trabajo en cada scrape.

### 7.4 Lectura crítica del dashboard

Valores a las 19:13:33, con la ventana de 30 s que trae cada panel. No son de base1: de esa hora Prometheus no tiene serie.

| Panel | ¿Qué pregunta responde? | Valor observado durante base1 | ¿La responde correctamente? ¿Cómo lo saben? |
| --- | --- | --- | --- |
| Lecturas por segundo | ¿Cuántas lecturas por segundo acepta? | 102.80 en esta corrida (base1 no está en Prometheus; allá el throughput fue 21.10) | Sí para esta corrida. `analizar.py` dio 101.49 y k6 102.32. La ventana de 30 s no tiene que salir idéntica. |
| Tráfico por réplica | ¿Cómo se reparte? | solo api-1, 102.80 | Sí. api-2 y api-3 no están levantadas. |
| p95 del sistema | ¿Cuál es el p95, en ms? | 432.0 | No. El p95 de las solicitudes es 161.3 ms. Con una sola réplica el `avg()` del panel no cambia el número. El error es el bucket de 0.1 a 0.5. |
| p50 y p99 del sistema | ¿Cuál es el p50 y el p99? | 70.8 y 486.4 | El p50 se acerca a la mediana del CSV (99.1) pero interpola dentro de 0 a 100 ms y no incluye a Traefik. El p99 no: en el CSV es 173.0 y el panel dice 486. |
| Porcentaje de fallas | ¿Qué porcentaje falla? | la consulta no devuelve serie | No del todo. k6 falló 0 de 12280. Si no existe la serie `estado!="201"`, el panel queda vacío y vacío no es 0%. |
| Traefik por código HTTP | ¿Cuántas solicitudes van por código? | HTTP 201 en 102.53 y HTTP 200 en 0 | Sí. Empata con las lecturas aceptadas. |
| Réplicas activas | ¿Cuáles reporta Prometheus? | api-1 = 1, api-2 = 0, api-3 = 0 | Sí. Solo api-1 está arriba. |

Método RED:

| Letra | Paneles |
| --- | --- |
| R (Rate) | Lecturas por segundo, tráfico por réplica y Traefik por código |
| E (Errors) | Porcentaje de fallas, y el código de Traefik si apareciera un 5xx |
| D (Duration) | p50, p95 y p99 |

## Parte C: escalamiento horizontal

### 8.4 Comparación

Fuente: `python analizar.py resultados/escalado.csv`

| Medida | base1 | escalado | Factor (escalado / base1) |
| --- | --- | --- | --- |
| Throughput (lecturas/s) |  |  |  |
| p50 (ms) |  |  |  |
| p95 (ms) |  |  |  |
| p99 (ms) |  |  |  |
| Máximo (ms) |  |  |  |

Cambio en la forma del histograma:

### 8.5 Por réplica

| Réplica | Solicitudes/s | p95 (ms) | CPU en `docker stats` | `cpus` en docker-compose |
| --- | --- | --- | --- | --- |
| api-1 |  |  |  |  |
| api-2 |  |  |  |  |
| api-3 |  |  |  |  |

Explicación:

Cambios en la tabla 7.4 con los datos de escalado:

## Parte D: fallas y SPOF

### 9.1 Réplica colgada

Prueba: `resultados/colgada.csv` (k6, 10 VUs, 2 min). Congelamiento de `api-2`
durante 45 s con `docker compose exec api-2 python congelar.py 45`.
Hora de inicio de k6: [HH:MM:SS]. Hora del comando de congelar:

| Herramienta | ¿Qué mostró sobre `api-2` durante el congelamiento? |
|---|---|
| Panel de Traefik | [Lo que viste. Si no se capturó: "No se capturó durante el congelamiento; solo hay captura con 3 servidores UP antes de la falla (capturas/traefik_3_servidores.png)".] |
| Objetivos de Prometheus | [Lo que muestre `up{job="api"}` en el rango del congelamiento, ej. "api-2 se mantuvo en 1" si así sale]. Captura: capturas/prometheus_up_colgada.png |
| Grafana: réplicas activas | [Lo que veas en el rango 20:21 a 20:26]. Captura: capturas/grafana_replicas_colgada.png |
| Grafana: tráfico por réplica | [Lo que veas: si api-2 cae a 0, cuánto dura]. Captura: capturas/grafana_trafico_colgada.png |
| Grafana: porcentaje de fallas | [Lo que veas. "No data" si no hubo fallas.] Captura: capturas/grafana_fallas_colgada.png |
| `analizar.py` (> 5000 ms, máximo, fallas) | 10 solicitudes mayores a 5000 ms; máximo 45379.1 ms; 0 fallas de 2870 solicitudes. Fuente: `python analizar.py resultados/colgada.csv --umbral 5000` (capturas/analizar_colgada.png) |


### 9.2 Caída del balanceador

Hora de `21:09:28`:  
Hora de `21:10:22`:  
Solicitudes fallidas según `analizar.py`:  
Segundos de la prueba con fallas (primera y última):  
| Panel de Grafana        | Qué mostró durante el corte |
|-------------------------|-----------------------------|
| Lecturas por segundo    | Subió a unas 35 lecturas/s, cayó a 0 alrededor de las 21:10 y volvió a subir a unas 20 lecturas/s antes de terminar la prueba. Hay un hueco sin tráfico entre los dos picos. |
| Tráfico por réplica     | Las tres réplicas (api-1, api-2, api-3) cayeron a 0 al mismo tiempo y volvieron juntas, con dos picos iguales a los del panel anterior. Ninguna réplica falló sola. |
| p95 del sistema         | La línea se interrumpe durante el corte: no hay valor mientras no llegan solicitudes. Antes del corte estaba entre unos 550 y 800 ms. |
| p50 y p99 del sistema   | Las dos líneas también se interrumpen durante el corte. El p99 pasó de cerca de 2 s antes del corte a un pico de ~1.3 s después. |
| Porcentaje de fallas    | **No data** en todo el rango, aunque k6 registró 70 fallas. |
| Traefik por código HTTP | La línea de HTTP 201 muestra el mismo patrón de dos picos, con un hueco durante el corte (la línea se corta y se reanuda unos segundos después). |
| Réplicas activas        | Las tres en 1 durante todo el rango (21:08 a 21:10), sin ningún cambio.


### 9.3 Dominios de falla del laboratorio

| Componente | ¿Qué pasa si falla? | ¿Es SPOF? | ¿Cómo se enterarían? (métrica o alerta) | ¿Cómo se eliminaría en la nube? |
| --- | --- | --- | --- | --- |
| Traefik |  |  |  |  |
| Una réplica de la API |  |  |  |  |
| PostgreSQL |  |  |  |  |
| Prometheus |  |  |  |  |
| Grafana |  |  |  |  |
| Docker Engine |  |  |  |  |
| La laptop completa |  |  |  |  |

Si las tres réplicas corren en la misma laptop, ¿contra qué falla protegen y contra cuál no?

### Dominios de falla del proyecto Smart City

| Componente | ¿Qué pasa si falla? | ¿Es SPOF? | ¿Cómo se enterarían? (métrica o alerta) | ¿Cómo se eliminaría en la nube? |
| --- | --- | --- | --- | --- |
|  |  |  |  |  |
|  |  |  |  |  |
|  |  |  |  |  |
|  |  |  |  |  |
|  |  |  |  |  |

## 10. Preguntas de defensa

### 1. Predicción

Hash del commit de la predicción:

### 2. Throughput al pasar de una a tres réplicas

### 3. p50 y p99 al escalar

Consulta PromQL que lo prueba:

```text

```

### 4. Panel con valor incorrecto en escalado

Consulta corregida:

```text

```

### 5. Significado del p95 de base1

### 6. Caída de Traefik: CSV contra Grafana

Según el CSV, fallaron 70 de 2401 solicitudes, entre el segundo 50 y el 104 (python analizar.py resultados/sin_traefik.csv). En Grafana, el panel de porcentaje de fallas mostró No data (captura: capturas/[archivo].png).

Diferencia: esas solicitudes nunca llegaron a la API, porque Traefik estaba detenido. El panel de fallas lee el histograma de la propia API, así que no las contó. k6 sí las vio porque mide desde el cliente.

Implicación para Smart City: el dashboard solo mide desde dentro de la aplicación y no detecta fallas de lo que está antes de ella. Hace falta monitorear también el balanceador (up{job="traefik"}), alertar cuando los datos desaparecen, medir desde fuera con pruebas como las de k6 y eliminar el SPOF con más de un balanceador.

### 7. Réplica congelada

Solicitudes mayores a 5 s: 10 (máximo 45379 ms, 0 fallas, porque el timeout de k6 es de 60 s y las solicitudes terminaron respondiendo al descongelarse). Fuente: python analizar.py resultados/colgada.csv --umbral 5000.
Prometheus: [activa (up = 1), siempre que tu captura de up{job="api"} lo muestre].
Por qué Traefik la sacó y Prometheus no lo notó: Traefik consulta /salud, que en app.py se retiene mientras la réplica está congelada, así que no respondió en 2 s. Prometheus consulta /metrics, que está exento del congelamiento, y por eso la siguió viendo activa.
Parámetros de Traefik: healthcheck.interval=5s y healthcheck.timeout=2s.
Costo de reducirlos: más tráfico de chequeos y más falsos positivos. Una réplica solo lenta, como api-3 con 0.25 CPU, podría sacarse sin estar rota.

### 8. Réplicas necesarias para (50 × grupo) lecturas/s

### 9. Mejora mínima para considerarla real


## Parte C: Escalamiento horizontal

### Base en la máquina de Cindy (1 réplica)

Fuente: `python analizar.py resultados/base_cindy.csv` (corrida de 20:26:24 a 20:28:24, capturas C1 y C1b).

| Medida | base_cindy |
| --- | --- |
| Throughput (lecturas/s) | 33.99 |
| Media (ms) | 291.7 |
| Mediana (ms) | 289.0 |
| p90 (ms) | 394.6 |
| p95 (ms) | 433.9 |
| p99 (ms) | 696.1 |
| Máximo (ms) | 1132.8 |

Comprobación con la ley de Little: L = 33.99 × 0.2917 ≈ 9.9 ≈ 10 VUs.

### 8.4 Comparación: 1 réplica contra 3 réplicas

Fuentes: `python analizar.py resultados/base_cindy.csv` (20:26:24 – 20:28:24, captura C1) y `python analizar.py resultados/escalado.csv` (20:49:41 – 20:51:41, capturas C7 y C7b). Ambas corridas en la misma máquina, con 10 VUs durante 2 minutos.

| Medida | base_cindy | escalado | Factor (escalado / base) |
| --- | --- | --- | --- |
| Throughput (lecturas/s) | 33.99 | 49.18 | 1.45 |
| p50 (ms) | 289.0 | 23.8 | 0.08 |
| p95 (ms) | 433.9 | 701.3 | 1.62 |
| p99 (ms) | 696.1 | 1096.0 | 1.57 |
| Máximo (ms) | 1132.8 | 2054.4 | 1.81 |

Ley de Little en escalado: L = 49.18 × 0.2015 ≈ 9.9 ≈ 10 VUs.

**¿Cambió la forma de la distribución?** Sí. En base_cindy el histograma tiene una sola montaña alrededor de 290 ms. En escalado la distribución se separa en dos grupos: la mayoría de las lecturas queda muy a la izquierda (decenas de ms) y aparece una cola larga hacia la derecha, entre 600 y 2000 ms. No solo se movió: pasó de un solo grupo a dos grupos con comportamientos distintos. Por eso la mediana bajó mucho mientras el p95, el p99 y el máximo subieron.

### 8.5 Investigación por réplica

Fuentes: consultas de Prometheus en la pestaña Table, evaluadas en 2026-10-25 02:51:41 UTC (20:51:41 hora local, fin de escalado), con ventana `[2m]` para cubrir toda la corrida (capturas C8b y C9b). Gráficas en el tiempo: capturas C8 y C9. CPU: `docker stats --no-stream` durante una carga idéntica de 60 s con las mismas 3 réplicas (captura C5).

Consulta de solicitudes por réplica:
`sum by (instance) (rate(smartcity_solicitud_duracion_segundos_count[2m]))`

Consulta de p95 por réplica:
`histogram_quantile(0.95, sum by (le, instance) (rate(smartcity_solicitud_duracion_segundos_bucket[2m]))) * 1000`

| Réplica | Límite de CPU (`docker-compose.yml`) | Solicitudes/s | p95 (ms) | CPU en `docker stats` | Uso de su límite |
| --- | --- | --- | --- | --- | --- |
| api-1 | 0.5 | 16.82 | 95.3 | 24.96 % | ~50 % |
| api-2 | 0.5 | 16.62 | 98.3 | 22.99 % | ~46 % |
| api-3 | 0.25 | 16.51 | 977.5 | 25.58 % | ~100 % |

**¿Reciben la misma cantidad de solicitudes?** Sí. Las tres reciben ~16.5 lecturas/s, porque Traefik reparte por round-robin con el mismo peso (Weight = 1 en el panel de Traefik, captura C3), sin considerar cuánta CPU tiene cada una.

**¿Responden con la misma latencia?** No. api-3 tiene un p95 unas 10 veces mayor que las otras dos. Tiene la mitad de CPU (`cpus: 0.25`) y recibe el mismo tráfico, así que trabaja al ~100 % de su límite y se le forma fila, mientras api-1 y api-2 usan solo la mitad de su CPU. PostgreSQL usa ~6 % de su CPU, así que no es el cuello de botella.

Nota sobre la precisión: el p95 de api-1 y api-2 (~95 ms) es una estimación de Prometheus. Sus solicitudes caen casi todas en el primer bucket (0 a 0.1 s), y Prometheus supone que se reparten uniformemente dentro de él, por eso da un valor cercano a 100 ms aunque la mediana medida por k6 es de 23.8 ms.

Observación para el panel "p95 del sistema" (pregunta 4): el panel promedia los p95 de cada réplica, (95.3 + 98.3 + 977.5) / 3 ≈ 390 ms, mientras el p95 real de escalado según `analizar.py` es 701.3 ms.

## Preguntas de defensa: Parte C (Cindy)

### 1. Predicción contra resultados

Commit de la predicción: `2cf2cd4 Prediccion antes de escalar` (`git log --oneline`, captura C0), hecho antes de `docker compose up -d api-2 api-3`.

| Medida | base_cindy | Predicción | escalado (medido) | ¿Acertamos? |
| --- | --- | --- | --- | --- |
| Throughput (lecturas/s) | 33.99 | ~50 (factor ~1.5) | 49.18 (factor 1.45) | Sí, casi exacto |
| p50 (ms) | 289.0 | ~60 | 23.8 | La dirección sí; bajó más de lo esperado |
| p99 (ms) | 696.1 | ~700 o más | 1096.0 | La dirección sí; empeoró más de lo esperado |

**En qué acertamos:** que el throughput no se triplicaría y quedaría cerca de 1.5 veces, porque Traefik reparte en partes iguales y api-3 (`cpus: 0.25`) limita al sistema. Que el p50 bajaría porque 2/3 de las lecturas van a réplicas con CPU de sobra. Que el p99 no mejoraría, porque la cola sale de api-3.

**En qué nos equivocamos:** sobreestimamos la latencia de las réplicas rápidas (predijimos ~60 ms y la mediana fue 23.8 ms). Con api-1 y api-2 al ~50 % de su CPU (captura C5) casi no hay fila, y cada lectura tarda poco más que su tiempo de servicio (1 / 34 s ≈ 29 ms con 0.5 CPU). También subestimamos cuánto empeoraría la cola: el p99 subió 57 % y el máximo pasó de 1132.8 a 2054.4 ms.

**Qué no consideramos:** que el p95 también empeoraría (433.9 → 701.3 ms). Como api-3 recibe 1/3 de las solicitudes y es la lenta, más del 5 % de todas las lecturas son lentas, así que el p95 ya cae dentro del grupo de api-3. Tampoco consideramos que el panel "p95 del sistema" de Grafana ocultaría este efecto al promediar los p95 de las réplicas (≈ 390 ms contra 701.3 ms reales).

### 2. ¿Por qué se triplicó el throughput?

No se triplicó. Según `analizar.py`, pasó de 33.99 a 49.18 lecturas/s: un factor de **1.45**, no 3.

Evidencia:
1. **Las tres réplicas reciben lo mismo.** `sum by (instance) (rate(smartcity_solicitud_duracion_segundos_count[2m]))` evaluada al final de escalado (captura C8b) da 16.82, 16.62 y 16.51 lecturas/s. Traefik usa round-robin con peso 1 para cada servidor (captura C3).
2. **api-3 tiene la mitad de CPU.** En `docker-compose.yml`, api-1 y api-2 tienen `cpus: 0.5` y api-3 tiene `cpus: 0.25`. El trabajo de cada lectura (`firmar`, 8400 iteraciones de SHA-256 para el grupo 1) es casi todo CPU.
3. **api-3 está saturada y las otras no.** `docker stats` durante la carga (captura C5): api-3 usa 25.58 % (~100 % de su límite), api-1 24.96 % y api-2 22.99 % (~50 % del suyo). PostgreSQL usa 5.82 %, así que no es el cuello de botella.

Con 1 réplica de 0.5 CPU el sistema acepta ~34/s, así que api-3 con 0.25 CPU aguanta ~17/s. Como el balanceador le manda 1/3 del tráfico a cada réplica, el sistema llega a su límite cuando api-3 se satura: 3 × ~17 ≈ 51/s, muy cerca de los 49.18 medidos. La CPU total subió 2.5 veces (de 0.5 a 1.25), pero el reparto parejo deja sin usar la mitad de la capacidad de api-1 y api-2.

Por la ley de Little (L = 10 VUs fijo): L = 49.18 × 0.2015 ≈ 9.9. Si λ no se triplicó, W tampoco se dividió entre 3: pasó de 291.7 a 201.5 ms.

Para acercarse a 3 veces habría que darle a api-3 la misma CPU (0.5), o usar pesos en el balanceador proporcionales a la CPU (2:2:1), que aprovecharían hasta ~2.5 veces.

### 3. ¿Qué pasó con el p50 y el p99?

Fuente: `analizar.py` sobre base_cindy.csv y escalado.csv (capturas C1 y C7).
- **p50:** 289.0 → 23.8 ms (mejoró ~12 veces).
- **p99:** 696.1 → 1096.0 ms (**empeoró** 57 %). El p95 también empeoró: 433.9 → 701.3 ms.

**Réplica responsable: api-3.** Consulta PromQL, evaluada en 2026-10-25 02:51:41 UTC (fin de escalado, captura C9b):

`histogram_quantile(0.95, sum by (le, instance) (rate(smartcity_solicitud_duracion_segundos_bucket[2m]))) * 1000`

| Réplica | p95 (ms) |
| --- | --- |
| api-1 | 95.3 |
| api-2 | 98.3 |
| api-3 | 977.5 |

La gráfica en el tiempo (captura C9) muestra lo mismo durante toda la prueba: api-3 se mantiene cerca de 1 s mientras las otras dos quedan cerca de 100 ms.

**Causa:** api-3 tiene `cpus: 0.25` y recibe la misma cantidad de lecturas que las otras (~16.5/s, captura C8b). Trabaja al ~100 % de su límite (25.58 % en `docker stats`, captura C5), así que las lecturas le hacen fila. Como 1/3 de todas las solicitudes pasa por api-3, la distribución se partió en dos grupos (histograma de escalado): la mediana cae en el grupo rápido (api-1 y api-2) y por eso mejoró, mientras el p95 y el p99 caen en el grupo lento (api-3) y por eso empeoraron. Agregar una réplica más lenta que las demás, con reparto parejo, mejora la experiencia típica pero empeora la de los usuarios con peor suerte.
