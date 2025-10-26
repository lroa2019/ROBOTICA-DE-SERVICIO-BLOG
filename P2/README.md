# Misión de Búsqueda y Rescate Autónoma con Drone

## 1. Fundamento Teórico: Arquitectura de Control y Detección

La robustez de la misión se basa en dos pilares teóricos: la gestión de estados para la resiliencia y la visión por computador para la detección de objetivos.

### 1.1. Arquitectura de Control: Máquina de Estados Finitos (FSM)

Se implementó una arquitectura de control basada en una **Máquina de Estados Finitos (FSM)** para gestionar de manera clara y robusta los diferentes modos operativos del drone. Esta estructura permite un **cambio de comportamiento** basado en eventos críticos (ej. bajo nivel de batería, llegada al destino, detección de un superviviente) y garantiza que el drone siempre sepa qué hacer a continuación.

Los estados definidos en la enumeración `State` son:

* **`APPROACH_ACCIDENT`**: Navegación inicial hacia las coordenadas GPS conocidas del accidente.
* **`PATROL`**: Ejecución de la búsqueda sistemática de supervivientes a lo largo de una ruta predefinida (waypoints).
* **`RETURN`**: Navegación de vuelta a la base (lancha) por una razón específica (batería baja o misión completada).
* **`RECHARGE`**: Período de simulación de recarga en la base.
* **`LAND`**: Aterrizaje y finalización de la misión.

La FSM gestiona la **continuidad de la misión** guardando un punto de reanudación (`resume_pos` y `resume_idx`) justo antes de pasar al estado `RETURN` por baja batería, permitiendo al drone retomar la patrulla exactamente donde la dejó después de recargar.

### 1.2. Visión por Computador para Detección de Supervivientes

La detección de las personas se realiza utilizando la cámara ventral del drone y la librería **OpenCV**.

1.  **Detector Haar Cascade:** Se emplea un clasificador **Haar Cascade** pre-entrenado (`haarcascade_frontalface_default.xml`) para la detección de características que se asemejen a caras.
2.  **Robustez de Detección:** Para compensar posibles orientaciones no óptimas de la víctima, el sistema aplica una técnica de robustez: si la detección inicial en la imagen original falla, se realizan **pruebas con rotaciones** de la imagen (45°, 90°, 135°, etc.) antes de descartar el *frame*.
3.  **Registro y Descarte de Duplicados:** Una vez que se detecta un superviviente, se registra su posición GPS $(x, y)$. Se implementa un mecanismo para **evitar el registro de duplicados** al verificar que la nueva detección se encuentre a una distancia mínima (`DETECT_RADIUS = 4.0 m`) de cualquier superviviente previamente anotado.

---

## 2. Gestión de Recursos Críticos: El Modelo de Batería

La autonomía en el SAR es crítica, por lo que el sistema incorpora un modelo de gestión energética que dicta los cambios de estado hacia `RETURN` y `RECHARGE`.

### 2.1. Simulación de Consumo y Monitoreo

El consumo de batería se modela en función de la **distancia recorrida**.

* **Tasa de Drenaje:** El drone pierde una cantidad fija de batería por metro recorrido (`BATT_RATE_PER_METER = 0.93%` por metro).
* **Umbrales de Alerta:** El sistema genera alertas informativas (`[BATTERY]`) en intervalos definidos (`ALERT_THRESHOLDS = [90, 80, ..., 10]`) para notificar el descenso gradual.
* **Umbral Crítico de Retorno:** Se define un umbral de batería baja (`LOW_BATTERY_THRESHOLD = 30%`). Al alcanzar este nivel en cualquier estado de navegación, la FSM realiza una transición inmediata al estado `RETURN`.

### 2.2. Procedimiento de Retorno y Recarga

El protocolo de recarga es no bloqueante y garantiza la máxima eficiencia en la reanudación:

1.  **Transición a `RETURN`:** Al detectar el umbral crítico del 30%, el drone registra sus coordenadas actuales (`resume_pos`) y el índice del *waypoint* pendiente (`resume_idx`).
2.  **Recarga:** Al aterrizar en la base, el estado transiciona a `RECHARGE`. La carga se simula en ticks, restaurando la batería hasta el $100\%$ a una tasa de **~5% por segundo simulado**.
3.  **Reanudación:** Una vez completada la recarga, el drone despega nuevamente y su **primer objetivo** se establece en la `resume_pos` guardada, retomando el estado `PATROL` y el índice `resume_idx` para continuar la búsqueda sin repetir áreas.

---

## 3. Implementación y Ejecución de la Misión (Fases Operacionales)

La misión se ejecuta como un bucle continuo de la FSM, con navegación definida por la función `Maps_step` que controla la posición del drone hacia el objetivo actual.

### 3.1. Fase Inicial: Despegue y Aproximación

1.  **Despegue:** El drone inicia la misión despegando a una altura de seguridad y navegación (`ALTITUDE = 3.5 m`). Se registra la posición de despegue como la **Base** $(x_{\text{base}}, y_{\text{base}})$.
2.  **Aproximación:** El sistema entra en el estado `APPROACH_ACCIDENT`, navegando al punto inicial de coordenadas conocidas del accidente (`accident\_goal`). Durante esta fase, la detección de supervivientes está **desactivada** (`search\_mode = False`) para priorizar la llegada rápida a la zona.

