Este conjunto de archivos en C implementa una aplicación básica utilizando el sistema operativo de tiempo real **FreeRTOS**. Según los comentarios del archivo `app.c`, el proyecto está diseñado como el esqueleto inicial para resolver el clásico problema de sincronización de **Productor-Consumidor** (basado en el libro *The Little Book of Semaphores* de Allen B. Downey).

Actualmente, el código se encuentra en un estado de **plantilla básica** o esqueleto funcional: las tareas están creadas y corren de forma independiente de manera cíclica, pero los mecanismos de sincronización (semáforos o colas) aún no han sido implementados.

A continuación, se detalla el análisis y funcionamiento de cada archivo en español:

---

## 1. `app.c` (Inicialización de la Aplicación)

Es el núcleo de la inicialización antes de que el planificador (*scheduler*) de FreeRTOS tome el control absoluto del procesador.

* **Variables Globales de Control:** Define e inicializa contadores globales útiles para el diagnóstico del sistema: ticks del sistema (`g_app_tick_cnt`), contador de la tarea Idle (`g_task_idle_cnt`), y desbordamientos de pila (`g_app_stack_overflow_cnt`).
* **Función `app_init()`:** * Muestra mensajes informativos en el log indicando qué aplicación se está ejecutando mediante macros de logging (`LOGGER_INFO`).
* **Creación de Tareas:** Utiliza la API `xTaskCreate` para instanciar dinámicamente dos tareas: **Task A** y **Task B**. Ambos hilos se configuran con la prioridad más baja por encima de la tarea inactiva (`tskIDLE_PRIORITY + 1ul`) y con un tamaño de stack mínimo (`configMINIMAL_STACK_SIZE`).
* **Validación:** Emplea `configASSERT(pdPASS == ret)` para colgar el sistema si alguna tarea no pudo ser creada (por ejemplo, por falta de memoria RAM en el *Heap*).
* Llama a la inicialización de interrupciones (`app_it_init()`) y del contador de ciclos de CPU (`cycle_counter_init()`).



---

## 2. `task_a.c` (Tarea A)

Representa el comportamiento del primer hilo de ejecución (`task_a`).

* **Estructura:** Se ejecuta dentro de un bucle infinito `for (;;)` (característico de las tareas en sistemas de tiempo real).
* **Funcionamiento Cíclico:**
1. Incrementa un contador local/global de ejecuciones (`g_task_a_cnt`).
2. Envía un mensaje al log: `"   ==> Task    A - Wait:   250mS"`.
3. **Bloqueo Voluntario:** Llama a `vTaskDelay(TASK_A_DEL_MAX)`, donde `TASK_A_DEL_MAX` equivale a **250 milisegundos**. Durante este tiempo, la tarea cede el procesador para que otras tareas (o la tarea Idle) puedan ejecutarse.



---

## 3. `task_b.c` (Tarea B)

Representa el comportamiento del segundo hilo de ejecución (`task_b`).

* **Estructura:** Al igual que la Tarea A, posee un bucle infinito.
* **Funcionamiento Cíclico:** Es idéntica en estructura a la Tarea A, pero con una diferencia crítica en su temporización:
1. Incrementa su propio contador (`g_task_b_cnt`).
2. Envía un mensaje al log (curiosamente el texto dice "Wait: 250mS", pero la constante real de tiempo es diferente).
3. **Bloqueo Voluntario:** Llama a `vTaskDelay(TASK_B_DEL_MAX)`. Aquí, `TASK_B_DEL_MAX` está definido como `pdMS_TO_TICKS(2500ul)`, lo que significa que **se bloquea por 2500 milisegundos (2.5 segundos)**.


* *Nota de diseño:* Se ejecuta 10 veces más lento que la Tarea A.



---

## 4. `freertos.c` (Funciones de Gancho o Hooks)

Este archivo contiene las funciones *Callback* o *Hook* de FreeRTOS. Son funciones que el propio sistema operativo invoca automáticamente cuando ocurren ciertos eventos globales si están activadas en la configuración (`FreeRTOSConfig.h`).

* **`vApplicationIdleHook()` (Gancho de Tarea Inactiva):** Se ejecuta repetidamente cuando **ninguna** tarea de la aplicación está lista para correr (es decir, cuando Task A y Task B están en estado *Delayed*). Aquí incrementa el contador `g_task_idle_cnt`. Es el lugar ideal para poner el microcontrolador en modo de bajo consumo (*Low Power*).
* **`vApplicationTickHook()` (Gancho de Tick de Reloj):** Se ejecuta dentro de la Interrupción del Reloj del Sistema (ISR del *SysTick*). Incrementa el contador general de tiempo del sistema `g_app_tick_cnt` en cada milisegundo.
* **`vApplicationStackOverflowHook()` (Detector de Desbordamiento de Pila):** Si una tarea consume más memoria de la asignada a su pila (Stack), FreeRTOS detecta la corrupción e invoca esta función. El código entra en una sección crítica y se congela deliberadamente (`configASSERT( 0 )`) para permitir al desarrollador conectar un depurador y analizar el fallo.

---

## 5. `app_it.c` (Interrupciones de la Aplicación)

Este archivo está destinado a manejar la inicialización de las interrupciones específicas de la aplicación (`app_it_init`).

* **Estado actual:** Actualmente está vacío de periféricos. Solo muestra un ejemplo de código en lenguaje ensamblador (`__asm("CPSID i")` y `__asm("CPSIE i")`) para deshabilitar y volver a habilitar las interrupciones globales, lo cual se utiliza frecuentemente para proteger "secciones críticas" de código frente a accesos concurrentes de hardware.

---

## Resumen del Comportamiento del Sistema

Al encenderse el sistema:

1. Se ejecuta `app_init()`, se configuran las variables y se crean **Task A** y **Task B** con la misma prioridad (Prioridad 1).
2. Cuando el planificador inicia, ambas tareas se ejecutan por primera vez imprimiendo sus mensajes.
3. **Task A** se duerme por 250ms y **Task B** se duerme por 2500ms.
4. Al estar ambas dormidas, FreeRTOS ejecuta la tarea **Idle**, la cual incrementa continuamente `g_task_idle_cnt`.
5. Cada 1ms, el reloj de hardware genera una interrupción, ejecutando `vApplicationTickHook` e incrementando `g_app_tick_cnt`.
6. Cada 250ms, **Task A** se despierta, incrementa su contador, imprime su mensaje y se vuelve a dormir.
7. Cada 2500ms (2.5 segundos), coincidirán el despertar de **Task A** y **Task B**. Al tener la misma prioridad, el planificador de FreeRTOS las alternará (Time-slicing / Round-robin) para que ambas procesen su bucle antes de volver a bloquearse.