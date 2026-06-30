# Práctica P5 – Laser Mapping

## 1. Introducción

La práctica P5 plantea el problema del mapeo autónomo de un entorno industrial desconocido mediante un sensor láser 2D y odometría. Mi robot debe:

1. Explorar de forma autónoma las zonas no visitadas del entorno.
2. Construir, de manera incremental, un mapa de ocupación probabilístico de la nave industrial.
3. Distinguir entre celdas libres, ocupadas y no observadas.
4. Permitir evaluar el efecto de una localización degradada (odometría 2 y 3) sobre la calidad del mapa generado.

El entorno es un almacén con estanterías, pasillos y obstáculos dispersos. Mi robot dispone de un sensor láser que cubre 360 grados con una apertura angular muy estrecha por haz, lo que le permite aproximar cada disparo láser como una línea recta (geometría axial), simplificando el trazado de cada rayo sobre la rejilla del mapa.

## 2. Modelo probabilístico del mapa: rejilla de ocupación en log-odds

Debo realizar la construcción de un **mapa de ocupación probabilístico en forma de rejilla (grid map)**, siguiendo el enfoque estándar de combinación de evidencias mediante la regla de Bayes.

### 2.1 Representación interna

En lugar de almacenar directamente una probabilidad de ocupación por celda, el mapa se representa internamente en el **espacio de log-odds**, definido para una celda con probabilidad de ocupación p como:

```
l = log( p / (1 − p) )
```

Esta representación es la habitual en mapeo probabilístico porque convierte la combinación bayesiana de evidencias sucesivas en una simple suma de incrementos, evitando inestabilidades numéricas cerca de p = 0 o p = 1.

Cada celda de la rejilla (970 × 1500) se inicializa a l = 0, lo que corresponde a p = 0.5, es decir, total incertidumbre (celda no observada).

### 2.2 Modelo sensorial y combinación de evidencias (regla de Bayes)

Por cada lectura láser se actualizan dos tipos de evidencia:

* **Evidencia de espacio libre**: todas las celdas atravesadas por el haz, desde el robot hasta el punto de impacto (o hasta el alcance máximo si no hay impacto), reciben un pequeño decremento en su log-odds. Esto reduce su probabilidad de ocupación.
* **Evidencia de obstáculo**: si el haz detectó un impacto real (distancia finita y dentro del alcance máximo), la celda final del rayo recibe un incremento mayor en su log-odds, aumentando su probabilidad de ocupación.

Cuando una misma celda es observada repetidamente, sus incrementos y decrementos se van sumando algebraicamente. Esta suma de log-odds es matemáticamente equivalente a aplicar la regla de Bayes de forma recursiva para combinar observaciones independientes sobre la misma celda.

### 2.3 Saturación de la inercia probabilística

Para evitar que una celda observada muchas veces como libre (u ocupada) acumule un log-odds extremo (lo que la haría casi imposible de corregir ante una observación contraria), el valor de cada celda se satura dentro de un rango simétrico acotado tras cada actualización. Este mecanismo de saturación lo uso para limitar la inercia probabilística del mapa.

### 2.4 Visualización del mapa

Para mostrar el mapa, el log-odds de cada celda se convierte de nuevo a probabilidad mediante la función sigmoide (inversa del log-odds), y esa probabilidad se traduce a una escala de grises: blanco para libre, negro para ocupado y gris intermedio para zonas aún no observadas (probabilidad cercana a 0.5).

## 3. Modelado del sensor láser

El láser proporciona 360 lecturas equiespaciadas angularmente alrededor del robot, con un alcance máximo configurado. Para cada lectura:

* Si la distancia es infinita o supera el alcance máximo, se interpreta como ausencia de obstáculo: el rayo se traza hasta el alcance máximo y se marca únicamente como evidencia de espacio libre.
* Si la distancia es finita y está dentro del alcance, se interpreta como un impacto real: el rayo se traza como espacio libre hasta el punto de impacto, y ese punto final se marca como evidencia de ocupación.

Dado que la apertura angular de cada haz es muy pequeña, he aproximado cada disparo como un segmento recto desde la posición del robot hasta el punto calculado mediante la pose del robot, el ángulo del haz y la distancia medida.

## 4. Trazado de rayos: algoritmo de Bresenham

Para recorrer eficientemente todas las celdas intermedias entre el robot y el punto final de cada rayo he empleado una variante discreta del algoritmo de Bresenham. Este algoritmo:

* Tiene coste lineal respecto a la longitud del rayo en celdas.
* Evita el uso de aritmética en coma flotante salvo en el cálculo inicial del punto final del rayo.
* Garantiza que se visite exactamente la secuencia de celdas que atraviesa la línea recta, sin huecos ni duplicados.

Cada celda intermedia recibe la actualización de "evidencia libre"; la celda final recibe la actualización de "evidencia ocupada" o "evidencia libre", según si hubo o no impacto real.

## 5. Exploración autónoma

La exploración combina dos comportamientos:

### 5.1 Evitación reactiva de obstáculos

Antes de decidir hacia dónde explorar, el robot comprueba si existe un obstáculo cercano en su sector frontal. Si la distancia es inferior a un umbral de seguridad, el robot reduce su velocidad lineal a cero, compara el espacio libre disponible a izquierda y derecha, y gira hacia el lado más despejado hasta liberar el frente.

### 5.2 Búsqueda activa de regiones desconocidas

