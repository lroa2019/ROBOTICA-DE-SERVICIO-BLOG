# Práctica 3 – Coche Autónomo con Maniobra de Aparcamiento Automatizado

## 1. Introducción Teórica

El objetivo de esta práctica es desarrollar un sistema de conducción autónoma capaz de ejecutar una maniobra completa de aparcamiento en línea utilizando únicamente sensores laterales y la orientación del vehículo. El sistema debe ser capaz de:

1. Detectar la orientación real de la calle.
2. Buscar un hueco adecuado entre vehículos estacionados.
3. Alinear el vehículo con la calle antes de iniciar la maniobra.
4. Ejecutar correctamente el giro y el retroceso para introducirse en el hueco.
5. Realizar un ajuste final de posición y orientación dentro del espacio.

Para resolver este problema se utiliza una combinación de:

* Interpretación geométrica del láser lateral.
* Filtrado robusto mediante mediana y RANSAC.
* Control cinemático para vehículos Ackermann.
* Una máquina de estados finitos (FSM) que estructura toda la lógica.
* Controladores específicos para movimientos en X, Y y yaw.

A diferencia de otras prácticas, **no se utiliza GPS absoluto**, ya que su precisión es insuficiente. El GPS solo se emplea para desplazamientos incrementales y para obtener el ángulo yaw del vehículo.

## 2. Sensores y Cinemática

### 2.1 Modelo Ackermann

El coche se controla mediante:

* Velocidad lineal (V)
* Velocidad angular (W)

Este modelo reproduce el comportamiento real de un vehículo, sin permitir desplazamientos laterales independientes.

### 2.2 Sensores disponibles

1. **Láser derecho (principal)**

   * 180 lecturas de distancia
   * Angulos entre 0° (parte trasera-derecha) y 179° (delantera-derecha)

2. **Láser frontal y trasero**

   * Utilizados principalmente para seguridad

3. **IMU / compás**

   * Proporciona yaw, fundamental para el control del vehículo

4. **GPS (solo incremental)**

   * No se usa para posicionamiento absoluto
   * Sí para medir desplazamientos relativos

## 3. Arquitectura del Sistema: Máquina de Estados

El sistema está organizado como una **FSM (Finite State Machine)**. Cada etapa aborda una fase distinta del proceso de aparcamiento:

1. **ESTIMATE_STREET**
   Estima la orientación de la calle utilizando datos del láser.

2. **SEARCHING_PARKING_SPACE**
   Busca un hueco con la anchura suficiente para aparcar.

3. **GOTO_PARK_ENTRY**
   Coloca el vehículo en la posición exacta para iniciar la maniobra.

4. **PARKING_MANEUVER**
   Maniobra principal: giro marcha atrás y retroceso profundo.

5. **FINAL_ALIGN_SHIFT_Y**
   Ajuste lateral fino dentro del hueco.

6. **FINAL_ALIGN_X_YAW**
   Cuadrado final: posición longitudinal y orientación paralela a la calle.

Este diseño modular permite depurar y analizar cada fase por separado, al igual que en la práctica anterior del dron SAR.

## 4. Estimación de la Calle

La primera tarea es determinar la orientación de la calle a partir de los datos del láser derecho.

### 4.1 Selección de puntos útiles

Se filtran:

* Lecturas infinitas
* Lecturas máximas (sin obstáculo cercano)
* Puntos demasiado próximos al sensor

Después se proyectan las lecturas en coordenadas (x, y).

### 4.2 Detección de la línea lateral mediante RANSAC

Se aplica un proceso RANSAC simplificado:

1. Selección aleatoria de dos puntos.
2. Cálculo de la recta candidata.
3. Conteo de inliers (puntos a menos de 8 cm).
4. Selección de la recta con mayor número de inliers.

La técnica es robusta frente a ruido, discontinuidades y curvas suaves.

### 4.3 Obtención del yaw de la calle

La orientación detectada se combina con el yaw actual del vehículo:

`TARGET_YAW = yaw_coche + yaw_detectado_laser`

Posteriormente se filtra mediante mediana para eliminar ruidos.

<div align="center">
  <img src="https://github.com/user-attachments/assets/92b55363-ce2a-412e-b848-a7655968e9ec" width="600">
</div>
<p align="center"><b>Figura 1: Estimación del ángulo de la calle mediante RANSAC.</b></p>

## 5. Detección del Hueco de Aparcamiento

Tras conocer la orientación de la calle, el vehículo mantiene yaw estable y recorre el lateral derecho buscando un hueco apto para aparcar.

### 5.1 Sector angular de interés

Se selecciona un rango alrededor de 55°–85°, donde el láser observa directamente hacia los vehículos aparcados o hacia el hueco.

### 5.2 Criterios de hueco válido

Un hueco es válido si:

* Las lecturas en ese sector exceden cierto umbral (6 m o 9 m según modo).
* La longitud acumulada del hueco supera 6 metros.
* El hueco permanece estable durante un intervalo de distancia.

Se mide el desplazamiento del coche para calcular la longitud real del hueco.

