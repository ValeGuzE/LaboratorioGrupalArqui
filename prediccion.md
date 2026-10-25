# Predicción antes de escalar

Escrita antes de levantar `api-2` y `api-3`. Base medida en la máquina de Cindy:
`python analizar.py resultados/base_cindy.csv` (corrida de 20:26:24 a 20:28:24).
El base1 de la Parte A se midió en otra máquina, por eso se repitió la base aquí.

| Medida | base (medido) | Predicción con 3 réplicas | Razonamiento |
| --- | --- | --- | --- |
| Throughput (lecturas/s) | 33.99 | ~50 (factor ~1.5, no 3) | Traefik reparte en partes iguales (round-robin). api-3 tiene `cpus: 0.25`, la mitad que las otras, así que solo aguanta ~17/s. Cuando api-3 se satura, el sistema queda limitado a ~3 × 17 ≈ 51/s. La CPU total sube 2.5 veces, pero el reparto parejo no la aprovecha. |
| p50 (ms) | 289.0 | ~60 | 2/3 de las solicitudes van a api-1 y api-2, que quedarían a ~50 % de uso y casi sin fila. La mediana cae en ese grupo rápido. |
| p99 (ms) | 696.1 | ~700 o más (casi sin mejora) | El 1 % más lento sale de api-3, que trabaja al 100 % y acumula a la mayoría de los sensores esperando. Por la ley de Little, W ≈ 10/51 ≈ 0.2 s en promedio, lo que deja ~0.5 s en promedio para las de api-3; su cola puede igualar o superar al p99 base. |
