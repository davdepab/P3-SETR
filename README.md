# Controlador en Tiempo Real para Máquina Expendedora en Arduino

<img width="1024" height="765" alt="image" src="https://github.com/user-attachments/assets/45df7206-1955-45e2-81ce-44de10ccdd82" />


En este proyecto he diseñado e implementado el sistema de control software para una máquina expendedora automática utilizando **Arduino UNO**. El objetivo central ha sido construir una arquitectura reactiva y determinista capaz de atender eventos físicos en tiempo real, leer sensores y gestionar la interfaz de usuario sin recurrir a esperas bloqueantes (`delay()`).

---

## 1. Esquemático y Montaje del Circuito

El montaje físico integra los periféricos exigidos: una pantalla LCD 16x2, un módulo joystick analógico con pulsador integrado, un sensor de temperatura y humedad DHT11, un sensor de distancia por ultrasonidos HC-SR04, un pulsador externo y dos LEDs indicadores (`LED1` y `LED2`).

<img width="2173" height="1559" alt="sketch_bb" src="https://github.com/user-attachments/assets/36f558f4-6283-4e91-82b6-d623c5e15815" />

### Distribución de Pines y Criterios de Conexión

| Componente | Pin Arduino | Justificación técnica de la conexión |
| :--- | :--- | :--- |
| **Pulsador externo** | D2 | En Arduino UNO solo los pines `D2` y `D3` disponen de interrupciones externas por hardware (`INT0`/`INT1`). Además utilizo la resistencia interna de Arduino `INPUT_PULLUP` para simplificar el montaje del circuito. |
| **Ultrasonidos (Trig / Echo)** | D3 / D4 | Pines digitales estándar para generar el pulso de disparo (`10 µs`) y medir el tiempo del eco de retorno. |
| **LED1 / LED2** | D5 / D6 | `LED2` está conectado a `~D6` porque requiere modulación por ancho de pulso (PWM) por hardware para variar su brillo progresivamente. |
| **Joystick (SW / VRx / VRy)** | D7 / A0 / A1 | Los potenciómetros de los ejes usan entradas analógicas (`A0/A1`) y el pulsador integrado usa `D7` aprovechando la resistencia interna `INPUT_PULLUP`. |
| **LCD 16x2 (RS, E, D4-D7)** | D13, D12, D11, D10, D9, D8 | Configurado en interfaz paralela de 4 bits para ahorrar cuatro pines digitales sin penalizar velocidad de escritura. |
| **Sensor DHT11** | A2 | Utilizo el pin analógico `A2` en modo digital para mantener ordenado y despejado el bloque de pines digitales. |

### Montaje Físico

* Adicionalmente he diseñado una carcasa capaz de albergar todos los componentes de forma unificada, útil y cómoda para el usuario utilizando el programa de diseño CAD: **OnShape**. El botón de acceso al modo administrador y reinicio del modo de servicio no está contemplado en la carcasa, permanece en la protoboard junto al cableado de todos los componentes, simulando un acceso restringido y fuera del alcance del usuario, siendo así únicamente accesible por un administrador.

<img width="3504" height="2478" alt="Part Studio 1" src="https://github.com/user-attachments/assets/7c39565b-2976-44f1-bfd9-bca59b6d8cc2" />  

* Montaje real sobre la carcasa impresa en 3D (PLA):

<img width="3071" height="2714" alt="image" src="https://github.com/user-attachments/assets/3c60ab1f-48b1-4ae5-b732-5a7dc3b86c80" />

<img width="2731" height="1987" alt="image" src="https://github.com/user-attachments/assets/daa26280-eb79-4362-8353-921f345d0c71" />


---

## 2. Descripción detallada de la implementación

Para evitar código monolítico difícil de depurar, he organizado el software de forma modular en **9 ficheros** (un `.ino` principal como núcleo coordinador y 8 ficheros `.hpp` independientes para los hilos y funciones auxiliares).

### A. Recursos Empleados y Justificación Técnica

1. **Concurrencia Cooperativa (`ArduinoThread` y `ThreadController`):**
   * En una máquina expendedora no se puede detener el procesador con `delay()` durante los 5 segundos de lectura ambiental o los 4–8 segundos de preparación de bebida, ya que el sistema dejaría de responder al pulsador de emergencia o administración. Mediante `ArduinoThread` he dividido cada responsabilidad en un hilo periódico independiente que ejecuta su tarea en microsegundos y cede el control de inmediato.
2. **Interrupciones Hardware (`attachInterrupt` en D2):**
   * El botón externo exige discriminar con precisión entre una pulsación de reinicio (2 a 3 segundos) y una de acceso/salida del modo Admin (≥ 5 segundos) **en cualquier instante**, independientemente del estado o hilo activo. Asociar `D2` a una rutina de interrupción por cambio de flanco (`CHANGE`) con antirrebote por software (`50 ms`) permite capturar el instante exacto de bajada y subida sin depender de la frecuencia de muestreo del `loop()`.
3. **Temporizador Hardware (`TimerOne`):**
   * La consola de administración requiere un contador de tiempo encendido preciso segundo a segundo. En lugar de acumular tiempos con `millis()` sujetos a pequeñas variaciones por la carga de los hilos, he configurado `Timer1` a `1.000.000 µs` (1 s) para disparar una interrupción interna periódica que incrementa `segundosEncendido` de forma exacta.
