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

### 7.2 Consultas PromQL

1. Contador crudo, qué información útil da:

2. Tasa por segundo comparada con el throughput de la Parte A:

### 7.3 Prometheus contra el cálculo a mano

| Corrida | p95 calculado con `analizar.py` (ms) | p95 estimado por Prometheus (ms) | Diferencia (%) |
| --- | --- | --- | --- |
| base1 |  |  |  |

1. Entre qué límites `le` cae el p95 y qué supone Prometheus dentro del bucket:

2. Tiempo que queda fuera de la medición del servidor:

3. Cuál p95 reportarían a un gerente y por qué:

4. Qué cambiarían en `app.py` y qué costo tendría:

### 7.4 Lectura crítica del dashboard

| Panel | ¿Qué pregunta responde? | Valor observado durante base1 | ¿La responde correctamente? ¿Cómo lo saben? |
| --- | --- | --- | --- |
| Lecturas por segundo |  |  |  |
| Tráfico por réplica |  |  |  |
| p95 del sistema |  |  |  |
| p50 y p99 del sistema |  |  |  |
| Porcentaje de fallas |  |  |  |
| Traefik por código HTTP |  |  |  |
| Réplicas activas |  |  |  |

Método RED:

| Letra | Paneles |
| --- | --- |
| R (Rate) |  |
| E (Errors) |  |
| D (Duration) |  |

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

| Herramienta | ¿Qué mostró sobre `api-2` durante el congelamiento? |
| --- | --- |
| Panel de Traefik |  |
| Objetivos de Prometheus |  |
| Grafana: réplicas activas |  |
| Grafana: tráfico por réplica |  |
| Grafana: porcentaje de fallas |  |
| `analizar.py` (solicitudes mayores a 5000 ms, máximo, fallas) |  |

### 9.2 Caída del balanceador

Hora de `stop`:  
Hora de `start`:  
Solicitudes fallidas según `analizar.py`:  
Segundos de la prueba con fallas (primera y última):  

| Panel de Grafana | Qué mostró durante el corte |
| --- | --- |
| Lecturas por segundo |  |
| Tráfico por réplica |  |
| p95 del sistema |  |
| p50 y p99 del sistema |  |
| Porcentaje de fallas |  |
| Traefik por código HTTP |  |
| Réplicas activas |  |

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

### 7. Réplica congelada

### 8. Réplicas necesarias para (50 × grupo) lecturas/s

### 9. Mejora mínima para considerarla real
