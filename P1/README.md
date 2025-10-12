
# Aspiradora Robótica con Algoritmo BSA

---

## 1. Introducción teórica

El objetivo de esta práctica es desarrollar una **aspiradora robótica autónoma** capaz de limpiar un entorno completo de forma sistemática y eficiente.  
El proyecto simula el funcionamiento de un robot doméstico de limpieza, similar a los modelos comerciales actuales, y permite entender cómo se combinan la **percepción del entorno**, la **planificación de rutas** y el **control de movimiento**.

El robot opera sobre un **mapa del entorno**, que incluye tanto las zonas transitables como los obstáculos fijos (paredes, muebles, etc.).  
A partir de este mapa, el sistema genera una **rejilla de navegación**, dividiendo la superficie en celdas cuadradas que representan áreas pequeñas del entorno.  
Esta discretización convierte un espacio continuo en un conjunto de posiciones alcanzables que el robot puede visitar una a una.

Para planificar la cobertura se utiliza el **algoritmo BSA (Backtracking Spiral Algorithm)**.  
Este método combina una exploración sistemática tipo espiral con la capacidad de **retroceder (backtracking)** hacia zonas no visitadas cuando no existen celdas libres adyacentes, asegurando así la **cobertura total del entorno**.

El sistema se completa con un **control de movimiento reactivo**, que traduce las decisiones del planificador en órdenes de velocidad lineal y angular, y con una **interfaz gráfica (WebGUI)** donde se visualiza el progreso en tiempo real: el mapa, los obstáculos, las zonas visitadas y la trayectoria del robot.

---

## 2. Fases del desarrollo

El desarrollo de la práctica se divide en varias fases principales que abarcan desde el procesamiento inicial del mapa hasta la ejecución final de la limpieza.

---

### 2.1 Registro y preprocesamiento del mapa

En primer lugar, el programa **carga el mapa del entorno** desde el simulador.  
Este mapa se transforma a escala de grises y se normaliza para distinguir entre las zonas libres (grises) y los obstáculos (negros).  
Después se divide en una **rejilla de celdas cuadradas** de **33×33 píxeles**, lo que define la resolución de la navegación.

La elección de este tamaño de celda es crucial: un valor demasiado grande provoca pérdida de precisión, mientras que uno demasiado pequeño aumenta la carga de cálculo.  
En este caso se busca un equilibrio entre realismo y eficiencia.

---

### 2.2 Creación de la rejilla y clasificación de celdas

Una vez segmentado el mapa, el sistema analiza cada celda para determinar si está **libre o bloqueada**.  
Si todos los píxeles del bloque son transitables, se considera una celda libre; si existe algún píxel negro, se marca como obstáculo.  
Esto asegura que el robot no se acerque en exceso a las paredes o muebles.

También se crea una correspondencia entre el **centro de cada celda** y sus coordenadas dentro del mapa, lo que permite transformar posiciones entre el mundo real y la representación digital.

---

### 2.3 Planificación de la ruta con el algoritmo BSA

Una vez que el mapa está listo y se conoce la posición inicial del robot, se ejecuta el algoritmo **BSA (Backtracking Spiral Algorithm)**.  
El robot comienza desde su celda actual y se expande a las celdas adyacentes libres siguiendo un orden de prioridad (izquierda, arriba, derecha, abajo).  
Cada celda visitada se marca en **verde**, indicando que ya fue recorrida.

Cuando no hay más celdas libres cercanas, el sistema localiza un **punto crítico** (en azul), que representa una celda libre no visitada, y planifica una ruta hacia ella mediante una búsqueda **BFS (Breadth-First Search)**.  
De esta manera, el robot retoma la exploración hasta que todo el entorno queda cubierto.

Durante este proceso, el mapa se actualiza en la interfaz WebGUI:
- Zonas visitadas: verde  
- Ruta planificada: amarillo  
- Puntos críticos: azul  

Esta visualización permite seguir paso a paso el razonamiento del robot y comprobar que se logra la cobertura completa.

---

### 2.4 Pilotaje y control del movimiento

La fase final consiste en la **ejecución física de la ruta**.  
El robot emplea un **control proporcional-derivativo (PD)** para orientar su dirección y ajustar su velocidad según la distancia y el ángulo respecto al siguiente destino.  
Este control garantiza movimientos suaves, sin giros bruscos ni sobrepasos.  
A medida que el robot avanza, cada celda alcanzada se marca como visitada, actualizando en tiempo real el mapa mostrado en la interfaz.

El proceso se repite de manera continua hasta cubrir completamente el entorno.  
Al finalizar, el sistema muestra un mensaje confirmando que la limpieza ha concluido correctamente.

---

## 3. Problemas durante el desarrollo

Durante la realización de la práctica se identificaron varios inconvenientes que afectaron al comportamiento del sistema:

1. **Diferencia entre el tamaño del robot y el tamaño de las celdas.**  
   El robot tiene un diámetro aproximado de **35 píxeles**, mientras que las celdas del mapa son de **33 píxeles**.  
   Este ajuste se realizó para que el mapa no quedara demasiado reducido, pero generó pequeños conflictos en zonas estrechas (como el área del sofá).  
   En estos casos, el robot podía quedarse atascado porque el algoritmo planifica trayectorias que rozan los límites sin dejar margen de seguridad.  
   Una posible mejora sería aumentar ligeramente el tamaño de celda o añadir una tolerancia de seguridad en el control de movimiento.

2. **Errores de mapeado en el entorno.**  
   En ciertos puntos, especialmente junto al sofá, el mapa parece **mal definido o desplazado algunos píxeles**.  
   Esto provoca que el robot detecte como libre una zona que realmente está parcialmente ocupada.  
   Como el sistema solo considera libres las celdas completamente despejadas, estas pequeñas imprecisiones pueden causar bloqueos o desvíos.

3. **Planificación lenta de la ruta.**  
   El algoritmo de planificación resultó **lento**, ya que el programa **pinta cada celda en la interfaz** a medida que la calcula.  
   Esto permite observar el progreso en tiempo real, pero consume muchos recursos gráficos y de procesamiento.  
   Una solución sería **actualizar la interfaz por lotes** (cada cierto número de celdas o al completar un tramo), lo que aceleraría considerablemente el proceso sin perder claridad visual.
