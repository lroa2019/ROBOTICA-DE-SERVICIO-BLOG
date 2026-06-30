# Práctica P6 - Localización Visual mediante Marcadores AprilTag

## 1. Introducción teórica

El objetivo es desarrollar un sistema de autolocalización para un robot móvil utilizando marcadores visuales AprilTag como referencias en el entorno. El problema central de la localización es poder estimar en todo momento la pose del robot, definida por su posición en el plano (x, y) y su orientación (yaw).

La localización basada únicamente en odometría presenta un problema bien conocido: los errores se acumulan con el tiempo debido al ruido de los sensores y al deslizamiento de las ruedas, provocando una deriva creciente. Para corregir esta deriva, se usa información visual: el robot detecta balizas AprilTag cuya posición en el mundo es conocida de antemano (almacenada en un mapa de balizas), y a partir de cada detección estima su propia pose absoluta mediante el algoritmo Perspective-n-Point (PnP).

La solución que he implementada combina ambas fuentes de información de la siguiente manera:

* Cuando el robot detecta una o varias balizas, su posición estimada se calcula directamente a partir de la pose relativa cámara-baliza obtenida por PnP, compuesta con la posición absoluta conocida de esa baliza en el mapa.
* Cuando el robot no detecta ninguna baliza, su posición se propaga aplicando, sobre la última pose conocida, el incremento de odometría acumulado desde la última detección. La odometría se utiliza así de forma exclusivamente incremental, nunca como medida de posición absoluta.

Además, el robot se desplaza por el entorno mediante un control reactivo basado en el sensor láser, lo que le permite explorar el escenario, evitar obstáculos y observar los marcadores desde distintas posiciones. El funcionamiento completo del sistema se visualiza en tiempo real mediante la interfaz gráfica del propio simulador.

## 2. Fases del desarrollo

### 2.1 Detección y procesamiento de marcadores AprilTag

El sistema obtiene la imagen de la cámara del robot y la convierte a escala de grises, ya que la detección de AprilTags no requiere información de color. A continuación, se utiliza la librería pyapriltags para detectar los marcadores visibles en la imagen, obteniendo las coordenadas en píxeles de las cuatro esquinas de cada marcador detectado.

Cada marcador detectado se identifica por su identificador (tag_id) y se contrasta con el mapa de balizas, un archivo de configuración que almacena, para cada baliza conocida, su posición y orientación absolutas en el mundo. Solo se consideran válidas las detecciones correspondientes a balizas presentes en ese mapa.

### 2.2 Modelo de cámara

Para poder aplicar PnP es necesario disponer de un modelo de cámara. Aquí uso una aproximación de los parámetros intrínsecos, calculada directamente a partir del tamaño de la imagen proporcionada por el simulador: la distancia focal se aproxima al ancho de la imagen en píxeles, y el punto principal se sitúa en el centro de la imagen. No se considera distorsión de la lente.

Dado que el movimiento del robot es planar, la pose final se reduce a una representación en 2D: posición (x, y) y orientación (yaw), descartando la componente de altura.

### 2.3 Estimación de pose mediante PnP y composición de transformaciones

Para cada marcador detectado se conocen, en su propio sistema de referencia local, los cuatro puntos 3D de sus esquinas (el marcador se modela como un cuadrado de lado conocido). Comparando esos puntos 3D con sus proyecciones 2D detectadas en la imagen, el algoritmo PnP (mediante el método iterativo de OpenCV) calcula la transformación que lleva el sistema de referencia del marcador al sistema de referencia de la cámara, expresada como una rotación y una traslación (matriz RT cámara-respecto-a-marcador).

A partir de esa transformación se obtiene, por inversión, la transformación inversa: la posición y orientación de la cámara respecto al marcador. Esta transformación se reexpresa en los ejes de referencia habituales del mundo (eje x hacia delante, eje y lateral), ya que los ejes nativos de una cámara siguen un convenio distinto (profundidad en z, lateral en x, vertical en y).

Finalmente, esta transformación cámara-respecto-a-marcador se compone con la transformación marcador-respecto-al-mundo, es decir, la posición y orientación absolutas de esa baliza tal y como están almacenadas en el mapa. El resultado de esta composición de transformaciones (RT del marcador en el mundo combinada con RT de la cámara respecto al marcador) es directamente la pose absoluta del robot en el mundo: su posición (x, y) y su orientación (yaw).

### 2.4 Selección de marcador y propagación odométrica

Cuando el robot detecta varios marcadores simultáneamente, el sistema se queda únicamente con la estimación obtenida a partir del marcador que ocupa mayor área en la imagen (el más cercano al robot), descartando el resto.

