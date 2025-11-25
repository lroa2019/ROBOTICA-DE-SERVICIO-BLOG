# Práctica 4 - Planificación de Trayectorias y Manipulación de Estanterías en un Almacén con OMPL

## 1. Introducción Teórica

El objetivo de esta práctica es programar un sistema de logística autónoma capaz de:

1. Navegar por un almacén con obstáculos fijos.
2. Localizar estanterías en posiciones determinadas.
3. Recoger cada estantería, transportarla y depositarla en su destino.
4. Repetir el proceso para todas las estanterías.
5. Ejecutar estas tareas utilizando **dos robots distintos**:

   * Un robot **holonómico** (desplazamiento omnidireccional).
   * Un robot **Ackermann** (cinemática de coche).

Para ello se emplean tres pilares principales:

* La librería **OMPL** para planificación de caminos en 2D (SE2).
* Un **modelo de colisión basado en footprint rectangular** que cambia según si el robot transporta o no la estantería.
* Una **máquina de estados finitos (FSM)** que separa la planificación, la navegación y la manipulación de estanterías.

Además, la práctica introduce un aspecto especialmente relevante:
**cuando el robot transporta una estantería, su tamaño efectivo aumenta significativamente**, por lo que debe planificar rutas considerando un footprint mayor.

## 2. Modelos de Robot: Holonómico vs Ackermann

### 2.1 Robot Holonómico

Características:

* Movimiento directo hacia cualquier dirección.
* Control basado en la geometría del vector relativo al waypoint.
* Capacidad de girar prácticamente sobre sí mismo.

Esto permite rutas más directas, y una navegación más simple.

<div align="center">
  <img src="https://github.com/user-attachments/assets/92455ff2-4577-4055-ab04-668940fb2859" width="400">
</div>


### 2.2 Robot Ackermann

Características:

* Cinemática similar a un automóvil.
* No puede desplazarse lateralmente.
* Su control de movimiento depende exclusivamente del **yaw**.
* Necesita radios de giro reales y mayores distancias para orientarse.

Esto hace que la planificación se apoye más en la generación de rutas suaves por OMPL y en controladores diferenciales del yaw.

<div align="center">
  <img src="https://github.com/user-attachments/assets/85c1395b-56e8-4c36-b816-54c2707ac270" width="400">
</div>

## 3. La Estantería como Carga: Impacto en el Footprint

El robot puede operar en dos modos:

1. **Robot sin carga**.
2. **Robot transportando la estantería**.

El código define el footprint base del robot (largo x ancho), pero cuando el robot lleva la estantería encima, el footprint pasa a ser:

```
footprint_combinado = max(robot, estantería)  en longitud y anchura
```

Así, la estantería modifica radicalmente:

* El espacio que debe considerar OMPL para la planificación.
* La detección de colisiones.
* La validez de los estados (SE2) en el mapa.

Por tanto, esta práctica no solo trata de planificación, sino también de **ajuste dinámico del modelo de colisión**.

## 4. El Mapa, la Escala y el Sistema de Coordenadas

El mapa del almacén es una imagen en escala de grises. El código convierte dicho mapa en una rejilla binaria en la que:

* Blanco/gris = espacio libre.
* Negro = obstáculo.

Las dimensiones reales del almacén se utilizan para calcular:

* Escalas de conversión metros → píxeles.
* Límites válidos del espacio SE2 para OMPL.
* Posiciones iniciales y posiciones destino de las estanterías.

Se implementan dos funciones fundamentales:

* **pixels2coordinates**
* **coordinates2pixels**

Ambas son esenciales para:

1. Pasar a OMPL los puntos en píxeles.
2. Dibujar sobre el mapa rutas planificadas en coordenadas reales.

<div align="center">
  <img src="https://github.com/user-attachments/assets/55cb2563-7bc6-4c7f-bed3-d0a66024e4e0" width="500">
</div>

## 5. Modelado de Colisiones: Footprint Rectangular Orientado

Uno de los aspectos más importantes y complejos del código es la función:

**isStateValid(state)**

Esta función determina si un estado SE2 está libre de colisiones. Utiliza:

