# sotri-tp3_03-application — Vehicular crossing

Implementación del problema **Vehicular crossing** (*Other problems*) sobre FreeRTOS:
un sistema de control de acceso que monitorea y controla el ingreso/egreso de
vehículos a un cruce vial de **capacidad limitada** (`G_TASKS_CNT_MAX = 3`), con
**dos puntos de ingreso** (entry_a, entry_b) y **dos de egreso** (exit_a, exit_b).

## Arquitectura de tareas

| Tarea | Prioridad | Rol |
|---|---|---|
| `task_entry_a` / `task_entry_b` | 3 | Monitoreo y control del ingreso de vehículos |
| `task_exit_a` / `task_exit_b` | 2 | Monitoreo y control del egreso de vehículos |
| `task_test` | 1 (se auto-eleva a 3) | Recupera estímulos de un array y excita a las demás tareas |

`task_test` recorre el array `e_task_test_array[]` y, por cada estímulo
(`Entry_A`, `Entry_B`, `Exit_A`, `Exit_B`), señaliza a la tarea correspondiente.
El estímulo a usar se selecciona con el macro `E_TASK_TEST_X` (la prueba que se
documenta abajo usa `E_TASK_TEST_X == 6`).

## Mecanismos de sincronización

| Primitiva | Tipo | Uso |
|---|---|---|
| `h_entry_a_bin_sem`, `h_entry_b_bin_sem` | Semáforo binario | Señalización `task_test → task_entry_*` (evento "llegó un vehículo a ingresar") |
| `h_exit_a_bin_sem`, `h_exit_b_bin_sem` | Semáforo binario | Señalización `task_test → task_exit_*` (evento "vehículo egresa") |
| `h_mutex_mut_sem` | Mutex | Exclusión mutua sobre el contador compartido `g_tasks_cnt` |
| `g_tasks_cnt` | Contador (`uint32_t`) | Cantidad de vehículos dentro del cruce (tope `G_TASKS_CNT_MAX`) |

**Por qué binarios "vacíos" para señalizar:** los semáforos de ingreso/egreso se
crean con `xSemaphoreCreateBinary()` (cuenta inicial 0), de modo que las tareas
de entry/exit quedan **bloqueadas** en `xSemaphoreTake(..., portMAX_DELAY)` hasta
que `task_test` haga `xSemaphoreGive()`. Es un uso de **sincronización por
eventos**, distinto del binario inicializado en 1 que se usaría para exclusión
mutua.

**Detección del límite de capacidad (contador + mutex):** al recibir el evento,
cada `task_entry_*` toma el mutex y evalúa `g_tasks_cnt < G_TASKS_CNT_MAX`:
- Si hay lugar → `g_tasks_cnt++`, semáforo vial **VERDE**, el vehículo ingresa.
- Si está lleno → semáforo vial **ROJO**, se **impide** el ingreso (no incrementa).

Las `task_exit_*` decrementan `g_tasks_cnt` (si > 0) y reabren la entrada (VERDE).

**Recurso compartido entre A y B:** `task_entry_a` y `task_entry_b` (y sus exit)
operan sobre el **mismo** `g_tasks_cnt` protegido por el **mismo**
`h_mutex_mut_sem`. Por eso el cruce es un **único recurso de capacidad 3** por el
que compiten ambos puntos de ingreso, garantizando exclusión mutua sobre el
contador.

## Mapa de la consigna (pasos recomendados)

- [x] Sincronización `task_test`↔`task_entry_a` para el ingreso (semáforo binario).
- [x] Sincronización `task_test`↔`task_exit_a` para el egreso (semáforo binario).
- [x] Detección de límite (`G_TASKS_CNT_MAX`) e impedir el ingreso (contador + mutex).
- [x] Control del semáforo vial (Rojo/Verde).
- [x] Repetición del procedimiento para `task_entry_b` y `task_exit_b`.
- [x] Acceso a recursos compartidos por las tareas de ingreso (mutex compartido).

## Comportamiento observado (`E_TASK_TEST_X == 6`)

Array de estímulos: `{Entry_A, Entry_B, Entry_A, Entry_B, Exit_A, Exit_B, Exit_A, Exit_B}`.

```
==> Entry A - VERDE  - ingresa  (1/3)
==> Entry B - VERDE  - ingresa  (2/3)
==> Entry A - VERDE  - ingresa  (3/3)
==> Entry B - ROJO   - lleno    (3/3) - ingreso impedido
==> Exit  A - egresa            (2/3) - Entry A: VERDE
==> Exit  B - egresa            (1/3) - Entry B: VERDE
==> Exit  A - egresa            (0/3) - Entry A: VERDE
==> Exit  B - cruce vacio       (0/3)
```

**Conclusión:** el sistema admite vehículos por orden de llegada hasta colmar la
capacidad (3). El 4° ingreso (`Entry_B`) encuentra el cruce lleno → semáforo en
**ROJO** e ingreso impedido. A medida que los vehículos egresan, los slots se
liberan y la entrada vuelve a **VERDE**. El mutex garantiza que el conteo sea
consistente aun cuando ingresos y egresos de A y B compiten por el mismo recurso.
