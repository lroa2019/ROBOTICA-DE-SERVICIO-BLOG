# Práctica P5 – Laser Mapping

## 1. Introducción

La práctica P5 plantea el problema fundamental del mapeo autónomo de un entorno industrial desconocido mediante el uso de un sensor láser 2D y odometría perfecta. El objetivo es que el robot:

1. Explore zonas desconocidas del mapa.
2. Actualice un mapa probabilístico de ocupación del almacén.
3. Distinga entre celdas libres, ocupadas y no observadas.
4. Mantenga un proceso continuo de mapeo incluso cuando la odometría deja de ser perfecta (odometría2 y odometría3).

El entorno de trabajo es un almacén con estanterías, pasillos, zonas abiertas y obstáculos dispersos. El robot cuenta con:

* Un sensor láser de 360º.
* Localización perfecta en la primera fase (mapeo base).
* Odometría degradada en fases posteriores (para observar la degradación del mapa).

La práctica exige diseñar:

1. Un algoritmo de exploración capaz de encontrar regiones desconocidas.
2. Un generador de mapa probabilístico basado exclusivamente en lecturas de láser.
3. Una estrategia de actualización del mapa capaz de diferenciar:

   * Zonas libres (FREE)
   * Obstáculos (OCC)
   * Regiones no observadas (UNK)

## 2. Mapa de Ocupación Probabilístico

### 2.1 Representación interna del mapa

El mapa se implementa como una matriz de tamaño:

```
1500 (ancho) × 970 (alto)
```

Cada celda puede tomar tres valores:

* `FREE = 255` → espacio libre confirmado
* `OCC = 0` → celda ocupada
* `UNK = 127` → desconocido

Esto constituye un mapa probabilístico simplificado, donde cada celda representa la probabilidad máxima de ocupación según observaciones acumuladas.

### 2.2 Filosofía de actualización

El mapa se actualiza siguiendo estos principios:

* El haz del láser se interpreta como información estructurada:

  * Todo rayo es principalmente una confirmación de celdas libres a lo largo de la trayectoria del haz.
  * El punto final del rayo puede ser:

    * Un obstáculo real → marca de celda ocupada.
    * Una zona fuera del rango → se marca el tramo como libre pero no obstaculizado.

* Este enfoque es equivalente a un modelo de celdas ocupadas sin filtros probabilísticos bayesianos.

### 2.3 Conversión mundo → mapa

Las funciones `poseToMap` convierten coordenadas métricas reales a índices en la matriz del mapa.

El robot marca:

* Su posición como libre.
* Los rayos como libres.
* Los impactos como ocupados.

Esto permite construir una representación cada vez más precisa del entorno.

## 3. Modelado del Sensor Láser

### 3.1 Características del láser

El láser posee:

* 360 lecturas.
* Alcance máximo configurable (`MAX_RANGE = 4.0`).
* Valores especiales:

  * `d = inf` → sin obstáculo.
  * `d > MAX_RANGE` → también considerado ausencia de obstáculo.

### 3.2 Razonamiento geométrico

Para cada rayo:

```
fx = x + r * cos(angle)
fy = y + r * sin(angle)
```

donde:

* `(x, y, yaw)` son las coordenadas del robot.
* `r` es la distancia recortada al rango máximo.
* `angle` se calcula como `yaw + i * angle_step`.

Cada rayo se traza con un algoritmo Bresenham modificado, que permite:

* Marcar cada celda atravesada como libre.
* Marcar el final del rayo como ocupado si hubo detección real.

Este proceso es fundamental para obtener un mapa coherente.

## 4. Trazado de Rayos: Implementación de Bresenham

La función `marcar_rayo` utiliza una variante discreta del algoritmo de Bresenham, que itera sobre los píxeles que atraviesa el rayo desde el robot hasta el punto final.

### 4.1 Razones para usar Bresenham

* Es eficiente: O(n) respecto a la longitud del rayo.
* No requiere floating point salvo para el cálculo inicial del rayo.
* Permite un control claro de colisiones entre celdas.

### 4.2 Efectos en el mapa

* Cada celda intermedia se marca como `FREE` salvo que ya fuera `OCC`.
* La celda final se marca como `OCC` únicamente si el sensor detectó un obstáculo real.

Una ventaja adicional de Bresenham en este contexto es que evita problemas de interpolación que generarían incertidumbre innecesaria en el mapa.

## 5. Exploración Autónoma Basada en Láser

La práctica requiere que el robot explore zonas no visitadas sin un objetivo fijo. Para ello se desarrolla una estrategia compuesta por dos mecanismos:

### 5.1 Movimiento reactivo por obstáculos

Antes de explorar, el robot debe garantizar seguridad:

* Si hay un obstáculo frontal a menos de `SAFE_DIST`, el robot:

  * Reduce velocidad.
  * Selecciona el lado (izquierda o derecha) con mayor espacio.
  * Gira para evitar colisión.

Este comportamiento garantiza supervivencia en entornos desconocidos.

### 5.2 Búsqueda activa de zonas desconocidas

La clave de la exploración está en la función:

```
find_best_unknown_heading()
```

que examina 36 direcciones angulares alrededor del robot y calcula:

* Si en esa dirección existen celdas UNK accesibles.
* A qué distancia aparece la primera celda desconocida.
* Si el camino requerido hasta ella es libre.

El robot elige la dirección cuya primera celda desconocida esté más cerca, de forma que maximiza el ritmo de descubrimiento del mapa.

Si no se encuentra ninguna dirección que conduzca a regiones desconocidas, el robot:

* Avanza recto.
* O sigue explorando al azar.

Este patrón de movimiento es un precursor simplificado de algoritmos clásicos como Frontier-Based Exploration.

## 6. Control de Movimiento durante la Exploración

### 6.1 Control angular

Se usa un regulador proporcional:

```
w_cmd = K_ANG * rel_angle
```

limitado por `ANGULAR_W`.

### 6.2 Control lineal

La velocidad depende del ángulo relativo:

* Si el robot está alineado con la celda desconocida → avanza rápido.
* Si debe girar mucho → reduce velocidad para mayor precisión.

La velocidad mínima se fuerza a `0.2 m/s` para evitar bloqueos.

## 7. Integración Completa del Sistema

El bucle principal ejecuta:

### 7.1 Lectura del láser y la pose

Con localización perfecta en esta primera fase.

### 7.2 Actualización del mapa

Mediante `update_map`, llamando internamente a `marcar_rayo`.

### 7.3 Estrategia de exploración

Con `random_exploration_controller`.

### 7.4 Visualización

`WebGUI.setUserMap` actualiza la interfaz con el mapa actual.

La ejecución periódica está regulada por `Frequency.tick(10)`.

## 8. Resultados Experimentales

### 8.1 Mapa inicial (todo desconocido)

<div align="center">
  <img src="https://github.com/user-attachments/assets/bdcce1c4-27ff-466e-a5f2-7810323b3896" width="500">
</div>  
<p align="center"><b>Figura 1: Estado inicial del mapa (UNK).</b></p>

### 8.2 Progresión del mapeo

<div align="center">
  <img src="https://github.com/user-attachments/assets/b7397590-a684-41c9-bf66-d9cad0a823bb" width="500">
</div>  
<p align="center"><b>Figura 2: Proceso de construcción del mapa.</b></p>

### 8.3 Mapa final tras exploración completa

<div align="center">
  <img src="https://github.com/user-attachments/assets/cd97c030-9591-4cdc-aab5-2c41313cc0d1" width="500">
</div>  
<p align="center"><b>Figura 3: Mapa final obtenido por el robot.</b></p>

### 8.4 Video de la ejecución completa (odometría2 y odometría3)

<div align="center">
  <img src="https://github.com/user-attachments/assets/6f519c18-133a-46bd-93ae-07c3635bcee1" width="500">
</div>  
<p align="center"><b>Figura 4: Demostración de la ejecución.</b></p>

## 9. Problemas Detectados

### 9.1 Irregularidades en el mapa derivadas del ruido del sensor láser

El sensor láser empleado presenta leves imprecisiones en la estimación de distancias, incluso en condiciones estáticas. Como consecuencia, al representar los resultados del trazado de rayos en el mapa de ocupación, aparecen bordes irregulares y patrones residuales que se manifiestan como finas prolongaciones lineales alrededor de paredes y objetos.
Estas irregularidades generan un mapa con contornos menos definidos y dificultan la interpretación precisa de la geometría del entorno. Además, dichas perturbaciones se ven amplificadas por el proceso discreto de Bresenham, que es especialmente sensible a pequeñas variaciones angulares y métricas en los impactos del láser.

### 9.2 Aproximaciones excesivas a obstáculos debido a imprecisiones en el sensor láser

Durante la exploración se observó que, debido a pequeñas imprecisiones en las lecturas del sensor láser, el robot tiende a aproximarse a los obstáculos más de lo previsto por el margen de seguridad establecido. Aunque el controlador reactivo implementa una distancia mínima de seguridad, la variabilidad en la medición provoca que esta distancia efectiva se reduzca en la práctica.
Si bien no se registraron colisiones, el comportamiento resultante muestra que el robot opera en rangos muy cercanos a paredes y objetos, reduciendo el margen operativo y comprometiendo la estabilidad de la navegación. Este fenómeno es especialmente evidente en zonas estrechas, donde la combinación de ruido sensorial y discretización del mapa hace que el robot interprete incorrectamente la distancia real al obstáculo más próximo, ajustando su movimiento con un margen inferior al deseado.

### 9.3 Degradación del mapa con odometría degradada

En los escenarios con odometría imperfecta (odometría2 y odometría3), la acumulación progresiva de error provoca deformaciones globales en el mapa: paredes duplicadas, desplazamientos relativos y distorsiones estructurales. Este efecto es especialmente notable en pasillos largos y superficies amplias, donde la incoherencia angular y posicional se acumula rápidamente.

### 9.4 Limitaciones de la estrategia de exploración reactiva

La política de exploración basada únicamente en la detección de sectores desconocidos y la evasión reactiva de obstáculos puede derivar en situaciones donde determinadas zonas del espacio permanecen sin explorar. Esto ocurre especialmente en áreas con geometría compleja o donde se requieren maniobras más deliberadas que un enfoque puramente reactivo no contempla.