* El footprint rectangular actual (robot solo o robot+estantería).
* Un margen adicional (EXTRA_MARGIN_M).
* La orientación yaw del robot.
* Un escaneo del área alrededor del centro del robot en píxeles.
* Transformación de coordenadas de los píxeles al marco local del robot.

La validación implica comprobar si alguna celda marcada como obstáculo cae dentro del rectángulo orientado del footprint.

Esto asegura que:

* El robot no atraviesa paredes.
* El robot no se mete entre patas de estantería.
* OMPL explore únicamente posiciones físicamente posibles.

## 6. Planificación con OMPL

La planificación se realiza utilizando:

```
ompl.base.SE2StateSpace
ompl.geometric.SORRTstar
```

El proceso es el siguiente:

1. Crear el espacio SE2 con límites según el tamaño del mapa.
2. Registrar la función de validez de estado (isStateValid).
3. Crear estados inicial y objetivo en píxeles.
4. Lanzar el planificador con un tiempo límite (5 segundos).
5. Simplificar la ruta obtenida.
6. Convertir la ruta a coordenadas del mundo.
7. Enviar la ruta al WebGUI para visualizarla.

OMPL trabaja directamente con **posición + yaw**, lo cual es correcto ya que los robots deben orientarse adecuadamente.

## 7. Manipulación de Estanterías

El robot interactúa con estanterías mediante:

* **HAL.lift()** → recoger
* **HAL.putdown()** → depositar

Al levantar una estantería:

* Se borran del mapa las patas originales, marcándolas como espacio libre.
* Se guarda una copia de los píxeles originales de la estantería.

Al depositarla:

* Se restauran exactamente en la nueva posición de destino.
* Se engordan ligeramente para que OMPL no planifique atravesando patas.

Este proceso hace posible que el robot replantee rutas sin que la estantería interfiera cuando está cargada.

## 8. Máquina de Estados Finita (FSM)

La lógica completa se estructura en tres estados principales:

### 8.1 TRAJECTORY_DEFINITION

* Selecciona el tamaño correcto del robot según si lleva carga o no.
* Llama al planificador OMPL.
* Genera la ruta hacia la estantería o hacia el destino final.

### 8.2 TRAJECTORY_EXECUTION

* El robot sigue la ruta generada:

  * Con **move_holo** si es holonómico.
  * Con **move_ackermann** si es Ackermann.
* Ajuste final del yaw en el caso holonómico.
* Transición al estado de manipulación cuando alcanza su destino.

### 8.3 SHELF_MANIPULATION

* Si el robot no lleva carga:

  * Levanta la estantería.
  * Actualiza el mapa eliminando patas en origen.
* Si lleva carga:

  * Deposita la estantería.
  * Restaura patas en destino.
* Avanza al siguiente objetivo.

## 9. Seguimiento de Trayectorias

### 9.1 Navegación con Robot Holonómico

La función **move_holo**:

* Calcula el vector relativo desde el robot al siguiente waypoint.
* Controla la velocidad lineal según la distancia.
* Controla la velocidad angular según la dirección de ese vector.
* La navegación es muy suave y directa.

Es capaz de llegar al waypoint en cualquier dirección sin necesidad de girar previamente.

### 9.2 Navegación con Robot Ackermann

La función **move_ackermann**:

* No utiliza coordenadas relativas.
* Calcula el yaw objetivo mediante atan2.
* Ajusta la velocidad proporcionalmente a la distancia.
* Ajusta la dirección proporcional al error de yaw.

Este controlador es físicamente consistente:

* No permite giro sobre sí mismo.
* Respeta la cinemática realista.
* Necesita más espacio para maniobrar.

## 10. Resultados Experimentales

A continuación se dejan los espacios para añadir imágenes, GIFs y vídeos de ejecución de la práctica.

### 10.1 Transporte y depósito (robot holonómico)

<div align="center">
  <img src="https://github.com/user-attachments/assets/31e201e7-3e72-4816-8d0d-71e8fbb5edea" width="450">
</div>
<p align="center"><b>Figura 1: Holonómico llevando la estantería por el pasillo.</b></p>

