Este conjunto de archivos en C implementa una aplicación embebida basada en el sistema operativo de tiempo real **FreeRTOS**. El sistema modela y resuelve de forma clásica el problema de sincronización de **Productor-Consumidor** (basado en el libro *The Little Book of Semaphores* de Allen B. Downey).

En este escenario, **Task A actúa como el Productor** y **Task B actúa como el Consumidor**. Ambas tareas se sincronizan y protegen los recursos compartidos utilizando dos primitivas de FreeRTOS: un **Mutex** (`mutex_buffer`) para la exclusión mutua y un **Semáforo Binario** (`sem_data_ready`) para la señalización de eventos.

A continuación, se presenta el análisis y la explicación detallada del funcionamiento de cada archivo:

---

## 1. `app.c` (Inicialización del Sistema)

Este archivo se encarga de la configuración inicial del entorno de la aplicación antes de que el planificador (*scheduler*) de FreeRTOS tome el control del procesador.

* **Variables Globales de Control y Estado:** Define contadores globales para el diagnóstico del sistema: ticks del sistema (`g_app_tick_cnt`), tiempo de inactividad (`g_task_idle_cnt`), y desbordamientos de pila (`g_app_stack_overflow_cnt`). También define la variable compartida `shared_data` (el recurso crítico).
* **Mecanismos de Sincronización:** Crea los semáforos que gobernarán el comportamiento concurrente:
* `mutex_buffer = xSemaphoreCreateMutex();`: Exclusión mutua para asegurar que solo una tarea a la vez acceda a `shared_data`.
* `sem_data_ready = xSemaphoreCreateBinary();`: Semáforo de sincronización que inicia en cero (bloqueado) y servirá para avisarle a Task B que hay datos nuevos disponibles.


* **Creación de Tareas (`xTaskCreate`):** Instancia dinámicamente en memoria las dos tareas principales (`task_a` y `task_b`). Ambas se configuran con la misma prioridad estática (`tskIDLE_PRIORITY + 1ul`) y el tamaño de pila mínimo (`configMINIMAL_STACK_SIZE`).
* **Seguridad:** Cada asignación crítica es evaluada con `configASSERT()`. Si falta memoria en el *Heap* al crear los semáforos o las tareas, el sistema detiene su ejecución inmediatamente para facilitar la depuración.

---

## 2. `task_a.c` (El Productor)

Representa el hilo de ejecución de la **Tarea A**, encargada de generar la información.

* **Bucle Infinito (`for(;;)`):** La tarea corre cíclicamente realizando los siguientes pasos:
1. **Acceso Seguro:** Intenta tomar el mutex (`xSemaphoreTake(mutex_buffer, portMAX_DELAY)`). Si Task B estuviera leyendo el recurso, Task A se bloquearía de forma indefinida (`portMAX_DELAY`) hasta que el mutex se libere.
2. **Producción del Dato:** Almacena el valor actual de su contador interno (`g_task_a_cnt`) dentro de la variable global compartida `shared_data` e incrementa su contador.
3. **Liberación y Señalización:** * Suelta el mutex (`xSemaphoreGive(mutex_buffer)`) finalizando la sección crítica.
* Entrega el semáforo binario (`xSemaphoreGive(sem_data_ready)`), notificando de manera efectiva al consumidor que el dato ha sido actualizado.


4. **Bloqueo Voluntario:** Llama a `vTaskDelay(pdMS_TO_TICKS(250ul))`, lo que la mueve al estado *Blocked* durante **250 milisegundos**. Esto cede el procesador para permitir el avance del resto del sistema.



---

## 3. `task_b.c` (El Consumidor)

Representa el hilo de ejecución de la **Tarea B**, encargada de procesar la información generada por la Tarea A.