Cuando no hay riesgo de colisión inminente, el robot evalúa un conjunto de direcciones distribuidas uniformemente a su alrededor (36 direcciones, cada 10 grados). Para cada dirección, se avanza virtualmente sobre el propio mapa ya construido, celda a celda, hasta encontrar la primera celda no observada accesible (es decir, sin atravesar antes una celda ya marcada como ocupada). El robot selecciona la dirección cuya celda desconocida más cercana está a menor distancia, de forma que prioriza siempre la exploración más eficiente del entorno restante. Si ninguna dirección conduce a una zona desconocida cercana, el robot avanza en línea recta para seguir cubriendo el entorno.

Esta estrategia es una versión simplificada de los algoritmos clásicos de exploración por fronteras (frontier-based exploration).

## 6. Control de movimiento

El giro hacia la dirección elegida se controla mediante un regulador proporcional sobre el ángulo relativo entre la orientación actual del robot y la dirección objetivo, limitado a una velocidad angular máxima. La velocidad lineal se modula en función de ese mismo ángulo relativo: cuanto más alineado está el robot con el objetivo, mayor es su velocidad de avance; cuanto mayor es el giro pendiente, más se reduce la velocidad lineal, sin bajar nunca de un mínimo que evita que el robot se quede bloqueado.

## 7. Integración del sistema

En cada ciclo de control mi robot:

1. Obtiene su pose (de la fuente de odometría seleccionada) y la lectura láser completa.
2. Actualiza el mapa de log-odds con el modelo sensorial descrito en la sección 2.
3. Decide su siguiente movimiento mediante el controlador de exploración reactiva.
4. Convierte el mapa de log-odds a una imagen de probabilidad y la envía a la interfaz web para su visualización en tiempo real.

El ciclo se ejecuta a una frecuencia regulada para mantener la sincronización entre percepción, mapeo y control.

## 8. Resultados Experimentales

### 8.1 Mapa inicial (todo desconocido)

<div align="center">
  <img src="https://github.com/user-attachments/assets/ec227a36-a89f-4ebc-a12c-cd77e5ee10a4" width="800">
</div>  
<p align="center"><b>Figura 1: Estado inicial del mapa (UNK).</b></p>

### 8.2 Progresión del mapeo

<div align="center">
  <img src="https://github.com/user-attachments/assets/74ed9025-7aa0-402b-b2dc-ee633a91bcaf" width="800">
</div>  
<p align="center"><b>Figura 2: Proceso de construcción del mapa.</b></p>

### 8.3 Mapa final tras exploración completa

<div align="center">
  <img src="https://github.com/user-attachments/assets/c4baff3c-313e-4474-847d-3c569c2c212f" width="800">
</div>  
<p align="center"><b>Figura 3: Mapa final obtenido por el robot.</b></p>

### 8.4 Video de la ejecución completa (odometría2 y odometría3)

[Ejecución completa video](https://github.com/user-attachments/assets/fbb4ce38-f6fc-4882-ad74-671035619bc2)
<p align="center"><b>Figura 4: Demostración de la ejecución.</b></p>

### 8.5 Mapa tras exploración con odometría2

Repití la construcción del mapa empleando las fuentes de odometría ruidosa proporcionadas. A medida que aumenta el ruido en la estimación de la pose, el mapa generado muestra una degradación progresiva, como: contornos de paredes duplicados o desdoblados, desplazamientos relativos entre regiones que deberían ser coherentes entre sí, y pérdida de nitidez en los límites entre zonas libres y ocupadas. Este resultado ilustra de forma directa la sensibilidad del mapeo probabilístico a la calidad de la localización: dado que cada actualización del mapa depende de proyectar la lectura láser desde la pose estimada, cualquier error acumulado en esa pose se traslada directamente a una distorsión geométrica del mapa, incluso cuando el modelo sensorial y la regla de combinación de evidencias funcionan correctamente.

<div align="center">
  <img src="https://github.com/user-attachments/assets/2cfb1293-d4b4-4e7f-9f41-7aa232f78874" width="800">
</div>  
<p align="center"><b>Figura 5: Mapa obtenido usando odometría2.</b></p>

## 9. Problemas detectados

### 9.1 Irregularidades en el mapa derivadas del ruido del sensor láser

El sensor láser presenta leves imprecisiones en la estimación de distancias incluso en condiciones estáticas. Al trazar los rayos correspondientes, estas imprecisiones se traducen en bordes irregulares y pequeñas prolongaciones lineales alrededor de paredes y objetos, lo que reduce la nitidez de los contornos del mapa. El carácter discreto del trazado de rayos amplifica ligeramente este efecto ante pequeñas variaciones angulares o métricas en los impactos del láser.

### 9.2 Aproximación excesiva a obstáculos

Pese a que el controlador reactivo implementa una distancia mínima de seguridad, la variabilidad de las lecturas del láser provoca que, en la práctica, esa distancia efectiva se reduzca en determinadas situaciones. Aunque no se registraron colisiones, el robot llega a operar en algunos tramos con un margen inferior al deseado, especialmente en pasillos estrechos donde el ruido sensorial y la discretización del mapa dificultan estimar con precisión la distancia real al obstáculo más cercano.

### 9.3 Degradación del mapa con odometría imperfecta

Como se describe en la sección 8.5, el uso de odom2 y odom3 provoca una acumulación progresiva de error de pose que se traduce en deformaciones globales del mapa, especialmente visibles en pasillos largos y superficies amplias, donde la incoherencia angular y posicional se acumula más rápidamente.

### 9.4 Limitaciones de la estrategia de exploración reactiva

La política de exploración, basada en la detección de la celda desconocida más cercana y en la evasión reactiva de obstáculos, puede dejar sin explorar determinadas zonas del entorno, en particular áreas con geometría compleja que requerirían maniobras más deliberadas que un enfoque puramente reactivo no contempla.