### 10.2 Movimiento Ackermann con carga

<div align="center">
  <img src="https://github.com/user-attachments/assets/e8b01a9b-6918-43e0-9d5b-e3f9be431cca" width="450">
</div>
<p align="center"><b>Figura 2: Robot Ackermann transportando la estantería.</b></p>

### 10.3 GIF demostrando la organizacion de las 6 estanterias

<div align="center">
  <img src="https://github.com/user-attachments/assets/35b2958d-66b5-46a2-92ca-3d4bee2055cb" width="450">
</div>
<p align="center"><b>Figuras 3: Ejecución completa en ambos modos.</b></p>

# 11. Problemas Detectados

Durante el desarrollo de la práctica surgieron varios problemas significativos relacionados tanto con la **planificación**, como con el **modelo geométrico del robot** y la **coherencia entre el mapa 2D y la escena 3D**.

A continuación se describen en detalle:

### 11.1 Desajuste entre el mapa 2D y el entorno 3D real

Uno de los problemas más graves fue que el mapa en el que OMPL planifica **no coincide exactamente con la geometría del mundo 3D del simulador**.
Este desajuste se aprecia de forma evidente cuando:

* Se observa al robot pintado sobre el mapa 2D.
* La orientación del robot en el mapa aparece **girada respecto a la orientación real del robot en el mundo 3D**.
* Las patas de las estanterías y los obstáculos no coinciden exactamente en posición.

Este desfase provoca errores sistemáticos en:

* El footprint necesario para validar colisiones.
* La ubicación real de pasillos y espacios libres.
* Las distancias de seguridad necesarias para evitar choques.

Como consecuencia, era habitual que OMPL considerara válido un estado que en la realidad colisionaba o, justo al contrario, descartara rutas perfectamente transitables.

### 11.2 Complejidad adicional en la planificación de rutas

El desajuste anterior generó dificultades como:

* Rutas que OMPL considera libres pero que el robot no puede ejecutar físicamente.
* Rutas excesivamente close-to-obstacles cuando en realidad el obstáculo estaba desplazado.
* Fallos en la verificación de colisiones al transformar entre coordenadas de píxeles y metros reales.

Esto obligó a depurar repetidamente:

* Límites del espacio SE2.
* Márgenes adicionales de seguridad.
* La función de validación de estado (isStateValid).

### 11.3 Necesidad de márgenes diferentes según el tipo de robot

A causa de estos desajustes, el tamaño del footprint efectivo no era el mismo en el mapa que en el mundo real.
Esto llevó a una situación inesperada:

* El **robot holonómico** necesitó **más margen** alrededor de su footprint porque

  * Se mueve lateralmente,
  * Puede orientarse en ángulos arbitrarios,
  * Sus maniobras dependen fuertemente de colisiones locales.

* El **robot Ackermann**, sorprendentemente, necesitó **menos margen**, porque:

  * Está mucho más limitado en sus direcciones de movimiento.
  * No adopta todas las orientaciones intermedias que el holonómico sí alcanza.
  * Sus giros suaves evitan muchas orientaciones en las que la colisión sí aparecería.

Esto contradice la intuición habitual (normalmente el Ackermann requiere más espacio), pero en este caso particular se explica por:

* El giro del robot en el mapa respecto al mundo real.
* La diferencia entre footprint dibujado y footprint real.
* La diferencia entre sus trayectorias físicas.

### 11.4 Sensibilidad extrema del footprint cuando el robot transporta estanterías

Al cargar la estantería, el footprint combinado robot+estantería era grande y rectangular.
El problema es que:

* Ese rectángulo en el mapa estaba **desalineado con respecto al 3D**.
* Las patas de la estantería no coinciden exactamente con los píxeles utilizados para borrarlas o restaurarlas.
* El algoritmo de colisiones detectaba falsos positivos al escanear regiones "engordadas" o desplazadas.

Esto derivó en:

* Rutas no planificables cuando sí deberían serlo.
* Necesidad de aumentar manualmente el número de iteraciones y tiempo de OMPL.
* Ajustes repetidos del margen EXTRA_MARGIN_M.
