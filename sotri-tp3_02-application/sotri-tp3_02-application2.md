Este nuevo conjunto de archivos implementa una aplicación embebida bajo el sistema operativo de tiempo real **FreeRTOS**. En esta versión del código, se modela y resuelve otro problema clásico de sincronización de la ciencia de la computación: el **Problema de los Lectores-Escritores** (*Readers-Writers Problem*), basado también en los patrones de diseño del libro *The Little Book of Semaphores* de Allen B. Downey.

En este escenario:

* **Task A actúa como el Lector (Reader)**: Puede haber múltiples lectores accediendo simultáneamente al recurso si no hay un escritor modificándolo.
* **Task B actúa como el Escritor (Writer)**: Requiere acceso exclusivo y absoluto; ningún lector ni otro escritor puede estar en la sección crítica al mismo tiempo.

A continuación, se detalla el análisis y funcionamiento de cada archivo en español:

---

### 1. `app.c` (Inicialización de la Aplicación)

Este archivo se encarga de la configuración global antes de que el planificador (*scheduler*) del sistema operativo tome el control.

* **Recursos Compartidos y Estado:** Define la variable global `shared_data` (el recurso crítico que se lee y escribe) y una variable `readers` que actúa como un contador para llevar un registro estricto de cuántos lectores están dentro de la habitación o sección crítica.
* **Mecanismos de Sincronización:** Para resolver el problema, se utilizan dos semáforos de FreeRTOS:
* `mutex`: Un semáforo (usualmente binario o mutex) que protege el contador `readers`. Asegura que cuando un lector incrementa o decrementa la cantidad de lectores activos, lo haga de forma atómica.
* `roomEmpty`: Un semáforo binario que representa si la "habitación" (la sección crítica de `shared_data`) está completamente vacía de lectores. Si el semáforo está disponible, un escritor puede entrar.


* **Creación de Tareas:** Llama a `xTaskCreate` para lanzar **Task A** (`task_a`) y **Task B** (`task_b`) con prioridades idénticas por encima de la tarea Idle (`tskIDLE_PRIORITY + 1ul`).

---

### 2. `task_a.c` (El Lector - Reader)

Representa el hilo de ejecución que consulta el recurso compartido. Su algoritmo implementa el patrón clásico para dar prioridad o permitir múltiples lectores concurrentes:

* **Fase de Entrada del Lector:**
1. Toma el `mutex` (`xSemaphoreTake(mutex, portMAX_DELAY)`) para modificar la variable global de control de forma segura.
2. Incrementa el contador `readers++`.
3. **Condición de Bloqueo al Escritor:** Si este es el **primer lector** que llega (`readers == 1`), significa que los lectores deben tomar control de la habitación. Por lo tanto, intenta tomar `roomEmpty`. Si un escritor estuviera escribiendo, el lector se bloqueará aquí. Si la habitación estaba vacía, el primer lector "cierra la puerta" para que ningún escritor pueda entrar.
4. Libera el `mutex` (`xSemaphoreGive(mutex)`) para que otros lectores puedan realizar este mismo proceso de entrada en paralelo sin trabarse entre sí.


* **Sección Crítica de Lectura:**
* Lee el valor de `shared_data`, lo almacena en una variable local (`local_data`) y lo reporta a través del sistema de logs (`LOGGER_INFO("Reader: %lu", local_data)`). Múltiples tareas lectoras podrían hacer esto al mismo tiempo.


* **Fase de Salida del Lector:**
1. Vuelve a tomar el `mutex` para actualizar el contador de forma segura.
2. Decrementa `readers--`.
3. **Condición de Liberación:** Si este es el **último lector** que sale de la habitación (`readers == 0`), significa que ya no hay nadie consultando el dato. Por lo tanto, devuelve el semáforo `roomEmpty` (`xSemaphoreGive(roomEmpty)`), "abriendo la puerta" nuevamente para que un escritor pueda ingresar.
4. Libera el `mutex`.



---

### 3. `task_b.c` (El Escritor - Writer)

Representa el hilo de ejecución encargado de modificar el recurso compartido. Su estructura es mucho más simple pero restrictiva:

* **Funcionamiento:**
1. **Exclusión Absoluta:** Al iniciar su ciclo, intenta tomar el semáforo `roomEmpty` (`xSemaphoreTake(roomEmpty, portMAX_DELAY)`). Si hay tan solo un lector dentro de la habitación (`readers > 0`), el escritor se quedará bloqueado en este punto indefinidamente sin consumir CPU.
2. **Sección Crítica de Escritura:** Una vez que obtiene el semáforo (lo que garantiza que la habitación está vacía), accede a `shared_data`, modifica su valor (por ejemplo, asignándole una nueva variable `value`) e incrementa sus propios registros.
3. **Liberación:** Al terminar de escribir, devuelve el semáforo `roomEmpty` (`xSemaphoreGive(roomEmpty)`) para permitir que los lectores (u otros escritores) vuelvan a competir por el acceso.



---

### 4. `freertos.c` (Callbacks y Monitores del Kernel)

Contiene las funciones *Hook* estándar de FreeRTOS que reaccionan a eventos internos del sistema:

* **`vApplicationIdleHook()`:** Incrementa el contador global `g_task_idle_cnt` en los momentos en que ni el lector ni el escritor requieran procesamiento (por estar bloqueados esperando un semáforo o en un estado de reposo).
* **`vApplicationTickHook()`:** Incrementa de forma síncrona `g_app_tick_cnt` con cada interrupción del *SysTick* (un incremento cada milisegundo) para mantener el tiempo base de la aplicación.
* **`vApplicationStackOverflowHook()`:** Si la pila de memoria (Stack) de Task A o Task B se llena de forma peligrosa y sobrepasa los límites configurados, FreeRTOS ejecuta este fragmento, congela el microcontrolador mediante un *assert* y evita que se corrompa el resto de la memoria RAM.

---

### 5. `app_it.c` (Gestión de Interrupciones de Hardware)

* Expone la función `app_it_init()`. Al igual que en versiones anteriores, se mantiene como una plantilla de control de bajo nivel. Muestra el uso de ensamblador inline (`__asm("CPSID i")` y `__asm("CPSIE i")`) para deshabilitar y habilitar las interrupciones globales de la CPU, una técnica utilizada en sistemas embebidos para crear secciones críticas ultra-rápidas a nivel de hardware cuando no se quiere depender del planificador del sistema operativo.

---

### Resumen de la Dinámica del Sistema

Este diseño implementa la solución conocida como **"Primera solución al problema de los lectores-escritores"** (o preferencia a los lectores). El comportamiento dinámico del firmware asegura lo siguiente:

1. **Lectura Concurrente:** Si el Escritor (Task B) no está activo, infinitas tareas lectoras (derivadas del código de Task A) podrían entrar a leer `shared_data` simultáneamente. El semáforo `roomEmpty` se toma una sola vez (por el primer lector) y se libera una sola vez (por el último lector).
2. **Protección contra Corrupción:** Mientras el Escritor (Task B) esté modificando el dato, ningún Lector podrá iniciar una lectura, evitando que se lean datos parciales o corruptos.
3. **Exclusión Mutua del Contador:** El semáforo `mutex` previene que dos lectores alteren la variable `readers` al mismo milisegundo, lo cual rompería la lógica de control.