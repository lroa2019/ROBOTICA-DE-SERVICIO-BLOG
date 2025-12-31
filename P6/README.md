# Localización Visual mediante Marcadores AprilTag

## 1. Introducción teórica

El objetivo de esta práctica es desarrollar un sistema de autolocalización para un robot móvil utilizando marcadores visuales AprilTag como referencias en el entorno.
El mayor problema de la localización es poder estimar en todo momento la pose del robot, la cual esta definida por su posición en el plano (x, y) y su orientación (yaw).

La localización basada únicamente en odometría presenta el siguiente problema: los errores se acumulan con el tiempo debido al ruido de los sensores y al deslizamiento de las ruedas.
Para reducir esta deriva, en esta práctica empleamos información visual, detectando balizas AprilTag cuya posición en el mundo es conocida previamente.

La solución que he propuesto combina ambas fuentes de información mediante un Filtro de Kalman Extendido (EKF), que permite fusionar:

* La predicción obtenida a partir de la odometría del robot.
* La corrección basada en observaciones visuales de los marcadores.

Además, el robot se desplaza por el entorno utilizando un control reactivo basado en el sensor láser, esto le permite explorar el escenario, evitar obstáculos y observar los marcadores desde distintas posiciones.
El funcionamiento total del sistema se visualiza en tiempo real con la interfaz gráfica del propio simulador.

## 2. Fases del desarrollo

En esta práctica he dividido el desarrollo en varias fases, que engloban desde la percepción visual hasta la estimación de la pose y el control del movimiento del robot.

### 2.1 Detección y procesamiento de marcadores AprilTag

En primer lugar, el sistema obtiene la imagen de la cámara del robot y la convierte a escala de grises, ya que la detección de AprilTags no requiere información de color.
A continuación, se utiliza la librería `pyapriltags` para detectar los marcadores visibles en la imagen y estimar su pose relativa respecto a la cámara.

A partir de esta estimación, se calculan:

* La distancia entre el robot y el marcador.
* El ángulo relativo (bearing) del marcador respecto al eje frontal del robot.

Cada marcador detectado se asocia con su posición conocida en el mundo, definida previamente en un archivo de configuración.
De esta forma, cada baliza proporciona una observación válida para el sistema de localización.

### 2.2 Modelo de cámara y sistema de referencia

Para poder estimar correctamente la pose relativa de los marcadores, es necesario que definamos un modelo de cámara.
En esta práctica utilizamos una aproximación de los parámetros intrínsecos, calculados a partir del tamaño de la imagen proporcionada por el simulador.

Dado que el movimiento del robot es planar, la información tridimensional se simplifica a una representación en 2D, considerando únicamente la posición (x, y) y la orientación (yaw).
Esta simplificación es suficiente para el tipo de entorno y movimiento considerado en la práctica.

### 2.3 Inicialización y predicción mediante odometría

El sistema de localización se basa en un Filtro de Kalman Extendido, cuyo estado está definido como:

x = [x, y, yaw]

En la primera iteración, el filtro se inicializa utilizando la odometría del robot, asumiendo una incertidumbre inicial moderada tanto en la posición como en la orientación.

En cada ciclo de ejecución, el EKF realiza una fase de predicción, en la que:

* Se calcula el incremento de odometría desde la última iteración.
* Se actualiza la estimación de la pose del robot.
* Se incrementa la incertidumbre mediante la matriz de ruido del proceso.

La odometría se utiliza únicamente de forma incremental, tal y como se indica en el enunciado de la práctica, evitando así su uso como estimación absoluta de la posición.

### 2.4 Corrección con observaciones visuales (EKF)

Cuando el robot detecta uno o varios marcadores AprilTag, el EKF realiza la fase de corrección.
Para cada marcador observado, se calcula la observación esperada a partir de la pose predicha del robot y se compara con la medida real obtenida desde la cámara.

Durante esta fase se calculan:

* La innovación entre la medida real y la esperada.
* La matriz Jacobiana del modelo de observación.
* La ganancia de Kalman, utilizada para actualizar el estado.

Además, se aplica un criterio de rechazo estadístico (gating) basado en la distancia de Mahalanobis, con el fin de descartar observaciones inconsistentes que podrían degradar la estimación.