<div align="center">
  <img src="https://github.com/user-attachments/assets/132b3675-1bd6-4cce-83d9-92a2c0bf9293" alt="Mapa" width="400"/>
</div>

<p align="center"><b>Figura 1: Drone despegando de la lancha (Base) o volando hacia el punto de accidente.</b></p>

* **Detalle Técnico:** Muestra el drone alcanzando la altitud de crucero ($3.5$ metros) y la estabilidad de la posición antes de la navegación.

### 3.2. Fase de Patrulla y Detección (`PATROL`)

Al llegar a la zona de accidente, la FSM pasa al estado `PATROL`.

1.  **Ruta de Búsqueda:** Se utiliza una secuencia predefinida de puntos de paso (`waypoints`) que aseguran un **barrido sistemático** de la zona de búsqueda.
2.  **Detección Activa:** En este estado, la detección por cámara ventral está **activada** (`search\_mode = True`), monitoreando continuamente el suelo o el agua en busca de patrones faciales.
3.  **Transición Condicional:** La navegación de *waypoint* a *waypoint* es continua. El estado solo cambia si se completa la lista de *waypoints* o si el nivel de batería desciende por debajo del umbral crítico.

<div align="center">
  <img src="https://github.com/user-attachments/assets/1e51125d-b2fb-4d4e-b31d-4d35a7bfe551" alt="Mapa" width="400"/>
</div>

<p align="center"><b>Figura 2: Video del drone ejecutando la ruta de patrulla.</b></p>

* **Detalle Técnico:** Documenta la secuencia de navegación punto a punto y la visualización de la detección de un superviviente (o simulacro) a través de la interfaz (WebGUI).

### 3.3. Fase de Retorno a Base y Conclusión

La misión concluye o se interrumpe con el estado `RETURN`.

1.  **Retorno:** El drone navega de vuelta a las coordenadas de la Base $(x_{\text{base}}, y_{\text{base}})$.
2.  **Aterrizaje y Conclusión:** Al llegar a la base, el drone aterriza (`HAL.land()`). Si la razón de retorno fue **misión completada**, el sistema entra en el estado `LAND` y finaliza. Si la razón fue **batería baja**, transiciona a `RECHARGE` para repetir el ciclo de reanudación.

<div align="center">
  <img src="https://github.com/user-attachments/assets/0fcffdf1-d98e-4cda-9136-196bde22a9f1" alt="Mapa" width="400"/>
</div>

<p align="center"><b>Figura 3: Captura de pantalla mostrando nivel de bateria bajo.</b></p>

<div align="center">
  <img src="https://github.com/user-attachments/assets/4a06c06d-adf4-461e-b5f2-4b0d02aaadba" alt="Mapa" width="400"/>
</div>

<p align="center"><b>Figura 4: Captura de pantalla final mostrando la lista de coordenadas de los supervivientes encontrados.</b></p>

* **Validación de Resultados:** La imagen certifica la salida final del sistema, incluyendo el número total y las coordenadas GPS precisas (redondeadas a dos decimales) de los supervivientes rescatados, proporcionando los datos esenciales para los equipos de rescate físicos.

---

## 4. Propuesta de Mejora: Gestión Predictiva de Energía (RTH Dinámico)

El sistema actual opera con un umbral de retorno fijo ($30\%$), lo que compromete la eficiencia y la seguridad. Se propone una mejora que implemente un **cálculo predictivo dinámico** del punto de retorno (RTH).

### 4.1. Metodología de Umbral Dinámico

El objetivo es que el drone calcule en cada instante la batería mínima necesaria para volver a la base y active el retorno cuando la batería actual iguale o caiga por debajo de ese nivel, más un margen de seguridad.

1.  **Cálculo de Distancia de Retorno ($D_{\text{retorno}}$):** En cada *tick* de la FSM, el sistema calcula la distancia euclidiana (GPS) desde la posición actual del drone $(x_{\text{actual}}, y_{\text{actual}})$ hasta la base $(x_{\text{base}}, y_{\text{base}})$.

    $$D_{\text{retorno}} = \sqrt{(x_{\text{base}} - x_{\text{actual}})^2 + (y_{\text{base}} - y_{\text{actual}})^2}$$

2.  **Cálculo del Consumo Requerido ($B_{\text{necesaria}}$):** Utilizando la tasa de consumo conocida por metro (BATT_RATE_PER_METER = 0.93%/m), se estima la batería mínima necesaria para cubrir el trayecto de vuelta:

    $$B_{\text{necesaria}} = D_{\text{retorno}} \cdot B_{\text{RATE\_PER\_METER}}$$

3.  **Umbral Dinámico de Retorno ($B_{\text{dinámico}}$):** Se establece un **margen de seguridad del $5\%$** sobre el consumo calculado. El drone iniciaría el estado `RETURN` tan pronto como su batería actual ($B_{\text{actual}}$) sea igual o inferior a este umbral dinámico:

    $$B_{\text{dinámico}} = B_{\text{necesaria}} + 5\%$$

La nueva condición de transición para el retorno en los estados `APPROACH_ACCIDENT` y `PATROL` sería:

$$\text{Si } B_{\text{actual}} \le B_{\text{dinámico}} \text{, entonces la FSM transiciona a } \text{State.RETURN}$$

Esta mejora convierte la gestión de energía en un sistema **predictivo**, maximizando el tiempo de patrulla útil sin comprometer el margen de seguridad de aterrizaje.
