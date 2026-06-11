Este conjunto de archivos implementa una aplicación basada en el sistema operativo de tiempo real **FreeRTOS**. De acuerdo con las referencias del código, está estructurado para resolver el clásico problema de sincronización de **Productor-Consumidor** empleando mecanismos de control concurrentes (Semáforos y Mutex).

A continuación, se detalla el análisis y el funcionamiento técnico de cada componente de software:

---

### 1. `app.c` (Inicialización de la Aplicación y Recursos)

Este archivo constituye el punto central de configuración del sistema antes de iniciar el planificador de tareas de FreeRTOS.

* **Variables de Diagnóstico Globales:** Define y expone contadores del sistema tales como `g_app_tick_cnt` (conteo de ticks de reloj), `g_task_idle_cnt` (ciclos ejecutados por la tarea inactiva o IDLE) y `g_app_stack_overflow_cnt` (registro de fallas por desbordamiento de pila).
* **Inicialización de Mecanismos de Sincronización:** En esta capa de inicialización se instancian dinámicamente dos herramientas de sincronización esenciales:
* **`mutex_buffer` (Mutex):** Garantiza la exclusión mutua para proteger la variable o recurso crítico compartido (`shared_data`) evitando colisiones de datos.
* **`sem_data_ready` (Semáforo Binario):** Utilizado para la señalización entre hilos, indicándole al consumidor cuándo hay un nuevo elemento listo para ser procesado.


* **Creación de Tareas (`xTaskCreate`):** Crea dos hilos con el mismo nivel de prioridad estática (`tskIDLE_PRIORITY + 1ul`) y asigna a cada uno el tamaño mínimo de pila (`configMINIMAL_STACK_SIZE`):
* **Task A** (asociado a la función `task_a`).
* **Task B** (asociado a la función `task_b`).


* **Validación de Memoria:** Utiliza la macro `configASSERT()` para asegurar que tanto la reserva de memoria para tareas como para semáforos en el *Heap* haya sido exitosa.

---

### 2. `task_a.c` (El rol del Productor)

Representa la implementación de la **Tarea A** y modela el comportamiento del elemento productor en el sistema.

* **Funcionamiento Cíclico e Ininterrumpido:** Se ejecuta de forma iterativa dentro de un bucle infinito `for (;;)`.
* **Sección Crítica:** 1. Intenta adquirir el token de exclusión mutua mediante `xSemaphoreTake(mutex_buffer, portMAX_DELAY)`. Si la Tarea B está usando el buffer, la Tarea A se suspende indefinidamente en estado *Blocked* hasta su liberación.
2. Al obtenerlo de forma segura, actualiza la variable global `shared_data` asignándole su contador local actual (`g_task_a_cnt`) e incrementa dicho contador.
3. Libera inmediatamente el recurso con `xSemaphoreGive(mutex_buffer)`.
* **Señalización de Eventos:** Llama a `xSemaphoreGive(sem_data_ready)`. Este paso es fundamental ya que incrementa el semáforo binario, lo cual despierta o notifica de forma inmediata a la Tarea B de que un nuevo dato fue introducido.
* **Temporización:** Utiliza `vTaskDelay(TASK_A_DEL_MAX)` para auto-bloquearse de manera voluntaria durante **250 milisegundos**, permitiendo que otros hilos utilicen el microcontrolador.

---

### 3. `task_b.c` (El rol del Consumidor)

Implementa la **Tarea B**, la cual funciona de manera reactiva bajo el principio de control por eventos.

* **Bloqueo Inicial Eficiente:** Al ingresar a su bucle infinito, la primera acción obligatoria es invocar a `xSemaphoreTake(sem_data_ready, portMAX_DELAY)`. Dado que el semáforo arranca vacío, **la Tarea B se suspende inmediatamente**. No consume tiempo de procesamiento en absoluto hasta que la Tarea A produzca un dato y "entregue" el semáforo.
* **Extracción Segura de Datos:** Tras recibir la señal del semáforo, la tarea procede a capturar el mutex (`mutex_buffer`), copia el valor de `shared_data` en una variable local (`local_data`) y devuelve el control del mutex de inmediato.
* **Procesamiento de Logs:** Una vez el dato está a salvo en el entorno local, emite los registros de depuración a través del sistema de logs.
* **Retardo de Control:** Llama a `vTaskDelay(TASK_B_DEL_MAX)` para forzar una pausa de **2500 milisegundos (2.5 segundos)** antes de volver a evaluar el semáforo de entrada.

---

### 4. `freertos.c` (Llamadas de Retorno y Ganchos del Sistema / Hooks)

Este archivo centraliza las funciones de gancho del Kernel, las cuales reaccionan de manera automática ante determinados estados globales del sistema operativo:

* **`vApplicationIdleHook()`:** Se invoca de manera repetitiva únicamente cuando ninguna tarea de la aplicación está en condiciones de ejecutarse (es decir, cuando tanto Task A como Task B están retenidas en sus respectivos retardos por tiempo). Incrementa la variable `g_task_idle_cnt`. En sistemas de producción reales, este gancho se utiliza estratégicamente para colocar el procesador en modo de bajo consumo energético.
* **`vApplicationTickHook()`:** Se ejecuta directamente dentro de la rutina de servicio de interrupción (ISR) del reloj del sistema (*SysTick*). Incrementa a cada milisegundo el contador `g_app_tick_cnt`, actuando como el reloj base del firmware.
* **`vApplicationStackOverflowHook()`:** Proporciona protección en tiempo de ejecución. Si un hilo sobrepasa el límite físico de su memoria asignada de pila, este callback atrapa el error incrementando `g_app_stack_overflow_cnt` y detiene deliberadamente el hardware a través de un assert, previniendo fallos críticos impredecibles.

---

### 5. `app_it.c` (Inicialización de Interrupciones de la Aplicación)

Este archivo provee la función `app_it_init()` para preparar las interrupciones del microcontrolador.

* En el fragmento provisto, contiene una sección demostrativa del control de secciones críticas en arquitecturas de hardware embebido. Emplea instrucciones de lenguaje ensamblador embebido (`__asm("CPSID i")` y `__asm("CPSIE i")`) para inhabilitar y rehabilitar las interrupciones globales a nivel de la CPU. Este recurso es habitual para proteger secciones críticas muy rápidas que no requieran del bloqueo avanzado provisto por los semáforos del sistema operativo.

---

### Análisis del Comportamiento y Flujo de Ejecución

Debido al diseño asíncrono y los tiempos configurados, el sistema se comporta dinámicamente del siguiente modo:

1. **Frecuencias diferentes:** La **Tarea A** genera información a una velocidad diez veces mayor (**cada 250ms**) en comparación con el ciclo de la **Tarea B**, que requiere **2500ms** para completar su rutina de espera.
2. **Efecto de Desacoplamiento:** Gracias a que se utiliza un semáforo binario para notificar el evento y un mutex para proteger el recurso, la Tarea B permanecerá bloqueada pacíficamente en `sem_data_ready`. Una vez que se cumpla su tiempo de delay, tomará el dato más reciente depositado en la variable compartida de manera totalmente segura y libre de condiciones de carrera (*race conditions*).