* **Bucle Infinito y Control por Eventos:**
1. **Espera de Señal:** La tarea inicia su ciclo intentando tomar el semáforo binario (`xSemaphoreTake(sem_data_ready, portMAX_DELAY)`). Al estar el semáforo inicialmente vacío, **Task B se bloquea de inmediato** y no consume ciclos de CPU de forma innecesaria. No se despertará hasta que Task A haga el respectivo `xSemaphoreGive`.
2. **Acceso Seguro al Dato:** Una vez despertada por el semáforo, toma el mutex de exclusión mutua (`xSemaphoreTake(mutex_buffer, portMAX_DELAY)`), copia el valor de `shared_data` a una variable local (`local_data`), y libera el mutex inmediatamente.
3. **Procesamiento:** Imprime en los logs el dato consumido de forma segura.
4. **Bloqueo Voluntario:** Llama a `vTaskDelay(pdMS_TO_TICKS(500ul))`, forzando una espera de **500 milisegundos** antes de volver a intentar esperar el siguiente dato.



---

## 4. `freertos.c` (Callbacks del Sistema u Hooks)

Contiene las funciones *Hook* de FreeRTOS, las cuales actúan como oyentes automáticos del comportamiento interno del núcleo.

* **`vApplicationIdleHook()`:** Se ejecuta continuamente cuando el procesador no tiene tareas prioritarias listas para ejecutar (por ejemplo, cuando tanto Task A como Task B están cumpliendo sus tiempos de `vTaskDelay`). Incrementa `g_task_idle_cnt` y es el espacio idóneo para activar modos de bajo consumo en el hardware.
* **`vApplicationTickHook()`:** Se ejecuta como una rutina de interrupción corta (ISR) disparada por el timer del sistema en cada milisegundo, incrementando el contador global de tiempo `g_app_tick_cnt`.
* **`vApplicationStackOverflowHook()`:** Es un mecanismo de resguardo. Si alguna de las tareas sufre un desbordamiento en su pila de memoria (*Stack Overflow*), esta función detiene el sistema en un bucle controlado (`configASSERT( 0 )`) para evitar corrupción de memoria impredecible en el microcontrolador.

---

## 5. `app_it.c` (Gestión de Interrupciones)

Este archivo está destinado a inicializar el comportamiento de las interrupciones específicas de la aplicación a través de `app_it_init()`.

* En su estado actual, no interactúa con periféricos de hardware, pero deja un ejemplo explícito de arquitectura de software embebida: el uso de instrucciones en ensamblador inline (`__asm("CPSID i")` y `__asm("CPSIE i")`) para deshabilitar y habilitar interrupciones a nivel global en núcleos ARM Cortex-M. Esto se utiliza habitualmente para crear secciones críticas rápidas a nivel de hardware.

---

## Dinámica Temporal del Sistema

El flujo e interacción del código se comporta de la siguiente manera a lo largo del tiempo:

1. **Arranque:** `app_init()` crea el Mutex, el Semáforo Binario y lanza ambas tareas con Prioridad 1.
2. **Ciclo Inicial:** Task A se ejecuta primero, toma el mutex, escribe en `shared_data`, libera el mutex, levanta el semáforo `sem_data_ready` y se bloquea por 250ms.
3. **Sincronización:** Al liberarse el semáforo, Task B (que estaba bloqueada en él) se despierta inmediatamente, toma el mutex, lee de forma segura el dato de Task A, lo imprime y se bloquea por 500ms.
4. **Asincronía de Tiempos:** Dado que el productor (Task A) trabaja cada 250ms y el consumidor (Task B) procesa cada 500ms, la Tarea A producirá datos más rápido de lo que la Tarea B puede consumirlos, demostrando un escenario típico de desacoplamiento temporal mediante semáforos en sistemas de tiempo real. Cuando no hay trabajo pendiente, la tarea *Idle* toma el control.

¿Te gustaría modificar los tiempos de retardo (`vTaskDelay`) o añadir una cola de mensajes (`Queue`) para evitar que se pierdan los datos intermedios que produce la Tarea A?