Gracias a este proceso, el sistema es capaz de corregir la deriva acumulada por la odometría y mantener una estimación estable de la pose del robot.

### 2.5 Control reactivo y exploración del entorno

Mientras se ejecuta la localización, el robot se desplaza por el entorno mediante un control reactivo basado en el sensor láser.
El comportamiento del robot se divide principalmente en dos estados:

* Estado GO: el robot avanza hacia delante a velocidad constante cuando no hay obstáculos cercanos.
* Estado TURN: cuando se detecta un obstáculo frontal, el robot gira sobre sí mismo hasta encontrar una zona libre.

Para mejorar la estabilidad del sistema, las lecturas del láser se procesan mediante un promediado de rayos en distintas zonas (frontal, izquierda y derecha), reduciendo el efecto del ruido y de lecturas erróneas.
El sentido del giro se decide comparando el espacio libre disponible a ambos lados del robot.

Este comportamiento permite que el robot explore el entorno de forma segura y aumente la probabilidad de detectar distintos marcadores.

### 2.6 Visualización de la ejecución

Durante toda la ejecución del sistema se nos muestra en tiempo real:

* La imagen capturada por la cámara, con los marcadores visibles.
* La pose estimada del robot, actualizada continuamente por el EKF.

Esta visualización facilita la comprobación del correcto funcionamiento del sistema y permite que observemos de forma clara cómo las observaciones visuales influyen en la estimación de la pose.

## 3. Resultados visuales de la ejecución

En este apartado incluimos capturas de pantalla y un GIF representativo del funcionamiento completo del sistema.

<p align="center">
  <img src="https://github.com/user-attachments/assets/dbf31637-9e65-4de5-80a2-708ffa952653" width="400"/>
</p>
<p align="center"><b>Figura X: Robot detectando un marcador AprilTag.</b></p>

<p align="center">
  <img src="https://github.com/user-attachments/assets/e5b44d81-9e2e-4d2b-b727-50652b22d125" width="400"/>
</p>
<p align="center"><b>Figura Y: Visualización de la pose estimada del robot.</b></p>

<p align="center">
  <img src="https://github.com/user-attachments/assets/64dddff5-861f-4b95-8ac8-15b6790d6c3e" width="600"/>
</p>
<p align="center"><b>Figura Z: Proceso completo de exploración y localización del robot.</b></p>

## 4. Problemas durante el desarrollo

Durante la realización de la práctica se han detectado varios inconvenientes relacionados principalmente con el rendimiento del simulador y la latencia de sensores, que afectaron al movimiento del robot:

1. Lag general durante la ejecución (bajo rendimiento del simulador).
   En algunos momentos la simulación presentaba un rendimiento irregular, con bajadas de FPS y “tirones”.
   Esto afectaba directamente al control del robot, ya que las órdenes de velocidad se aplicaban con retraso o de forma poco continua.
   Como consecuencia, el robot podría realizar movimientos menos suaves y más difíciles de predecir.

2. Lecturas del láser con latencia (“medidas tardías”).
   He observado que, en ocasiones, las lecturas del láser llegaban tarde respecto al movimiento real del robot.
   Esto provocaba que el robot avanzase demasiado antes de detectar correctamente un obstáculo.
   En esos casos el robot podía llegar a “comerse” parcialmente una pared (acercarse demasiado o incluso solaparse visualmente en el simulador) y solo después reaccionaba.

3. Giros bruscos por detección tardía de obstáculos.
   Debido a la latencia ya comentada, el robot a veces pasaba de una situación “aparentemente libre” a detectar de golpe una distancia frontal muy pequeña.
   Al activarse el modo de giro, el robot comenzaba a girar de forma brusca cerca del obstáculo, lo que empeoraba la estabilidad del movimiento y podía hacer que se quedase girando durante más tiempo del necesario.

Posibles mejoras sugeridas:
Para mitigar estos problemas, una opción sería suavizar la información del láser (por ejemplo mediante promedios en una ventana de rayos o filtrado), y reducir la velocidad lineal cuando el robot esté cerca de obstáculos. También ayudaría disminuir la carga de visualización o prints para reducir el lag y conseguir ciclos de control más estables.