4. **Mecanismo de Recuperación (`Watchdog Timer` con `avr/wdt.h`):**
   * En sistemas empotrados que interactúan con sensores físicos, una lectura anómala o un fallo eléctrico en el DHT11 o el ultrasonidos (`pulseIn`) podría dejar el flujo detenido. Al habilitar el Watchdog Timer a 4 segundos (`WDTO_4S`) y refrescarlo en cada iteración del `loop()` (`wdt_reset()`), garantizo que el microcontrolador se reinicie automáticamente ante cualquier bloqueo imprevisto.

---

### B. Flujo de Funcionamiento y Arquitectura por Módulos

El sistema opera como una **máquina de estados finitos** gobernada por la activación y desactivación dinámica de hilos (`thread.enabled`) y supervisada desde el `loop()` principal:

1. **Inicialización y Modo Arranque (`main.ino` e `initThread.hpp`):**
   * En el `setup()` desactivo temporalmente el watchdog, configuro entradas/salidas, inicializo el LCD con el mensaje `CARGANDO...`, enlazo los 7 hilos al `ThreadController` (dejando habilitado únicamente `initThread`), arranco `Timer1` y activo el watchdog.
   * `initThread` se ejecuta cada **500 ms** alternando `LED1` 6 veces para completar exactamente 3 parpadeos de 1 segundo. Al finalizar, apaga `LED1`, limpia pantalla y cede el control al modo Servicio habilitando `usThread`.

2. **Modo Servicio (`usThread.hpp`, `humidityThread.hpp`, `menuThread.hpp`, `preparingThread.hpp`):**
   * **Detección de presencia (`usThread`, cada 200 ms):** Invoca `medicion()`, que dispara el pulso ultrasónico y calcula la distancia en centímetros (`duration * 0.0343 / 2.0`). Mientras no haya nadie (`>= 100 cm`), mantiene en pantalla `ESPERANDO CLIENTE` usando la bandera `flagUS` para no reescribir el LCD innecesariamente. Al detectar presencia (`< 100 cm`), desactiva `usThread` y activa `humidityThread`.
   * **Monitorización ambiental (`humidityThread`, cada 1000 ms):** Llama a `humidity()` durante 5 ciclos (5 segundos) para mostrar temperatura y humedad del DHT11 descartando lecturas corruptas (`isnan()`). Al quinto segundo, renderiza el catálogo mediante `print_lcd()` y activa `menuThread`.
   * **Navegación y selección (`menuThread`, cada 200 ms):** Lee los ejes analógicos del joystick para desplazarse cómodamente por los 5 productos y detecta la pulsación del botón integrado (`sw`) mediante detección de flanco (`sw == LOW && swAnterior == HIGH`) para filtrar rebotes por software. Al confirmar una bebida, genera un tiempo pseudoaleatorio entre 4 y 8 segundos (`random(4000, 8000)`), calcula el incremento proporcional de brillo por ciclo (`255.0 / (tiempoPrep / 20.0)`) y cede paso a `preparingThread`.
   * **Preparación y entrega (`preparingThread`, cada 20 ms):** Muestra `Preparando Cafe...` e incrementa progresivamente el ciclo de trabajo PWM de `LED2` mediante `analogWrite()` hasta alcanzar la intensidad máxima al cumplirse el tiempo aleatorio. Acto seguido apaga `LED2`, muestra `RETIRE BEBIDA` durante 150 ciclos de 20 ms (3 segundos exactos) y retorna el sistema al estado inicial de reposo (`usThread`).

3. **Interrupciones y Control Central (`funciones_auxiliares.hpp` y `loop()`):**
   * Las variables modificadas en las interrupciones (`reiniciar`, `admin`, `enModoAdmin`, `segundosEncendido`) están declaradas como `volatile` para evitar optimizaciones erróneas del compilador.
   * Cuando `interrupcionBoton()` detecta una pulsación de **2 a 3 segundos**, levanta `reiniciar`; si es de **≥ 5 segundos**, levanta `admin` y conmuta `enModoAdmin`. En el siguiente ciclo, el `loop()` intercepta estas banderas, detiene todos los hilos activos, resetea contadores, banderas internas y brillo de LEDs, y habilita limpiamente el estado de destino sin solapamientos.

4. **Modo Administración (`adminThread.hpp` y `modpreciosThread.hpp`):**
   * Al entrar en Admin se encienden ambos LEDs (`LED1` y `LED2`) como indicador visual de estado y se despliega el menú técnico en `adminThread` (**200 ms**):
     * **Ver temperatura:** Refresca dinámicamente temperatura y humedad en tiempo real.
     * **Ver distancia sensor:** Muestra la lectura continua en centímetros del ultrasonidos.
     * **Ver contador:** Muestra `segundosEncendido` comparándolo con `segundoAnterior` para refrescar el LCD únicamente cuando cambia el segundo, eliminando parpadeos.
     * **Modificar precios:** Transfiere el control a `modpreciosThread` (**200 ms**). Al elegir un producto, guarda su precio actual en `varMemory` y permite incrementarlo o decrementarlo en tramos de `0.05 €` con el eje `Y`. Pulsando `sw` se confirma el cambio directamente sobre el array global `precios[]` (reflejándose al instante al volver al modo Servicio); moviendo el joystick a la izquierda (`vx < 400`) se restaura `varMemory` descartando los cambios y volviendo al menú anterior.
