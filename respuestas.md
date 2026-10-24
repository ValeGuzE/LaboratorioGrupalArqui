# Laboratorio 04: respuestas

**Grupo:**  
**Integrantes:**  

Regla: toda respuesta numérica cita su fuente (CSV y comando, consulta PromQL o captura con hora).

## Parte A: estadística del rendimiento

Horas de las corridas:

| Corrida | Inicio | Fin |
| --- | --- | --- |
| base1 |  |  |
| base2 |  |  |
| escalado |  |  |
| colgada |  |  |
| sin_traefik |  |  |

### 6.3 Estadísticas

Fuente: `python analizar.py resultados/base1.csv` y `python analizar.py resultados/base2.csv`

| Medida | base1 | base2 | Diferencia (%) |
| --- | --- | --- | --- |
| Throughput (lecturas/s) |  |  |  |
| Media (ms) |  |  |  |
| Mediana (ms) |  |  |  |
| p90 (ms) |  |  |  |
| p95 (ms) |  |  |  |
| p99 (ms) |  |  |  |
| Máximo (ms) |  |  |  |

### 6.4 Interpretación

1. p95 de k6 contra p95 de `analizar.py`:

2. Forma del histograma de base1, media contra mediana:

3. "El ___ % de las lecturas fue aceptado en ___ ms o menos; 1 de cada ___ tardó más."

4. Ley de Little (L = λ · W):

5. Variabilidad entre base1 y base2:

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