Cuando el robot no detecta ningún marcador, su pose no puede recalcularse mediante PnP. En ese caso, se mantiene como referencia la última pose absoluta conocida (la última vez que se vio una baliza), junto con la lectura de odometría que había en ese instante. La pose actual se obtiene aplicando a esa referencia el incremento de posición y orientación medido por la odometría desde entonces, corrigiendo la orientación de ese incremento para expresarlo en el sistema de referencia de la pose anclada (y no en el sistema de referencia interno, potencialmente desviado, de la propia odometría). De este modo, la odometría se emplea exclusivamente para medir desplazamientos relativos entre observaciones visuales, nunca como estimación absoluta de la posición.

Como medida adicional de robustez, antes de aceptar una nueva estimación PnP como ancla válida se comprueba que su salto respecto a la última pose conocida no sea excesivo (umbral de distancia). Esto permite descartar detecciones puntuales claramente espurias —por ejemplo, debidas a una mala asociación de esquinas— sin necesidad de un filtro probabilístico completo.

### 2.5 Control reactivo y exploración del entorno

Mientras se ejecuta la localización, el robot se desplaza por el entorno mediante un autómata reactivo de dos estados basado en el sensor láser:

* Estado GO: el robot avanza hacia delante a velocidad constante mientras no detecta obstáculos cercanos en su sector frontal.
* Estado TURN: en cuanto la distancia frontal cae por debajo de un umbral de seguridad, el robot detiene su avance y gira sobre sí mismo, eligiendo el sentido de giro hacia el lado (izquierda o derecha) con mayor espacio libre disponible, hasta recuperar margen suficiente por delante.

Para mejorar la estabilidad de estas decisiones, las lecturas del láser no se usan de forma puntual, sino promediadas en pequeñas ventanas angulares en las zonas frontal, izquierda y derecha, reduciendo el efecto de lecturas ruidosas o erróneas aisladas.

Este comportamiento de choca-gira permite que el robot recorra el entorno de forma seudoaleatoria y segura, aumentando la probabilidad de observar distintas balizas desde múltiples posiciones.

### 2.6 Visualización de la ejecución

Durante toda la ejecución se muestra en tiempo real:

* La imagen capturada por la cámara, con los marcadores visibles.
* La pose estimada del robot, actualizada continuamente según el procedimiento que he descrito anteriormente.

Esta visualización permite comprobar el correcto funcionamiento del sistema y observar de forma directa cómo cada observación visual corrige la pose estimada del robot.

## 3. Resultados visuales de la ejecución

En este apartado incluimos capturas de pantalla y un vídeo representativo del funcionamiento completo del sistema.

<p align="center">
  <img src="https://github.com/user-attachments/assets/18e640ca-53e7-46da-8490-6e000af07a76" width="600"/>
</p>
<p align="center"><b>Figura 1: Robot detectando un marcador AprilTag.</b></p>

<p align="center">
  <img src="https://github.com/user-attachments/assets/e5b44d81-9e2e-4d2b-b727-50652b22d125" width="600"/>
</p>
<p align="center"><b>Figura 2: Visualización de la pose estimada del robot.</b></p>

[Ejecución completa video](https://github.com/user-attachments/assets/5f779dec-e489-49ee-a1c3-b21a0ccb321e)
<p align="center"><b>Figura 3: Proceso completo de exploración y localización del robot.</b></p>

## 4. Problemas durante el desarrollo

Durante la realización de la práctica se detectaron varios inconvenientes relacionados principalmente con el rendimiento del simulador y la latencia de los sensores, que afectaron al movimiento del robot:

1. **Lag general durante la ejecución.** En algunos momentos la simulación presentaba un rendimiento irregular, con bajadas de FPS y "tirones". Esto afectaba directamente al control del robot, ya que las órdenes de velocidad se aplicaban con retraso o de forma poco continua, provocando movimientos menos suaves y más difíciles de predecir.

2. **Lecturas del láser con latencia.** En ocasiones, las lecturas del láser llegaban tarde respecto al movimiento real del robot, lo que provocaba que avanzase demasiado antes de detectar correctamente un obstáculo, llegando a aproximarse en exceso a una pared antes de reaccionar.

3. **Giros bruscos por detección tardía de obstáculos.** Debido a la latencia comentada, el robot pasaba en ocasiones de una situación aparentemente libre a detectar de golpe una distancia frontal muy pequeña, lo que provocaba giros más bruscos de lo deseable cerca del obstáculo y, en algunos casos, tiempos de giro mayores de lo necesario.

**Mejoras propuestas:** suavizar aún más la información del láser (por ejemplo, ampliando la ventana de promediado o aplicando un filtrado temporal), reducir la velocidad lineal cuando el robot se aproxima a obstáculos, y disminuir la carga de visualización o de mensajes de depuración para aligerar el ciclo de control y conseguir iteraciones más estables.