### 5.3 Confirmación y transición

Una vez detectado el hueco:

* Se registra la posición actual.
* Se inicializan los parámetros de entrada.
* Se pasa al estado GOTO_PARK_ENTRY.

<div align="center">
  <img src="https://github.com/user-attachments/assets/b54d13b9-e8cf-4f2f-b15b-5e68f4bf975d" width="600">
</div>
<p align="center"><b>Figura 2: Detección de hueco válido en el lateral derecho.</b></p>

## 6. Fase de Entrada al Hueco

Esta etapa coloca al vehículo en la posición exacta para iniciar la maniobra de aparcamiento.

### 6.1 Cálculo de la distancia óptima al bordillo

Se utiliza la mediana de lecturas alrededor de 90° para obtener una medida fiable de la distancia al lateral.

### 6.2 Desplazamiento lateral controlado (Shift Y)

Mediante un controlador específico:

* Se mantiene el yaw constante.
* Se realiza una corrección suave del error lateral.
* Se evita la oscilación en yaw mediante correcciones proporcionales.

### 6.3 Avance en X

Tras corregir la posición lateral, el vehículo avanza aproximadamente 4.2 metros hacia delante, preparando el espacio necesario para iniciar la maniobra marcha atrás.

<div align="center">
  <img src="https://github.com/user-attachments/assets/f6850795-9f85-4dd7-9b6b-9ee7e0833a2f" width="600">
</div>
<p align="center"><b>Figura 3: Vehículo alineado y en posición de entrada.</b></p>

## 7. Maniobra Principal de Aparcamiento

La maniobra consta de dos etapas:

1. Giro marcha atrás hasta alcanzar un ángulo objetivo negativo.
2. Retroceso profundo hasta situar el vehículo dentro del hueco.

### 7.1 Control simultáneo de X y Yaw

El controlador go_to_x combina:

* Error longitudinal.
* Error angular (yaw).
* Velocidad mínima para asegurar control.

Se establece un ángulo objetivo inicial de aproximadamente -0.85 rad.

### 7.2 Retroceso controlado

La maniobra termina cuando:

* La posición X coincide con la esperada dentro del hueco.
* El ángulo de yaw coincide con el objetivo.

<div align="center">
  <img src="https://github.com/user-attachments/assets/e04b1c4a-215c-4dd7-983c-d89ff8724ad6" width="600">
</div>
<p align="center"><b>Figura 4: Secuencia de la maniobra de entrada marcha atrás.</b></p>

## 8. Alineación Final

Tras realizar la maniobra principal, se aplican dos correcciones finales.

### 8.1 Ajuste lateral

Se emplea de nuevo el controlador go_to_y con:

* Marcha atrás.
* Control angular estricto.
* Velocidad mínima personalizada.

### 8.2 Cuadrado final en X y yaw

Se realiza la corrección final para dejar el vehículo:

* Paralelo a la calle.
* A la distancia adecuada del bordillo.
* En la posición correcta dentro del hueco.

Se utiliza un suelo de velocidad para evitar que el vehículo entre en un régimen donde resulta difícil controlar la orientación.

<div align="center">
  <img src="https://github.com/user-attachments/assets/2b688100-44aa-417b-83c6-ccb76a7c42d7" width="600">
</div>
<p align="center"><b>Figura 5: Vehículo completamente alineado dentro del hueco.</b></p>

## 9. Controladores Implementados

### 9.1 Controlador go_to_y

Especializado en corrección lateral:

* Mantiene yaw constante.
* Calcula un yaw de referencia basado en el error lateral.
* Reduce velocidad para alcanzar precisión.

### 9.2 Controlador go_to_x

Especializado en desplazamientos hacia adelante o atrás:

* Control proporcional en X.
* Control proporcional en yaw.
* Suelo de velocidad configurable.
* Manejo de tolerancias independientes.

### 9.3 Controlador general GOTO

Permite configurar:

* Tolerancias.
* Velocidades máxima y mínima.
* Sentido de avance.
* Tipo de movimiento (posición o shift lateral).

## 10. Resultados

<div align="center">
  <img src="https://github.com/user-attachments/assets/0ee89f32-b463-4d57-a014-da8d4308179f" width="600">
</div>

---

<div align="center">
  <img src="https://github.com/user-attachments/assets/dc2a0973-9842-46f2-ab79-6c764ff84d77" width="600">
</div>

---

<div align="center">
  <img src="https://github.com/user-attachments/assets/09e2ca83-2ddb-4505-8bba-8e5b70658dee" width="600">
</div>

## 11. Problemas Encontrados

1. **Ruido en mediciones del láser**
   Solución: mediana + filtrado por rangos.

2. **Oscilaciones de yaw en maniobras lentas**
   Solución: suelos de velocidad y tolerancias adaptativas.

3. **Lecturas infinitas confundidas con hueco**
   Solución: descartar mediciones cercanas al rango máximo del sensor.

4. **Inestabilidad en alineación final**
   Solución: ajuste de tolerancias, velocidades y control proporcional.


